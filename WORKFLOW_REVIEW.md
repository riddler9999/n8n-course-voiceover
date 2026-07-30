# n8n Workflow Review — Course Voiceover Generator

**Workflow:** `n8n Course Voiceover Generator` (`hjay2RRs99eypw6c`) · 57 nodes · single webhook + action router
**Reviewer role:** Senior n8n Architect
**Goal:** Make it stable, maintainable, and production-ready — without redesigning the system.

---

## Current Score: 6.5 / 10

This is a genuinely well-architected pipeline, not a beginner workflow. The deterministic Scene Compiler, the "AI proposes / code disposes" pattern in scene planning, the consistent use of `responseMimeType: application/json`, and the guarded `JSON.parse` in most extractors are all senior-level moves. It loses points on **production hardening**, not on design: failures hang the webhook instead of returning an error, garbage results get committed to storage without validation, and a few unguarded response accesses can crash a run. Fix those and this is an 8+.

---

## Keep (do NOT change these)

These are correct and load-bearing. Leave them alone.

1. **Single webhook + Action Router.** Seven actions behind one entry point is the right amount of structure. Do **not** split this into micro-workflows — there is no technical reason to, and it would multiply the credential/deploy surface.
2. **Deterministic Scene Compiler (IR).** Splitting scenes on the Burmese full stop `။` in code, then *snapping* cuts to ffmpeg-measured silence, is excellent. Timing is reproducible; the LLM never decides structure. This is the strongest part of the workflow.
3. **"AI proposes, code disposes" in `Extract Scene Plan`.** Re-imposing the IR's `scene_id/start/end/duration/narration` after Gemini and only accepting validated art-direction fields (`director_drift` counter included) is exactly how to make an LLM step reliable. Keep this pattern everywhere.
4. **Structured JSON output on every Gemini call** (`responseMimeType: application/json`) plus **guarded `JSON.parse`** in `Parse Transcript`, `Extract Scene Plan`, `Extract QA Report`, `Merge Script + Assets`. Correct.
5. **Render isolated in a sub-workflow** (`Render via SUB`, `waitForSubWorkflow: true`). Heavy VPS render kept out of the main graph. Good separation — keep it.
6. **`Detect Pauses` fails soft** (`onError: continueRegularOutput`) and the compiler treats pauses as optional evidence. Correct: a missing silence map degrades quality, not availability.
7. **Graceful asset branch.** `Analyze Assets` (`neverError: true`) → `Merge Script + Assets` (try/catch → empty asset summary). The script action still returns 200 even if asset analysis fails. Good.

---

## Fix (critical — do these before calling it production-ready)

### F1. Failures hang the webhook — no error response on 6 of 7 paths **[highest priority]**
Only `Respond Error` (unknown action) returns an error. If any Gemini / GitHub / SSH node throws after its retries are exhausted, **nothing** is connected to a `Respond to Webhook` node, so the HTTP request stays open until it times out. The frontend then waits the full 900 s and shows a generic "Timed out."

**Fix (simple, no redesign):** add one `Error Trigger` → `Respond to Webhook (500)` workflow, *or* wire the error outputs of the terminal-risk nodes (`Gemini Script Generator`, `Gemini Audio STT`, `Scene Planner Agent`, `Gemini QA Reviewer`, `Upload to GitHub`, `SSH Extract Stills`, `Render via SUB`) into a single shared **`Respond Failure`** node returning `{ status:"error", stage, message }` with code 500. The client already knows how to display a JSON error — give it one.

### F2. Unguarded audio-response access can crash a run
`PCM to WAV` does `item.candidates[0].content.parts[0].inlineData.data` with **no try/catch**. Gemini TTS can return a 200 with no audio (safety block, empty candidates, or a text part). The HTTP error output only catches non-2xx, so a 200-without-audio throws here → unhandled → hang (see F1).

**Fix:** guard the access; if `candidates[0]…inlineData.data` is missing, route to the retry path (or throw a typed error that F1 can respond to). Same defensive read `Save New Components` already uses should be applied here.

### F3. Validate before saving to storage
The repo contains proof this is already happening in production:
- `visual_scripts/lesson_12_1783889512504.json` → `{"scenes": []}`
- `visual_scripts/lesson_1_1783971518213.json` → `{"error": "No transcript provided…"}`
- older `{"duration": "undefineds", …}` artifacts

`Prep Visual Upload` uploads whatever it parsed — including `{ error: "json_parse_failed" }` and empty `scenes: []` — straight to GitHub. `Concat WAV` → `Upload to GitHub` has no check that the WAV has real samples.

**Fix:** add a cheap gate before each upload:
- Visual: only commit if `Array.isArray(parsed.scenes) && parsed.scenes.length > 0` and no `parsed.error`. Otherwise return a 422 via F1.
- Audio: only commit if `pcm.length > <min_bytes>` and `chunks_merged === total_chunks`.

This stops the repo from filling with dead files and stops the frontend from "succeeding" on garbage.

### F4. Verify / standardize Gemini model IDs
Model IDs are inconsistent across nodes and at least one looks wrong:
| Node | Model |
|---|---|
| Gemini Script Generator, Scene Planner, Visual Script | `gemini-3.6-flash` |
| Gemini TTS ×4 | `gemini-3.1-flash-tts-preview` |
| Analyze Assets | `gemini-3.1-flash-lite` |
| Audio STT, QA Reviewer | `gemini-2.5-flash` |

A wrong or deprecated model ID returns **404**, which is then retried 5× (25 s wasted) and finally hangs (F1). Confirm every ID resolves against the live API, pin them in **one place** (a `Set` node or workflow static data referenced by expression), and prefer the same generation family unless a node genuinely needs a different one. `gemini-3.6-flash` in particular should be double-checked.

