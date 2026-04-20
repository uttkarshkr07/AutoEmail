# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a personal recruitment toolkit for PM internship outreach, containing:
- **`email_generator.html`** — A standalone browser app for AI-powered cold email generation
- **`main.tex`** — LaTeX resume source (compiled with `pdflatex main.tex` via TeX Live)
- **Excel files** (`CV_Company_Match.xlsx`, `aLUM cOMPANIES.xlsx`, etc.) — Company outreach data

## Running the App

No build step or server required. Open `email_generator.html` directly in any modern browser. The app is fully self-contained with CDN-loaded dependencies.

To compile the LaTeX CV:
```
pdflatex main.tex
```

## Architecture: `email_generator.html`

Single HTML file with inline CSS and vanilla JavaScript. No framework, no bundler, no backend.

**State** lives in three variables:
- `companies[]` — loaded from Excel upload or hardcoded defaults (21 entries)
- `emailState{}` — maps company ID → `{ status, subject, body, error }`
- `selectedId` — currently viewed company

**Persistence** via `localStorage` keys: `email_api_key`, `email_companies`, `email_state`.

**Data flow:**
1. User uploads an Excel file → parsed with the XLSX library (CDN) → populates `companies[]`
2. User triggers generation → `generateEmail(id)` calls Google Gemini API
3. Response parsed for `SUBJECT: ...` / `BODY: ...` format → stored in `emailState`
4. UI re-renders the right panel; status badge in sidebar updates

**Gemini API details:**
- Endpoint: `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent`
- Temperature 0.7, max tokens 8192
- API key entered by user in the UI and persisted in localStorage

**Excel schema** expected by the parser (columns by name, case-insensitive): `ID`, `Name`, `POC`, `Sector`, `Email`, `LinkedIn`.

**Hardcoded prompt constraints** (important when editing the prompt): no em-dashes, no contractions, plain-text output only, sector-specific tone rules (e.g., no healthcare terminology for companies like ScanAid).
