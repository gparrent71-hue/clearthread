# Tags, Dates & Sort Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add free-form tags, start dates, and due-date pickers to the import preview and edit modal, display tags as pills on item cards, and add a sort dropdown to the dashboard toolbar.

**Architecture:** All changes are to the single `index.html` file. Tags are stored as `string[]` and `startDate` as an ISO string in Firestore. Sorting is applied in `getFilteredItems()` using a new `state.sortBy` field; no persistence.

**Tech Stack:** Vanilla HTML/CSS/JS, Firebase Firestore (compat SDK v10), no build step.

---

**To run locally:** `cd /Users/gp/Documents/clearthread-repo && npx serve .` — then open the URL shown in a browser. Sign in with your existing account to see real data.

---

### Task 1: CSS + date helpers

**Files:**
- Modify: `index.html` — CSS section (~line 224), JS helpers section (~line 1111)

- [ ] **Step 1: Add `.tag-badge` CSS after `.due-badge` (around line 224)**

```css
.tag-badge {
  font-size: 11px; padding: 2px 7px; border-radius: 3px;
  background: #f1f5f9; color: #475569; font-weight: 500;
  flex-shrink: 0;
}
```

Add `.preview-edit-panel` CSS in the same CSS block (e.g. after the `.dupe-notice` block around line 450):

```css
/* ── Preview item edit panel ── */
.preview-edit-panel {
  margin-top: 10px; padding: 10px; background: #f8fafc;
  border-radius: 6px; border: 1px solid var(--border);
  grid-template-columns: 1fr 1fr 1fr; gap: 10px;
}
```

- [ ] **Step 2: Add `isIsoDate` and `formatDate` helpers**

After the `sanitizeJson` function (~line 1128), add:

```js
function isIsoDate(s) { return /^\d{4}-\d{2}-\d{2}$/.test(s || ''); }
function formatDate(iso) {
  const d = new Date(iso + 'T00:00:00');
  return d.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
}
```

- [ ] **Step 3: Verify**

Open the app in browser. No visible change expected — just confirm it loads without console errors.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add tag-badge CSS and date formatting helpers"
```

---

### Task 2: Sort state + getFilteredItems()

**Files:**
- Modify: `index.html` — state object (~line 1049), `getFilteredItems()` (~line 1217)

- [ ] **Step 1: Add `sortBy` to state**

In the `state` object (~line 1049), add `sortBy` after `selectedIds`:

```js
const state = {
  items: [],
  filterCat: 'all',
  filterStatus: 'open',
  filterOutlookCat: 'all',
  model: localStorage.getItem('clearthread_model') || 'claude-sonnet-4-6',
  previewItems: [],
  previewDupeCount: 0,
  extractedThreadName: '',
  importStep: 'input',
  importTab: 'paste',
  fileContent: null,
  worker: null,
  selectMode: false,
  selectedIds: new Set(),
  sortBy: 'date-added',
  firestoreUnsub: null,
};
```

- [ ] **Step 2: Replace `getFilteredItems()` with a version that sorts**

Replace the existing `getFilteredItems()` function (~line 1217) with:

```js
function parseSortDate(d) { return isIsoDate(d) ? d : null; }

