# ClearThread — Tags, Dates & Sort Design

**Date:** 2026-05-12
**Status:** Approved

## Overview

Add tags, start dates, and due dates to the preview step (so items can be annotated before saving to the dashboard), extend the edit modal with the same fields, display tags as pills on item cards, and add a sort dropdown to the dashboard toolbar.

## Data Model

Two new fields added to each Firestore item document in `clearthread_items`:

| Field | Type | Description |
|---|---|---|
| `tags` | `string[]` | Array of free-form labels, e.g. `["Project-X", "Q2"]`. Empty array if no tags. |
| `startDate` | `string` | ISO date string (e.g. `"2026-05-15"`), or `""` if not set. |

The existing `dueDate` field remains a string in Firestore. No migration. Old free-text due date values continue to display. New values set via the date picker will be ISO format.

## Feature 1: Preview Step — Editable Tags & Dates

### Interaction
Each preview item card gets a small **✏ Edit** button on the right, alongside the existing ✕ remove button. Clicking it reveals an edit panel inline below the item. Clicking it again collapses the panel.

### Edit Panel Fields
- **Tags** — single comma-separated text input (e.g. `Project-X, Q2`). Parsed into an array on save by splitting on commas and trimming whitespace.
- **Start Date** — `<input type="date">` date picker.
- **Due Date** — `<input type="date">` date picker. Replaces the current read-only due date display. Left empty by default — Claude's extracted due dates are free text ("Friday", "March 31") which cannot pre-populate an ISO date picker. The original extracted text continues to display as read-only metadata next to the picker so you know what Claude found.

### Save Behavior
When `doSave()` runs, `tags` and `startDate` are included with the item data sent to `saveItemsBatch()`. `dueDate` uses the date picker value if set, otherwise the originally extracted value.

## Feature 2: Edit Modal — Tags & Dates

The existing Edit modal (`edit-overlay`) gets two new fields:

- **Tags** — comma-separated text input, populated by joining the item's `tags` array with `", "`.
- **Start Date** — `<input type="date">` date picker, populated from `item.startDate`.
- **Due Date** — switches from `<input type="text">` to `<input type="date">`. Items with ISO-format due dates (set via the new picker) will pre-populate correctly. Items with old free-text due dates ("Friday", "March 31") will show an empty picker — the stored value is not lost until the user saves a new date.

On "Save Changes", `tags` is split/trimmed back into an array and `startDate` saved as-is.

## Feature 3: Dashboard Sort

### Sort Dropdown
A `<select>` dropdown added to the main toolbar, to the right of the existing controls above the items list.

| Option | Behavior |
|---|---|
| Date Added (default) | Current behavior — `createdAt` descending |
| Due Date ↑ | Earliest due date first; items with no due date sort to bottom |
| Due Date ↓ | Latest due date first; items with no due date sort to bottom |
| Start Date ↑ | Earliest start date first; items with no start date sort to bottom |
| Start Date ↓ | Latest start date first; items with no start date sort to bottom |
| Tag A→Z | Alphabetical by first tag; untagged items sort to bottom |

Sort is applied inside `getFilteredItems()` after filtering, before rendering. It is global (applies across all category/status tabs). Sort selection resets to "Date Added" on page reload.

## Feature 4: Item Card Display

Tags and start date are shown on saved item cards:

- **Tags** — each tag rendered as a small gray pill (`tag-badge`) in the item card header, after the category badge.
- **Start Date** — shown with a 🗓 icon in the item footer info row, alongside the existing 📅 due date.

## Out of Scope

- Tag autocomplete / suggestions from previously used tags
- Persistent sort preference (localStorage or Firestore)
- Filter by tag in the sidebar
