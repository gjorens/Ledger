# myLedger design document

How myLedger is built and why. Written for the owner, for future work with Claude, and as a reference for other projects (see also `PATTERNS.md`, which has reusable code patterns, and `ROADMAP.md`, which lists planned work). Describes version 4.54. Keep this file updated when behaviour changes (the CHANGELOG records *what* changed; this file records *how it works now*).

## 1. Purpose and constraints

A personal bank ledger: record transactions, project the balance forward from recurring items, reconcile against statements, report by category.

Design constraints that explain most decisions:
- **One file, no server.** The whole app is a single `index.html` (HTML, CSS and vanilla JavaScript, no libraries or build step). It is hosted as static files on GitHub Pages and updated by uploading the file.
- **The data is the user's file.** Ledger data lives in one JSON file in iCloud Drive (folder `myLedger`). The app never sends data anywhere. GitHub holds only the app code.
- **Works on Mac, iPad and iPhone.** Chrome and Edge can read and write the file directly (File System Access API). Brave (disabled by default), Safari and iOS cannot, so a manual open / save-to-Files path exists for them.
- **Safe by default.** Silent browser-side draft copy, conflict checks before overwriting, in-app confirmation for risky actions, nothing leaves the ledger until it is known to be saved elsewhere (archives).

## 2. Files in the repository

| File | Role |
|---|---|
| `index.html` | The whole app. Version and build time are constants `APP_VERSION` / `APP_BUILD`, shown beside the app name and in Settings > About. |
| `sw.js` | Service worker (offline use, see section 11). |
| `manifest.webmanifest`, `*.png` icons | PWA install information. |
| `install-mac.command` | Optional Mac installer: creates the iCloud `myLedger` folder and a small `myLedger.app` launcher that opens the site in its own window and falls back to an offline copy. |
| `README.md` | User-facing description. |
| `CHANGELOG.md` | Version history, newest first. |
| `ROADMAP.md` | Planned work (multiple ledgers, investments, app lock, native iOS). |
| `PATTERNS.md` | Reuse guide for other projects. |
| `DESIGN.md` | This file. |
| `ledger-test.json` | Sample data for testing. |

## 3. Data model (file format version 4)

```
{ version, createdAt, savedAt,
  settings:  { horizonMonths, catSort?, testData?, catTypes?, ruleSort?, showDeleted? },
  categories:[ names ],
  accounts:  [ { id, name, type, currency?, institution, number, startingBalance, openedOn, warnBelow?, closed? } ],
  recurring: [ rule ],
  reconciliations: { "accountId|YYYY-MM": { accountId, month, statementBalance, difference, itemCount, completedAt } },
  archives:  [ { file, through, count, from, to, at } ],
  records:   [ record ] }
```

**Record:** `{ id, accountId, text, category, amount, date, createdAt, deleted, deletedAt, reconciled?, reconAmt?, recurringId?, recurringDate?, transferId?, unposted?, editedAt? }`
- `amount` is a signed number. **Money out is negative in every account type**, including credit cards.
- `date` is a plain `YYYY-MM-DD` string (no time zones).
- `deleted` records stay in the file (so they can be restored) and no longer affect balances. Settings > Display can hide or show them.
- `reconciled` holds the statement month (`YYYY-MM`) the entry was ticked against; `reconAmt` the amount at that time.
- `transferId` ties the two legs of a transfer (one entry in each account).
- `recurringId` + `recurringDate` mean the entry was posted from a recurring rule; `recurringDate` is the *original* scheduled date, even if the entry was later moved. `unposted: true` marks a posted entry that was undone, so its occurrence returns to the projected list.

**Recurring rule:** `{ id, accountId, text, category, amount, frequency, startDate, endDate, skipped[], overrides{}, changes[], transferTo?, autoText?, createdAt, deleted?, deletedAt? }`
- `frequency`: `weekly`, `biweekly`, `monthly`, `monthlyNoDec` (every month except December), `quarterly`, `yearly`.
- `skipped[]`: original dates of occurrences that will not happen.
- `overrides{ originalDate: { amount, date, editedAt, confirmedAt | estimate } }`: change one occurrence only. `estimate: true` means "my estimate for this month only"; `confirmedAt` means confirmed by the biller.
- `changes[ { from, amount } ]`: scheduled amount changes; the estimate in force on a date is the latest change on or before it.
- `transferTo`: makes the rule a scheduled transfer from `accountId` to that account; `autoText` means the description is generated ("Transfer: A to B") and is regenerated if accounts change.