### F5. `Fetch Registry (Assets)` and `SSH Extract Stills` have no error handling
`Fetch Registry (Assets)` (GET raw.githubusercontent) throws on 404 by default; the library repo (`moehtetofficial/n8n-remotion-library`) is a *different* account than the output repo, so an outage/rename there breaks the **script** action. `SSH Extract Stills` has no `onError` — a VPS/ffmpeg/npx hiccup hangs `qa_check`.

**Fix:** set `neverError`/`onError: continueRegularOutput` on `Fetch Registry (Assets)` (downstream already tolerates empty `components`), and give `SSH Extract Stills` an error route into F1's responder.

---

## Improve Later (nice-to-have, not blocking)

- **I1. Duplicated IR logic.** `Build Scene IR` and `Prep QA` each re-implement `PATTERN_BY_TYPE`, `FALLBACK_WORKFLOW`, `firstSentence`, and `pickWorkflow`. Change one and the QA preview silently diverges from the real render. Extract into a tiny shared sub-workflow or a single Code node both call, or at minimum add a comment forcing them to stay in sync.
- **I2. Two parallel visual subsystems.** The `plan_scenes` path (deterministic compiler + art direction) and the legacy `visual_script` path (`Build Visual Request` → full-creative Gemini with `ui[]`) both exist. The compiler path is clearly the better V2 design. Once it's proven on real lessons, retire the legacy path to cut ~5 nodes and a very large prompt from maintenance. Don't remove it yet — just plan to.
- **I3. TTS throughput.** The loop is sequential (`batchSize` default = 1) with a 20 s wait after every chunk (`Wait 15s` node is actually `amount: 20`). The 4-key `Key Router` only helps on *retry*, not throughput, because `chunk_index % 4` is constant per chunk. A 20-chunk lesson takes ~8 min. If speed matters, either batch the keys for real parallelism or drop the fixed 20 s wait in favor of the existing retry/backoff. Also: on retry the same key is reused — rotate to the next key instead.
- **I4. Trim the `Build Visual Request` system prompt.** It's very long with heavy repetition and blank-line padding — real token cost on every visual call. A ~40 % trim won't change output quality.
- **I5. `maxOutputTokens: 32000`.** Confirm each target model actually supports that ceiling; otherwise it's silently clamped. Harmless but misleading.
- **I6. Rename `Wait 15s` → `Wait 20s`.** Small, but the name lies and that costs debugging time later.

---

## Storage Review

**Verdict: keep GitHub — with one caveat, not a replacement.**

GitHub Contents API for scripts/JSON is a good fit: free, versioned, raw URLs play directly in `<audio>`, and it doubles as the GitHub-Pages host. Do **not** swap it out for the JSON/script artifacts.

The one real limitation is **WAV in git history**. The `audio/` directory is already ~490 MB across ~80 files, and every WAV is stored in git *forever* — the repo only grows, clones get slower, and you'll eventually hit push/size friction. This is the single storage issue worth acting on, and only later:
- Keep JSON/scripts in the repo as-is.
- Move WAV output to something that doesn't bloat history: a GitHub **Release asset**, a dedicated audio branch you periodically squash, or object storage (R2/S3) with the raw URL returned to the client exactly as today. The frontend only needs a playable URL — the source is transparent to it.

No urgency, but budget for it before the repo gets painful to clone.

---

## AI Prompt Review

**Overall: strong.** Every call requests structured JSON, temperatures are sensible (STT 0.1, QA 0.2, planners 0.5), and the authoritative-IR re-imposition makes drift non-fatal. Specific notes:

- **Consistency:** excellent by design — `Extract Scene Plan` discards any timing/narration the model changes, so output is stable even when Gemini misbehaves. `director_drift` gives you an observability signal; surface it in logs.
- **JSON reliability:** good. All extractors guard `JSON.parse`. `Save New Components` even falls back to a `\{[\s\S]*\}` regex extract. Add the same guard to `PCM to WAV`'s candidate read (see F2).
- **Token usage:** `Build Visual Request` is the outlier — oversized, repetitive system prompt (I4). Everything else is proportionate. `Analyze Assets` correctly uses the cheap `flash-lite` for planning.
- **Prompt correctness:** the STT prompt (Burmese-only, no translation, contiguous timed segments) and the Scene Director prompt (strict 1:1 mapping, "you are NOT a scene planner") are well-constrained. No changes needed.

---

## Recommended Next Steps (prioritized)

1. **F1 — add a shared error responder.** One `Error Trigger` (or shared `Respond Failure` node) wired from the risky nodes. Biggest reliability win, smallest effort.
2. **F3 — validate before every GitHub upload.** Stop committing empty/error artifacts; return 422 instead. Prevents storage rot and false "success" in the UI.
3. **F2 — guard `PCM to WAV`** against 200-without-audio; route to retry or typed error.
4. **F4 — verify & pin all Gemini model IDs** in one place (check `gemini-3.6-flash` first).
5. **F5 — soft-fail `Fetch Registry (Assets)`; error-route `SSH Extract Stills`.**
6. **I3 — fix TTS retry to rotate keys**, and reconsider the fixed 20 s per-chunk wait if generation speed matters.
7. **I1 — de-duplicate the IR logic** between `Build Scene IR` and `Prep QA`.
8. **Later:** trim the visual prompt (I4), retire the legacy `visual_script` path (I2), and plan WAV off-loading out of git history (Storage).

**Rule of thumb going forward:** every path that opens the webhook must close it — with either a success *or* an error response — and nothing should reach GitHub until it's been validated. Fix those two invariants (F1 + F3) and the pipeline stops surprising you in production.
