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
3. **Vetting** — Images are compressed client-side, all files are base64-encoded and POSTed directly to the Make.com webhook; an AbortController allows cancellation
4. **Report** — Split-view layout: left side shows document tabs with PDF/image preview; right side shows the AI-generated eligibility report with 6 criteria

**Client-side state (key variables in JS):**
- `files[]` — original uploaded File objects
- `processedFiles[]` — compressed versions of images (same index as `files[]`; PDFs are passed through unchanged); used for upload and preview in the split view
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

**Webhook response shape** (flat fields read by `renderReport()`):
```json
{
  "name": "Applicant Full Name",
  "overall_status": "Pass|Fail|Clarify",
  "cross_check_status": "...", "cross_check_reason": "...",
  "academic_status":   "...", "academic_reason":    "...",
  "membership_status": "...", "membership_reason":  "...",
  "higher_degree_status": "...", "higher_degree_reason": "...",
  "experience_status": "...", "experience_reason":  "...",
  "declaration_status": "...", "declaration_reason": "..."
}
```

The `norm()` function maps any status string to `Pass`, `Fail`, or `Clarify` using keyword matching.

## Key Design Decisions

- **Direct webhook call:** The Make.com webhook URL is hardcoded in the `<input>` inside a `display:none` config row (visible in the DOM, hidden from users). The app POSTs directly to `hook.eu2.make.com` — no CORS proxy is needed because Make.com allows cross-origin requests.
- **Image compression:** Before tagging, all image files (JPEG, PNG, TIFF, etc.) are re-encoded as JPEG at ≤1920px and 0.78 quality via Canvas. The compressed `processedFiles[]` replaces originals for upload; the split-view preview also uses compressed blobs. PDFs are never compressed.
- **5 MB payload guard:** `estimatePayloadBytes()` accounts for base64 expansion (~33% overhead). The "Run" button is disabled if the estimated payload exceeds 5 MB. The size bar shows current estimated payload vs. limit.
- **Partial submissions allowed:** Only `application_form`, `first_degree_cert`, `first_degree_transcript`, `higher_degree_cert`, and `higher_degree_transcript` are in `REQUIRED`. Missing required docs generate a warning but don't block submission; the AI flags them as "Needs Clarification".
- **PDF download:** Uses `window.print()` with print-specific CSS (not a real PDF library).
- `hkps_vetting.html` is the legacy v1.0 file; all active work is in `index.html`.
