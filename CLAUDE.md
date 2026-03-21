# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A single-page static HTML/JavaScript application for the Hong Kong Psychological Society (HKPS) Registration Board. It's an AI-assisted vetting tool that helps assess psychologist registration applications. There is no build system, no package.json, no frameworks — just `index.html`.

## Running the App

Open `index.html` directly in a browser, or serve it locally:

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

Deployment is via GitHub Pages (push to `main` branch).

## Architecture

The entire application lives in `index.html` as a single file with inline `<style>` and `<script>` blocks.

**Application flow (4 phases):**
1. **Upload** — User drops/selects files (PDF, JPEG, PNG, TIFF, BMP, WebP)
2. **Tag** — User assigns each file a document role (application form, degree certificate, transcript, etc.)
3. **Vetting** — Files are base64-encoded and POSTed to a Make.com webhook via corsproxy.io (CORS bypass); an AbortController allows cancellation
4. **Report** — Split-view layout: left side shows document tabs with PDF/image preview; right side shows the AI-generated eligibility report with 6 criteria

**Client-side state (key variables in JS):**
- `files[]` — uploaded File objects
- `assignments{}` — maps file index → assigned role string
- `blobUrls{}` — maps file index → blob URL for in-browser preview
- `abortCtrl` — AbortController for canceling the in-flight webhook request

**Webhook payload shape:**
```json
{
  "documents": [
    { "role": "...", "fileName": "...", "mimeType": "...", "fileData": "<base64>" }
  ]
}
```

**Webhook response shape** (from Make.com):
```json
{
  "applicantName": "...",
  "criteria": [
    { "name": "...", "status": "Pass|Fail|Clarify", "details": "..." }
  ]
}
```

## Key Design Decisions

- **CORS proxy:** The webhook is hosted on Make.com; `corsproxy.io` is used to bypass browser CORS restrictions. The webhook URL itself is hardcoded in the script and hidden from the UI.
- **No AI summary section:** Was removed in a recent update; report renders individual criteria only.
- **PDF download:** Uses `window.print()` with print-specific CSS (not a real PDF library).
- **Demo disclaimer:** Footer marks this as a demo/beta for internal HKPS use only.
- `hkps_vetting.html` is the legacy v1.0 file; all active work is in `index.html`.
