# Reuse guide: patterns from myLedger

Source: myLedger v4.39, a single-file `index.html` (vanilla JS, no build step, no libraries) that keeps its data in one JSON file in the user's cloud folder (iCloud Drive) and is hosted on GitHub Pages as an installable PWA.

How to use this file: paste it into the other project and say which sections to apply. Code is shown with `APP` placeholders (app name, file name, data type). Replace `state.data` and the `render()` function with your own. The patterns do not depend on ledger or accounting logic.

Contents
1. Architecture and conventions
2. Opening and saving files (the important part)
3. Keeping the file in step across devices (conflict handling)
4. Safety nets (autosave, draft copy, leaving the page)
5. Test data flag
6. In-app confirm / alert dialogs
7. Look and feel (colour tokens, buttons, icons, tabs, tables, dialogs)
8. PWA (manifest, service worker, icons) and deploying to GitHub Pages
9. Mac helper: Automator script for downloads, backups and archives
10. Testing (Node harness and Playwright)
11. Gotchas learned the hard way
12. Checklist for a new project

---

## 1. Architecture and conventions

- **One file**: `index.html` with `<style>` and one `<script>`; extra files are only the manifest, the service worker and icons. Easy to host, upload through the GitHub web UI and cache offline.
- **State**: one `state` object. `state.data` is the whole document (what gets saved). `render()` rebuilds the UI from state (plain DOM, no framework). Anything that changes data calls `changed()`, which sets `dirty`, writes a draft and re-renders.
- **Data file format**: JSON with a `version` number and `savedAt` (ISO timestamp written on every save). `migrate(d)` upgrades older files in memory (saved in the new format on the next save). `fileProblem(d)` refuses files that are not yours or are from a newer version (message: reload to update).
- **Version in the UI**: `APP_VERSION` and `APP_BUILD` constants shown next to the title; bump them on every change. Keep a `CHANGELOG.md` (newest first) and a `README.md`; list future ideas in `ROADMAP.md`.
- **Dates** are stored as plain `YYYY-MM-DD` strings, never Date objects; money is kept in whole cents for arithmetic (`Math.round(x * 100)`).
- **Safe DOM**: build text with `textContent`, never `innerHTML` with data.

```js
'use strict';
const $ = id => document.getElementById(id);
const FS = ('showOpenFilePicker' in window) && ('showSaveFilePicker' in window);   // File System Access API (Chrome, Edge; NOT Brave by default, NOT iOS Safari)
const TOUCH = navigator.maxTouchPoints > 1;     // iPhone / iPad (iPadOS reports as Mac but has touch)
if (TOUCH) document.documentElement.classList.add('touch');   // CSS gives touch devices roomier buttons
const DATA_VERSION = 1;
const state = { data:null, dirty:false, handle:null, fileName:null, baseSavedAt:null, draftAt:null, msg:null, msgKind:'err', needsReconnect:false };
let saving = false;

function changed() { state.dirty = true; state.msg = null; persistDraft(); render(); }
function setMsg(m, kind) { state.msg = m; state.msgKind = kind || 'err'; render(); }
const uid = () => (crypto.randomUUID ? crypto.randomUUID() : Date.now().toString(36) + Math.random().toString(36).slice(2));
```

---

## 2. Opening and saving files

Three environments behave differently. The app detects which one it is in (`FS`, `TOUCH`) and offers the right buttons.

| Environment | How it works |
|---|---|
| Chrome / Edge (desktop) | File System Access API: pick a file or folder once, keep the handle, autosave silently. |
| Brave (desktop) | The API is disabled by default (`brave://flags/#file-system-access-api` enables it). Falls back to the "manual" path: Open file uses a file input, Save downloads a file. |
| iPhone / iPad Safari and the installed PWA | No File System Access. Open via file input; Save via the Web Share sheet (Save to Files) or a download. |

### 2.1 Browser-side storage (draft copy and file handle)

IndexedDB holds two things: a **draft** (a copy of the data on every change, so a crash or closed tab loses nothing) and the **file/folder handle** (so the user does not have to pick the file again). Every call is wrapped so a blocked IndexedDB (private mode) just means "no draft", never an error.

```js
function idb() { return new Promise((res, rej) => { const r = indexedDB.open('APP', 1); r.onupgradeneeded = () => r.result.createObjectStore('kv'); r.onsuccess = () => res(r.result); r.onerror = () => rej(r.error); }); }
async function kvGet(k) { try { const db = await idb(); return await new Promise((res, rej) => { const q = db.transaction('kv').objectStore('kv').get(k); q.onsuccess = () => res(q.result); q.onerror = () => rej(q.error); }); } catch (e) { return undefined; } }
async function kvSet(k, v) { try { const db = await idb(); await new Promise((res, rej) => { const tx = db.transaction('kv', 'readwrite'); tx.objectStore('kv').put(v, k); tx.oncomplete = res; tx.onerror = () => rej(tx.error); }); } catch (e) { /* storage unavailable */ } }
function persistDraft() {
  if (!state.data) return Promise.resolve();
  state.draftAt = new Date().toISOString();
  return kvSet('draft', { data:state.data, dirty:state.dirty, baseSavedAt:state.baseSavedAt, fileName:state.fileName, draftAt:state.draftAt });
}
```

