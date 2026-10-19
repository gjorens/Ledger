# myLedger design document

How myLedger is built and why. Written for the owner, for future work with Claude, and as a reference for other projects (see also `PATTERNS.md`, which has reusable code patterns, and `ROADMAP.md`, which lists planned work). Describes version 4.81. Keep this file updated when behaviour changes (the CHANGELOG records *what* changed; this file records *how it works now*).

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
| `ledger-test.json` | Sample data for testing (includes two vehicles and two sample portfolios). |
| `price-test.html` | One-off test page that checks which price sources a browser can reach (not part of the app). |

## 3. Data model (file format version 6)

```
{ version, createdAt, savedAt,
  settings:  { horizonMonths, catSort?, testData?, catTypes?, ruleSort?, showDeleted? },
  categories:[ names ],
  accounts:  [ { id, name, type, currency?, institution, number, startingBalance, openedOn, warnBelow?, closed? } ],
  recurring: [ rule ],
  reconciliations: { "accountId|YYYY-MM": { accountId, month, statementBalance, difference, itemCount, completedAt } },
  archives:  [ { file, through, count, from, to, at } ],
  records:   [ record ],
  vehicles:  [ { id, name, plate, vin, unit:'km'|'mi', annual, readings:[{date,odo}], schedule:{ "service name":{months,km} }, deleted? } ],
  serviceItems: [ names, kept A-Z ],
  services:  [ { id, vehicleId, date, odo, items:[names], by, cost, note, deleted, editedAt } ] }
```

**Record:** `{ id, accountId, text, category, amount, date, createdAt, deleted, deletedAt, reconciled?, reconAmt?, recurringId?, recurringDate?, transferId?, unposted?, editedAt? }`
- `amount` is a signed number. **Money out is negative in every account type**, including credit cards.
- `date` is a plain `YYYY-MM-DD` string (no time zones).
- `deleted` records stay in the file (so they can be restored) and no longer affect balances. Settings > Display can hide or show them.
- `reconciled` holds the statement month (`YYYY-MM`) the entry was ticked against; `reconAmt` the amount at that time.
- **Statement closing day.** An account may carry `stmtDay` (1-30; absent means the last day of the month). The reconciliation key and `reconciled` still use `YYYY-MM`, meaning the month the statement *closes* in. `stmtEnd(a, ym)` gives its closing date (clamped to short months), `stmtOf(a, date)` the statement a date belongs to, `stmtStart` the first day of its period, `lastStmtMonth(a)` the newest closed statement. Reconciliation, the missed-item date and archive eligibility use these instead of `monthEnd`. Changing the day after reconciling leaves earlier months labelled as they were.
- `transferId` ties the two legs of a transfer (one entry in each account).
- `recurringId` + `recurringDate` mean the entry was posted from a recurring rule; `recurringDate` is the *original* scheduled date, even if the entry was later moved. `unposted: true` marks a posted entry that was undone, so its occurrence returns to the projected list.

**Recurring rule:** `{ id, accountId, text, category, amount, frequency, startDate, endDate, skipped[], overrides{}, changes[], transferTo?, autoText?, createdAt, deleted?, deletedAt? }`
- `frequency`: `once` (a single projected item with an estimate: `startDate` = `endDate`; created by New item > Repeats > "One time, estimated amount"), `weekly`, `biweekly`, `monthly`, `monthlyNoDec` (every month except December), `quarterly`, `yearly`.
- `skipped[]`: original dates of occurrences that will not happen. Skipped dates are listed under the rule with a restore button (`reinstateOcc`); dates in months already archived are not offered, because archiving also adds the dates of archived posted entries to `skipped[]`.
- `overrides{ originalDate: { amount, date, editedAt, confirmedAt | estimate } }`: change one occurrence only. `estimate: true` means "my estimate for this month only"; `confirmedAt` means confirmed by the biller.
- `changes[ { from, amount } ]`: scheduled amount changes; the estimate in force on a date is the latest change on or before it.
- `transferTo`: makes the rule a scheduled transfer from `accountId` to that account; `autoText` means the description is generated ("Transfer: A to B") and is regenerated if accounts change.

