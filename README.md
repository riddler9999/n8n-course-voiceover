# n8n Course Voiceover Generator

GitHub Pages frontend for the n8n Course Voiceover automation pipeline.

## Architecture

```
GitHub Pages (this repo)
   ↓ POST webhook
n8n Workflow (Single Webhook + Action Router)
   ├── action: script → Gemini 3.1 Flash Lite → return text
   └── action: audio  → Gemini 3.1 TTS (Charon) → PCM-to-WAV → GitHub Contents API → return raw URL
```

## Workflow

1. Pick a lesson from the dropdown
2. Pick a style (calm / technical / excited / warning)
3. Click **Generate Script** — Gemini 3.1 Flash Lite returns a voiceover script
4. Edit the script in the textarea if needed
5. Click **Confirm and Generate Audio** — generates WAV via Gemini TTS and uploads to this repo
6. Play in browser or click **Download WAV**

## Setup

### n8n Side
- Workflow ID: `hjay2RRs99eypw6c`
- Webhook URL: `https://n8n.srv1695682.hstgr.cloud/webhook/n8n-course-voiceover`
- Required credentials (configure in n8n UI):
  - **Gemini API Key Header** (`httpHeaderAuth`)
    - Name: `x-goog-api-key`
    - Value: `<your Gemini API key>`
  - **GitHub Token Header** (`httpHeaderAuth`)
    - Name: `Authorization`
    - Value: `Bearer <your GitHub PAT>` (with `repo` scope)

### GitHub Side
- Repo: `riddler9999/n8n-course-voiceover`
- Pages: enable on `main` branch
- Audio files land in `audio/lesson_{id}_{timestamp}.wav`

## Course Outline

Module 1 — Foundations (Lessons 1–5)
Module 2 — Core Components (Lessons 6–14)
Module 3 — Integration Systems (Lessons 15–18)
Module 4 — AI Automation Projects (Lessons 19–21)
Module 5 — Production Build (Lessons 22–24)
