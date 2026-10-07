# myLedger

A personal bank-transaction ledger that runs entirely in your browser. One HTML file, no server, no account, no App Store. Your data lives in a single JSON file that you keep in iCloud Drive (or anywhere else), so the same ledger works on Mac, iPad and iPhone.

Live app: https://gjorens.github.io/Ledger/

## How it works

- `index.html` is the whole app (HTML, CSS and JavaScript in one file), served by GitHub Pages.
- Your transactions are **never sent anywhere**. The app reads and writes a JSON file you choose, plus a safety copy in the browser's own storage.
- Because it is a web page, it can be used on any device with a modern browser.

## Saving your data

| Browser | What happens |
|---|---|
| Chrome / Edge on Mac | Choose a save file once. The app then **autosaves** to it every 30 seconds, when you switch away, and when you close the page. After a restart, a **Reconnect** button re-grants access to the file. |
| Brave | Brave switches the file-saving feature off by default, so it behaves like Safari (**Save file**). To get autosave, open `brave://flags/#file-system-access-api`, set it to Enabled and relaunch. |
| Safari (Mac) | Changes are kept in the browser. **Save file** puts `ledger.json` in your Downloads folder; move it into your iCloud folder and replace the old one. |
| Safari on iPhone / iPad | **Open file** loads your ledger from Files. **Save to Files** saves it back: choose the folder in the Files sheet and tap Replace. |

Safety features: unsaved-changes banner, warning before closing with unsaved changes, a browser-side draft copy after every change, and a check that stops you overwriting a file that was changed from another device (`savedAt` comparison).

Tip: Safari's setting "Ask for each download" avoids a pile of `ledger (1).json` copies. On a Mac you can also script moving the download into iCloud with Automator.

## Settings (the gear icon)

- **Test data.** There is no switch. A file is test data when it says so inside itself (`settings.testData: true`, as in `ledger-test.json`). Opening it shows an orange stripe and a **TEST DATA** tag on any device. A file whose name contains "test" but is not marked asks you to confirm once, and is then marked. Opening any other file shows no stripe.
- **Archive old data.** Moves old reconciled entries (and deleted ones) into a separate `ledger-archive-YYYY-MM.json` so the ledger stays small; balances do not change. Entries of the latest reconciled month and anything unreconciled always stay. **View an archive...** opens one or more archive files read-only (account picker, search, running balances). The archive files are the only copy of those entries, so keep them with your ledger. The ledger records which archives exist (`archives` in the data file).
- **About myLedger.** The version and build time, whether test data is open, and where the app is saving.
- **User guide.** A short built-in guide to every feature.
- **Categories.** Add, rename, delete (entries become uncategorised) or merge categories (renaming to an existing name offers to merge). Choose the dropdown order: your own order (with up/down arrows), A–Z, or most used first.
- **Where your ledger file is kept.** Any folder you like: iCloud Drive, Dropbox, OneDrive or a local folder. If the standard file name is not there, the app uses a look-alike (for example `ledger-test (1).json`; test files are only matched when test data is open), and **Open a file...** in Settings lets you pick one directly. On Chrome, Edge and Brave (Mac) choose a folder, and optionally type a name to create a new folder inside it. The app saves the ledger file in that folder and autosaves there. The page always shows the folder name and file name it is saving to. Browsers only reveal the folder's name, not its full path. On Safari and iOS the browser cannot remember a folder, so you pick the folder each time in the Files sheet when you tap **Save to Files** or **Save file**.

## Features

**Accounts**
- Any number of accounts, each with a type (chequing, savings, credit card, line of credit, loan, investment, cash, other), institution, optional name and optional account number (only the last four digits are ever shown).
- Credit cards, lines of credit and loans are **displayed the way the bank shows them**: amount owing is positive, a charge is positive, a payment is negative. Column headings and form labels change to match ("Owing", "Charges / payments"). The stored numbers are the same as for every other account.
- Each account can have a **currency** (CAD, USD, EUR, GBP, MXN and a dozen more, stored as `currency` on the account). Balances, warnings and the reconciliation summary show the symbol; list rows stay plain numbers. There is **no conversion**: transferring between accounts in different currencies asks you to confirm (the same number goes into both), and an all-accounts report across several currencies carries a warning that totals are not meaningful.
- Transfers are described automatically as `From → To` (read-only; follows account renames) and take positive amounts only: a swap button between the two account boxes reverses the direction.
- Transfers between accounts create linked, offsetting entries in both accounts. Deleting or editing one side keeps the other in step.

**Entering items** (one **+ New item** popup for everything)
- Payment or deposit, or a transfer between accounts.
- One-off or repeating (weekly, every 2 weeks, monthly, every 3 months, yearly, with an optional last date). Repeating items, including recurring transfers, become **recurring items**.
- Category dropdown (with "add new"), and an autocomplete list of earlier descriptions that fills in the category and amount.
- Amount boxes use a **fixed decimal point**: type the digits only (1, 2, 5, 0 gives 12.50). The **±** button switches the sign.

**Transactions list**
- Sorted by transaction date, credits before debits, then description.
- Running balance, soft delete (deleted entries stay in the file and stop affecting balances), edit of any entry.
- Opens scrolled to the latest past transaction.

