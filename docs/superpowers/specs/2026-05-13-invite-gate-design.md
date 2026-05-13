# ClearThread Invite Gate — Design Spec
**Date:** 2026-05-13
**Status:** Approved

## Overview

Add a single-use invite code gate to ClearThread so the app can be shared with specific people without being open to anyone with the URL. The admin (gparrent71@gmail.com) generates codes from inside the app; recipients enter a code during account creation.

## Data Model

**Firestore collection:** `invite_codes`

Document ID is the code itself (e.g., `KX7P2Q4M`) — enables O(1) lookup without a query.

| Field | Type | Default | Description |
|---|---|---|---|
| `createdAt` | Timestamp | now() | When admin generated the code |
| `used` | Boolean | false | Flips to true on redemption |
| `usedBy` | String | null | Firebase Auth userId of redeemer |
| `usedByEmail` | String | null | Email of redeemer (captured at redemption) |
| `usedAt` | Timestamp | null | When the code was redeemed |

Codes are 8-character uppercase alphanumeric (e.g., `KX7P2Q4M`), randomly generated in the browser by the admin panel.

## Sign-up Flow

The existing "Create Account" form gains one required **Invite Code** field. On submit:

1. Validate invite code field is non-empty (client-side)
2. Read `invite_codes/{code}` from Firestore
   - If document doesn't exist or `used === true`: show error *"That invite code isn't valid or has already been used."* Halt — do not create account
3. Create Firebase Auth account (existing logic unchanged)
4. Firestore transaction: set `used: true`, `usedBy: uid`, `usedByEmail: email`, `usedAt: serverTimestamp()`
5. Continue into app as normal

**Existing users:** Unaffected. The invite check runs only in the create-account path, not on sign-in.

**Enforcement note:** UI-level only. A user who creates a Firebase Auth account outside the app would only access their own isolated Firestore data — not a concern for this personal shared tool.

## Admin Panel

A new **Admin** tab inside the existing Settings modal, visible only when `firebase.auth().currentUser.email === 'gparrent71@gmail.com'`.

### Generate Code Area
- "Generate Invite Code" button
- On click: generate random 8-char uppercase alphanumeric code, write to `invite_codes/{code}`, auto-copy to clipboard, show "Copied!" confirmation toast

### Code List
- Table: **Code** | **Created** | **Status**
- Status column:
  - Unused codes: "Unused" + Delete button
  - Used codes: "Used by [usedByEmail] on [usedAt date]" — no delete button
- Delete removes the Firestore document (unused codes only)
- Empty state: "No invite codes yet. Generate one above."
- List updates in real-time via Firestore listener (admin-only onSnapshot)

## Firestore Security Rules

Added to the authoritative rules file at `/Users/gp/Documents/Projects/milestone-tracker/firestore.rules`:

```
match /invite_codes/{code} {
  // Public read — required for pre-auth code validation during sign-up
  allow read: if true;

  // Authenticated users can redeem an unused code (mark it used)
  allow update: if request.auth != null
    && resource.data.used == false
    && request.resource.data.used == true;

  // Admin can create and delete codes
  allow create, delete: if request.auth != null
    && request.auth.token.email == 'gparrent71@gmail.com';
}
```

All existing rules (clearthread collections, milestone tracker collections) remain unchanged.

## Out of Scope
- Bulk code generation
- Code expiry dates
- Granting admin access to other users
- Server-side enforcement (Cloud Function auth check)
