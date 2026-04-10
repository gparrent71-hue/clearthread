# ClearThread: Auth + File Upload Expansion — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Firebase Auth (email/password + Google), a shared Claude API key via Cloud Function proxy, and .docx/.pdf file upload support to ClearThread.

**Architecture:** A Firebase Cloud Function (`callClaude`) proxies all Claude API calls using a server-side API key, removing the need for users to supply their own. Firebase Auth gates the app behind a login overlay; all Firestore reads/writes are scoped to `userId`. File parsing for .docx (mammoth.js) and PDF (pdf.js) runs client-side before feeding the existing chunking pipeline.

**Tech Stack:** Firebase Auth (compat SDK v10), Firebase Functions v1 (Node 20), mammoth.js 1.6.0 (CDN), pdf.js 3.11.174 (CDN), vanilla HTML/CSS/JS

---

## File Map

| File | Action | Purpose |
|------|--------|---------|
| `firebase.json` | Create | Firebase project config (functions source) |
| `.firebaserc` | Create | Firebase project alias |
| `.gitignore` | Create | Ignore node_modules, secrets |
| `functions/package.json` | Create | Functions dependencies |
| `functions/index.js` | Create | `callClaude` HTTPS function — Claude proxy |
| `index.html` | Modify | Auth UI, Cloud Function calls, .docx/.pdf, Firestore scoping |

---

## Task 1: Firebase Project Files

**Files:**
- Create: `firebase.json`
- Create: `.firebaserc`
- Create: `.gitignore`

- [ ] **Step 1: Create firebase.json**

```json
{
  "functions": {
    "source": "functions"
  },
  "emulators": {
    "functions": {
      "port": 5001
    }
  }
}
```

Save to `/Users/gp/Documents/clearthread-repo/firebase.json`

- [ ] **Step 2: Create .firebaserc**

```json
{
  "projects": {
    "default": "milestone-tracker-955f4"
  }
}
```

Save to `/Users/gp/Documents/clearthread-repo/.firebaserc`

- [ ] **Step 3: Create .gitignore**

```
node_modules/
functions/node_modules/
.env
*.local
```

Save to `/Users/gp/Documents/clearthread-repo/.gitignore`

- [ ] **Step 4: Enable Firebase Auth providers (manual — browser)**

1. Go to https://console.firebase.google.com/project/milestone-tracker-955f4/authentication/providers
2. Enable **Email/Password** provider
3. Enable **Google** provider — it will ask for a support email, use your email address
4. For Google sign-in to work from GitHub Pages, go to **Authentication → Settings → Authorized domains** and confirm `gparrent71-hue.github.io` is listed (Firebase usually adds it automatically; if not, add it manually)

- [ ] **Step 5: Commit**

```bash
cd /Users/gp/Documents/clearthread-repo
git add firebase.json .firebaserc .gitignore
git commit -m "chore: Firebase project config files"
```

---

## Task 2: Cloud Function — callClaude

**Files:**
- Create: `functions/package.json`
- Create: `functions/index.js`

- [ ] **Step 1: Create functions/package.json**

```json
{
  "name": "clearthread-functions",
  "version": "1.0.0",
  "engines": { "node": "20" },
  "main": "index.js",
  "dependencies": {
    "cors": "^2.8.5",
    "firebase-admin": "^12.0.0",
    "firebase-functions": "^4.9.0"
  }
}
```

Save to `/Users/gp/Documents/clearthread-repo/functions/package.json`

- [ ] **Step 2: Create functions/index.js**

```javascript
const functions = require('firebase-functions');
const admin = require('firebase-admin');
const cors = require('cors')({ origin: true });

admin.initializeApp();

exports.callClaude = functions
  .runWith({ secrets: ['ANTHROPIC_API_KEY'] })
  .https.onRequest((req, res) => {
    cors(req, res, async () => {
      if (req.method === 'OPTIONS') return res.status(204).send('');

      // Verify Firebase Auth token
      const authHeader = req.headers.authorization;
      if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'Unauthorized' });
      }
      const idToken = authHeader.split('Bearer ')[1];
      try {
        await admin.auth().verifyIdToken(idToken);
      } catch {
        return res.status(401).json({ error: 'Invalid token' });
      }

      const { messages, system, model, max_tokens = 8192 } = req.body;

      try {
        const response = await fetch('https://api.anthropic.com/v1/messages', {
          method: 'POST',
          headers: {
            'x-api-key': process.env.ANTHROPIC_API_KEY,
            'anthropic-version': '2023-06-01',
            'content-type': 'application/json',
          },
          body: JSON.stringify({ model, max_tokens, system, messages }),
        });
        const data = await response.json();
        if (!response.ok) return res.status(response.status).json(data);
        return res.json(data);
      } catch (e) {
        return res.status(500).json({ error: e.message });
      }
    });
  });
```

Save to `/Users/gp/Documents/clearthread-repo/functions/index.js`

- [ ] **Step 3: Install dependencies**

```bash
cd /Users/gp/Documents/clearthread-repo/functions
npm install
```

Expected: `node_modules/` created, no errors.

- [ ] **Step 4: Commit**

```bash
cd /Users/gp/Documents/clearthread-repo
git add functions/
git commit -m "feat: callClaude Cloud Function — Claude API proxy with auth verification"
```

---

## Task 3: Deploy Cloud Function + Store API Secret

**Files:** none (deploy + secret setup)

- [ ] **Step 1: Verify Firebase CLI is logged in**

```bash
firebase login --no-localhost
```

If already logged in you'll see your account. If not, follow the login flow.

- [ ] **Step 2: Store the Anthropic API key as a secret**

```bash
cd /Users/gp/Documents/clearthread-repo
firebase functions:secrets:set ANTHROPIC_API_KEY
```

When prompted, paste your Anthropic API key (`sk-ant-…`) and press Enter.

- [ ] **Step 3: Deploy the function**

```bash
firebase deploy --only functions
```

Expected output contains:
```
✔  functions[callClaude(us-central1)]: Successful create operation.
Function URL (callClaude): https://us-central1-milestone-tracker-955f4.cloudfunctions.net/callClaude
```

- [ ] **Step 4: Verify the function rejects unauthenticated requests**

```bash
curl -X POST https://us-central1-milestone-tracker-955f4.cloudfunctions.net/callClaude \
  -H "Content-Type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","messages":[{"role":"user","content":"hi"}]}'
```

Expected: `{"error":"Unauthorized"}` with HTTP 401.

---

## Task 4: Firebase Auth SDK + State Wiring

**Files:**
- Modify: `index.html:8-10` (add Auth SDK)
- Modify: `index.html:931-932` (add auth init)
- Modify: `index.html:937-953` (update state)
- Modify: `index.html:1988-1994` (update init block)