### 2.2 Opening a file

- **FS path**: `showOpenFilePicker()` with **no file-type filter**. A filter greys out files the user wants (iCloud adds " (1)" suffixes, `.bak` copies, and so on). Keep the returned handle (`kvSet('handle', h)`) so later saves go straight to the same file.
- **Manual path**: a hidden `<input type="file" id="fileInput">`. On a computer, do **not** set `accept` (same greying problem). On touch devices set `accept=".json,.txt,application/json,text/plain"`, because without it iOS offers the Photo Library. Provide an **"Open any file"** button that removes the filter, as a fallback if the file is greyed out.
- After reading: `fileProblem(d)` for validation, then `adopt(d, name)`.

```js
const ACCEPT = '.json,.txt,application/json,text/plain';
function confirmDiscard() { return !state.dirty || ask({ ok:'Discard changes', danger:1 }, 'You have unsaved changes. Opening a file will replace them. Continue?'); }   // async: see section 6

async function openFile() {
  if (!await confirmDiscard()) return;
  if (FS) {
    try {
      const [h] = await window.showOpenFilePicker();
      const d = JSON.parse(await (await h.getFile()).text());
      const bad = fileProblem(d); if (bad) return setMsg(bad, 'err');
      state.handle = h; await kvSet('handle', h);
      adopt(d, h.name);
    } catch (e) { if (e.name !== 'AbortError') setMsg('Could not open file: ' + e.message, 'err'); }
  } else {
    if (TOUCH) $('fileInput').accept = ACCEPT; else $('fileInput').removeAttribute('accept');
    $('fileInput').click();
  }
}
async function openAnyFile() { if (!await confirmDiscard()) return; $('fileInput').removeAttribute('accept'); $('fileInput').click(); }

$('fileInput').addEventListener('change', async ev => {
  const f = ev.target.files[0]; ev.target.value = ''; if (!f) return;
  try {
    const d = JSON.parse(await f.text());
    const bad = fileProblem(d); if (bad) return setMsg(bad, 'err');
    // manual path cannot watch the file, so protect against loading an OLDER copy over a newer one
    if (state.data && state.baseSavedAt && d.savedAt && d.savedAt < state.baseSavedAt &&
        !await ask({ ok:'Load anyway' }, 'This file is OLDER than the version currently loaded.\n\nFile saved: ' + fmtTime(d.savedAt) + '\nLoaded version saved: ' + fmtTime(state.baseSavedAt) + '\n\nLoad it anyway?')) return;
    adopt(d, f.name);
  } catch (e) { setMsg('Could not read file: ' + e.message, 'err'); }
});

function adopt(d, name) {                    // make a loaded file the current document
  state.data = migrate(d); state.baseSavedAt = d.savedAt || null; state.dirty = false; state.msg = null;
  if (name) state.fileName = name;
  persistDraft(); render();
}
```

### 2.3 Saving, FS path (write to the same file)

Before overwriting, re-read the file; if its `savedAt` is newer than the one you loaded, someone (another device) saved since, so warn. Write with `createWritable()`; stamp `savedAt`; remember it as `baseSavedAt`.

```js
async function saveToHandle(auto) {
  if (saving || !state.handle || !state.data) return false;
  saving = true;
  try {
    const h = state.handle;
    let perm = await h.queryPermission({ mode:'readwrite' });
    if (perm !== 'granted') {                                   // permission can lapse; asking needs a user tap, so autosave only reports it
      if (auto) { setMsg('Folder access needs re-approval. Tap Save now.', 'warn'); return false; }
      perm = await h.requestPermission({ mode:'readwrite' });
      if (perm !== 'granted') return false;
    }
    if (state.baseSavedAt) {                                    // conflict check
      try {
        const t = await (await h.getFile()).text();
        if (t.trim()) {
          const disk = JSON.parse(t);
          if (disk.savedAt && disk.savedAt > state.baseSavedAt) {
            if (auto) { setMsg('The file on disk is newer (saved elsewhere). Tap Save now to decide.', 'warn'); return false; }
            if (!await ask({ ok:'Overwrite', danger:1 }, 'The file was saved more recently from another device (' + fmtTime(disk.savedAt) + ').\n\nOverwrite it with this version?')) return false;
          }
        }
      } catch (e) { /* unreadable or empty file: proceed */ }
    }
    const savedAt = new Date().toISOString();
    const w = await h.createWritable();
    await w.write(JSON.stringify({ ...state.data, savedAt }, null, 2));
    await w.close();
    state.data.savedAt = savedAt; state.baseSavedAt = savedAt; state.dirty = false; state.msg = null; state.needsReconnect = false;
    await persistDraft();
    return true;
  } catch (e) { setMsg('Save failed: ' + e.message, 'err'); return false; }
  finally { saving = false; render(); }
}
```