function getFilteredItems() {
  let items = state.items.filter(item => {
    const catMatch = state.filterCat === 'all' || item.category === state.filterCat;
    const statusMatch = state.filterStatus === 'all' || item.status === state.filterStatus;
    const outlookMatch = state.filterOutlookCat === 'all' || item.outlookCategory === state.filterOutlookCat;
    return catMatch && statusMatch && outlookMatch;
  });

  const sort = state.sortBy;
  if (sort === 'due-asc' || sort === 'due-desc') {
    items = items.slice().sort((a, b) => {
      const da = parseSortDate(a.dueDate), db = parseSortDate(b.dueDate);
      if (!da && !db) return 0;
      if (!da) return 1;
      if (!db) return -1;
      return sort === 'due-asc' ? da.localeCompare(db) : db.localeCompare(da);
    });
  } else if (sort === 'start-asc' || sort === 'start-desc') {
    items = items.slice().sort((a, b) => {
      const da = parseSortDate(a.startDate), db = parseSortDate(b.startDate);
      if (!da && !db) return 0;
      if (!da) return 1;
      if (!db) return -1;
      return sort === 'start-asc' ? da.localeCompare(db) : db.localeCompare(da);
    });
  } else if (sort === 'tag-az') {
    items = items.slice().sort((a, b) => {
      const ta = (a.tags && a.tags[0]) ? a.tags[0].toLowerCase() : null;
      const tb = (b.tags && b.tags[0]) ? b.tags[0].toLowerCase() : null;
      if (!ta && !tb) return 0;
      if (!ta) return 1;
      if (!tb) return -1;
      return ta.localeCompare(tb);
    });
  }
  return items;
}
```

- [ ] **Step 3: Verify**

Open app. Existing filtering (category, status tabs) should still work identically. No visible change yet.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add sortBy state and sorting logic to getFilteredItems"
```

---

### Task 3: Sort dropdown HTML + event listener

**Files:**
- Modify: `index.html` — toolbar HTML (~line 576), event listeners section (~line 2116)

- [ ] **Step 1: Add sort dropdown to toolbar**

Find the toolbar `<div>` that wraps the bulk-bar and select buttons (~line 576):

```html
      <div style="display:flex;align-items:center;gap:8px">
        <div class="bulk-bar" id="bulk-bar">
```

Insert the sort select **before** the `<div class="bulk-bar"` line:

```html
      <div style="display:flex;align-items:center;gap:8px">
        <select id="sort-select" class="form-control" style="width:auto;font-size:13px;padding:6px 10px">
          <option value="date-added">Date Added</option>
          <option value="due-asc">Due Date ↑</option>
          <option value="due-desc">Due Date ↓</option>
          <option value="start-asc">Start Date ↑</option>
          <option value="start-desc">Start Date ↓</option>
          <option value="tag-az">Tag A→Z</option>
        </select>
        <div class="bulk-bar" id="bulk-bar">
```

- [ ] **Step 2: Wire up the event listener**

After the sidebar filter event listeners (~line 2116), add:

```js
// Sort dropdown
document.getElementById('sort-select').addEventListener('change', e => {
  state.sortBy = e.target.value;
  renderItems();
});
```

- [ ] **Step 3: Verify**

