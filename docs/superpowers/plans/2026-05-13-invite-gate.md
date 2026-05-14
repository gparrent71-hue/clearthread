# ClearThread Invite Gate Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a single-use invite code gate so new accounts require a code, with an admin panel inside Settings for generating and managing codes.

**Architecture:** Three changes to `index.html` (invite code field + validation in sign-up, settings modal restructured with tabs, admin panel JS/HTML/CSS) and one change to the shared Firestore rules file. No new Cloud Function needed — code validation uses a direct Firestore read with a public-read rule on the `invite_codes` collection.

**Tech Stack:** Firebase Firestore (compat SDK 10.12.0), Firebase Auth, vanilla JS/HTML/CSS

---

### Task 1: Firestore security rules for invite_codes

**Files:**
- Modify: `/Users/gp/Documents/Projects/milestone-tracker/firestore.rules`

- [ ] **Step 1: Add invite_codes rule block**

Open `/Users/gp/Documents/Projects/milestone-tracker/firestore.rules`. Add the following block inside `match /databases/{database}/documents { }`, before the final closing `}`:

```
    // Invite gate
    match /invite_codes/{code} {
      // Public read — required for pre-auth code validation during sign-up
      allow read: if true;
      // Any authenticated user can redeem an unused code
      allow update: if request.auth != null
        && resource.data.used == false
        && request.resource.data.used == true;
      // Admin can create and delete codes
      allow create, delete: if request.auth != null
        && request.auth.token.email == 'gparrent71@gmail.com';
    }
```

- [ ] **Step 2: Deploy the rules**

```bash
cd /Users/gp/Documents/Projects/milestone-tracker
firebase deploy --only firestore:rules
```

Expected output ends with: `✔  Deploy complete!`

- [ ] **Step 3: Commit the rules change**

```bash
cd /Users/gp/Documents/Projects/milestone-tracker
git add firestore.rules
git commit -m "feat: add invite_codes Firestore security rules"
git push
```

---

### Task 2: Invite code field in Create Account form + validation

**Files:**
- Modify: `/Users/gp/Documents/clearthread-repo/index.html`
  - HTML: lines ~649–653 (Create Account panel)
  - HTML: line ~656 (remove Google create button)
  - HTML: line ~2373 (remove Google create listener)
  - JS: lines ~2310–2323 (`doCreateAccount`)

- [ ] **Step 1: Add invite code field to Create Account panel**

In `index.html`, find the Create Account panel (around line 649). Replace the block ending with the error div:

```html
      <div class="form-group">
        <label class="form-label">Confirm Password</label>
        <input type="password" class="form-control" id="auth-create-confirm" placeholder="Repeat password" autocomplete="new-password">
      </div>
      <div id="auth-create-error" class="auth-error"></div>
```

With:

```html
      <div class="form-group">
        <label class="form-label">Confirm Password</label>
        <input type="password" class="form-control" id="auth-create-confirm" placeholder="Repeat password" autocomplete="new-password">
      </div>
      <div class="form-group">
        <label class="form-label">Invite Code</label>
        <input type="text" class="form-control" id="auth-create-invite" placeholder="Enter your invite code" autocomplete="off" style="text-transform:uppercase;letter-spacing:2px">
      </div>
      <div id="auth-create-error" class="auth-error"></div>
```

- [ ] **Step 2: Remove "Continue with Google" from Create Account panel**

The Google sign-in popup creates accounts silently, bypassing the invite check. Remove it from the Create Account panel only (keep it on the Sign In panel for existing users).

In `index.html`, remove this button from the Create Account panel (around line 655–656):

```html
      <div class="auth-divider"><span>or</span></div>
      <button class="btn btn-secondary btn-full" id="btn-google-create">Continue with Google</button>
```

Also remove its event listener (around line 2373):

```javascript
document.getElementById('btn-google-create').addEventListener('click', doGoogleSignIn);
```

- [ ] **Step 3: Replace doCreateAccount() with invite-code-aware version**