After a browser restart the saved handle needs a **tap** to regain permission (browsers only allow the permission prompt from a click). Show a "Reconnect" button (`needsReconnect`) that calls `h.requestPermission()`, then re-reads the file and adopts it if it is newer.

### 2.4 Saving, manual path (iPhone / iPad / Brave)

A function called directly from a button tap (iOS requires that). Touch devices try the Web Share sheet with the file only (adding a title or text makes iOS save a stray extra file); otherwise download.

```js
async function exportFile() {
  const savedAt = new Date().toISOString();
  const file = new File([JSON.stringify({ ...state.data, savedAt }, null, 2)], outName(), { type:'application/json' });
  let done = false;
  if (TOUCH && navigator.canShare && navigator.canShare({ files:[file] })) {
    try { await navigator.share({ files:[file] }); done = true; }
    catch (e) { if (e.name === 'AbortError') return; }          // user cancelled: not saved, stay dirty
  }
  if (!done) {
    const a = document.createElement('a'); a.href = URL.createObjectURL(file); a.download = file.name;
    document.body.appendChild(a); a.click(); a.remove(); setTimeout(() => URL.revokeObjectURL(a.href), 5000); done = true;
  }
  state.data.savedAt = savedAt; state.baseSavedAt = savedAt; state.dirty = false; state.msg = null;
  await persistDraft(); render();
}
// Save under the name the file was opened with. Safari and Brave add " (1)", " (2)" to repeated downloads, so strip that.
function outName() {
  const n = (state.fileName || '').replace(/ \(\d+\)(?=\.json$)/i, '');
  return /^APPNAME[^\\\/]*\.json$/i.test(n) ? n : 'APPNAME.json';
}
function saveNow() { if (!state.data) return; if (FS && state.handle) saveToHandle(false); else if (FS) openSettings(); else exportFile(); }
const SAVE_LABEL = FS ? 'Save now' : (TOUCH ? 'Save to Files' : 'Save file');   // button text follows the environment
```

Note for Brave users: every manual save is a new download, and the browser names repeats "file (1).json". The Mac helper in section 9 files these away and keeps the standard name.

### 2.5 Choosing a cloud folder (FS path)

`showDirectoryPicker({ mode:'readwrite', id:'APP' })` lets the user pick iCloud Drive / Dropbox / OneDrive or any folder once; the app creates and autosaves its standard file there. Keep the directory handle in IndexedDB. When looking for the file inside the folder, try the exact standard name first, then a sensible look-alike (e.g. `APP (1).json`), skipping archive files:

```js
async function fileInDir(dir) {
  try { return await dir.getFileHandle(STD_NAME); } catch (e) { if (e.name !== 'NotFoundError') throw e; }
  const names = [];
  for await (const [n, h] of dir.entries()) if (h.kind === 'file' && /\.json$/i.test(n) && !/archive/i.test(n)) names.push(n);
  const best = names.filter(n => /APPNAME/i.test(n)).sort((a, b) => a.length - b.length)[0] || (names.length === 1 ? names[0] : null);
  return best ? await dir.getFileHandle(best) : null;
}
```

Guard against clobbering: if the app is about to create the standard file in a folder where it already exists (for example after "start a new, empty file"), ask before replacing it (`getFileHandle(name)` without `create`, catch `NotFoundError`).

### 2.6 Start a new, empty document

Clear `state` (data, handle, fileName, baseSavedAt, dirty, messages), clear the stored handle (`kvSet('handle', null)`), reset any test flag, then open the "create first item" dialog. The previously open file is not touched. Confirm with the user first (section 6).

### 2.7 Archive pattern (optional)

Old data can be moved to separate read-only archive files (`APP-archive-YYYY-MM.json`, with a `type:'APP-archive'` field so the main loader refuses them) while the main file keeps a short list of its archives, so totals stay unchanged. A separate read-only viewer opens one or several archive files. Only worth copying if the data grows without bound.

---

## 3. Keeping the file in step across devices

The same file is opened on a Mac, an iPad and a phone through iCloud. The rule: **`savedAt` inside the JSON decides what is newer**, not the file system's modified time (iCloud sync can change the file time without a real edit).

