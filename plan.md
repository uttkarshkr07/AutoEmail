# Project Plan — PM Intern Email Generator

---

## 1. Project Overview

A standalone, single-file browser app (`email_generator.html`) that helps Uttkarsh Kumar (pre-final year Dual Degree, IIT Kharagpur) generate tailored cold outreach emails for Product Management Intern applications. The app is preloaded with 21 target companies and uses the Gemini API to write a customised email for each contact. The user can upload a new Excel sheet at any time to replace the company list, generate emails one by one or in bulk, and copy them out for sending. The end goal is to add direct Gmail integration so emails can be sent from within the app without copy-pasting.

---

## 2. Tech Stack

| Layer | Choice |
|-------|--------|
| Runtime | Browser (open HTML file directly — no server) |
| Language | Vanilla JavaScript (ES2020+), HTML5, CSS3 |
| AI | Google Gemini API — model `gemini-3-flash-preview` |
| Excel parsing | SheetJS (CDN: `https://cdn.jsdelivr.net/npm/xlsx/dist/xlsx.full.min.js`) |
| Persistence | `localStorage` (API key, companies array, per-company email state) |
| Gmail sending | **Planned** — Gmail API via OAuth 2.0 (not yet implemented) |
| CV source | LaTeX (`main.tex`), compiled with `pdflatex` via TeX Live |

---

## 3. Architecture

```
intern dhundho/
├── email_generator.html   ← entire app (CSS + JS inline, no build step)
├── main.tex               ← LaTeX CV source
├── IITKGP_CV__Template___ ← compiled CV PDF
├── CV_Company_Match.xlsx  ← company outreach data (original)
├── CV_Company_Match_v2.xlsx
├── aLUM cOMPANIES.xlsx
├── Book1.xlsx
├── plan.md                ← this file
└── CLAUDE.md              ← Claude Code instructions
```

### `email_generator.html` internals

```
State variables
  companies[]         parsed company objects {id, name, poc, sector, email, linkedin}
  emailState{}        map of company id → {status, subject, body, error}
  selectedId          currently viewed company

Data flow
  1. Load → restore from localStorage or use DEFAULT_COMPANIES (21 entries)
  2. User selects company → renderPanel() shows status
  3. Generate → buildPrompt(c) → callGemini(c) → setState(id, result)
  4. Parse: plain-text SUBJECT: / BODY: delimiters (not JSON)
  5. Copy actions → navigator.clipboard
  6. Upload .xlsx → parseExcel() → replace companies, reset all state

localStorage keys
  email_api_key       Gemini API key string
  email_companies     JSON array of current companies
  email_state         JSON object mapping company id → state
```

---

## 4. Current State

### (a) Working
- App loads with hardcoded 21-company dataset
- Sidebar with company list, status dots (pending/loading/done/error)
- Per-company and bulk ("Generate all") email generation with 1s delay between calls
- Stop button to cancel bulk generation mid-run
- Plain-text SUBJECT:/BODY: parsing of Gemini response
- System instruction to suppress model thinking/preamble
- Copy body / Copy subject / Copy all / Regenerate buttons
- Excel upload with filtering (skips blank/invalid emails, non-URL LinkedIn)
- localStorage persistence across page refresh
- API key input persisted in localStorage

### (b) In Progress / Partially Done
- **Gemini model stability**: currently on `gemini-3-flash-preview`. The model sometimes outputs thinking steps before the email even with system instruction. Not yet confirmed fully stable.
- **Email generation reliability**: SUBJECT:/BODY: parsing is working but hasn't been tested across all 21 companies yet.

### (c) Broken / Blocked
- **Gmail send integration**: not implemented. User wants to send emails directly from the app. Requires Gmail API OAuth 2.0 flow, which is non-trivial in a local HTML file (needs a client ID from Google Cloud Console and either a redirect URI or a popup flow).
- **Gemini model for new API keys**: `gemini-2.0-flash` and `gemini-2.0-flash-lite` are deprecated for new users. `gemini-3-flash-preview` is the current working model but is a preview and may change.

---

## 5. Active Task

**Implement Gmail send integration** — allow the user to authenticate with Gmail via OAuth 2.0 and send the generated email (with CV attached) directly from the app, without leaving the browser.

---

## 6. Backlog

