# Mind Grapes browser extension — spec

Chrome MV3 extension to bookmark the current page into Mind Grapes. Client half
of the feature; the server `POST /capture` endpoint lives in the `openbrain`
(mindgrapes-server) repo — see MindgrapesApp/mindgrapes-server#35.

## What it does

Click the extension action → extract readable page text → send URL + text to the
Mind Grapes `POST /capture` endpoint → the server summarizes and stores it → the
popup shows "Saved ✓" and the summary.

## User flow (popup, 3 states, explicit Save click)

```
┌─ not authed ────────┐   ┌─ saving ────────────┐   ┌─ saved ─────────────┐
│  🍇 Mind Grapes     │   │  🍇 Saving…         │   │  🍇 Saved ✓         │
│  [ Connect ]        │   │  ▸ summarizing page │   │  "<summary…>"       │
│                     │   │                     │   │  [ View in brain ]  │
└─────────────────────┘   └─────────────────────┘   └─────────────────────┘
```

- Popup opens showing the current page; user clicks **Save** (no auto-save on open).
- First use: **Connect** runs the OAuth flow, then Save is available.
- Feedback is mandatory — summarization takes a beat, never a dead popup.
- Accessible: keyboard-reachable, `aria-label` on the action, status via `aria-live`.

## Architecture

```
manifest.json (MV3)
background.js
  · chrome.identity.launchWebAuthFlow  → OAuth 2.1 PKCE (public client)
  · token cache in chrome.storage
popup.html / popup.js                  → 3-state UI
content: Readability.js                → {title, text} from the active tab
        │
        ▼  HTTPS, Authorization: Bearer <JWT>, { url, title, text }
   POST /capture  (mindgrapes-server#35)
        │
        ◀  { experience_id, summary }
```

- **Auth** reuses the existing Mind Grapes OAuth 2.1 PKCE public-client setup;
  the OAuth JSON endpoints already send permissive CORS. Use
  `chrome.identity.launchWebAuthFlow` — the native MV3 primitive, no library.
- **Extraction** via vendored Readability.js (native platform + one dep, no
  server-side scraping).
- **No secrets in the extension** — public client, PKCE, no client secret.

## Files

- `manifest.json`, `background.js`, `popup.html`, `popup.js`, vendored `readability.js`
- `tests/e2e/` — Playwright, loads the unpacked extension into Chrome

## Acceptance / stop condition (the loop exits here)

End-to-end validation test: with the dev `openbrain` stack running, the extension
bookmarks **https://claude.com/blog/getting-started-with-loops**, and the test
asserts an experience with that `source_ref` and a non-empty summary appears in
the dev brain.

- Inner build loop uses a **dev JWT** for speed (headless, no interactive OAuth).
- The **real OAuth PKCE flow** is covered by one separate test, not per-iteration.

## Build technique

Built as a goal-based loop (https://claude.com/blog/getting-started-with-loops):
iterate implement → run e2e → read failure → fix, until the stop-condition
assertion goes green.

## Out of scope (v1)

Tags/folders, in-popup summary editing, offline queue, non-Chrome browsers,
server-side background summarization loop.