```js
async function refreshFromFile() {           // runs when the tab becomes visible, after reconnect, and every 45 s while open
  if (!state.handle || saving) return;
  try {
    if (await state.handle.queryPermission({ mode:'readwrite' }) !== 'granted') { if (!state.dirty) { state.needsReconnect = true; render(); } return; }
    state.needsReconnect = false;
    const d = JSON.parse(await (await state.handle.getFile()).text());
    if (fileProblem(d) || !d.savedAt) { render(); return; }
    const newer = !state.baseSavedAt || d.savedAt > state.baseSavedAt;
    if (!newer) { if (state.newerDisk) { state.newerDisk = null; render(); } return; }
    if (!state.dirty) { adopt(d); state.msg = 'Loaded a newer version of ' + state.fileName + ' (saved ' + fmtTime(d.savedAt) + ').'; state.msgKind = 'ok'; render(); return; }
    if (state.ignoreDisk === d.savedAt) return;                       // user already chose "Keep my version" for this one
    state.newerDisk = d.savedAt; state.newerData = d; render();       // unsaved edits here AND a newer file: let the user decide
  } catch (e) { /* ignore */ }
}
function loadNewerDisk() { const d = state.newerData; if (!d) return; ask({ ok:'Load newer file', danger:1 }, 'Load the newer file version? Your unsaved changes here will be replaced.').then(y => { if (y) { state.dirty = false; state.newerDisk = null; state.newerData = null; adopt(d); } }); }
function keepMine() { state.ignoreDisk = state.newerDisk; state.newerDisk = null; state.newerData = null; render(); }
setInterval(() => { if (document.visibilityState === 'visible' && FS && state.handle) refreshFromFile(); }, 45000);
```

Banner in `render()` when `state.newerDisk` is set: text "A newer version of FILE was saved elsewhere (time). You also have unsaved changes here." with buttons **Load newer file** (primary) and **Keep my version**.

The status banner has four states: green "All changes saved to FILE (time)", amber "Unsaved changes. Autosave in progress", amber "needs Reconnect", red errors. Keep the banner function in one place.

---

## 4. Safety nets

```js
function flush() { persistDraft(); if (FS && state.handle && state.dirty) saveToHandle(true); }
setInterval(() => { if (document.visibilityState === 'visible' && state.dirty && FS && state.handle) saveToHandle(true); }, 30000);   // autosave every 30 s
document.addEventListener('visibilitychange', () => { if (document.visibilityState === 'hidden') flush(); else refreshFromFile(); });
window.addEventListener('pagehide', flush);
window.addEventListener('blur', flush);
window.addEventListener('beforeunload', ev => { if (state.dirty) { ev.preventDefault(); ev.returnValue = ''; } });   // browser's own "leave site?" prompt
```

Startup: restore the draft from IndexedDB first (so work in progress survives a reload), then load the saved handle, `render()`, then `await refreshFromFile()`.

```js
(async () => {
  if (navigator.storage && navigator.storage.persist) { try { await navigator.storage.persist(); } catch (e) {} }   // ask the browser not to evict our storage
  const draft = await kvGet('draft');
  if (draft && valid(draft.data)) { state.data = migrate(draft.data); state.dirty = !!draft.dirty; state.baseSavedAt = draft.baseSavedAt || null; state.fileName = draft.fileName || null; state.draftAt = draft.draftAt || null; }
  if (FS) { const h = await kvGet('handle'); if (h) { state.handle = h; state.fileName = h.name; } }
  render();
  await refreshFromFile();
})();
```

---

## 5. Test data flag

Problem: the user tries the app with sample data and later uses the real file; the two must never be confused.

- The **file itself says** it is test data: `settings.testData: true`. This travels with the file, so it survives being opened on any device (no separate "mode" setting).
- `setTest(on)` toggles `body.test` (an orange stripe across the top via CSS), a "TEST DATA" badge next to the title, and the page title.
- When opening a file whose **name** contains "test" but which is not flagged, ask once whether it is test data; Yes writes the flag into the file.
- The standard file name follows the flag (`ledger.json` vs `ledger-test.json`), so saves from test data never overwrite the real file.

```js
const PROFILES = { main:{ file:'APP.json' }, test:{ file:'APP-test.json' } };
let PROFILE = 'main', PFILE = PROFILES.main.file;
function setTest(on) {
  PROFILE = on ? 'test' : 'main'; PFILE = PROFILES[PROFILE].file;
  document.body.classList.toggle('test', !!on);
  const b = $('profBadge'); if (b) b.hidden = !on;
  document.title = on ? 'APP (test)' : 'APP';
}
// in adopt(): if (st.testData) setTest(true); else if (name && /test/i.test(name)) { setTest(false); ask({ok:'Yes, test data', cancel:'No, my real data'}, ...).then(yes => { if (yes) { data.settings.testData = true; setTest(true); state.dirty = true; persistDraft(); render(); } }); } else setTest(false);
```
```css
body.test { border-top:6px solid #e08a00; }
.badge { background:#e08a00; color:#fff; border-radius:4px; padding:1px 6px; font-size:.65rem; letter-spacing:.05em; }
```

---