- [ ] **Step 1: Add Firebase Auth SDK script tag**

In `index.html`, after line 10 (after the Firestore script tag), add:

```html
  <script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-auth-compat.js"></script>
```

- [ ] **Step 2: Add auth init and CLOUD_FUNCTION_URL constant**

In `index.html`, replace lines 931-932:
```javascript
firebase.initializeApp(firebaseConfig);
const db = firebase.firestore();
```

With:
```javascript
firebase.initializeApp(firebaseConfig);
const db = firebase.firestore();
const auth = firebase.auth();

const CLOUD_FUNCTION_URL = 'https://us-central1-milestone-tracker-955f4.cloudfunctions.net/callClaude';
```

- [ ] **Step 3: Remove apiKey from state, add currentUser**

In `index.html`, replace the state object (lines ~937-953):
```javascript
const state = {
  items: [],
  filterCat: 'all',
  filterStatus: 'open',
  filterOutlookCat: 'all',
  apiKey: localStorage.getItem('clearthread_api_key') || '',
  model: localStorage.getItem('clearthread_model') || 'claude-opus-4-6',
  previewItems: [],
  previewDupeCount: 0,
  extractedThreadName: '',
  importStep: 'input',
  importTab: 'paste',
  fileContent: null,
  worker: null,
  selectMode: false,
  selectedIds: new Set(),
};
```

With:
```javascript
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
  firestoreUnsub: null,
};
```

- [ ] **Step 4: Replace init block with auth state listener**

In `index.html`, replace the init block at the bottom of the `<script>` (lines ~1986-1994):
```javascript
// ─────────────────────────────────────────────
// Init
// ─────────────────────────────────────────────
listenItems();

// Prompt settings if no API key
if (!state.apiKey) {
  setTimeout(() => showToast('Add your Claude API key in ⚙ Settings to get started'), 1500);
}
```

With:
```javascript
// ─────────────────────────────────────────────
// Init — Auth State Listener
// ─────────────────────────────────────────────
auth.onAuthStateChanged(user => {
  if (user) {
    // Signed in — show app, update header, start Firestore listener
    document.getElementById('auth-overlay').style.display = 'none';
    document.getElementById('user-email-display').textContent = user.email || user.displayName || 'Signed in';
    if (state.firestoreUnsub) state.firestoreUnsub();
    listenItems();
  } else {
    // Signed out — stop Firestore, clear items, show auth overlay
    if (state.firestoreUnsub) { state.firestoreUnsub(); state.firestoreUnsub = null; }
    state.items = [];
    renderAll();
    document.getElementById('auth-overlay').style.display = 'flex';
  }
});
```

- [ ] **Step 5: Update listenItems() to store unsubscribe and scope by userId**

Replace `listenItems()` (lines ~958-968):
```javascript
function listenItems() {
  db.collection('clearthread_items')
    .orderBy('createdAt', 'desc')
    .onSnapshot(snap => {
      state.items = snap.docs.map(d => ({ id: d.id, ...d.data() }));
      renderAll();
    }, err => {
      console.error('Firestore error:', err);
      showToast('Database connection error', 'error');
    });
}
```

With:
```javascript
function listenItems() {
  const uid = auth.currentUser.uid;
  state.firestoreUnsub = db.collection('clearthread_items')
    .where('userId', '==', uid)
    .orderBy('createdAt', 'desc')
    .onSnapshot(snap => {
      state.items = snap.docs.map(d => ({ id: d.id, ...d.data() }));
      renderAll();
    }, err => {
      console.error('Firestore error:', err);
      showToast('Database connection error', 'error');
    });
}
```

- [ ] **Step 6: Update saveItemsBatch() to include userId**

Replace `saveItemsBatch()` (lines ~970-987):
```javascript
async function saveItemsBatch(items, threadId, threadTitle) {
  const batch = db.batch();
  // Save thread record
  const threadRef = db.collection('clearthread_threads').doc(threadId);
  batch.set(threadRef, { title: threadTitle, importedAt: firebase.firestore.FieldValue.serverTimestamp() });
  // Save items
  items.forEach(item => {
    const ref = db.collection('clearthread_items').doc();
    batch.set(ref, {
      ...item,
      threadId,
      threadTitle,
      status: 'open',
      createdAt: firebase.firestore.FieldValue.serverTimestamp(),
    });
  });
  await batch.commit();
}
```

With:
```javascript
async function saveItemsBatch(items, threadId, threadTitle) {
  const uid = auth.currentUser.uid;
  const batch = db.batch();
  const threadRef = db.collection('clearthread_threads').doc(threadId);
  batch.set(threadRef, { title: threadTitle, userId: uid, importedAt: firebase.firestore.FieldValue.serverTimestamp() });
  items.forEach(item => {
    const ref = db.collection('clearthread_items').doc();
    batch.set(ref, {
      ...item,
      threadId,
      threadTitle,
      userId: uid,
      status: 'open',
      createdAt: firebase.firestore.FieldValue.serverTimestamp(),
    });
  });
  await batch.commit();
}
```

- [ ] **Step 7: Verify the page still loads without JS errors**

Open `index.html` in Chrome (via `cd /Users/gp/Documents/clearthread-repo && npx serve .` → `http://localhost:PORT/index.html`). The auth overlay won't exist yet so the page will look the same, but check the browser console — there should be no errors about undefined `auth` or missing `currentUser`.