Open app with items in the dashboard. The sort dropdown should appear in the toolbar. Changing it to "Due Date ↑" should reorder items (items without ISO due dates will fall to the bottom — that's expected). "Date Added" should restore original order.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add sort dropdown to dashboard toolbar"
```

---

### Task 4: Item card — tag pills and start date

**Files:**
- Modify: `index.html` — `renderItemCard()` (~line 1302)

- [ ] **Step 1: Update `renderItemCard()` to show tags and start date**

Replace the entire `renderItemCard` function (~lines 1302–1354) with:

```js
function renderItemCard(item) {
  const isDone = item.status === 'done';
  const dueDisplay = item.dueDate
    ? (isIsoDate(item.dueDate) ? formatDate(item.dueDate) : item.dueDate)
    : null;
  const dueHtml = dueDisplay
    ? `<span class="due-badge">📅 ${escHtml(dueDisplay)}</span>`
    : '';
  const ownerHtml = item.owner
    ? `<span>👤 ${escHtml(item.owner)}</span>` : '';
  const startHtml = item.startDate
    ? `<span>🗓 ${escHtml(formatDate(item.startDate))}</span>` : '';
  const tagsHtml = (item.tags && item.tags.length)
    ? item.tags.map(t => `<span class="tag-badge">${escHtml(t)}</span>`).join('')
    : '';
  const outlookCatHtml = item.outlookCategory
    ? `<span class="outlook-cat-badge">🏷 ${escHtml(item.outlookCategory)}</span>` : '';
  const flagHtml = item.flagged
    ? `<span class="flag-badge" title="Flagged for follow-up">🚩</span>` : '';
  const threadHtml = item.threadTitle
    ? `<span>📎 ${escHtml(item.threadTitle)}</span>` : '';
  const notesHtml = item.notes
    ? `<div class="item-notes">💬 ${escHtml(item.notes)}</div>` : '';
  const completeBtn = isDone
    ? `<button class="action-btn reopen" onclick="event.stopPropagation();markStatus('${item.id}','open')">Reopen</button>`
    : `<button class="action-btn complete" onclick="event.stopPropagation();markStatus('${item.id}','done')">✓ Done</button>`;
  const isSelected = state.selectMode && state.selectedIds.has(item.id);
  const selectedClass = isSelected ? ' selected' : '';
  const cbHtml = `<input type="checkbox" class="item-select-cb" style="${state.selectMode ? '' : 'display:none'}" ${isSelected ? 'checked' : ''} onclick="event.stopPropagation();toggleSelectItem('${item.id}')">`;

  return `
    <div class="item-card status-${item.status}${selectedClass}" id="card-${item.id}" onclick="handleCardClick('${item.id}',event)" style="${state.selectMode ? 'cursor:pointer' : ''}">
      <div class="item-card-header">
        ${cbHtml}
        <span class="cat-badge ${item.category}">${CAT_LABELS[item.category]}</span>
        ${tagsHtml}
        ${flagHtml}${outlookCatHtml}
        ${dueHtml}
        <div class="item-meta-right">
          <span class="status-badge ${item.status}">${item.status}</span>
        </div>
      </div>
      <div class="item-description">${escHtml(item.description)}</div>
      ${notesHtml}
      <div class="item-footer">
        <div class="item-info">
          ${ownerHtml}${startHtml}${threadHtml}
          ${item.sourceSnippet
            ? `<button class="source-toggle" onclick="event.stopPropagation();toggleSource(this)">view source</button>`
            : ''}
        </div>
        <div class="item-actions">
          ${completeBtn}
          <button class="action-btn edit" onclick="event.stopPropagation();openEdit('${item.id}')">Edit</button>
          <button class="action-btn del" onclick="event.stopPropagation();confirmDelete('${item.id}')">Delete</button>
        </div>
      </div>
      ${item.sourceSnippet
        ? `<div class="source-snippet">"${escHtml(item.sourceSnippet)}"</div>`
        : ''}
    </div>`;
}
```

- [ ] **Step 2: Verify**

Open app. Item cards should look the same as before for items with no tags/startDate. For any item that has a `tags` array (none yet since they're new), pills would appear. ISO due dates should format as "May 12, 2026"; free-text values like "Friday" should display unchanged.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: show tag pills and start date on item cards"
```

---

### Task 5: Preview edit panel

**Files:**
- Modify: `index.html` — `renderPreview()` (~line 1793), add `togglePreviewEdit()` and `updatePreviewItem()` after `removePreviewItem()`

- [ ] **Step 1: Replace `renderPreview()` to add ✏ button and edit panel**

Replace the existing `renderPreview()` function (~lines 1793–1821):

```js
function renderPreview() {
  const list = document.getElementById('preview-list');
  let countLabel = `${state.previewItems.length} item${state.previewItems.length !== 1 ? 's' : ''} found`;
  if (state.previewDupeCount > 0) countLabel += ` · ${state.previewDupeCount} duplicate${state.previewDupeCount !== 1 ? 's' : ''} removed`;
  document.getElementById('preview-count').textContent = countLabel;

  const dupeNotice = state.previewDupeCount > 0
    ? `<div class="dupe-notice">⚠ ${state.previewDupeCount} item${state.previewDupeCount !== 1 ? 's were' : ' was'} skipped as possible duplicate${state.previewDupeCount !== 1 ? 's' : ''} of items already in your dashboard</div>`
    : '';

  if (state.previewItems.length === 0) {
    list.innerHTML = dupeNotice + '<div style="text-align:center;color:var(--text-muted);padding:20px">No items were extracted. Try importing a different thread.</div>';
    return;
  }

  list.innerHTML = dupeNotice + state.previewItems.map(item => `
    <div class="preview-item" id="preview-${item._id}">
      <div class="preview-item-body">
        <span class="cat-badge ${item.category}" style="font-size:10px">${CAT_LABELS[item.category]}</span>
        <div class="preview-item-desc">${escHtml(item.description)}</div>
        <div class="preview-item-meta">
          ${item.owner ? `<span>👤 ${escHtml(item.owner)}</span>` : ''}
          ${item.dueDate && !isIsoDate(item.dueDate) ? `<span>📅 ${escHtml(item.dueDate)}</span>` : ''}
          ${item.sourceSnippet ? `<span title="${escHtml(item.sourceSnippet)}">📌 has source</span>` : ''}
        </div>
        <div id="preview-edit-${item._id}" class="preview-edit-panel" style="display:none">
          <div>
            <label style="font-size:10px;color:var(--text-muted);display:block;margin-bottom:3px;text-transform:uppercase;font-weight:600">Tags</label>
            <input type="text" class="form-control" style="font-size:12px" placeholder="e.g. Q2, Project-X"
              value="${item.tags ? item.tags.join(', ') : ''}"
              oninput="updatePreviewItem(${item._id}, 'tags', this.value)">
          </div>
          <div>
            <label style="font-size:10px;color:var(--text-muted);display:block;margin-bottom:3px;text-transform:uppercase;font-weight:600">Start Date</label>
            <input type="date" class="form-control" style="font-size:12px"
              value="${item.startDate || ''}"
              oninput="updatePreviewItem(${item._id}, 'startDate', this.value)">
          </div>
          <div>
            <label style="font-size:10px;color:var(--text-muted);display:block;margin-bottom:3px;text-transform:uppercase;font-weight:600">Due Date${item.dueDate && !isIsoDate(item.dueDate) ? ` <span style="font-weight:normal;color:var(--text-light)">(${escHtml(item.dueDate)})</span>` : ''}</label>
            <input type="date" class="form-control" style="font-size:12px"
              value="${isIsoDate(item.dueDate || '') ? item.dueDate : ''}"
              oninput="updatePreviewItem(${item._id}, 'dueDate', this.value)">
          </div>
        </div>
      </div>
      <div style="display:flex;flex-direction:column;gap:4px;flex-shrink:0;padding-top:2px">
        <button class="preview-delete" onclick="togglePreviewEdit(${item._id})" title="Add tags & dates" style="color:var(--text-muted)">✏</button>
        <button class="preview-delete" onclick="removePreviewItem(${item._id})" title="Remove this item">✕</button>
      </div>
    </div>
  `).join('');
}
```

- [ ] **Step 2: Add `togglePreviewEdit` and `updatePreviewItem` functions**

Add these immediately after `removePreviewItem` (~line 1830):

```js
function togglePreviewEdit(id) {
  const panel = document.getElementById(`preview-edit-${id}`);
  if (!panel) return;
  panel.style.display = panel.style.display === 'grid' ? 'none' : 'grid';
}

function updatePreviewItem(id, field, value) {
  const item = state.previewItems.find(i => i._id === id);
  if (!item) return;
  if (field === 'tags') {
    item.tags = value.split(',').map(t => t.trim()).filter(Boolean);
  } else {
    item[field] = value || null;
  }
}
```

- [ ] **Step 3: Verify**

Import a thread. In the preview screen:
- Each item should show a ✏ button alongside the ✕
- Clicking ✏ should reveal the Tags / Start Date / Due Date panel below that item
- Clicking ✏ again should collapse it
- Typing in the Tags field and then saving to dashboard should persist tags (visible as pills on the card after save)
- Picking a due date should override the extracted free-text date on that item
- Removing an item (✕) should still work

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: add inline edit panel (tags, start date, due date) to preview items"
```

---

### Task 6: Edit modal — tags and start date

**Files:**
- Modify: `index.html` — edit modal HTML (~line 739), `openEdit()` (~line 1864), `saveEdit()` (~line 1883)

- [ ] **Step 1: Add Tags and Start Date fields to the edit modal HTML**

Find the edit modal Owner/Due row (~lines 762–771):

```html
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:12px">
        <div class="form-group">
          <label class="form-label">Owner</label>
          <input type="text" class="form-control" id="edit-owner" placeholder="e.g. John, Sarah">
        </div>
        <div class="form-group">
          <label class="form-label">Due Date</label>
          <input type="text" class="form-control" id="edit-due" placeholder="e.g. Friday, March 31">
        </div>
      </div>
```

Replace it with:

```html
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:12px">
        <div class="form-group">
          <label class="form-label">Owner</label>
          <input type="text" class="form-control" id="edit-owner" placeholder="e.g. John, Sarah">
        </div>
        <div class="form-group">
          <label class="form-label">Start Date</label>
          <input type="date" class="form-control" id="edit-start">
        </div>
      </div>
      <div style="display:grid;grid-template-columns:1fr 1fr;gap:12px">
        <div class="form-group">
          <label class="form-label">Tags</label>
          <input type="text" class="form-control" id="edit-tags" placeholder="e.g. Q2, Project-X">
        </div>
        <div class="form-group">
          <label class="form-label">Due Date</label>
          <input type="date" class="form-control" id="edit-due">
          <div id="edit-due-hint" style="margin-top:4px;font-size:11px;color:var(--text-muted)"></div>
        </div>
      </div>
```

- [ ] **Step 2: Update `openEdit()` to populate the new fields**

Replace `openEdit()` (~line 1864):

```js
function openEdit(id) {
  const item = state.items.find(i => i.id === id);
  if (!item) return;

  document.getElementById('edit-id').value          = id;
  document.getElementById('edit-category').value    = item.category || 'action';
  document.getElementById('edit-description').value = item.description || '';
  document.getElementById('edit-owner').value       = item.owner || '';
  document.getElementById('edit-tags').value        = (item.tags || []).join(', ');
  document.getElementById('edit-start').value       = item.startDate || '';
  document.getElementById('edit-status').value      = item.status || 'open';
  document.getElementById('edit-notes').value       = item.notes || '';

  const dueVal = item.dueDate || '';
  const dueInput = document.getElementById('edit-due');
  dueInput.value = isIsoDate(dueVal) ? dueVal : '';
  dueInput.dataset.legacyValue = !isIsoDate(dueVal) ? dueVal : '';
  document.getElementById('edit-due-hint').textContent =
    (!isIsoDate(dueVal) && dueVal) ? `Current value: "${dueVal}"` : '';

  document.getElementById('edit-overlay').classList.add('visible');
}
```

- [ ] **Step 3: Update `saveEdit()` to include new fields**

Replace the `data` object inside `saveEdit()` (~line 1885):

```js
  const dueInput = document.getElementById('edit-due');
  const data = {
    category:    document.getElementById('edit-category').value,
    description: document.getElementById('edit-description').value.trim(),
    owner:       document.getElementById('edit-owner').value.trim() || null,
    tags:        document.getElementById('edit-tags').value.split(',').map(t => t.trim()).filter(Boolean),
    startDate:   document.getElementById('edit-start').value || null,
    dueDate:     dueInput.value || dueInput.dataset.legacyValue || null,
    status:      document.getElementById('edit-status').value,
    notes:       document.getElementById('edit-notes').value.trim() || null,
  };
```

- [ ] **Step 4: Verify**

Open the Edit modal on an existing item:
- Tags field should appear (empty for existing items)
- Start Date picker should appear (empty for existing items)
- Due Date is now a date picker; items with old free-text values ("Friday") should show an empty picker with a hint "Current value: 'Friday'"
- Adding tags, setting a start date, and saving should persist to Firestore and show immediately on the card
- Items with ISO due dates (set via preview) should pre-populate the due date picker correctly

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: add tags and start date to edit modal"
```