## 6. In-app confirm / alert dialogs

Browser `confirm()` / `alert()` boxes are headed "yourname.github.io says", cannot be styled and cannot have custom button labels. Replace them with one `<dialog>` and an `ask()` that returns a Promise. **Every caller becomes `async` and uses `await`**; a function called from tests must be awaited too.

```html
<dialog id="askDlg">
  <h3 style="margin:0 0 8px">APP</h3>
  <p id="askMsg" style="white-space:pre-line; margin:0 0 14px; line-height:1.45"></p>
  <div style="display:flex; justify-content:flex-end; gap:10px"><button type="button" id="askNo">Cancel</button><button type="button" id="askOk" class="primary">OK</button></div>
</dialog>
```
```js
function ask(o, msg) {               // resolves true (OK) / false (Cancel, Esc); {alert:1} shows only OK
  if (globalThis.__askHook) return Promise.resolve(globalThis.__askHook(msg, o));   // test hook (section 10)
  return new Promise(res => {
    const dlg = $('askDlg'), ok = $('askOk'), no = $('askNo');
    $('askMsg').textContent = msg;
    ok.textContent = o.alert ? 'OK' : (o.ok || 'OK'); ok.className = o.danger ? 'danger' : 'primary';
    no.textContent = o.cancel || 'Cancel'; no.hidden = !!o.alert;
    let done = false;
    const fin = v => { if (done) return; done = true; ok.onclick = no.onclick = null; dlg.onclose = null; if (dlg.open) dlg.close(); res(v); };
    ok.onclick = () => fin(true); no.onclick = () => fin(false); dlg.onclose = () => fin(false);
    dlg.showModal(); (o.danger ? no : ok).focus();       // dangerous actions start with Cancel focused
  });
}
// usage: if (!await ask({ ok:'Delete', danger:1 }, 'Delete "' + name + '"?')) return;
//        await ask({ alert:1 }, 'Enter an amount first.');
```

Use verbs on the OK button (Delete, Undo reconciliation, Overwrite, Load newer file); red for destructive ones. Submit handlers that `await ask` must call `ev.preventDefault()` **before** the first await. A promise-returning function called through Playwright `page.evaluate` will wait for the dialog: wrap the call in `() => { fn(); }`.

---

## 7. Look and feel

Design rules we settled on after trying others:
- **One accent colour (blue) only for the primary action** on a screen. Everything else is a calm neutral "tonal" button (Material Design 3 hierarchy: filled / tonal / outlined). Destructive actions use a soft red.
- Icon-only buttons for compact row actions (edit, delete, post, undo, skip, search), with a title tooltip and aria-label. Icons are single SVG paths drawn with the button's text colour (`stroke:currentColor`).
- Rectangular, not pill-shaped, tabs; thin 1px borders around each section card; month / group separator bars in long tables.
- Dark mode via `prefers-color-scheme`; all colours are CSS variables, so a dark palette is one block.
- Touch devices get bigger hit areas via the `.touch` class.
- Numbers right-aligned; negative amounts red (`.neg`); inputs use `inputmode="numeric"` with a fixed-decimal "money" behaviour (digits typed fill from the right, so "1234" becomes 12.34), with a ± button for the sign on phones.
- `input[type=date]` needs `-webkit-appearance:none; min-width:0; width:100%` or iOS draws it wider than its neighbours.

