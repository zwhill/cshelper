# CLAUDE.md — CSHelper

This file provides guidance to Claude Code when working in this repository.
It is also the session context file for Claude.ai — upload this alongside `index.html` at the start of every Claude.ai chat session.

**Standing instruction:** At the end of every session, before the final commit and push, update this file to reflect what changed (move shipped items from In Progress to Features Shipped, update version, note any new gaps or in-progress work) and include `CLAUDE.md` in the commit alongside `index.html` and `version.json`.

---

## What This Is

CSHelper (`zwhill.github.io/cshelper`) is a tech-support knowledgebase for Holley's Domestic Tuning brands — DiabloSport, Edge Products, Superchips, Range Technology, and Training/Policies. Built for green CS techs to self-serve product knowledge and reduce engineer escalations.

It is a **single self-contained file** — `index.html` (~1600+ lines) with all HTML, CSS, and JavaScript inline. No build step, no dependencies, no package manager, no tests.

`_headers` is a Cloudflare Pages headers file that sets `frame-ancestors *` so the app can be iframed anywhere — it is designed to eventually be embedded in Microsoft Teams.

---

## Developing / Running

- **Run locally:** open `index.html` in a browser, or `python3 -m http.server`. Graph sync and Teams SSO only work under the registered redirect URI.
- **Deploy:** push to repo — Cloudflare Pages deploys automatically. No compile/lint/test pipeline.
- All edits happen directly in `index.html`. Inline `<script>` starts at ~line 330; CSS is in `<head>`.

---

## Stack

| Layer | Detail |
|---|---|
| Hosting | Cloudflare Pages — `github.com/zwhill/cshelper` · local: `~/cshelper` |
| Proxy | Cloudflare Worker — `cshelper-proxy.zwhill.workers.dev` (CORS proxy for Anthropic + GAS) |
| AI | Anthropic API `claude-sonnet-4-20250514` — key stored as Cloudflare secret (not hardcoded) |
| Analytics | Google Sheets + Apps Script (sheets: Sessions, AIQueries, DocViews, Searches, Feedback) |
| Graph Auth | MSAL / Teams SSO — Azure AD app registration pending IT approval |
| Dev | Claude Code — local repo at `~/cshelper` on Zach's MacBook |

GAS URL: `https://script.google.com/macros/s/AKfycbyVdmcUve0LV3K8i60qIffHyiBUccbHNn6r3I-2RGNrcKd58SM68-nDH1ukx6j1POY61Q/exec`
Cloudflare account: Zwhill@hotmail.com (personal — moving to Holley org billing is a pending action)

---

## Architecture (all inside index.html)

Script is organized into commented sections (`// ── SECTION ──`). Key ones:

**CONFIG / SEED DATA** (~lines 330–700)
Hardcoded constants and the document catalog. `DRIVE_ID`, `FOLDER_IDS` (one SharePoint folder per brand), `BRANDS`, `TAG_MAP`. `SEED_DOCS` is ~226 documents built via `sd(id, name, brand, folder, type, uid)` — each points at a SharePoint embed URL (`SP_EMBED` + uid). These are the offline fallback shown before any live sync.

**GRAPH AUTH** (`getToken`)
Acquires a Microsoft Graph token. Prefers Teams SSO (`microsoftTeams.authentication`) when embedded, falls back to MSAL popup/silent (`User.Read`, `Files.Read.All`).

**DELTA SYNC** (`runDeltaSync` / `processFolderDelta`)
Admin-only. Walks each brand folder's Graph `delta` endpoint, merges changes into `allDocs`, persists per-brand delta tokens to `localStorage` (`cs_kb_delta`, `cs_kb_sync_meta`) for incremental syncs. `buildDoc` / `getDocType` map Graph items to the same doc shape as seed data.

**RENDER / SEARCH / MODAL**
`renderGrid`, `applyFilters` / `filterBrand` (brand sidebar), `localSearch`, `openDoc` (opens SharePoint embed in modal).

**AI** (`ask`)
Client-side retrieval + LLM. Filters `allDocs` for matches, builds a system prompt with doc names/snippets, POSTs to Anthropic via Cloudflare Worker proxy (`claude-sonnet-4-20250514`). `addFeedback` / `submitFeedback` capture thumbs up/down. Negative feedback requires a note before submitting. `addEscalateBtn` adds an escalation summary button that auto-copies to clipboard.

**ANALYTICS LOGGING** (`gasLog`)
Fires-and-forgets to GAS via Cloudflare Worker proxy. `sessionId` generated once per page load via `crypto.randomUUID()`. `aiCtx` fully removed — `browseBrandFilter` (value of `activeBrand` at query time) is used instead. Sheet schemas:

- **Sessions**: `[ts, sessionId, agentName, platform, appVersion]` — fired once after name is set.
- **AIQueries**: `[ts, sessionId, agentName, query, browseBrandFilter, hitsCount, hitsIds, responseMs, isFollowUp, queryIndex]`
- **DocViews**: `[ts, sessionId, agentName, docName, brand, docType]`
- **Searches**: `[ts, sessionId, agentName, searchTerm, resultCount, eventType]` — eventType is `search` or `search_click`
- **FeedbackAI** (renamed from Feedback): `[ts, sessionId, agentName, query, browseBrandFilter, aiResponse, rating, freeText, queryIndexInSession, platform, currentDocId, currentDocName, hitsCount, hitsIds, responseMs]`
- **FeedbackGeneral** (new): `[ts, sessionId, agentName, platform, feedbackType, detail, activeBrand, lastQuery, currentDocId, currentDocName]` — opened via Feedback button in topbar header