1. **Gmail OAuth + send** — Add "Send via Gmail" button on done emails. Flow: OAuth 2.0 popup → get access token → Gmail API `users.messages.send` with email body + CV attachment (PDF base64). Requires user to set up a Google Cloud project with Gmail API enabled and provide an OAuth client ID in the UI.
2. **Confirm Gemini model stability** — Test all 21 companies end-to-end. If `gemini-3-flash-preview` still produces thinking preamble, try `gemini-2.5-flash` as fallback.
3. **"Send all" bulk send** — After generating all emails, a button to queue and send all pending emails via Gmail with a delay between sends.
4. **Sent status tracking** — Add a `"sent"` status alongside pending/loading/done/error. Show a distinct dot colour (blue) and timestamp for sent emails.
5. **CV attachment** — Auto-attach the CV PDF when sending via Gmail. Since we can't read local files without a file picker, add a one-time "Select CV PDF" button that stores the base64 in localStorage.
6. **Export to CSV** — Button to download all generated emails as a CSV (company, poc, email, subject, body).
7. **Edit email before sending** — Make the body editable inline before copy/send so minor tweaks don't require regeneration.
8. **Model selector** — Small dropdown in the top bar to switch Gemini model without editing code.
9. **Error retry on "Generate all"** — Currently Generate All skips done entries; make it also re-queue errored entries automatically after a short backoff.

---

## 7. Known Issues / Bugs

| # | Description | Severity |
|---|-------------|----------|
| 1 | `gemini-3-flash-preview` sometimes outputs thinking/reasoning steps before the email, causing SUBJECT/BODY parse to fail. System instruction reduces this but doesn't eliminate it. | Medium |
| 2 | `maxOutputTokens: 8192` is set but the free-tier model may have a lower effective output limit per call. If the email body is still truncated, the parse will silently fail. | Low |
| 3 | localStorage has a ~5MB limit. With 21 companies × long email bodies, it should be fine, but large Excel uploads with 100+ companies and full emails could approach the limit. | Low |
| 4 | No loading indicator in sidebar during "Generate all" for companies not yet started — they show grey (pending) until their turn starts. | Low |

---

## 8. Key Decisions

| Decision | Reasoning |
|----------|-----------|
| Single HTML file, no framework | Personal tool, zero setup friction — open file and use |
| Plain-text SUBJECT:/BODY: parsing instead of JSON | JSON requires the model to escape all newlines as `\n` inside a string, which inflates token usage and causes truncation. Plain text is more token-efficient and robust |
| Sequential generation with 1s delay | Gemini free tier enforces per-minute rate limits; parallelism causes 429 errors |
| `gemini-3-flash-preview` as default model | `gemini-2.0-flash` and `gemini-2.0-flash-lite` deprecated for new API keys as of April 2026 |
| System instruction to suppress thinking | `gemini-3-flash-preview` is a reasoning model that outputs thinking steps; without suppression it consumes the full token budget before outputting the email |
| localStorage for persistence | No backend, no server — localStorage is the only reasonable option for a local single-file app |
| Gmail integration via OAuth 2.0 popup | A local HTML file cannot use a server-side OAuth flow; the implicit/popup flow is the only viable approach without a backend |

---

## 9. Claude Notes

- **Gemini API endpoint pattern**: `https://generativelanguage.googleapis.com/v1beta/models/{MODEL}:generateContent?key={KEY}`
- **Model name is the most fragile thing in this codebase** — it's a single string constant `GEMINI_URL` at the top of the `<script>` block. When the model breaks, change only that constant.
- **The prompt has strict constraints**: no em dashes, no contractions (write "I am"/"I have"), no "healthcare" mention, no "AI-driven healthcare chatbots". These are in the `RULES FOR THE TAILORED PARAGRAPH` section of `buildPrompt()`.
- **Fixed paragraphs must not change** — paragraphs 1, 2, and 4 of the email body are hardcoded in the prompt and must be reproduced exactly by the model. Only the tailored paragraph (paragraph 3) is generated.
- **ScanAid framing rule**: must be described as "translating operational gaps into structured product decisions" — never as a healthcare product.
- **Excel column mapping**: col B=name, C=poc, D=sector, E=email, F=linkedin (0-indexed: row[1]–row[5]). Header row is skipped (i starts at 1).
- **Gmail OAuth for local HTML files**: use `https://accounts.google.com/o/oauth2/v2/auth` with `response_type=token` (implicit flow) or `response_type=code` + PKCE. The redirect URI must be set to the exact file URL or `http://localhost` — this is a known friction point.
- **User's email**: youttkarsh07@gmail.com (used as the Gmail "from" address when send is implemented)
- **User's phone**: +91 7303549462 (appears in email signature — do not change)

---

## 10. Session Log

- **2026-04-18** — Built `email_generator.html` from scratch per `prompt.md` spec. Implemented full app: sidebar, panel, Gemini API call, Excel upload, localStorage persistence, copy buttons, stop button. Fixed model deprecation issues (`gemini-2.0-flash` → `gemini-2.0-flash-lite` → `gemini-1.5-flash` → `gemini-3-flash-preview`). Fixed JSON truncation by switching output format from JSON to plain-text SUBJECT:/BODY: delimiters. Added system instruction to suppress model thinking preamble. Gmail send integration identified as next priority feature.