```css
:root { color-scheme: light dark; --bg:#fff; --fg:#1b1b1f; --muted:#6b6b76; --line:#c9ccd6; --accent:#0a66d6; --warn-bg:#fff4d6; --warn-fg:#6b4a00; --ok-bg:#e3f5e6; --ok-fg:#14532d; --card:#f6f6f9;
  --btn:#eef0f4; --btn-bd:#c2c8d4; --btn-fg:#2a3040; --btn-hv:#e1e5ed; --ico:#46526b; --tab:#e3ebf8; --tab-bd:#4a6fa5; --tab-on-bd:#07408a; --dng-bg:#fdf0f0; --dng-bd:#e3a9a6; --dng-fg:#b3261e; --ok-ico:#1b7f4b; }
@media (prefers-color-scheme: dark) { :root { --bg:#16161a; --fg:#ececf1; --muted:#9a9aa6; --line:#44444e; --accent:#5aa2ff; --warn-bg:#3a2e0a; --warn-fg:#ffd98a; --ok-bg:#12301b; --ok-fg:#9be3b0; --card:#202026;
  --btn:#2b2f3a; --btn-bd:#4b5160; --btn-fg:#e4e7ef; --btn-hv:#363b49; --ico:#b9c3d8; --tab:#243a5a; --tab-bd:#7fa3d9; --tab-on-bd:#a9c8f5; --dng-bg:#3a2223; --dng-bd:#7a4545; --dng-fg:#ff9a92; --ok-ico:#7ad9a0; } }
* { box-sizing:border-box; }
body { margin:0; background:var(--bg); color:var(--fg); font:16px/1.3 -apple-system, system-ui, sans-serif; }
main { max-width:920px; margin:0 auto; padding:10px 12px; }
.banner { display:flex; gap:10px; align-items:center; justify-content:space-between; padding:7px 10px; border-radius:10px; margin:6px 0; font-size:.92rem; }
.banner.warn { background:var(--warn-bg); color:var(--warn-fg); } .banner.ok { background:var(--ok-bg); color:var(--ok-fg); } .banner.err { background:#fde2e2; color:#7a1212; }
button { font:inherit; padding:6px 12px; border-radius:9px; border:1px solid var(--btn-bd); background:var(--btn); color:var(--btn-fg); cursor:pointer; }
button:hover { background:var(--btn-hv); }
button.primary { background:var(--accent); color:#fff; border-color:var(--tab-on-bd); } button.primary:hover { filter:brightness(1.08); }
button.danger { background:var(--dng-bg); color:var(--dng-fg); border-color:var(--dng-bd); }
button.icon { color:var(--ico); padding:3px 7px; line-height:0; } button.icon.danger { color:var(--dng-fg); } button.icon.ok { color:var(--ok-ico); }
button.mini { padding:2px 7px; font-size:.82rem; } button:disabled { opacity:.5; }
.touch button.icon { padding:7px 11px; }
.ico { width:16px; height:16px; fill:none; stroke:currentColor; stroke-width:1.8; stroke-linecap:round; stroke-linejoin:round; }
@media (prefers-color-scheme: dark) { button.primary { color:#06203d; } }
input, select { min-width:0; font:inherit; padding:6px 9px; border-radius:9px; border:1px solid var(--line); background:var(--bg); color:var(--fg); width:100%; }
input[type=date] { -webkit-appearance:none; appearance:none; min-width:0; width:100%; text-align:left; min-height:2.3em; }
.card { background:var(--card); border:1px solid var(--line); border-radius:12px; padding:9px 12px; margin:8px 0; }
.tab { border-radius:4px; padding:5px 14px; background:var(--tab); color:var(--fg); border:2px solid var(--tab-bd); font-weight:500; }
.tab.on { background:var(--accent); color:#fff; border-color:var(--tab-on-bd); font-weight:700; }
dialog { border:1px solid var(--line); border-radius:14px; background:var(--bg); color:var(--fg); padding:12px; width:min(420px, calc(100% - 32px)); }
dialog::backdrop { background:rgba(0,0,0,.45); }
.neg { color:#c62828; } @media (prefers-color-scheme: dark) { .neg { color:#ff8a80; } }
/* tables: scroll inside a wrapper, header row and first column stay in view */
.tablewrap { overflow:auto; max-height:75vh; border:1px solid var(--line); border-radius:10px; }
th, td { padding:3px 8px; text-align:left; border-bottom:1px solid var(--line); }
th { position:sticky; top:0; z-index:2; background:var(--card); font-size:.78rem; text-transform:uppercase; letter-spacing:.04em; color:var(--muted); box-shadow:0 1px 0 var(--line); }
td:first-child, th:first-child { position:sticky; left:0; background:var(--bg); z-index:1; box-shadow:1px 0 0 var(--line); white-space:nowrap; }
th:first-child { background:var(--card); z-index:3; }
```

Icon helper (one SVG path per icon; add more from any line-icon set such as Feather, 24x24 viewBox):

```js
const ICONS = {
  edit:'M12 20h9M16.5 3.5a2.1 2.1 0 0 1 3 3L7 19l-4 1 1-4z',
  trash:'M3 6h18M8 6V4h8v2M19 6l-1 14H6L5 6M10 11v6M14 11v6',
  check:'M5 13l4 4L19 7',
  x:'M6 6l12 12M18 6L6 18',
  swap:'M7 4L3 8l4 4M3 8h14M17 20l4-4-4-4M21 16H7'
};
function iconBtn(icon, title, fn, cls) {
  const b = document.createElement('button'); b.className = 'mini icon' + (cls ? ' ' + cls : ''); b.title = title; b.setAttribute('aria-label', title); b.onclick = fn;
  const NS = 'http://www.w3.org/2000/svg', svg = document.createElementNS(NS, 'svg'), p = document.createElementNS(NS, 'path');
  svg.setAttribute('viewBox', '0 0 24 24'); svg.setAttribute('class', 'ico'); svg.setAttribute('aria-hidden', 'true');
  p.setAttribute('d', ICONS[icon]); svg.appendChild(p); b.appendChild(svg); return b;
}
```

Other UI patterns worth reusing: a **Settings dialog** with collapsible `<details class="sset">` sections; a **search dialog** with results, a "go to" button that scrolls the main list to the item and flashes it (`scrollIntoView` + a temporary highlight class), and a **Today** button that jumps to the current date; a **balance sparkline** drawn as inline SVG; a month/group divider row in long lists.