`currentDocId` and `currentDocName` are session-level vars set on doc open (`openDoc()`), reset to `null` on modal close (`closeModal()`). All `gasLog('Feedback', ...)` references renamed to `gasLog('FeedbackAI', ...)`.

**Manual steps required:** Create `Sessions`, `FeedbackGeneral` sheets in Google Sheet with correct headers (see schema above).

Feedback buttons (header topbar + doc viewer bottom bar) use an outlined accent style: `1.5px solid var(--red)`, transparent background, red text, fills red on hover. Rating values corrected to `'up'` / `'down'` (was `thumbs_up` / `thumbs_down`). Console filter accepts both for backward compatibility.

**Console view**
Admin-only analytics tab. `renderTopQueries`, `renderPainPoints`, `renderBrandChart`, `renderTopDocs`, `renderSyncHealth`, `renderQueryLog`. Two views toggled by `switchTab`: `view-kb` and `view-console`.

**Roles**
Two roles: `agent` (default) and `admin`. Admin unlocked via `promptAdminAccess`, exposes sync controls and Console tab (`applyRoleUI`).

---

## Doc Object Shape

All doc objects — seed and live — share this shape. Preserve it when adding code paths:

```js
{ id, name, brand, url, embedUrl, type, snippet, live, modified?, size?, changeType? }
```

Brand keys (`DiabloSport`, `Edge`, `Superchips`, `Range`, `Training`) are used as object keys across `FOLDER_IDS`, `BRANDS`, and `deltaTokens` — keep them consistent.

`esc` / `escId` are the only escaping helpers — always use them when injecting user or doc strings into `innerHTML`.

---

## Current Version

<!-- VERSION --> 1.2.10

This line must stay in sync with `const VERSION` in `index.html` and `version.json` in the repo root. All three update together in every commit.

---

## Features Shipped

- 226 docs seeded across all brands with SharePoint embed previews
- AI assistant: brand context, markdown rendering, auto-growing textarea
- Thumbs up/down feedback — negative requires a free-text note before submitting
- Escalation summary button — sends fixed prompt to AI, auto-copies result to clipboard
- Admin/console tab: session + all-time analytics, pain points (zero-hit queries + thumbs down), query log
- Collapsible sidebar (starts open on desktop), horizontally resizable AI panel (drag left border)
- Agent name modal on load — stored in `agentName`, prepended to all GAS log rows
- Version polling every 60s against `version.json` — banner prompts hard reload on new deploy
- Mobile layout: tab-based (Docs / AI / Info), 36px topbar, full-screen search overlay, 3x2 brand filter grid, fixed bottom tab bar
- Mobile doc viewing uses `window.open()` to bypass Safari third-party cookie iframe block
- Delta sync scaffolded — blocked on Azure AD
- Citation pills in AI responses — inline `.cite-pill` elements after each AI reply, clicking opens SharePoint embed modal
- Revision history card in console — versioned CHANGELOG constant, rendered on console open with tags and timeline dots
- Version number in sidebar footer is clickable — opens revision history in the doc modal

---

## In Progress

- **AI policy guardrails** — `AI_POLICY.md` + system prompt additions to explicitly block emissions defeat device support and non-compliant tuning scenarios
- **Org API key** — cost case being built to move from personal Cloudflare secret to Holley-owned Anthropic billing account (~$200/mo cap to start)

---

## Known Gaps / Placeholders

- `clientId` in `getToken` is `'YOUR_CLIENT_ID'` — must be set for Graph sync / Teams SSO to work
- `ADMIN_PASSWORD` is hardcoded (`holley2025`) — fine for POC
- Azure AD app registration pending IT — blocks delta sync and proper Graph API auth
- SharePoint iframe embeds blocked on mobile Safari (workaround in place)
- Anthropic API key on personal Cloudflare account — needs org billing before production handoff

---

## Long-Term Vision

- Blessing queue on DomesticTuning-Wiki (draft/pending/live status on pages) — only live pages feed CSHelper AI context
- CSHelper AI pulls live wiki content via shared Worker instead of static SharePoint seed docs
- Azure AD auth → Graph API sync → identity on logs
- Engineer ping/queue when AI can't resolve a ticket
- Power BI analytics on SharePoint
- Teams embedding (headers file already in place)
- Clean handoff to a developer with this file and a tidy repo

---

## Related Repo

**DomesticTuning-Wiki** — `github.com/zwhill/DomesticTuning-Wiki` · local: `~/DomesticTuning-Wiki`
Single-file internal wiki on same Cloudflare Worker and origin. Cloudflare KV storage (`WIKI_KV`, key: `wiki:pages`). No auth. Shares `cshelper-proxy` worker.

---

## Conventions

- Single-file app — all changes go in `index.html` unless explicitly noted
- Every session must bump `const VERSION` in `index.html`, update `version.json`, and update the version line in this file — all three must match and commit together
- All Claude Code prompts delivered to Zach must be wrapped in a code block for easy copying
- Never use markdown formatting inside Claude Code prompts
- POC mode — pragmatic over perfect, but flag anything that won't survive a real production handoff
- **At the end of every session, before final commit and push: update this file and include it in the commit**

---

## Session Start Checklist (Claude.ai)

1. Zach uploads `index.html` and `CLAUDE.md`
2. Confirm current VERSION matches the version line above
3. Review In Progress — confirm what's still outstanding
4. Ask Zach what the priority is for this session