**Recurring items and projections**
- Projected entries appear after today for the look-ahead period you choose (3, 6, 12 months...).
- The amount is an **estimate** (italic). When the biller confirms the real amount, **Edit** it and it shows in **bold**. **Post** turns an occurrence into a real entry (also bold); **Skip** drops one occurrence; **Stop** ends the item.
- **Change amount** schedules a future change to the estimate from a given date.
- Renaming an account updates the default description ("Transfer to ...") on its recurring transfers.

**Monthly reconciliation** (per account)
- Compare the month to your statement: tick items, enter the closing balance, see the difference.
- Edit unticked items, add a missed item (with the same New item popup), undo the last reconciliation. Only months that still need reconciling are listed.
- **After the first reconciled month, balances are calculated from the latest reconciled statement balance.** The opening balance is only the starting point until then.

**Low-balance warning**
- Optional per account: "warn me if the balance is projected to drop below ..." (for a card: "amount owing goes above ...").
- A popup appears when a change makes a future balance (real future-dated entries plus projections) fall below the level, and the account's tab gets a **red ring** for as long as that is true.

**Balance graph**
- The **Balance graph** button (next to Look ahead) shows a compact line graph of the 12 months ending at the look-ahead date, with the same 12 months a year earlier as a second line. The projected part is dashed, and the account's warning level is drawn as a red dotted line.

**Reports**
- By category, or by category and month, for this month, last month, this year, last year, the last 12 months, all time or a custom range, for one account or all. Transfers, deleted entries and projections are excluded.

## Data file format

A JSON file, currently `version: 4`:

```
{
  version, createdAt, savedAt,
  settings: { horizonMonths, catSort?, testData? },
  categories: [ ... ],
  archives:   [ { file, through, count, from, to, at } ],
  accounts:   [ { id, name, type, currency?, institution, number, startingBalance, openedOn, warnBelow, closed } ],
  recurring:  [ { id, accountId, text, category, amount, frequency, startDate, endDate,
                  skipped[], overrides{}, changes[], transferTo?, autoText? } ],
  reconciliations: { "accountId|YYYY-MM": { statementBalance, difference, itemCount, completedAt } },
  records:    [ { id, accountId, text, category, amount, date, createdAt, deleted, deletedAt,
                  reconciled?, reconAmt?, recurringId?, recurringDate?, transferId? } ]
}
```

- Amounts are plain signed numbers: money out is negative in every account, including credit cards.
- Projections are computed on the fly and are not stored.
- Older files (v1 to v3) are upgraded in memory when opened and saved as v4 on the next save. Files from a **newer** version than the app understands are refused, so a stale cached copy of the app cannot damage them.
- With roughly 500 records a year the file stays small, so a single JSON file is a good fit.

## Installing it as an app

The app is a web page, so "installing" means giving it an icon and its own window. All of these files need to sit in the repository root: `index.html`, `manifest.webmanifest`, `sw.js`, `apple-touch-icon.png`, `icon-192.png`, `icon-512.png`, `favicon.png` and `install-mac.command`.

**iPhone and iPad** (there is no script on iOS; Apple only allows this through Safari)
1. Open https://gjorens.github.io/Ledger/ in **Safari**.
2. Tap the Share button, then **Add to Home Screen**, then **Add**.
3. Open myLedger from the home screen icon. It runs full screen without Safari's address bar, and opens offline once it has been loaded once.
4. Tap **Open file** and pick `ledger.json` from iCloud Drive > myLedger. Use **Share to Files** to save it back.

**Mac** (installer script)
1. Download `install-mac.command` from this repository, then right-click it and choose **Open** (the first time only, because macOS is cautious about downloaded scripts). Or paste this into Terminal:
   `curl -fsSL https://raw.githubusercontent.com/gjorens/Ledger/main/install-mac.command | bash`
2. The script creates an **iCloud Drive > myLedger** folder for your data, keeps an offline copy of the app in `~/Library/Application Support/myLedger`, and builds **~/Applications/myLedger.app** with the myLedger icon. Drag it to the Dock.
3. The app opens in its own window using Chrome, Edge or Brave if installed (these can autosave), otherwise your default browser. With no internet it opens the offline copy.
4. First run: click **Choose save file...** and save `ledger.json` in the iCloud Drive myLedger folder.
5. To remove it: `bash install-mac.command --uninstall`. Your `ledger.json` is never touched.

Alternatively, in Chrome or Edge open the app and use the browser menu > **Install myLedger** (or Save and Share > Install page as app).

## Updating the app

See [CHANGELOG.md](CHANGELOG.md) for the version history; add an entry at the top each time you publish a new `index.html`.

1. Replace `index.html` in this repository (Add file, Upload files, Commit).
2. Wait a minute for GitHub Pages to publish.
3. Reload the page on each device. The version and build time are shown next to the name at the top of the page.

All devices should run the same version, since older versions do not understand newer files.

## Known limits

- Ticks made during a reconciliation are not stored until you press **Mark month reconciled**.
- Reports use the stored signs, so spending always counts as "out", including on credit cards.
- Safari on Mac and iOS cannot autosave to a file; use Save file (Mac) or Save to Files (iPhone/iPad).

## Development notes

There is no build step. Open `index.html` in a browser to run it. The logic was checked with a Node `vm` harness against generated test data (three accounts, about 650 entries over three years, recurring items, transfers and reconciliations). Test data and scripts are not part of this repository.