---

## 8. PWA and deploying

Files in the repo root next to `index.html`: `manifest.webmanifest`, `sw.js`, `icon-192.png`, `icon-512.png`, `apple-touch-icon.png`, `favicon.png`.

```html
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#1f3a8f">
<meta name="apple-mobile-web-app-capable" content="yes">
<link rel="manifest" href="manifest.webmanifest">
<link rel="apple-touch-icon" href="apple-touch-icon.png">
<link rel="icon" href="favicon.png">
```
```json
{ "name":"APP", "short_name":"APP", "start_url":"./", "scope":"./", "display":"standalone", "background_color":"#ffffff", "theme_color":"#1f3a8f",
  "icons":[ {"src":"icon-192.png","sizes":"192x192","type":"image/png"}, {"src":"icon-512.png","sizes":"512x512","type":"image/png"}, {"src":"icon-512.png","sizes":"512x512","type":"image/png","purpose":"any"} ] }
```
```js
// sw.js: network first (a new version is picked up as soon as you are online), cached copy when offline
const CACHE = 'app-cache';
const ASSETS = ['./', './index.html', './manifest.webmanifest', './icon-192.png', './icon-512.png', './apple-touch-icon.png'];
self.addEventListener('install', e => { e.waitUntil(caches.open(CACHE).then(c => c.addAll(ASSETS)).then(() => self.skipWaiting())); });
self.addEventListener('activate', e => { e.waitUntil(self.clients.claim()); });
self.addEventListener('fetch', e => {
  const req = e.request;
  if (req.method !== 'GET' || new URL(req.url).origin !== self.location.origin) return;
  e.respondWith(fetch(req).then(res => { if (res.ok) { const copy = res.clone(); caches.open(CACHE).then(c => c.put(req, copy)); } return res; })
    .catch(() => caches.match(req, { ignoreSearch:true }).then(m => m || caches.match('./index.html'))));
});
```
```js
if ('serviceWorker' in navigator && (location.protocol === 'https:' || location.hostname === '127.0.0.1' || location.hostname === 'localhost'))
  window.addEventListener('load', () => navigator.serviceWorker.register('sw.js').catch(() => {}));
```

Deploy: GitHub repo, Settings > Pages > deploy from branch `main` / root. Updating = upload the changed files through the GitHub web UI (drag and drop, commit). The URL is `https://USER.github.io/REPO/`. On iPhone/iPad: Safari > Share > Add to Home Screen gives the installed app. Users get a new version on next open when online (network-first service worker). Bump the version/build constants so it is obvious which version is running.

---

## 9. Mac helper: Automator script (downloads, backups, archives)

Needed only because Brave (and some other browsers) cannot write straight to the cloud folder. An Automator **Folder Action** on `~/Downloads` ("Run Shell Script", input: as arguments) moves each downloaded data file into the iCloud folder under its standard name, keeps dated backups, and files archive files into a subfolder. It ignores every other download. (Give Automator / Folder Actions access under System Settings > Privacy & Security if prompted.)

```bash
DEST="$HOME/Library/Mobile Documents/com~apple~CloudDocs/APP"     # iCloud Drive/APP
BAK="$DEST/Backups"
ARC="$DEST/Archive"
mkdir -p "$DEST" "$BAK" "$ARC"

for f in "$@"; do
  name=$(basename "$f")
  sleep 1                                   # wait until the browser has finished writing
  case "$name" in
    *archive*.json)                         # archives keep their own name, minus the browser's " (n)"
      clean=$(echo "$name" | sed -E 's/ \([0-9]+\)\.json$/.json/')
      mv -f "$f" "$ARC/$clean"; continue ;;
    APP-test*.json) target="APP-test.json" ;;
    APP*.json)      target="APP.json" ;;
    *) continue ;;                          # ignore every other download
  esac
  if [ -f "$DEST/$target" ]; then           # dated backup of what is being replaced
    stamp=$(date +%Y-%m-%d_%H%M%S)
    cp "$DEST/$target" "$BAK/${target%.json}_$stamp.json"
  fi
  mv -f "$f" "$DEST/$target"
done

for base in APP APP-test; do                # keep only the 20 newest backups of each file
  ls -1t "$BAK"/${base}_2*.json 2>/dev/null | tail -n +21 | while read -r old; do rm -f "$old"; done
done
```

Caveat: if one app has several real data files, they must have different standard names and the `case` must route each of them separately, or a second file will overwrite the first (the backups would still hold the old copy).

---

## 10. Testing

**Logic tests in Node** (fast, no browser): load the HTML, take the text between `<script>` and `</script>`, and run it in a `vm` context with stub DOM elements. Elements are `Proxy` stubs that record `addEventListener` handlers, so a test can fill fields (`E.amount.value = '50'`) and call `handlers['newForm:submit']({ preventDefault(){} })`.