**Versioning.** `DATA_VERSION = 4`. Older files are upgraded in memory on open (`migrate`) and written as v4 on the next save. A file from a *newer* version is refused so an out-of-date cached app cannot damage it. `savedAt` inside the JSON (not the file's modification time) decides which copy is newer, because iCloud can change file times.

## 4. How the numbers work

- **Balance** of an account = opening (anchor) balance + the sum of all non-deleted records for it, ordered by date, then credits before debits, then description.
- **Reconciled anchor.** The anchor is the latest reconciled statement's closing balance minus the `reconAmt` of every entry ticked against statements (those entries are then added back in by the normal sum). With no reconciliation the anchor is the account's `startingBalance`.
- **Credit cards, lines of credit and loans** (liability accounts) are stored with the same signs as everything else (what you owe is negative) but are *displayed* the way the bank shows them: amount owing positive, a charge positive, a payment negative. The helpers `isLiab`, `sgn`, `dfmt`, `dmoney` do the flip, and forms use "+ = charge, - = payment". On liability accounts negative displayed amounts (payments and credits) are red.
- **Projection.** Recurring rules are expanded on the fly for `horizonMonths` ahead (3, 6, 12 and so on) and are **never stored** as records. Changing the look-ahead therefore deletes nothing. An occurrence becomes a real record only when you **Post** it, or when its date passes and it is posted. Posting adds a record with `recurringId`/`recurringDate`; the pair `rule|originalDate` then counts as "handled" and no projection appears for it.
- **Occurrence changes** (`overrides`) and **scheduled changes** (`changes`) let one month differ without touching the others.
- **Finished rules** (end date passed and everything posted) move to a "finished / stopped" group and can be restored or cleaned up.
- **Low-balance warnings:** a per-account `warnBelow` level (for liability accounts, a level of amount owing); with none set, ordinary accounts warn below zero and liability accounts do not warn.

## 5. Categories and checks

- Categories are a list of names; each has a **type** (`in`, `out`, `any`) in `settings.catTypes`, otherwise guessed from the name (for example anything with "tax" is expense, "income" or "salary" is income).
- `amountSane` asks for confirmation when an amount goes the opposite way to its category (a positive expense or a negative income). It runs when adding an item and when editing the amount or category.
- Renaming a category updates every record and rule; renaming it to an existing name merges them; deleting leaves the entries uncategorised.

## 6. Storing and saving data

Three places hold data:

1. **The ledger file** (iCloud Drive `myLedger/ledger.json`; `ledger-test.json` for test data). This is the real data.
2. **A browser draft** in IndexedDB (database `ledger-test`, store `kv`, keys `draft`, `handle`, `dir`). The full ledger is written there after **every change** and when the page is hidden, blurred or closed. It is one copy that is overwritten each time. It exists to survive a crash or a lost file connection, not to give history. The app asks the browser for persistent storage.
3. Small preferences in localStorage (selected account, folder label).

**With the File System Access API (Chrome, Edge):**
- The app keeps a file handle (and optionally a folder handle) in IndexedDB. The browser may require re-approval of folder access; the app shows "Folder access needs re-approval" and the Save button re-asks.
- **Autosave** runs every 30 seconds while the page is visible and there are unsaved changes, and again when the page is hidden. A save rewrites the whole file with a new `savedAt`.
- **Conflict check before saving:** if the file on disk has a `savedAt` newer than the one this copy was loaded from, autosave stops and the user must choose (overwrite, or open the newer file).
- **Polling:** every 45 seconds, and whenever the app becomes visible, `refreshFromFile` reads the file. If it is newer and there are no unsaved changes, it is loaded automatically with a message. If there are unsaved changes, a banner offers "load newer file" or "keep mine".

**Without it (Brave, Safari, iPhone, iPad):** the file is opened through a file picker and saved with **Save file** (Mac download) or **Save to Files** (Web Share sheet on touch devices). Changes are unsaved until the user does this, and the app warns when closing with unsaved changes.

**Test data** is flagged `settings.testData` (orange stripe, "TEST DATA" tag, title "myLedger (test)"). Real and test data use different standard file names so they cannot be confused.

**Starting empty:** "Start a new ledger" creates an empty v4 file; one real ledger file is supported at a time (multiple ledgers are on the roadmap).

## 7. Backups and recovery

The app itself keeps **no versioned backups**. What exists:

- The overwritten-each-time **browser draft** (section 6).
- **Soft deletes**: deleted entries and rules can be restored until removed.
- **Unpost** returns a posted occurrence to the projection.
- **Archives** (section 8) are separate files, not backups.
- **iCloud Drive and Time Machine** keep older versions of `ledger.json` (Mac: File > Revert To > Browse All Versions). This is the practical recovery route.
- **Optional Mac Automator Folder Action** (script in `PATTERNS.md` section 9): moves downloaded ledger files from Downloads into the iCloud folder and keeps dated copies of each file it replaces in `myLedger/Backups` (newest 20 kept) and archive files in `myLedger/Archive`. It exists for Brave, which cannot write to the folder directly. It only runs on the Mac where it is installed, and only on files downloaded to Downloads, not on writes made directly by Chrome or Edge.

Possible improvement: an automatic dated backup and a "Download backup" button (not built).

## 8. Archiving

Settings > Archive old data moves **reconciled** entries up to a chosen month out of the ledger into a separate file, to keep the working file small. Balances do not change because the reconciled anchor carries them.

- **File name:** `ledger-archive-YYYY-MM.json` (`ledger-test-archive-YYYY-MM.json` for test data), where the month is the last month archived; `-2`, `-3` are added if the name was used before. Extension `.json`.
- **Contents:** `{ type:"myLedger-archive", version, createdAt, archivedThrough, testData, accounts, reconciliations, records }`.
- **Where:** chosen folder (Chrome/Edge), else a Save dialog; on Brave a download; on iOS the Share sheet.
- **Safety:** the app reads the file back to check the length (when it can write directly) or asks "Was the archive saved?" (download / share). Only then are records removed from the ledger. A note `{ file, through, count, from, to, at }` is added to `archives[]`. Recurring occurrences already posted and archived are added to the rule's `skipped[]` so they do not reappear as projections.
- The **archive viewer** opens archive files read-only (accounts, entries, search). The archive file is the only copy of those entries.

## 9. User interface

- **Layout:** account tabs, an account summary, the transaction table with month divider rows, projected rows (shaded) after the actual ones, collapsible cards for Recurring items, Reconciliation and Reports, and a gear for Settings.
- **Dialogs:** all confirmations, alerts and text prompts are in-app `<dialog>` elements (`ask`, `askText`), titled "myLedger", not browser `confirm()`/`prompt()` (which show "<website> says"). All callers are `async`. A test hook `__askHook` answers dialogs automatically in tests.
- **Money boxes:** digits only with the decimal point fixed (typing 1 2 5 0 gives 12.50); a minus sign or the ± button flips the sign; zero is allowed.
- **Settings** is a full-width dialog with a sticky X button; sections (File location, Display, Categories, Archive old data, About, User guide) open one at a time.
- **Colours** are CSS variables with a dark-mode set. Negative amounts are red; on liability accounts the displayed negative (payments) is red. Buttons are tonal grey with icons, the primary action is blue.
- **Touch:** on touch devices rows and buttons are larger.
- **Search:** searches all entries in the account (and optionally deleted ones), with a "show items exceeding a threshold" filter.
- **Reports:** income and expense by category and month, with a frozen category column, categories sorted, for one account or all.

## 10. Adding, editing and repeating items

- **New item:** payment/deposit or transfer (two linked entries); "Repeats" choice makes a recurring rule instead of an entry; "Add another" keeps the dialog open, keeps the date, and resets Repeats to "Does not repeat"; the last date of a repeating item cannot be before the first.
- **Edit entry:** date, amount, description, category; for a transfer the from and to accounts and direction. An entry that is not part of a recurring rule can be made repeating with the **Repeats** choice: it becomes the first posted occurrence of a new rule.
- **Edit recurring item** (all fields; the first date is locked once any occurrence is posted), **change amount from a date**, **edit this occurrence** (estimate or confirmed), **skip**, **post now**, **unpost**.
- Editing a reconciled entry's date or amount asks first and clears its reconciled mark.

## 11. PWA, service worker and hosting

- Hosted on GitHub Pages (`https://gjorens.github.io/Ledger/`). Updating means uploading changed files to the repository.
- `manifest.webmanifest` + icons let the site be installed (Add to Home Screen from **Safari** on iPhone / iPad; install from Chrome/Edge on a Mac; `install-mac.command` for a Dock app).
- `sw.js` is **network first**: when online, files are fetched fresh and the cache is refreshed; when offline, the cached copy is served. A new version therefore appears on the next load when online. It caches only the app files, never ledger data.
- Third-party libraries: none.

## 12. Testing

- A Node `vm` harness (`t4.js`, kept outside the repository) loads the script against a stub DOM and runs the logic and form handlers (about everything that does not need real layout). Run with `TZ=America/Toronto node t4.js`; it must print `ALL PASSED`.
- Playwright + Chromium drive the real page from a local web server for screenshots and dialog behaviour, including a dark-mode check.
- Not tested: Brave, iPhone and iPad hardware, File System Access API against a real iCloud folder. These are the usual places for surprises.

## 13. Decisions and rationale

- **JSON file, not a database:** a year is about 500 records, so a single readable file is simple, portable and easy to back up.
- **Projections are computed, not stored:** changing the look-ahead or a rule never leaves stale data.
- **Soft delete:** mistakes are recoverable; the Display setting keeps the list uncluttered.
- **Same sign convention in all accounts, display flip for liabilities:** one rule for storage and transfers, familiar numbers on screen.
- **`savedAt` in the file:** reliable across iCloud and devices, unlike file times.
- **Custom dialogs, one file, no libraries:** consistent look, easy to host, easy to reuse.

## 14. Known limits

- One real ledger file at a time.
- No automatic versioned backups (section 7).
- No app lock or encryption; anyone using an unlocked device can open the app and a saved file.
- No live bank feeds, investments, multi-currency conversion or dividends (roadmap).
- No cross-device real-time sync; sharing is by the iCloud file plus the 45-second check.