In `index.html`, replace the entire `doCreateAccount` function (around lines 2310–2323):

```javascript
async function doCreateAccount() {
  const email = document.getElementById('auth-create-email').value.trim();
  const password = document.getElementById('auth-create-password').value;
  const confirm = document.getElementById('auth-create-confirm').value;
  const inviteCode = document.getElementById('auth-create-invite').value.trim().toUpperCase();
  const errEl = document.getElementById('auth-create-error');
  errEl.textContent = '';
  if (password !== confirm) { errEl.textContent = 'Passwords do not match.'; return; }
  if (password.length < 6) { errEl.textContent = 'Password must be at least 6 characters.'; return; }
  if (!inviteCode) { errEl.textContent = 'An invite code is required to create an account.'; return; }
  let codeDoc;
  try {
    codeDoc = await db.collection('invite_codes').doc(inviteCode).get();
  } catch (e) {
    errEl.textContent = 'Could not validate invite code. Please try again.';
    return;
  }
  if (!codeDoc.exists || codeDoc.data().used) {
    errEl.textContent = "That invite code isn't valid or has already been used.";
    return;
  }
  let userCredential;
  try {
    userCredential = await auth.createUserWithEmailAndPassword(email, password);
  } catch (e) {
    errEl.textContent = friendlyAuthError(e.code);
    return;
  }
  try {
    await db.collection('invite_codes').doc(inviteCode).update({
      used: true,
      usedBy: userCredential.user.uid,
      usedByEmail: email,
      usedAt: firebase.firestore.FieldValue.serverTimestamp()
    });
  } catch (e) {
    console.warn('Failed to mark invite code as used:', e);
  }
}
```

- [ ] **Step 4: Manual test — no code entered**

Open `index.html` in Chrome. Click "Create account". Fill in email/password/confirm, leave Invite Code blank. Click Create Account.
Expected: error "An invite code is required to create an account."

- [ ] **Step 5: Manual test — invalid code**

Enter `BADCODE1` as the invite code. Click Create Account.
Expected: error "That invite code isn't valid or has already been used."

- [ ] **Step 6: Commit**

```bash
cd /Users/gp/Documents/clearthread-repo
git add index.html
git commit -m "feat: require invite code on account creation"
```

---

### Task 3: Admin panel (settings tabs + generate/list/delete)

**Files:**
- Modify: `/Users/gp/Documents/clearthread-repo/index.html`
  - State object: line ~1098 (add `adminUnsub`)
  - CSS: after line ~338 (add `.code-table` rules)
  - HTML: lines ~827–850 (restructure settings modal)
  - JS: lines ~2052–2059 (replace `openSettings`/`closeSettings`)
  - JS: after `closeSettings` (add admin JS functions)
  - JS: around line ~2227 (add tab + generate button listeners)

- [ ] **Step 1: Add adminUnsub to state object**

In `index.html`, find `state` (line ~1082). Add `adminUnsub: null,` after `firestoreUnsub: null,`:

```javascript
  firestoreUnsub: null,
  adminUnsub: null,
```

- [ ] **Step 2: Add code-table CSS**

In `index.html`, find the `.tab-btn.active` rule (line ~338). Add the following immediately after it:

```css
    .code-table { width:100%; border-collapse:collapse; font-size:13px; margin-top:12px; }
    .code-table th { text-align:left; padding:6px 8px; color:var(--text-muted); font-weight:500; border-bottom:1px solid var(--border); }
    .code-table td { padding:8px; border-bottom:1px solid var(--border); vertical-align:middle; }
    .code-table tr:last-child td { border-bottom:none; }
    .code-status-used { color:var(--text-muted); font-size:12px; }
```

- [ ] **Step 3: Restructure settings modal HTML**

In `index.html`, replace the entire settings modal (lines ~827–850):