```js
const vm = require('vm'), fs = require('fs');
const code = fs.readFileSync('index.html', 'utf8').split('<script>')[1].split('</script>')[0];
const handlers = {}, els = {};
function stub(id) {
  const store = id ? { addEventListener: (t, fn) => { handlers[id + ':' + t] = fn; } } : {};
  return new Proxy(function () {}, {
    get(_, k) { if (k === 'then') return undefined; if (k === Symbol.toPrimitive) return () => ''; if (k in store) return store[k]; return (store[k] = stub()); },
    set(_, k, v) { store[k] = v; return true; },
    apply() { return stub(); },
  });
}
let alerts = [], confirmAnswer = true;
const ctx = {
  document: { documentElement: { classList: { add(){} } }, getElementById: id => els[id] || (els[id] = stub(id)), createElement: () => stub(), createTextNode: () => stub(), querySelectorAll: () => [] },
  window: { addEventListener() {}, showOpenFilePicker: undefined }, navigator: { maxTouchPoints: 0 }, crypto: { randomUUID: () => require('crypto').randomUUID() },
  setInterval() {}, setTimeout, console, URL, Date, JSON, Math, Set, Promise,
  __askHook: (msg, o) => o.alert ? (alerts.push(msg), true) : confirmAnswer,     // makes ask() synchronous-answer in tests
};
ctx.window.window = ctx.window;
vm.createContext(ctx); vm.runInContext(code, ctx);
const run = s => vm.runInContext(s, ctx);
const tick = () => new Promise(r => setTimeout(r, 0));
const H = async n => { const r = handlers[n + ':submit']({ preventDefault(){} }); await tick(); await tick(); return r; };   // await after anything that calls ask()
const ok = (c, m) => console.log(c ? 'ok  ' : 'FAIL', m);
```
Run with `TZ=America/Toronto node tests.js` (date logic depends on the time zone). Wrap the test body in an `async` function so it can `await H('form')`. Put these in the first tests: balances/totals after each kind of edit, migration of an old file, refusal of newer-version and wrong-type files.

**Browser checks with Playwright** (real layout, dialogs, screenshots): a Chromium is available at `/opt/pw-browsers/chromium`; launch with `args=['--no-proxy-server']`, serve the folder with `python3 -m http.server 8765 --bind 127.0.0.1`, load data by calling the app's own `adopt(data, 'file.json')` through `page.evaluate`. Use `viewport={'width':430,'height':800}` to check the phone layout and take a screenshot to look at. A mock File System Access directory can be built from the browser's OPFS (`navigator.storage.getDirectory()`) by overriding `showDirectoryPicker`.

---

## 11. Gotchas learned the hard way

- A file-type filter on the open dialog greys out files with suffixes like " (1)" or `.bak`. Use no filter on desktop; filter only on touch; always offer "Open any file".
- iOS: file/share actions must run directly inside the tap handler; sharing with a title or text adds an extra file; the Photo Library appears unless the input has `accept`.
- Brave: File System Access is off by default; saves arrive as downloads named "x (1).json". Strip the suffix when naming, and use the Mac helper.
- Do not compare file modified times across devices; compare a `savedAt` stored inside the file.
- `queryPermission` is silent, `requestPermission` needs a tap. After a restart the first save needs a "Reconnect" tap.
- Native `confirm()` titles show the site name; use the in-app `ask()` and remember every caller is async.
- A handler that awaits must call `preventDefault()` before the first `await`.
- All of a user's GitHub Pages repos share one origin (`USER.github.io`), so IndexedDB and localStorage are shared between apps. **Prefix the IndexedDB database name and every localStorage key with the app name**, or two apps will overwrite each other's draft and file handle.
- Always show the version/build in the UI; it is the first thing to check when something "didn't change" (cached old file).
- Dark mode: re-check every colour on both themes; text on filled buttons needs its own dark override.

---

## 12. Checklist for a new project

1. Decide the document shape; add `version` and `savedAt`; write `valid`, `fileProblem`, `migrate`.
2. Copy: state/`changed`/`setMsg`, IndexedDB helpers, open/save (FS and manual), `refreshFromFile`, safety nets, startup (sections 1 to 4).
3. Copy the `ask()` dialog (section 6) and use it instead of `confirm`/`alert` from the start.
4. Copy the CSS tokens, buttons, icons (section 7); change `--accent` to the project's colour.
5. Add the test-data flag if the project needs sample data (section 5).
6. Add manifest, service worker, icons; publish on GitHub Pages (section 8).
7. If using Brave on a Mac, set up the Automator folder action (section 9).
8. Set up the Node harness and one Playwright script (section 10) before the first feature is added.
9. Keep `CHANGELOG.md`, `README.md`, `ROADMAP.md` current; bump the version constants every time.