**Versioning.** `DATA_VERSION = 5` (5 added the Autos data). Older files are upgraded in memory on open (`migrate`, which adds empty `vehicles`/`services` and the default `serviceItems`) and written as v5 on the next save. A file from a *newer* version is refused so an out-of-date cached app cannot damage it. `savedAt` inside the JSON (not the file's modification time) decides which copy is newer, because iCloud can change file times.

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

**Share-sheet saving (iPhone, iPad, Android; `exportFile`).** The file is passed to `navigator.share`; the operating system picks the starting folder and the page is told only that the sheet closed (or was cancelled), not where the file went. So after a successful share `confirmShared` asks the owner whether it was saved to the right folder and replaced; only a Yes sets `savedAt` / `baseSavedAt` and clears `dirty`. A No leaves the ledger flagged unsaved with a warning banner. The folder name shown in the reminders comes from the per-device setting `localStorage myLedger.saveLoc` (File window > Open and save); it is never written to the ledger file because each device differs (iCloud Drive, Google Drive, Dropbox...). The plain-download fallback (Mac Brave) is not asked, since the Automator folder action handles that file. The archive export has a similar share path that is not yet confirmed this way.

**Starting empty:** "Start a new ledger" creates an empty v5 file; one real ledger file is supported at a time (multiple ledgers are on the roadmap).

## 7. Backups and recovery

The app itself keeps **no versioned backups**. What exists:

- The overwritten-each-time **browser draft** (section 6).
- **Soft deletes**: deleted entries and rules can be restored until removed.
- **Unpost** returns a posted occurrence to the projection.
- **Archives** (section 8) are separate files, not backups.
- **iCloud Drive and Time Machine** keep older versions of `ledger.json` (Mac: File > Revert To > Browse All Versions). This is the practical recovery route.
- **Optional Mac Automator Folder Action** (script in `PATTERNS.md` section 9): moves downloaded ledger files from Downloads into the iCloud folder and keeps dated copies of each file it replaces in `myLedger/Backups` (newest 20 kept) and archive files in `myLedger/Archive`. It exists for Brave, which cannot write to the folder directly. It only runs on the Mac where it is installed, and only on files downloaded to Downloads, not on writes made directly by Chrome or Edge.

**One folder on a Mac.** The app does not read or use the Automator script's folder; the two are independent. The script's destination is hard-coded (`DEST` in the script, `iCloud Drive/<APP>`, here myLedger) and it only acts on files that arrive in `~/Downloads`. So: (a) with Brave, which cannot write to a folder, the user must open the file from the same folder the script writes to, or an old copy may be opened; (b) with Chrome or Edge folder mode the app writes directly into whichever folder was chosen and the script never runs, so those saves get no dated backups, and if the chosen folder differs from the script's, two copies can drift apart. Guidance for the owner: choose one folder, use it on every device, and point the script's `DEST` at it. This is explained in the in-app user guide (Saving).

Possible improvement: an automatic dated backup and a "Download backup" button (not built).

## 8. Archiving

File window > Archive old data moves **reconciled** entries up to a chosen month out of the ledger into a separate file, to keep the working file small. Balances do not change because the reconciled anchor carries them.

- **File name:** `ledger-archive-YYYY-MM.json` (`ledger-test-archive-YYYY-MM.json` for test data), where the month is the last month archived; `-2`, `-3` are added if the name was used before. Extension `.json`.
- **Contents:** `{ type:"myLedger-archive", version, createdAt, archivedThrough, testData, accounts, reconciliations, records }`.
- **Where:** chosen folder (Chrome/Edge), else a Save dialog; on Brave a download; on iOS the Share sheet.
- **Safety:** the app reads the file back to check the length (when it can write directly) or asks "Was the archive saved?" (download / share). Only then are records removed from the ledger. A note `{ file, through, count, from, to, at }` is added to `archives[]`. Recurring occurrences already posted and archived are added to the rule's `skipped[]` so they do not reappear as projections.
- The **archive viewer** opens archive files read-only (accounts, entries, search). The archive file is the only copy of those entries.

## 9. User interface

- **Layout:** account tabs, an account summary, the transaction table with month divider rows, projected rows (shaded) after the actual ones, collapsible cards for Recurring items, Reconciliation and Reports, and a gear for Settings.
- **Dialogs:** all confirmations, alerts and text prompts are in-app `<dialog>` elements (`ask`, `askText`), titled "myLedger", not browser `confirm()`/`prompt()` (which show "<website> says"). All callers are `async`. A test hook `__askHook` answers dialogs automatically in tests.
- **Money boxes:** digits only with the decimal point fixed (typing 1 2 5 0 gives 12.50); a minus sign or the ± button flips the sign; zero is allowed.
- **Header:** app name with the short version (`v4.58`; the build time is in Settings > About and the tooltip), the Save button (shown when a ledger is open), the **File** icon and the gear. The banner under it reports file actions taken or needed.
- **File window** (file icon): Open a file, Open any file (touch), choose folder / single file (Chrome, Edge), Start a new ledger, Archive old data and View an archive. All file-related actions live here.
- **Settings** (gear) is a full-width dialog with a sticky X button; sections (Display, Categories, About, User guide) open one at a time. The File window works the same way.
- **Pages:** a drop-down (`#viewSel`) replaces the "Accounts" heading and chooses the page: Accounts, Investments, Loans and mortgages, Autos (registry `VIEWS` in the code; choice kept in localStorage `myLedger.view`). Accounts is the original screen. The others are placeholder pages (`renderViewPage`) that list planned content and, where a page has `types`, the real accounts of those types. Design rule: a page is a *view* of the single data file. Investment, loan and mortgage accounts stay in `accounts` (types Investment, Loan, Line of credit), so the existing transfer machinery connects them to every other account; a page adds its own tools and, later, its own record types (positions, prices, vehicles, service history) in the same file.
- **Accounts:** a + button (add) and a pencil button (edit selected) beside the heading; tabs are a fixed-size grid (about 12 characters, longer names truncated with a tooltip).
- **Appearance:** Settings > Display > Appearance (Default / Light / Dark), stored per device in localStorage `myLedger.theme`, not in the ledger file. A small script in `<head>` sets `data-theme="light|dark"` on the html element before first paint; with no `data-theme` the CSS follows `prefers-color-scheme`. Every dark rule therefore exists twice: inside the media query scoped to `:root:not([data-theme])`, and as `:root[data-theme="dark"]`. Add any new dark-only rule in both places.
- **Colours** are CSS variables with a dark-mode set. Negative amounts are red; on liability accounts the displayed negative (payments) is red. Buttons are tonal grey with icons, the primary action is blue.
- **Balance graph:** full-width dialog; an SVG drawn at the dialog's pixel width and redrawn on resize, with a hover / touch read-out (guide line, dots and a box with the date, balance and the balance a year earlier). Points are weekly samples from `balanceSeries`.
- **iPhone layout:** `PHONE` (user agent contains iPhone or iPod; not iPad) adds the `phone` class to the page. The item list then hides the Category column, shows dates without the year (`fmtShort`; the month bars carry the year), tightens cell padding and stacks the action buttons so all columns fit without horizontal scrolling.
- **Touch:** on touch devices rows and buttons are larger.
- **Search:** searches all entries in the account (and optionally deleted ones), with a "show items exceeding a threshold" filter.
- **Reports:** income and expense by category and month, with a frozen category column, categories sorted, for one account or all.

**Autos page (v4.67).** `renderAutos` draws one vehicle at a time (a vehicle drop-down that is always shown, even for one vehicle, so the name is visible; + and pencil buttons). Nothing derived is stored; it is all computed from the three arrays:
- **Odometer model** (`odoModel`): all points = stand-alone `readings` plus the mileage on non-deleted service records. Distance per day is measured between the first point inside the last 730 days and the latest point once they are at least 90 days apart; otherwise the owner's `annual` estimate / 365; otherwise unknown. `estOdo` projects the current odometer from the latest point.
- **Due rows** (`dueRows`): per schedule item, find the latest service record containing it; next date = date + months; next distance = its mileage + km; the distance date = when the estimated odometer reaches that distance. Due = the earlier of the two. Status: overdue (date passed, or the estimated odometer is past the distance), soon (within 45 days), ok, unknown (no usable date), none (never recorded). Only the *next* occurrence is shown per item.
- **Service list:** `serviceItems` is shared by all vehicles, kept A-Z (`addCatalog`, case-insensitive duplicate check); names found on records are re-added on open. Schedules are per vehicle (`schedule`), edited in the vehicle dialog. New vehicles start with oil, tire rotation, brake inspection and brake pads/rotors defaults.
- **Managing the list:** `renameServiceItem(old, new)` rewrites `items` on every service record (including deleted ones), moves the vehicle schedule entry, and removes the old name from `serviceItems`; if the new name already exists (case-insensitive) it is a merge and the schedule already on the surviving name wins. `deleteServiceItem` refuses a name still referenced by any record (deleted ones too, so a restore cannot bring it back) or schedule. UI: `openSvcList` / `drawSvcList`; a + button (`addSvcListItem`) adds a name.
- **Costs:** `cost` is a plain number per visit; totals are summed in cents (`svcTotal`).
- Deleted records and removed vehicles are flagged (`deleted`), never dropped. Vehicles and services are not part of archiving and have no link to ledger entries (a service payment is entered in the account as usual).
- iPhone: the due table drops the "Last done" column (shown under the service name) and the history table puts the mileage under the date; a global `table{min-width:720px}` rule had to be overridden (`.autot{min-width:0}`).

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

### Browser and device matrix (draft: to be verified on real devices)

Written from general knowledge on 2026-10-12; cells marked (verify) are unchecked. Only Chromium (Playwright) has been used for testing. Also shown in the in-app user guide (Settings > User guide > Browsers and devices); keep the two in step.

**Mac**

| Browser | Saves into a folder | How you save | Install as app | Main weakness |
|---|---|---|---|---|
| Chrome | Yes, after picking the folder once (Reconnect after a restart) | Autosave every 30 s | Yes (Dock app) | Direct saves get no dated backups |
| Edge | Same as Chrome | Same | Yes | Same |
| Brave | No: File System Access is blocked by default (verify whether a setting enables it) | Save file downloads; then Open a file | Yes (verify) | No autosave; needs the Automator script for backups |
| Safari | No | Download, then Open a file | Add to Dock in recent macOS (verify) | No autosave; manual saves only |

**iPhone and iPad** (every browser uses Apple's WebKit engine, so capabilities match; in the EU, Apple's rules may allow other engines: verify)

| Browser | Saves into a folder | How you save | Install as app | Main weakness |
|---|---|---|---|---|
| Safari tab | No | Share sheet > Save to Files; the owner confirms and chooses Replace | Add to Home Screen (share menu) | The phone picks the folder; script-writable storage may be cleared after ~7 days unused (verify) |
| Home Screen app (from Safari) | No | Same | It is the installed app | Its storage is separate from the Safari tab's; use one or the other |
| Chrome on iOS | No | Same | Home Screen (verify) | Same limits as Safari |
| Brave on iOS | No | Same | Home Screen (verify) | Same limits as Safari |

**Common to phones and tablets:** no automatic dated backups; the app cannot see where a shared file went (section 6, share-sheet saving); one app, tab or icon should be used consistently because storage is not shared between them.

**To verify when time permits:** (1) can Brave on Mac enable the file-system API in a setting, and does it offer Install app; (2) can Safari on macOS add the site to the Dock and does it keep a folder; (3) which iOS browsers offer Add to Home Screen; (4) whether data in the Safari tab and the Home Screen app are separate on the current iOS; (5) whether stored drafts survive about a week of non-use in a Safari tab versus the Home Screen app. Update both tables with what you find and remove the "(verify)" marks.

## Investments (v4.77)

- **Data (format 6):** `portfolios[{id, institution, name, accounts[{id,name,type}], deleted?}]` (the institution is a text label on the portfolio and the page groups by it), `invDays{portfolioId:{'YYYY-MM-DD':{v?, dep?, note?}}}` (`v` = portfolio value in CAD, `dep` = deposit (+) or withdrawal (-)), `invStart{portfolioId:{'YYYY': value}}` (optional January 1 override). Setting `settings.invCal`: `either` (default), `tsx`, `nyse`, `both`.
- **Net vs gross.** Change on a day = value - previous value - deposits since that value (a deposit-only day carries into the next value day). Gross YTD % = (value - Jan 1 value) / Jan 1 value; net YTD % = cumulative net change / Jan 1 value. % week = the week's net change / the value at the end of the previous week (or the Jan 1 value). Average per day = YTD net change / trading days so far; projected year end = value + average x remaining trading days. These reproduce the owner's sheet.
- **January 1 value:** `invStart` if set, otherwise the last value entered before January 1; each year therefore carries forward, and the owner can correct it against the year-end statement.
- **Combined view:** the portfolios' values are added (each carried forward from its last entry), so the daily change is the sum; a deposit in one portfolio is kept out of the combined change.
- **Trading days:** `holidays(year)` returns TSX and US closures by rule (Family Day, Good Friday, Victoria Day, Canada Day, Civic Holiday, Labour Day, Thanksgiving, Christmas and Boxing Day with Monday/Tuesday observance; MLK, Presidents, Memorial, Juneteenth, Independence Day, Labor Day, Thanksgiving, Christmas with Friday/Monday observance; a Saturday New Year is not observed by the NYSE). Special one-off closures are not modelled.
- **CSV import:** `parseInvCsv(text)` reads the owner's yearly sheets (year in cell A1, value column = the one before "Change", a column whose header starts with "Deposit", a "Forward" row, dates as "Jan 02"); rows with a "total" label are not deposits; extra columns after the deposits column become optional extra portfolios. Verified against the 2024 sheet in the test script.
- **Not built yet:** prices, index columns, holdings and share-count true-ups, dividend projection, transfers to bank accounts (see ROADMAP).

## 12. Testing

- A Node `vm` harness (`t4.js`, kept outside the repository) loads the script against a stub DOM and runs the logic and form handlers (about everything that does not need real layout). Run with `TZ=America/Toronto node t4.js`; it must print `ALL PASSED`.
- Playwright + Chromium drive the real page from a local web server for screenshots and dialog behaviour, including a dark-mode check.
- Not tested: Brave, iPhone and iPad hardware, File System Access API against a real iCloud folder. These are the usual places for surprises.

## 13. Security and privacy

**Where data lives.** Ledger data exists only in the user's own files (iCloud Drive folder, Downloads, Time Machine) and in the browser's IndexedDB draft on each device. It is never sent to a server. GitHub holds only the app code. No analytics, accounts or third-party libraries.

**What the data contains, and its sensitivity** (assessment by the owner, Oct 2026):
- Account numbers: the most sensitive item. Roughly as exposed as the numbers printed on a paper cheque.
- Balances and transaction history: could help someone impersonate the owner or target them, and are private in themselves.
- Not stored: online banking passwords, PINs, SIN / SSN, card security codes.

**Current protection.** None inside the app (no lock, no encryption). Protection comes from the device (passcode, Face ID, FileVault) and from iCloud account security. Anyone using an unlocked device can open the app and the saved file. See section 15.

**Planned (see ROADMAP.md, Extra security):** (1) an app lock screen; (2) optional encryption of the file and draft; (3) a privacy screen.

**Design notes for optional encryption (not built):**
- *Layer in the app, not the disk.* Use the browser's Web Crypto API (AES-GCM, random IV per save, key from the passphrase via PBKDF2 with a high iteration count, or Argon2 if a library is accepted). The file in iCloud and the IndexedDB draft are then ciphertext, so Apple, Microsoft and GitHub never hold a key. Keep `version` and a small plain header (format, salt, iteration count, wrapped keys) readable so the app can recognise an encrypted file.
- *Envelope encryption.* A random data key encrypts the ledger. The data key is stored wrapped twice: once by a key derived from the passphrase, once by a generated **recovery key**. Either unlocks the file, and the passphrase can change without re-encrypting the data.
- *Lost passphrase is the main risk.* Mitigations: the recovery key is shown once at setup and must be stored offline (printed, password manager or safe); setup forces a confirmation (re-enter part of the key); an unencrypted export stays available; encryption is optional with a plain "no recovery without the key" warning; optionally split the recovery key between trusted people (Shamir secret sharing); a hint is only a weak help. Face ID / Keychain can unlock on Apple devices as a convenience, but never as the only copy.
- *Lock and encryption are separate layers.* A lock screen stops a casual onlooker but does not protect the stored file or draft. Encryption protects the stored data, not a device that is unlocked and running. Face ID from a plain web page is limited (WebAuthn platform authenticator at best); full biometric support is more likely in the native iOS version.
- *Side effects to handle:* the Automator backup script only moves files, so it works with encrypted ones; the 45-second `refreshFromFile` check and the `savedAt` comparison need the header to carry `savedAt` in the clear (or a decrypt step); archives must be encrypted with the same key; search and reports work on decrypted data in memory only.

**Apple and Microsoft disk encryption (background, not legal or security advice; verify current documentation).** No public evidence of an intentional back door in FileVault / APFS or BitLocker; the ciphers (AES-XTS) are standard. The practical exposure is **key escrow**: consumer BitLocker recovery keys are normally backed up to the user's Microsoft account and have been produced under legal process; FileVault can optionally unlock through iCloud; iCloud data is readable by Apple under standard data protection, but end-to-end encrypted with **Advanced Data Protection** turned on. Most real-world compromise comes from implementation bugs, malware, or someone who knows the device passcode. This is why in-app encryption is preferred for a file stored in iCloud.

## 14. Decisions and rationale

- **JSON file, not a database:** a year is about 500 records, so a single readable file is simple, portable and easy to back up.
- **Projections are computed, not stored:** changing the look-ahead or a rule never leaves stale data.
- **Soft delete:** mistakes are recoverable; the Display setting keeps the list uncluttered.
- **Same sign convention in all accounts, display flip for liabilities:** one rule for storage and transfers, familiar numbers on screen.
- **`savedAt` in the file:** reliable across iCloud and devices, unlike file times.
- **Custom dialogs, one file, no libraries:** consistent look, easy to host, easy to reuse.

## 15. Known limits

- One real ledger file at a time.
- No automatic versioned backups (section 7).
- No app lock or encryption; anyone using an unlocked device can open the app and a saved file (section 13).
- No live bank feeds, investments, multi-currency conversion or dividends (roadmap).
- No cross-device real-time sync; sharing is by the iCloud file plus the 45-second check.
