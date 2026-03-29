# Commitment Radar — Design Spec
**Date:** 2026-03-29
**Status:** Approved
**File:** `commitment-radar.html`
**Repo:** https://github.com/gparrent71-hue/clearthread
**Live URL (after deploy):** https://gparrent71-hue.github.io/clearthread/commitment-radar.html

---

## Overview

Commitment Radar is a personal operating system for promises, follow-ups, and obligations hidden inside meetings, emails, and notes. It replaces the question "what am I forgetting?" with a single screen that shows what's hot, what's approaching, what's parked, and what deserves long-term investment.

It is a superset of ClearThread — ClearThread's thread extraction engine becomes the data ingestion layer. ClearThread itself stays live as a simpler alternative.

---

## Architecture

### File & Stack
- Single file: `commitment-radar.html`
- Vanilla HTML/CSS/JS, no build system
- Claude API (browser fetch, same pattern as ClearThread)
- Firebase Firestore (same project: `milestone-tracker-955f4`, new collections)
- Open in Chrome or Edge (not Safari)

### Transplanted from ClearThread (verbatim)
- Firebase config + Firestore helpers (`listenItems`, `saveItemsBatch`, `updateItem`, `deleteItem`)
- Claude API fetch + `sanitizeJson` + JSON parsing/retry logic
- Chunked Web Worker (`<script type="javascript/worker">`) for file imports
- Email preprocessing (`preprocessEmails`, `stripSignature`, deduplication, Outlook metadata)
- Import modal (paste tab + file tab)
- Settings modal (API key, model selector — extended with Workspace Context)
- Button/form CSS design system (variables, `.btn`, `.modal`, `.overlay`)

### New in Commitment Radar
- 4-column dashboard (Hot / Approaching / Parked / Invest)
- Updated item data model (6 categories, zone, confidence, roleTag, followUpDate)
- Updated system prompt (new categories, confidence scoring, Invest suggestions, Workspace Context injection)
- Aging engine (runs on load + every 5 min, escalates items between zones)
- Workspace Context: free-text field in Settings, prepended to every Claude system prompt

### Firestore Collections
| Collection | Purpose |
|---|---|
| `radar_items` | Commitment items (new) |
| `radar_threads` | Import session records (new) |
| `clearthread_items` | ClearThread items — untouched |
| `clearthread_threads` | ClearThread sessions — untouched |

**Note:** New Firestore collections must be added to Firebase security rules or all reads/writes will be silently blocked.

### Deployment
Push `commitment-radar.html` to `main` branch of `gparrent71-hue/clearthread` repo. Live within ~60 seconds.

---

## Data Model

Each item in `radar_items`:

```json
{
  "id": "firestore-generated",
  "threadId": "string",
  "threadTitle": "string",
  "createdAt": "timestamp",
  "lastActivityAt": "timestamp",

  "category": "my_commitment | their_commitment | waiting | risk | decision | followup",
  "zone": "hot | approaching | parked | invest",
  "zoneLocked": "boolean",

  "description": "string",
  "owner": "string | null",
  "dueDate": "string | null",
  "followUpDate": "string | null",
  "sourceSnippet": "string",
  "notes": "string",

  "confidence": "high | medium | low",
  "investSuggested": "boolean",

  "status": "open | done | snoozed",
  "roleTag": "pm | mop | csl | personal | null",

  "outlookCategory": "string | null",
  "flagged": "boolean"
}
```

### Zone Assignment Logic

**At import time** (before save):
- `dueDate` within 7 days → `hot`
- `dueDate` 8–30 days → `approaching`
- `dueDate` 30+ days or no date → `parked`
- `investSuggested: true` AND user accepted in preview → `invest`

**Note on dueDate parsing:** `dueDate` is stored as natural language (e.g., "Thursday", "Mar 31", "EOD April 5"). The zone assignment logic and aging engine must parse these strings into actual dates relative to the current date. Use a lightweight date parsing approach (e.g., `Date.parse()` with normalization, or a small inline parser). Items where `dueDate` cannot be parsed are treated as "no date" → `parked`.

**Aging engine** (on load + every 5 min), skips `zoneLocked: true` and all `invest` items:

| Condition | Zone |
|---|---|
| `dueDate` within 7 days | `hot` |
| No `lastActivityAt` update in 10+ days | `hot` |
| `dueDate` 8–30 days | `approaching` |
| `dueDate` 30+ days, recently active | `parked` |
| `zone === 'invest'` | Exempt — never moved |
| `zoneLocked === true` | Exempt — user chose it |

### Confidence Scoring (Claude-assigned)
- `high` — explicit, named commitment ("I will send the draft by Friday")
- `medium` — implied or soft ("we should probably get that reviewed")
- `low` — loose discussion point or speculative reference

### roleTag
Included in schema from day one for future v2 role-lens feature (PM / MOP / CSL / Personal view switching). Claude does NOT extract roleTag in v1 — it always defaults to `null`. User can set it manually in the edit modal. Not surfaced in v1 UI beyond the edit modal field.