```html
<div class="overlay" id="settings-overlay">
  <div class="modal" style="width:460px">
    <div class="modal-header">
      <span class="modal-title">Settings</span>
      <button class="modal-close" id="settings-close">✕</button>
    </div>
    <div class="modal-body">
      <div class="tab-bar">
        <button class="tab-btn active" id="settings-tab-settings">Settings</button>
        <button class="tab-btn" id="settings-tab-admin" style="display:none">Admin</button>
      </div>
      <div id="settings-pane-settings">
        <div class="form-group">
          <label class="form-label">Model</label>
          <select class="form-control" id="model-select">
            <option value="claude-sonnet-4-6">Claude Sonnet 4.6 (recommended)</option>
            <option value="claude-haiku-4-5-20251001">Claude Haiku 4.5 (fastest)</option>
          </select>
          <div style="margin-top:6px;font-size:12px;color:var(--text-muted)">
            Sonnet gives better extraction quality. Haiku is faster for large files.
          </div>
        </div>
      </div>
      <div id="settings-pane-admin" style="display:none">
        <div style="margin-bottom:16px">
          <button class="btn btn-primary" id="btn-generate-code">Generate Invite Code</button>
          <div style="margin-top:8px;font-size:12px;color:var(--text-muted)">
            Generates a single-use code and copies it to your clipboard.
          </div>
        </div>
        <div id="admin-code-list"></div>
      </div>
    </div>
    <div class="modal-footer">
      <button class="btn btn-secondary" id="settings-cancel">Cancel</button>
      <button class="btn btn-primary" id="settings-save">Save Settings</button>
    </div>
  </div>
</div>
```

- [ ] **Step 4: Replace openSettings() and closeSettings() with tab-aware versions**

In `index.html`, replace `openSettings()` and `closeSettings()` (lines ~2052–2059):

```javascript
const ADMIN_EMAIL = 'gparrent71@gmail.com';

function switchSettingsTab(tab) {
  document.getElementById('settings-pane-settings').style.display = tab === 'settings' ? '' : 'none';
  document.getElementById('settings-pane-admin').style.display = tab === 'admin' ? '' : 'none';
  document.getElementById('settings-tab-settings').classList.toggle('active', tab === 'settings');
  document.getElementById('settings-tab-admin').classList.toggle('active', tab === 'admin');
  document.getElementById('settings-save').style.display = tab === 'settings' ? '' : 'none';
  document.getElementById('settings-cancel').textContent = tab === 'admin' ? 'Close' : 'Cancel';
  if (tab === 'admin') loadAdminCodes();
}

function openSettings() {
  document.getElementById('model-select').value = state.model;
  const isAdmin = auth.currentUser && auth.currentUser.email === ADMIN_EMAIL;
  document.getElementById('settings-tab-admin').style.display = isAdmin ? '' : 'none';
  switchSettingsTab('settings');
  document.getElementById('settings-overlay').classList.add('visible');
}

function closeSettings() {
  document.getElementById('settings-overlay').classList.remove('visible');
  if (state.adminUnsub) { state.adminUnsub(); state.adminUnsub = null; }
}
```

- [ ] **Step 5: Add admin panel JS functions after closeSettings()**

In `index.html`, add the following block immediately after `closeSettings()`:

```javascript
function generateCodeStr() {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
  return Array.from({length: 8}, () => chars[Math.floor(Math.random() * chars.length)]).join('');
}

async function generateInviteCode() {
  const btn = document.getElementById('btn-generate-code');
  btn.disabled = true;
  try {
    const code = generateCodeStr();
    await db.collection('invite_codes').doc(code).set({
      createdAt: firebase.firestore.FieldValue.serverTimestamp(),
      used: false,
      usedBy: null,
      usedByEmail: null,
      usedAt: null
    });
    await navigator.clipboard.writeText(code);
    showToast('✓ Code copied: ' + code, 'success');
  } catch (e) {
    showToast('Failed to generate code', 'error');
  } finally {
    btn.disabled = false;
  }
}

function loadAdminCodes() {
  if (state.adminUnsub) { state.adminUnsub(); state.adminUnsub = null; }
  const listEl = document.getElementById('admin-code-list');
  listEl.innerHTML = '<div style="font-size:13px;color:var(--text-muted)">Loading…</div>';
  state.adminUnsub = db.collection('invite_codes').orderBy('createdAt', 'desc').onSnapshot(snap => {
    if (snap.empty) {
      listEl.innerHTML = '<div style="font-size:13px;color:var(--text-muted)">No invite codes yet. Generate one above.</div>';
      return;
    }
    const rows = snap.docs.map(doc => {
      const d = doc.data();
      const code = doc.id;
      const created = d.createdAt ? formatDate(d.createdAt.toDate().toISOString().slice(0, 10)) : '—';
      const statusHtml = d.used
        ? `<span class="code-status-used">Used by ${d.usedByEmail || d.usedBy} on ${d.usedAt ? formatDate(d.usedAt.toDate().toISOString().slice(0, 10)) : '—'}</span>`
        : `<span style="color:var(--text-muted)">Unused</span>&nbsp;&nbsp;<button class="btn btn-secondary" style="padding:2px 8px;font-size:12px" onclick="deleteInviteCode('${code}')">Delete</button>`;
      return `<tr><td><code>${code}</code></td><td>${created}</td><td>${statusHtml}</td></tr>`;
    }).join('');
    listEl.innerHTML = `<table class="code-table"><thead><tr><th>Code</th><th>Created</th><th>Status</th></tr></thead><tbody>${rows}</tbody></table>`;
  }, () => {
    listEl.innerHTML = '<div style="font-size:13px;color:#dc2626">Failed to load codes.</div>';
  });
}

async function deleteInviteCode(code) {
  if (!confirm('Delete invite code ' + code + '?')) return;
  try {
    await db.collection('invite_codes').doc(code).delete();
    showToast('✓ Code deleted', 'success');
  } catch (e) {
    showToast('Failed to delete code', 'error');
  }
}
```

- [ ] **Step 6: Add tab and generate button event listeners**

In `index.html`, find the settings event listeners block (around line 2227):

```javascript
document.getElementById('btn-settings').addEventListener('click', openSettings);
document.getElementById('settings-close').addEventListener('click', closeSettings);
document.getElementById('settings-cancel').addEventListener('click', closeSettings);
```

Add three new listeners immediately after:

```javascript
document.getElementById('settings-tab-settings').addEventListener('click', () => switchSettingsTab('settings'));
document.getElementById('settings-tab-admin').addEventListener('click', () => switchSettingsTab('admin'));
document.getElementById('btn-generate-code').addEventListener('click', generateInviteCode);
```

- [ ] **Step 7: Manual test — Settings tab still works**

Reload `index.html` in Chrome. Open Settings. Confirm model dropdown still appears and Save/Cancel work as before. Confirm no "Admin" tab is visible when signed in as a non-admin user.

- [ ] **Step 8: Manual test — Admin tab (signed in as gparrent71@gmail.com)**

Sign in as gparrent71@gmail.com. Open Settings. Confirm "Admin" tab appears next to "Settings". Click Admin. Confirm "Generate Invite Code" button appears and "No invite codes yet." message.

- [ ] **Step 9: Manual test — generate a code**

Click "Generate Invite Code". Confirm:
- Toast appears: "✓ Code copied: XXXXXXXX"
- Code appears in the table below with "Unused" status and a Delete button
- Delete button removes the code after confirmation

- [ ] **Step 10: Manual test — full invite flow**

Sign out. Click "Create account". Fill email/password/confirm. Paste the generated code into the Invite Code field. Click Create Account. Confirm:
- Account created and lands in the app
- Sign back in as gparrent71@gmail.com, open Settings → Admin
- Code row now shows "Used by [new email] on [today's date]" with no Delete button

- [ ] **Step 11: Commit and push**

```bash
cd /Users/gp/Documents/clearthread-repo
git add index.html
git commit -m "feat: add invite code admin panel to settings"
git push
```

Expected: GitHub Pages live site updates within ~60 seconds. Hard refresh (`Cmd+Shift+R`) on https://gparrent71-hue.github.io/clearthread/ to verify.
