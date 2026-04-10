# ClearThread: Multi-User Auth + File Upload Expansion

**Date:** 2026-04-10
**Status:** Approved

---

## Overview

Two features added to ClearThread (`index.html`):

1. **Multi-user authentication** — Firebase Auth with email/password and Google sign-in, password recovery, per-user data scoping in Firestore
2. **File upload expansion** — `.docx` and `.pdf` support alongside existing `.txt`, parsed client-side

A Firebase Cloud Function replaces direct browser-to-Anthropic calls, allowing a shared API key so friends don't need their own Anthropic accounts.

---

## Architecture

### What changes

- **Firebase Auth SDK** added to `index.html` alongside existing Firestore SDK
- **Login overlay** shown on load if user is not authenticated; dismissed on successful auth
- **Cloud Function** (`functions/index.js`) — single HTTPS callable `callClaude` that proxies all Claude API calls
- **Firestore data model** — all writes scoped with `userId: currentUser.uid`; queries filter by `userId`; security rules updated to require auth and ownership
- **File parsing** — mammoth.js (`.docx`) and pdf.js (`.pdf`) added via CDN; file handler routes by extension before feeding into existing chunking pipeline

### What stays the same

Dashboard, import modal, edit modal, Web Worker chunking, duplicate detection, Outlook category features, select mode, bulk delete — all unchanged.

---

## Section 1: Authentication UI

The login overlay is a centered card shown over the app (rest of page blurred/hidden) until the user is authenticated. Three states toggle in place:

**Sign In**
- Email + password fields
- Sign In button
- Google Sign In button
- Links: "Create account" | "Forgot password?"

**Create Account**
- Email + password + confirm password fields
- Create Account button
- Google Sign In button
- Link: "Back to sign in"

**Forgot Password**
- Email field only
- Send Reset Email button — triggers Firebase's built-in password reset email
- Confirmation message shown on submit
- Link: "Back to sign in"

**Google Sign In** works identically on Sign In and Create Account states. Firebase handles the popup, creates the account if new, signs in if existing.

**Header** — once authenticated, user's email and a Sign Out link appear in the header alongside the existing Settings button.

**Settings modal** — API key field removed (users no longer provide their own key).

---

## Section 2: Cloud Function Proxy

**Function name:** `callClaude`
**Type:** HTTPS Callable (Firebase Functions v2)

**Request shape:**
```json
{ "messages": [...], "system": "...", "model": "claude-haiku-4-5-20251001" }
```

**Behavior:**
1. Verifies Firebase Auth token — rejects unauthenticated calls with a 401
2. Reads `ANTHROPIC_API_KEY` from Firebase Functions environment config
3. Forwards request to `https://api.anthropic.com/v1/messages`
4. Returns response to client

**Model options:** `claude-haiku-4-5-20251001` and `claude-sonnet-4-6` only. Opus not offered. Model selector in Settings stays, capped to these two.

**Auth token flow:**
- Paste-tab extractions: main thread calls `firebase.auth().currentUser.getIdToken()` inline before the Claude call
- File imports (Web Worker): main thread fetches token once before spawning the worker, passes it via `postMessage` alongside existing config; worker uses it as `Authorization: Bearer <token>` header on Cloud Function requests

**API key storage:** Firebase Functions environment variable (`ANTHROPIC_API_KEY`). Never in Firestore, never sent to browser. Set once via Firebase CLI:
```
firebase functions:secrets:set ANTHROPIC_API_KEY
```

---

## Section 3: File Upload Expansion

**Libraries (CDN, no build step):**
- `mammoth.js` — extracts plain text from `.docx`
- `pdf.js` 3.11.174 — extracts plain text from PDF (same version used in Weekly Burn Monitor)

**Flow:**
1. File picker `accept` attribute expands to `.txt,.docx,.pdf`
2. On file select, extension checked → routed to appropriate extractor
3. All three extractors produce plain text → feeds into existing `preprocessEmails()` + chunking pipeline unchanged

**Error handling:**
- Image-only (non-text) PDFs: show clear error — "This PDF doesn't contain extractable text. Try exporting from your meeting notes app as a text-based PDF."
- Malformed .docx: show error — "Could not read this Word file. Try saving as .docx from Word or Google Docs."

**File info panel** — shows detected file type (TXT / DOCX / PDF) alongside existing character count and chunk count.

**Limitations:**
- Scanned/image-only PDFs not supported
- Heavily formatted .docx (complex tables, embedded objects) may lose formatting; plain meeting notes extract cleanly

---

## Section 4: Firestore Data Model & Security Rules

### Data model

All `saveItemsBatch()`, `updateItem()`, and thread save calls add `userId: currentUser.uid`.

`listenItems()` query adds `.where('userId', '==', currentUser.uid)`.

No new collections. No schema migration — existing unscoped data is orphaned (not deleted, just invisible to all logged-in users).

### Security rules

```
match /clearthread_items/{itemId} {
  allow read, update, delete: if request.auth != null
    && request.auth.uid == resource.data.userId;
  allow create: if request.auth != null
    && request.auth.uid == request.resource.data.userId;
}

match /clearthread_threads/{threadId} {
  allow read, update, delete: if request.auth != null
    && request.auth.uid == resource.data.userId;
  allow create: if request.auth != null
    && request.auth.uid == request.resource.data.userId;
}
```

### Indexes

Firestore composite index on `userId + createdAt` created automatically on first query (Firebase console will prompt if needed).

---

## Deployment Notes

- `index.html` continues to be served from GitHub Pages — no change
- Cloud Function deployed separately via `firebase deploy --only functions`
- Firebase project: `milestone-tracker-955f4` (existing)
- Firebase Auth must be enabled in the Firebase console (Email/Password + Google providers)
- Google Auth requires an OAuth consent screen configured in Google Cloud Console

---

## Out of Scope

- User profile management (display name, avatar)
- Admin view across users
- Usage tracking / rate limiting per user
- Invite-only registration (open registration; anyone can create an account, but usage is covered by the shared API key)