---

## Dashboard UI

### Layout
Header → Filter bar → 4-column radar board (full viewport height, columns scroll independently).

### Columns
| Column | Color | Meaning |
|---|---|---|
| 🔴 Hot | Red | Needs attention now — overdue, aging, or within 7 days |
| 🟡 Approaching | Amber | On the horizon — 8–30 days out |
| 🔵 Parked | Blue | Future or no deadline — tracked but not pressing |
| 🟣 Invest | Purple | No urgency, high long-term value — relationships, mentoring, growth |

### Item Cards
Each card shows:
- **Left border color** — category at a glance
- **Category badge** — My Commitment / Their Commitment / Waiting On / Risk / Decision Needed / Follow-up
- **Confidence dot** — ● High (green) / ● Med (amber) / ● Low (grey)
- **Description** (1–2 sentences)
- **Owner** (if set)
- **Due date** (color-coded: red = overdue, amber = soon, grey = normal)
- **Age badge** — "Xd no activity" — appears when `lastActivityAt` > 7 days (early warning); items escalate to Hot zone at 10 days. Invest items are exempt from both.

### Filter Bar
Category chips (All / My Commitment / Their Commitment / Waiting On / Risk / Decision Needed / Follow-up) + status chips (Open / Done / Snoozed). Filters apply across all four columns simultaneously.

### Card Actions (on hover or click)
- Mark Done / Reopen
- Edit (opens edit modal)
- Move to zone (manual zone override, sets `zoneLocked: true`)
- Delete

### Edit Modal
Fields: Category, Description, Owner, Due Date, Follow-up Date, Zone (with lock), Status, Notes, Role Tag (hidden until v2 UI, but editable).

---

## Claude Extraction

### Model Selection
Settings modal offers three models (stored in `localStorage` under `radar_model`):
- `claude-opus-4-6` — most capable, ~$0.01–0.05/thread
- `claude-sonnet-4-6` — balanced, ~$0.003–0.01/thread
- `claude-haiku-4-5-20251001` — fastest/cheapest, ~$0.001–0.005/thread

**File imports always use Haiku** regardless of selection (rate limit safety). Paste tab uses selected model.

### System Prompt (key additions over ClearThread)

**Categories:**
- `my_commitment` — user explicitly committed or was directly assigned
- `their_commitment` — another named person committed
- `waiting` — user is blocked pending someone else's action
- `risk` — unaddressed threat to schedule, scope, budget, or relationship
- `decision` — a choice needed before work can proceed
- `followup` — a conversation, check-in, or relationship touch that should happen

**Confidence:** `high` (explicit + specific) / `medium` (implied/soft) / `low` (vague/speculative)

**Invest flagging:** `investSuggested: true` when item has no urgency but clear long-term value — mentoring opportunities, relationship investments, learning moments, visibility opportunities, strategic follow-ups with no deadline.

### Workspace Context Injection
Free-text field in Settings (saved to `localStorage` under `radar_workspace_context`). Prepended to system prompt as:
```
--- YOUR WORKSPACE CONTEXT ---
[user text]
--- END CONTEXT ---
```
Suggested length: under ~500 words. Helps Claude make better ownership and priority judgments using knowledge of your specific projects, people, and role context.

### Extracted Item JSON (per item)
```json
{
  "category": "my_commitment",
  "description": "string",
  "owner": "string | null",
  "dueDate": "string | null",
  "followUpDate": "string | null",
  "sourceSnippet": "string",
  "confidence": "high | medium | low",
  "investSuggested": false,
  "outlookCategory": "string | null",
  "flagged": false
}
```

---

## Import Flow

Carries over from ClearThread with two additions:

1. **Zone preview chip** — each item in the preview list shows where it will land (🔴/🟡/🔵/🟣) based on `dueDate` and `investSuggested`
2. **Invest badge toggle** — items with `investSuggested: true` show a purple `Invest?` badge; clicking toggles acceptance. Default is accepted.

Everything else unchanged: chunked file processing (50k char chunks, Haiku, Web Worker), deduplication, Outlook metadata, cancel button, thread naming, duplicate detection against existing items.

---

## Workspace Context

- Location: Settings modal, below model selector
- Label: "Workspace Context"
- Input: `<textarea>`, ~6 rows
- Saved to: `localStorage` under `radar_workspace_context`
- Helper text: "Describe your key projects, people, and roles. Claude uses this to make better judgments about ownership and priority. Keep it under ~500 words."
- Injected into every Claude call (both paste and file/worker)

---

## Out of Scope (v1)

- Role lens UI switching (PM / MOP / CSL / Personal views) — `roleTag` included in data model, UI deferred to v2
- Conversation prep brief generator — v2
- Tone-aware follow-up drafting — v2
- Pattern learning / analytics — future
- Multi-user / shared workspaces — not planned