(Note: `currentUser` will be null until the auth overlay is added and a user signs in — that's expected.)

- [ ] **Step 8: Commit**

```bash
cd /Users/gp/Documents/clearthread-repo
git add index.html
git commit -m "feat: Firebase Auth SDK wiring — auth state listener, userId-scoped Firestore reads and writes"
```

---

## Task 5: Login Overlay HTML + CSS

**Files:**
- Modify: `index.html` (add auth overlay HTML before import modal, add CSS in `<style>`)

- [ ] **Step 1: Add auth overlay CSS**

In `index.html`, inside the `<style>` block, add before the closing `</style>` tag:

```css
/* ── Auth Overlay ── */
.auth-overlay {
  position: fixed; inset: 0; background: rgba(15,23,42,0.7);
  display: flex; align-items: center; justify-content: center;
  z-index: 1000;
}
.auth-card {
  background: var(--surface); border-radius: 12px;
  padding: 36px 32px; width: 380px;
  box-shadow: 0 20px 60px rgba(0,0,0,0.25);
}
.auth-logo {
  font-size: 22px; font-weight: 700; color: var(--text);
  letter-spacing: -0.3px; text-align: center; margin-bottom: 28px;
}
.auth-logo span { color: var(--action); }
.auth-title {
  font-size: 18px; font-weight: 600; color: var(--text);
  margin-bottom: 20px;
}
.auth-subtitle {
  font-size: 13px; color: var(--text-muted); margin-bottom: 16px;
}
.auth-error {
  font-size: 12px; color: #dc2626; margin-bottom: 10px; min-height: 16px;
}
.auth-success {
  font-size: 13px; color: #059669; margin-bottom: 12px;
  padding: 10px 12px; background: #ecfdf5; border-radius: var(--radius);
}
.btn-full { width: 100%; justify-content: center; margin-bottom: 10px; }
.auth-divider {
  display: flex; align-items: center; gap: 12px;
  margin: 14px 0; color: var(--text-muted); font-size: 12px;
}
.auth-divider::before, .auth-divider::after {
  content: ''; flex: 1; height: 1px; background: var(--border);
}
.auth-links {
  display: flex; justify-content: space-between; margin-top: 16px;
}
.auth-links a {
  font-size: 12px; color: var(--action); text-decoration: none;
}
.auth-links a:hover { text-decoration: underline; }
.auth-panel { display: block; }
```

- [ ] **Step 2: Add auth overlay HTML**

In `index.html`, immediately before the line `<!-- ═══ IMPORT MODAL ═══ -->`, insert:

```html
<!-- ═══ AUTH OVERLAY ═══ -->
<div id="auth-overlay" class="auth-overlay" style="display:none">
  <div class="auth-card">
    <div class="auth-logo">Clear<span>Thread</span></div>

    <!-- Sign In panel -->
    <div id="auth-panel-signin" class="auth-panel">
      <h2 class="auth-title">Sign in</h2>
      <div class="form-group">
        <label class="form-label">Email</label>
        <input type="email" class="form-control" id="auth-email" placeholder="you@example.com" autocomplete="email">
      </div>
      <div class="form-group">
        <label class="form-label">Password</label>
        <input type="password" class="form-control" id="auth-password" placeholder="Password" autocomplete="current-password">
      </div>
      <div id="auth-signin-error" class="auth-error"></div>
      <button class="btn btn-primary btn-full" id="btn-signin">Sign In</button>
      <div class="auth-divider"><span>or</span></div>
      <button class="btn btn-secondary btn-full" id="btn-google-signin">Continue with Google</button>
      <div class="auth-links">
        <a href="#" id="link-to-create">Create account</a>
        <a href="#" id="link-to-forgot">Forgot password?</a>
      </div>
    </div>

    <!-- Create Account panel -->
    <div id="auth-panel-create" class="auth-panel" style="display:none">
      <h2 class="auth-title">Create account</h2>
      <div class="form-group">
        <label class="form-label">Email</label>
        <input type="email" class="form-control" id="auth-create-email" placeholder="you@example.com" autocomplete="email">
      </div>
      <div class="form-group">
        <label class="form-label">Password</label>
        <input type="password" class="form-control" id="auth-create-password" placeholder="At least 6 characters" autocomplete="new-password">
      </div>
      <div class="form-group">
        <label class="form-label">Confirm Password</label>
        <input type="password" class="form-control" id="auth-create-confirm" placeholder="Repeat password" autocomplete="new-password">
      </div>
      <div id="auth-create-error" class="auth-error"></div>
      <button class="btn btn-primary btn-full" id="btn-create-account">Create Account</button>
      <div class="auth-divider"><span>or</span></div>
      <button class="btn btn-secondary btn-full" id="btn-google-create">Continue with Google</button>
      <div class="auth-links">
        <a href="#" id="link-to-signin-from-create">Back to sign in</a>
      </div>
    </div>

    <!-- Forgot Password panel -->
    <div id="auth-panel-forgot" class="auth-panel" style="display:none">
      <h2 class="auth-title">Reset password</h2>
      <p class="auth-subtitle">Enter your email and we'll send you a reset link.</p>
      <div class="form-group">
        <label class="form-label">Email</label>
        <input type="email" class="form-control" id="auth-forgot-email" placeholder="you@example.com" autocomplete="email">
      </div>
      <div id="auth-forgot-error" class="auth-error"></div>
      <div id="auth-forgot-success" class="auth-success" style="display:none">
        Check your email for a reset link.
      </div>
      <button class="btn btn-primary btn-full" id="btn-send-reset">Send Reset Email</button>
      <div class="auth-links">
        <a href="#" id="link-to-signin-from-forgot">Back to sign in</a>
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 3: Add user indicator to header**

In `index.html`, replace the header section (lines ~458-465):
```html
<!-- ═══ HEADER ═══ -->
<header>
  <div class="logo">Clear<span>Thread</span></div>
  <div class="header-actions">
    <button class="btn btn-primary" id="btn-import">＋ Import Thread</button>
    <button class="btn-icon" id="btn-settings" title="Settings">⚙</button>
  </div>
</header>
```

With:
```html
<!-- ═══ HEADER ═══ -->
<header>
  <div class="logo">Clear<span>Thread</span></div>
  <div class="header-actions">
    <span id="user-email-display" style="font-size:12px;color:var(--text-muted);max-width:200px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap"></span>
    <button class="btn btn-secondary" id="btn-signout" style="font-size:12px;padding:5px 10px">Sign Out</button>
    <button class="btn btn-primary" id="btn-import">＋ Import Thread</button>
    <button class="btn-icon" id="btn-settings" title="Settings">⚙</button>
  </div>
</header>
```

- [ ] **Step 4: Verify in browser**

Reload the local server. The page should load with the auth overlay visible (since no user is signed in). The overlay should be centered and styled. Console should have no errors.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: auth overlay HTML + CSS — sign in, create account, forgot password panels"
```

---

## Task 6: Auth Flow JavaScript

**Files:**
- Modify: `index.html` (add auth functions + event listeners to the JS section)

- [ ] **Step 1: Add auth panel switcher and auth flow functions**

In `index.html`, in the `<!-- ═══ JAVASCRIPT ═══ -->` section, add immediately after the `// Init — Auth State Listener` block at the bottom of the script (before `</script>`):

```javascript
// ─────────────────────────────────────────────
// Auth UI
// ─────────────────────────────────────────────
function showAuthPanel(name) {
  ['signin','create','forgot'].forEach(p => {
    document.getElementById(`auth-panel-${p}`).style.display = p === name ? 'block' : 'none';
  });
  document.getElementById('auth-signin-error').textContent = '';
  document.getElementById('auth-create-error').textContent = '';
  document.getElementById('auth-forgot-error').textContent = '';
  document.getElementById('auth-forgot-success').style.display = 'none';
}

async function doSignIn() {
  const email = document.getElementById('auth-email').value.trim();
  const password = document.getElementById('auth-password').value;
  const errEl = document.getElementById('auth-signin-error');
  errEl.textContent = '';
  try {
    await auth.signInWithEmailAndPassword(email, password);
  } catch (e) {
    errEl.textContent = friendlyAuthError(e.code);
  }
}

async function doCreateAccount() {
  const email = document.getElementById('auth-create-email').value.trim();
  const password = document.getElementById('auth-create-password').value;
  const confirm = document.getElementById('auth-create-confirm').value;
  const errEl = document.getElementById('auth-create-error');
  errEl.textContent = '';
  if (password !== confirm) { errEl.textContent = 'Passwords do not match.'; return; }
  if (password.length < 6) { errEl.textContent = 'Password must be at least 6 characters.'; return; }
  try {
    await auth.createUserWithEmailAndPassword(email, password);
  } catch (e) {
    errEl.textContent = friendlyAuthError(e.code);
  }
}

async function doSendReset() {
  const email = document.getElementById('auth-forgot-email').value.trim();
  const errEl = document.getElementById('auth-forgot-error');
  const successEl = document.getElementById('auth-forgot-success');
  errEl.textContent = '';
  successEl.style.display = 'none';
  if (!email) { errEl.textContent = 'Please enter your email address.'; return; }
  try {
    await auth.sendPasswordResetEmail(email);
    successEl.style.display = 'block';
    document.getElementById('btn-send-reset').disabled = true;
  } catch (e) {
    errEl.textContent = friendlyAuthError(e.code);
  }
}

async function doGoogleSignIn() {
  const provider = new firebase.auth.GoogleAuthProvider();
  try {
    await auth.signInWithPopup(provider);
  } catch (e) {
    document.getElementById('auth-signin-error').textContent = friendlyAuthError(e.code);
  }
}

function doSignOut() {
  auth.signOut();
}

function friendlyAuthError(code) {
  const msgs = {
    'auth/invalid-email': 'Please enter a valid email address.',
    'auth/user-not-found': 'No account found with that email.',
    'auth/wrong-password': 'Incorrect password.',
    'auth/email-already-in-use': 'An account with this email already exists.',
    'auth/weak-password': 'Password must be at least 6 characters.',
    'auth/too-many-requests': 'Too many attempts. Please try again later.',
    'auth/invalid-credential': 'Incorrect email or password.',
    'auth/popup-closed-by-user': '',
  };
  return msgs[code] || 'Something went wrong. Please try again.';
}

// Auth event listeners
document.getElementById('btn-signin').addEventListener('click', doSignIn);
document.getElementById('btn-create-account').addEventListener('click', doCreateAccount);
document.getElementById('btn-send-reset').addEventListener('click', doSendReset);
document.getElementById('btn-google-signin').addEventListener('click', doGoogleSignIn);
document.getElementById('btn-google-create').addEventListener('click', doGoogleSignIn);
document.getElementById('btn-signout').addEventListener('click', doSignOut);
document.getElementById('link-to-create').addEventListener('click', e => { e.preventDefault(); showAuthPanel('create'); });
document.getElementById('link-to-forgot').addEventListener('click', e => { e.preventDefault(); showAuthPanel('forgot'); });
document.getElementById('link-to-signin-from-create').addEventListener('click', e => { e.preventDefault(); showAuthPanel('signin'); });
document.getElementById('link-to-signin-from-forgot').addEventListener('click', e => { e.preventDefault(); showAuthPanel('signin'); });

// Submit on Enter key in auth inputs
document.getElementById('auth-password').addEventListener('keydown', e => { if (e.key === 'Enter') doSignIn(); });
document.getElementById('auth-create-confirm').addEventListener('keydown', e => { if (e.key === 'Enter') doCreateAccount(); });
document.getElementById('auth-forgot-email').addEventListener('keydown', e => { if (e.key === 'Enter') doSendReset(); });
```

- [ ] **Step 2: Verify auth flow in browser**

1. Reload the local server page
2. Auth overlay should be visible
3. Click "Create account" — panel should switch to create account form
4. Click "Back to sign in" — panel should return to sign in form
5. Click "Forgot password?" — panel should switch to forgot password form
6. Click "Back to sign in" — panel should return to sign in form
7. Try creating an account with mismatched passwords — error "Passwords do not match." should appear
8. Create a real account with your email + a test password — overlay should disappear and the dashboard should appear
9. Your email should appear in the header; "Sign Out" button should be visible
10. Click Sign Out — overlay should reappear

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat: auth flows — sign in, create account, forgot password, Google sign-in, sign out"
```

---

## Task 7: Settings Modal Cleanup

**Files:**
- Modify: `index.html:672-708` (settings modal HTML)
- Modify: `index.html:1806-1828` (settings JS functions)
- Modify: `index.html:1947-1957` (settings event listeners)

- [ ] **Step 1: Remove API key field from Settings HTML**

Replace the settings modal HTML (lines ~672-708):
```html
<!-- ═══ SETTINGS MODAL ═══ -->
<div class="overlay" id="settings-overlay">
  <div class="modal" style="width:460px">
    <div class="modal-header">
      <span class="modal-title">Settings</span>
      <button class="modal-close" id="settings-close">✕</button>
    </div>
    <div class="modal-body">
      <div class="form-group">
        <label class="form-label">Claude API Key</label>
        <div class="key-input-wrap">
          <input type="password" class="form-control" id="api-key-input"
            placeholder="sk-ant-…" style="padding-right:40px">
          <button class="key-toggle" id="key-toggle">👁</button>
        </div>
        <div style="margin-top:6px;font-size:12px;color:var(--text-muted)">
          Your key is stored only in this browser's localStorage and never sent anywhere except Anthropic's API.
          Get a key at <strong>console.anthropic.com</strong> → API Keys.
        </div>
      </div>
      <div class="form-group">
        <label class="form-label">Model</label>
        <select class="form-control" id="model-select">
          <option value="claude-opus-4-6">Claude Opus 4.6 (most capable)</option>
          <option value="claude-haiku-4-5">Claude Haiku 4.5 (fastest, lowest cost)</option>
        </select>
        <div style="margin-top:6px;font-size:12px;color:var(--text-muted)">
          Opus 4.6: ~$0.01–0.05 per thread. Haiku 4.5: ~$0.001–0.005 per thread.
        </div>
      </div>
    </div>
    <div class="modal-footer">
      <button class="btn btn-secondary" id="settings-cancel">Cancel</button>
      <button class="btn btn-primary" id="settings-save">Save Settings</button>
    </div>
  </div>
</div>
```

With:
```html
<!-- ═══ SETTINGS MODAL ═══ -->
<div class="overlay" id="settings-overlay">
  <div class="modal" style="width:460px">
    <div class="modal-header">
      <span class="modal-title">Settings</span>
      <button class="modal-close" id="settings-close">✕</button>
    </div>
    <div class="modal-body">
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
    <div class="modal-footer">
      <button class="btn btn-secondary" id="settings-cancel">Cancel</button>
      <button class="btn btn-primary" id="settings-save">Save Settings</button>
    </div>
  </div>
</div>
```

- [ ] **Step 2: Update Settings JS functions**

Replace `openSettings()`, `closeSettings()`, `saveSettings()` (lines ~1806-1828):
```javascript
// ─────────────────────────────────────────────
// Settings Modal
// ─────────────────────────────────────────────
function openSettings() {
  document.getElementById('api-key-input').value = state.apiKey;
  document.getElementById('model-select').value  = state.model;
  document.getElementById('settings-overlay').classList.add('visible');
}

function closeSettings() {
  document.getElementById('settings-overlay').classList.remove('visible');
}

function saveSettings() {
  const key   = document.getElementById('api-key-input').value.trim();
  const model = document.getElementById('model-select').value;
  state.apiKey = key;
  state.model  = model;
  localStorage.setItem('clearthread_api_key', key);
  localStorage.setItem('clearthread_model', model);
  closeSettings();
  showToast('✓ Settings saved', 'success');
}
```

With:
```javascript
// ─────────────────────────────────────────────
// Settings Modal
// ─────────────────────────────────────────────
function openSettings() {
  document.getElementById('model-select').value = state.model;
  document.getElementById('settings-overlay').classList.add('visible');
}

function closeSettings() {
  document.getElementById('settings-overlay').classList.remove('visible');
}

function saveSettings() {
  const model = document.getElementById('model-select').value;
  state.model = model;
  localStorage.setItem('clearthread_model', model);
  closeSettings();
  showToast('✓ Settings saved', 'success');
}
```

- [ ] **Step 3: Remove API key event listeners**

In `index.html`, remove the API key toggle listener block (lines ~1953-1957):
```javascript
// API key show/hide toggle
document.getElementById('key-toggle').addEventListener('click', () => {
  const inp = document.getElementById('api-key-input');
  inp.type = inp.type === 'password' ? 'text' : 'password';
});
```

Delete those 4 lines entirely.

- [ ] **Step 4: Verify Settings modal**

1. Sign in (if needed) to dismiss the auth overlay
2. Click the ⚙ Settings button
3. Modal should show only the Model dropdown (no API key field)
4. Select Haiku, click Save Settings — toast "✓ Settings saved" appears
5. Reopen Settings — Haiku should still be selected

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: settings modal — remove API key field, update model options to Sonnet/Haiku"
```

---

## Task 8: Firestore Security Rules Update

**Files:** none (manual step in Firebase console)

- [ ] **Step 1: Update Firestore security rules (manual — browser)**

1. Go to https://console.firebase.google.com/project/milestone-tracker-955f4/firestore/rules
2. Replace the existing rules with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

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

    // Keep existing rules for other collections (Milestone Tracker, etc.)
    // Add them back here if they existed before
  }
}
```

3. Click **Publish**

Note: If other apps (Milestone Tracker) were using open rules on different collections, add those rules back above the `}` that closes the database block. Check the existing rules before publishing.

---

## Task 9: Update Main-Thread Claude Calls

**Files:**
- Modify: `index.html:1052-1099` (extractFromText function)
- Modify: `index.html:1647-1691` (doExtraction function — remove apiKey check)

- [ ] **Step 1: Replace extractFromText() to call Cloud Function**

Replace `extractFromText()` (lines ~1052-1099):
```javascript
async function extractFromText(text, modelOverride) {
  const apiKey = state.apiKey;
  if (!apiKey) throw new Error('No API key set. Please add your Claude API key in Settings.');

  const CHAR_LIMIT = 380000;
  if (text.length > CHAR_LIMIT) text = text.slice(0, CHAR_LIMIT);

  const resp = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': apiKey,
      'anthropic-version': '2023-06-01',
      'anthropic-dangerous-direct-browser-access': 'true',
    },
    body: JSON.stringify({
      model: modelOverride || state.model,
      max_tokens: 4096,
      system: SYSTEM_PROMPT,
      messages: [{ role: 'user', content: text }],
    }),
  });

  if (!resp.ok) {
    const err = await resp.json().catch(() => ({}));
    const msg = err?.error?.message || `API error ${resp.status}`;
    const error = new Error(msg);
    if (resp.status === 429) error.isRateLimit = true;
    throw error;
  }

  const data = await resp.json();
  const raw = data.content?.[0]?.text || '';

  let jsonStr = raw.replace(/^```json\s*/i, '').replace(/^```\s*/i, '').replace(/```\s*$/, '').trim();
  const firstBrace = jsonStr.indexOf('{');
  const lastBrace = jsonStr.lastIndexOf('}');
  if (firstBrace !== -1 && lastBrace > firstBrace) jsonStr = jsonStr.slice(firstBrace, lastBrace + 1);
  jsonStr = sanitizeJson(jsonStr);

  let parsed;
  try { parsed = JSON.parse(jsonStr); }
  catch { throw new Error(`Claude returned unexpected output: "${raw.slice(0, 120)}…"`); }

  return parsed;
}
```

With:
```javascript
async function extractFromText(text, modelOverride) {
  const CHAR_LIMIT = 380000;
  if (text.length > CHAR_LIMIT) text = text.slice(0, CHAR_LIMIT);

  const idToken = await auth.currentUser.getIdToken();
  const resp = await fetch(CLOUD_FUNCTION_URL, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${idToken}`,
    },
    body: JSON.stringify({
      model: modelOverride || state.model,
      max_tokens: 4096,
      system: SYSTEM_PROMPT,
      messages: [{ role: 'user', content: text }],
    }),
  });

  if (!resp.ok) {
    const err = await resp.json().catch(() => ({}));
    const msg = err?.error?.message || `API error ${resp.status}`;
    const error = new Error(msg);
    if (resp.status === 429) error.isRateLimit = true;
    throw error;
  }

  const data = await resp.json();
  const raw = data.content?.[0]?.text || '';

  let jsonStr = raw.replace(/^```json\s*/i, '').replace(/^```\s*/i, '').replace(/```\s*$/, '').trim();
  const firstBrace = jsonStr.indexOf('{');
  const lastBrace = jsonStr.lastIndexOf('}');
  if (firstBrace !== -1 && lastBrace > firstBrace) jsonStr = jsonStr.slice(firstBrace, lastBrace + 1);
  jsonStr = sanitizeJson(jsonStr);

  let parsed;
  try { parsed = JSON.parse(jsonStr); }
  catch { throw new Error(`Claude returned unexpected output: "${raw.slice(0, 120)}…"`); }

  return parsed;
}
```

- [ ] **Step 2: Remove API key check from doExtraction()**

In `doExtraction()` (lines ~1647-1691), find and remove:
```javascript
  if (!state.apiKey) {
    showImportError('Please add your Claude API key in Settings first.');
    return;
  }
```

Delete those 4 lines entirely.

- [ ] **Step 3: Verify paste-tab extraction end-to-end**

1. Sign in, click ＋ Import Thread
2. Paste a short meeting notes sample (anything — a few sentences about tasks)
3. Click Extract
4. Extraction should succeed and show the preview screen with items
5. Save — items should appear in the dashboard

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: extractFromText routes through Cloud Function with Firebase auth token"
```

---

## Task 10: Update Web Worker to Use Cloud Function

**Files:**
- Modify: `index.html:831-869` (extractChunk in worker)
- Modify: `index.html:871-914` (worker onmessage)
- Modify: `index.html:1605-1645` (runFileWorker in main thread)

- [ ] **Step 1: Update extractChunk() in the Web Worker**

In the `<script id="extraction-worker" type="javascript/worker">` block, replace `extractChunk()` (lines ~831-869):
```javascript
async function extractChunk(chunkText, apiKey, systemPrompt) {
  const resp = await fetch('https://api.anthropic.com/v1/messages', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'x-api-key': apiKey,
      'anthropic-version': '2023-06-01',
      'anthropic-dangerous-direct-browser-access': 'true',
    },
    body: JSON.stringify({
      model: 'claude-haiku-4-5-20251001',
      max_tokens: 8192,
      system: systemPrompt,
      messages: [{ role: 'user', content: chunkText }],
    }),
  });
  if (!resp.ok) {
    const err = await resp.json().catch(() => ({}));
    const msg = err?.error?.message || `API error ${resp.status}`;
    const error = new Error(msg);
    if (resp.status === 429) error.isRateLimit = true;
    throw error;
  }
  const data = await resp.json();
  if (data.stop_reason === 'max_tokens') {
    throw new Error('Response truncated — chunk produced too many items to fit in one response. Try a smaller file or contact support.');
  }
  const raw = data.content?.[0]?.text || '';
  let jsonStr = raw.replace(/^```json\s*/i, '').replace(/^```\s*/i, '').replace(/```\s*$/, '').trim();
  const firstBrace = jsonStr.indexOf('{');
  const lastBrace = jsonStr.lastIndexOf('}');
  if (firstBrace !== -1 && lastBrace > firstBrace) jsonStr = jsonStr.slice(firstBrace, lastBrace + 1);
  jsonStr = sanitizeJson(jsonStr);
  let parsed;
  try { parsed = JSON.parse(jsonStr); }
  catch { throw new Error(`JSON parse failed. Response start: "${raw.slice(0, 80)}" / end: "${raw.slice(-80)}"`); }
  return parsed;
}
```

With:
```javascript
async function extractChunk(chunkText, authToken, cloudFunctionUrl, systemPrompt) {
  const resp = await fetch(cloudFunctionUrl, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${authToken}`,
    },
    body: JSON.stringify({
      model: 'claude-haiku-4-5-20251001',
      max_tokens: 8192,
      system: systemPrompt,
      messages: [{ role: 'user', content: chunkText }],
    }),
  });
  if (!resp.ok) {
    const err = await resp.json().catch(() => ({}));
    const msg = err?.error?.message || `API error ${resp.status}`;
    const error = new Error(msg);
    if (resp.status === 429) error.isRateLimit = true;
    throw error;
  }
  const data = await resp.json();
  if (data.stop_reason === 'max_tokens') {
    throw new Error('Response truncated — chunk produced too many items to fit in one response. Try a smaller file or contact support.');
  }
  const raw = data.content?.[0]?.text || '';
  let jsonStr = raw.replace(/^```json\s*/i, '').replace(/^```\s*/i, '').replace(/```\s*$/, '').trim();
  const firstBrace = jsonStr.indexOf('{');
  const lastBrace = jsonStr.lastIndexOf('}');
  if (firstBrace !== -1 && lastBrace > firstBrace) jsonStr = jsonStr.slice(firstBrace, lastBrace + 1);
  jsonStr = sanitizeJson(jsonStr);
  let parsed;
  try { parsed = JSON.parse(jsonStr); }
  catch { throw new Error(`JSON parse failed. Response start: "${raw.slice(0, 80)}" / end: "${raw.slice(-80)}"`); }
  return parsed;
}
```

- [ ] **Step 2: Update worker onmessage to receive authToken + cloudFunctionUrl**

In the worker block, replace `self.onmessage` (lines ~871-914):
```javascript
self.onmessage = async function(e) {
  const { apiKey, text, systemPrompt } = e.data;
  ...
        result = await extractChunk(chunks[i], apiKey, systemPrompt);
  ...
```

Change the destructuring and extractChunk call:
```javascript
self.onmessage = async function(e) {
  const { authToken, cloudFunctionUrl, text, systemPrompt } = e.data;
  const { text: processedText, emailsRemoved } = preprocessEmails(text);
  const chunks = splitIntoChunks(processedText, CHUNK_SIZE);
  let allItems = [];
  let suggestedTitle = '';

  for (let i = 0; i < chunks.length; i++) {
    self.postMessage({ type: 'progress', current: i + 1, total: chunks.length });
    const chunkStart = Date.now();

    let result;
    for (let attempt = 0; attempt <= 3; attempt++) {
      try {
        result = await extractChunk(chunks[i], authToken, cloudFunctionUrl, systemPrompt);
        break;
      } catch (err) {
        const isRetryable = err.isRateLimit || err.message === 'Failed to fetch' || err instanceof TypeError;
        if (isRetryable && attempt < 3) {
          const label = err.isRateLimit
            ? `Rate limit hit — retrying chunk ${i + 1}`
            : `Network error — retrying chunk ${i + 1}`;
          await waitWithCountdown(30, label);
        } else {
          self.postMessage({ type: 'error', message: err.message || 'Extraction failed' });
          return;
        }
      }
    }

    if (!suggestedTitle && result.suggestedTitle) suggestedTitle = result.suggestedTitle;
    allItems = allItems.concat(result.items || []);

    if (i < chunks.length - 1) {
      const elapsed = Date.now() - chunkStart;
      const remaining = Math.ceil(Math.max(0, RATE_WINDOW_MS - elapsed) / 1000);
      if (remaining > 0) {
        await waitWithCountdown(remaining, `Chunk ${i + 1} of ${chunks.length} complete`);
      }
    }
  }

  self.postMessage({ type: 'done', suggestedTitle, items: allItems, chunkCount: chunks.length, emailsRemoved });
};
```

- [ ] **Step 3: Update runFileWorker() to fetch token and pass it**

Replace `runFileWorker()` in the main JS section (lines ~1605-1645):
```javascript
function runFileWorker(text) {
  return new Promise((resolve, reject) => {
    const blob = new Blob(
      [document.getElementById('extraction-worker').textContent],
      { type: 'application/javascript' }
    );
    const worker = new Worker(URL.createObjectURL(blob));
    state.worker = worker;
    ...
    worker.postMessage({ apiKey: state.apiKey, text, systemPrompt: SYSTEM_PROMPT });
  });
}
```

With:
```javascript
async function runFileWorker(text) {
  const idToken = await auth.currentUser.getIdToken();
  return new Promise((resolve, reject) => {
    const blob = new Blob(
      [document.getElementById('extraction-worker').textContent],
      { type: 'application/javascript' }
    );
    const worker = new Worker(URL.createObjectURL(blob));
    state.worker = worker;

    worker.onmessage = function(e) {
      const msg = e.data;
      if (msg.type === 'progress') {
        document.getElementById('extracting-text').textContent =
          msg.total === 1 ? 'Extracting actions, deadlines, and more…'
                          : `Processing chunk ${msg.current} of ${msg.total}…`;
        document.getElementById('extracting-sub').textContent =
          msg.total === 1 ? 'This usually takes 5–15 seconds'
                          : `Using Haiku — ${Math.round((msg.current - 1) / msg.total * 100)}% complete`;
      } else if (msg.type === 'tick') {
        document.getElementById('extracting-text').textContent =
          `${msg.label} — resuming in ${msg.secondsLeft}s`;
      } else if (msg.type === 'done') {
        state.worker = null;
        worker.terminate();
        resolve({ suggestedTitle: msg.suggestedTitle, items: msg.items, _chunkCount: msg.chunkCount, _emailsRemoved: msg.emailsRemoved });
      } else if (msg.type === 'error') {
        state.worker = null;
        worker.terminate();
        reject(new Error(msg.message));
      }
    };

    worker.onerror = function(e) {
      state.worker = null;
      worker.terminate();
      reject(new Error(e.message || 'Worker error'));
    };

    worker.postMessage({ authToken: idToken, cloudFunctionUrl: CLOUD_FUNCTION_URL, text, systemPrompt: SYSTEM_PROMPT });
  });
}
```

- [ ] **Step 4: Verify file import end-to-end**

1. Sign in, click ＋ Import Thread, switch to the File tab
2. Upload a `.txt` meeting notes file
3. Click Extract — chunking should proceed through the Cloud Function
4. Preview should appear with extracted items
5. Save — items appear in dashboard scoped to your user

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: Web Worker routes through Cloud Function — authToken replaces apiKey"
```

---

## Task 11: .docx Support (mammoth.js)

**Files:**
- Modify: `index.html:8-11` (add mammoth CDN)
- Modify: `index.html:565` (update file input accept)
- Modify: `index.html:1895-1940` (replace readFile, add readDocxFile + processFileContent)

- [ ] **Step 1: Add mammoth.js CDN**

In `index.html` `<head>`, after the Firebase Auth script tag, add:

```html
  <!-- File parsing -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/mammoth/1.6.0/mammoth.browser.min.js"></script>
```

- [ ] **Step 2: Update file input accept attribute**

In `index.html`, find line ~565:
```html
<input type="file" id="file-input" accept=".txt">
```

Replace with:
```html
<input type="file" id="file-input" accept=".txt,.docx,.pdf">
```

- [ ] **Step 3: Replace readFile() and add helpers**

Replace `readFile()` and everything through `reader.readAsText(file);` (lines ~1895-1940):
```javascript
function readFile(file) {
  if (!file.name.endsWith('.txt')) {
    showToast('Please upload a .txt file', 'error');
    return;
  }
  const reader = new FileReader();
  reader.onload = e => {
    state.fileContent = e.target.result;
    document.getElementById('file-name').textContent = `✓ ${file.name} loaded`;

    // Character count and chunk info
    const total = state.fileContent.length;
    const { text: processedText, emailsRemoved } = preprocessEmails(state.fileContent);
    const chunks = splitIntoChunks(processedText, 50000);
    const sizeEl = document.getElementById('file-info-size');
    let sizeText = `${total.toLocaleString()} chars`;
    if (emailsRemoved > 0) sizeText += ` · ${emailsRemoved} duplicate email${emailsRemoved !== 1 ? 's' : ''} removed`;
    sizeText += chunks.length > 1
      ? ` — will be processed in ${chunks.length} chunks using Haiku`
      : ` — full file will be sent to Claude`;
    sizeEl.textContent = sizeText;
    sizeEl.className = 'file-info-size';

    // Last email details
    const email = parseLastEmail(state.fileContent);
    const emailSection = document.getElementById('file-info-email');
    const emailRows    = document.getElementById('file-info-email-rows');
    if (email) {
      const row = (label, val) => val
        ? `<div class="file-info-row"><strong>${label}</strong><span>${escHtml(val)}</span></div>`
        : '';
      emailRows.innerHTML = row('From:', email.from) + row('Sent:', email.sent) + row('Subject:', email.subject);
      emailSection.style.display = '';
    } else {
      emailSection.style.display = 'none';
    }
    document.getElementById('file-info').style.display = 'block';

    // Pre-fill thread name from filename
    const threadName = document.getElementById('thread-name-input');
    if (!threadName.value) {
      threadName.value = file.name.replace(/\.txt$/i, '').replace(/[-_]/g, ' ');
    }
  };
  reader.readAsText(file);
}
```

With:
```javascript
function readFile(file) {
  const ext = file.name.split('.').pop().toLowerCase();
  if (!['txt', 'docx', 'pdf'].includes(ext)) {
    showToast('Please upload a .txt, .docx, or .pdf file', 'error');
    return;
  }
  if (ext === 'txt') {
    const reader = new FileReader();
    reader.onload = e => processFileContent(e.target.result, file.name, 'TXT');
    reader.readAsText(file);
  } else if (ext === 'docx') {
    readDocxFile(file);
  } else {
    readPdfFile(file);
  }
}

async function readDocxFile(file) {
  const arrayBuffer = await file.arrayBuffer();
  try {
    const result = await mammoth.extractRawText({ arrayBuffer });
    if (!result.value.trim()) {
      showToast('Could not read this Word file. Try saving as .docx from Word or Google Docs.', 'error');
      return;
    }
    processFileContent(result.value, file.name, 'DOCX');
  } catch {
    showToast('Could not read this Word file. Try saving as .docx from Word or Google Docs.', 'error');
  }
}

async function readPdfFile(file) {
  const arrayBuffer = await file.arrayBuffer();
  try {
    const pdf = await pdfjsLib.getDocument({ data: new Uint8Array(arrayBuffer) }).promise;
    let fullText = '';
    for (let i = 1; i <= pdf.numPages; i++) {
      const page = await pdf.getPage(i);
      const content = await page.getTextContent();
      fullText += content.items.map(item => item.str).join(' ') + '\n';
    }
    if (!fullText.trim()) {
      showToast("This PDF doesn't contain extractable text. Try exporting from your meeting notes app as a text-based PDF.", 'error');
      return;
    }
    processFileContent(fullText, file.name, 'PDF');
  } catch {
    showToast("This PDF doesn't contain extractable text. Try exporting from your meeting notes app as a text-based PDF.", 'error');
  }
}

function processFileContent(text, fileName, fileType) {
  state.fileContent = text;
  document.getElementById('file-name').textContent = `✓ ${fileName} loaded`;

  const total = text.length;
  const { text: processedText, emailsRemoved } = preprocessEmails(text);
  const chunks = splitIntoChunks(processedText, 50000);
  const sizeEl = document.getElementById('file-info-size');
  let sizeText = `${fileType} · ${total.toLocaleString()} chars`;
  if (emailsRemoved > 0) sizeText += ` · ${emailsRemoved} duplicate email${emailsRemoved !== 1 ? 's' : ''} removed`;
  sizeText += chunks.length > 1
    ? ` — will be processed in ${chunks.length} chunks using Haiku`
    : ` — full file will be sent to Claude`;
  sizeEl.textContent = sizeText;
  sizeEl.className = 'file-info-size';

  const email = parseLastEmail(text);
  const emailSection = document.getElementById('file-info-email');
  const emailRows = document.getElementById('file-info-email-rows');
  if (email) {
    const row = (label, val) => val
      ? `<div class="file-info-row"><strong>${label}</strong><span>${escHtml(val)}</span></div>`
      : '';
    emailRows.innerHTML = row('From:', email.from) + row('Sent:', email.sent) + row('Subject:', email.subject);
    emailSection.style.display = '';
  } else {
    emailSection.style.display = 'none';
  }
  document.getElementById('file-info').style.display = 'block';

  const threadName = document.getElementById('thread-name-input');
  if (!threadName.value) {
    threadName.value = fileName.replace(/\.(txt|docx|pdf)$/i, '').replace(/[-_]/g, ' ');
  }
}
```

- [ ] **Step 4: Verify .docx upload**

1. Create a short Word document with meeting notes content (or save any .docx)
2. Sign in to the app, click ＋ Import Thread → File tab
3. Upload the .docx file
4. File info panel should show "DOCX · N chars"
5. Click Extract — items should be extracted successfully

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "feat: .docx file support via mammoth.js — readDocxFile + processFileContent"
```

---

## Task 12: PDF Support (pdf.js)

**Files:**
- Modify: `index.html` head (add pdf.js CDN)
- Modify: `index.html` JS init area (set workerSrc)

Note: `readPdfFile()` was already added in Task 11 Step 3.

- [ ] **Step 1: Add pdf.js CDN script tags**

In `index.html` `<head>`, after the mammoth script tag, add:

```html
  <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
```

- [ ] **Step 2: Set pdf.js worker source**

In `index.html`, in the main `<script>` block, immediately after the Firebase config/init lines (after `const CLOUD_FUNCTION_URL = ...;`), add:

```javascript
// pdf.js worker
pdfjsLib.GlobalWorkerOptions.workerSrc = 'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js';
```

- [ ] **Step 3: Verify PDF upload**

1. Export a meeting notes document as PDF from Word, Google Docs, or another app (must be text-based, not a scan)
2. Sign in to the app, click ＋ Import Thread → File tab
3. Upload the PDF
4. File info panel should show "PDF · N chars"
5. Click Extract — items should be extracted successfully
6. Test error case: try uploading a scanned/image-only PDF — toast should say "This PDF doesn't contain extractable text."

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: PDF file support via pdf.js — text-based PDFs extract via readPdfFile"
```

---

## Task 13: Deploy + Final Verification

- [ ] **Step 1: Push to GitHub Pages**

```bash
cd /Users/gp/Documents/clearthread-repo
git push
```

Wait ~60 seconds, then hard-refresh `https://gparrent71-hue.github.io/clearthread/` (`Cmd+Shift+R`).

- [ ] **Step 2: Verify live site — auth**

1. Open the live URL in an incognito window
2. Auth overlay should appear
3. Create a new account with email + password
4. Overlay should dismiss; dashboard should load (empty)
5. Sign out — overlay reappears
6. Sign back in with same credentials
7. Try "Forgot password?" with your email — check that a reset email arrives
8. Try Google sign-in (if the OAuth consent screen is configured)

- [ ] **Step 3: Verify live site — extraction**

1. Sign in, click ＋ Import Thread → Paste tab
2. Paste a paragraph of meeting notes
3. Click Extract — result should come back via Cloud Function (not direct Anthropic call)
4. Save — items visible in dashboard

- [ ] **Step 4: Verify live site — file uploads**

1. Upload a `.txt` file — should work as before
2. Upload a `.docx` file — should show "DOCX · N chars" and extract correctly
3. Upload a PDF — should show "PDF · N chars" and extract correctly

- [ ] **Step 5: Verify multi-user isolation**

1. Create a second account (use a different browser or incognito)
2. Import some items with that account
3. Sign in as the first account — should see only that account's items

- [ ] **Step 6: Google OAuth consent screen (if Google sign-in shows an error)**

If Google sign-in fails with "This app hasn't been verified":
1. Go to https://console.cloud.google.com/apis/credentials/consent (project: milestone-tracker-955f4)
2. Under "Test users", add the email addresses of users who will use Google sign-in
3. This is enough for a personal tool — no need to go through full verification

---

## Notes for Future Reference

- **Cloud Function URL:** `https://us-central1-milestone-tracker-955f4.cloudfunctions.net/callClaude`
- **To update API key:** `firebase functions:secrets:set ANTHROPIC_API_KEY` then `firebase deploy --only functions`
- **To add a new user:** They create their own account at the live URL — registration is open
- **Firebase Auth console:** https://console.firebase.google.com/project/milestone-tracker-955f4/authentication/users
