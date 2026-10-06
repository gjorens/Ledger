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
| Safari (Mac) | Changes are kept in the browser. **Download file** saves `ledger.json`; move it into your iCloud folder and replace the old one. |
| Safari on iPhone / iPad | **Open file** loads your ledger from Files. **Share to Files** saves it back. |

Safety features: unsaved-changes banner, warning before closing with unsaved changes, a browser-side draft copy after every change, and a check that stops you overwriting a file that was changed from another device (`savedAt` comparison).

Tip: Safari's download setting "Ask for each download" avoids a pile of `ledger (1).json` copies. On a Mac you can also script moving the download into iCloud with Automator.

## Features

**Accounts**
- Any number of accounts, each with a type (chequing, savings, credit card, line of credit, loan, investment, cash, other), institution, optional name and optional account number (only the last four digits are ever shown).
- Credit cards, lines of credit and loans are **displayed the way the bank shows them**: amount owing is positive, a charge is positive, a payment is negative. Column headings and form labels change to match ("Owing", "Charges / payments"). The stored numbers are the same as for every other account.
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

**Reports**
- By category, or by category and month, for this month, last month, this year, last year, the last 12 months, all time or a custom range, for one account or all. Transfers, deleted entries and projections are excluded.

## Data file format

A JSON file, currently `version: 4`:

```
{
  version, createdAt, savedAt,
  settings: { horizonMonths },
  categories: [ ... ],
  accounts:   [ { id, name, type, institution, number, startingBalance, openedOn, warnBelow, closed } ],
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

## Updating the app

1. Replace `index.html` in this repository (Add file, Upload files, Commit).
2. Wait a minute for GitHub Pages to publish.
3. Reload the page on each device. The version and build time are shown next to the name at the top of the page.

All devices should run the same version, since older versions do not understand newer files.

## Known limits

- Categories can be added but not yet renamed or deleted.
- Ticks made during a reconciliation are not stored until you press **Mark month reconciled**.
- Reports use the stored signs, so spending always counts as "out", including on credit cards.
- Safari on Mac and iOS cannot autosave to a file; use Download or Share.

## Development notes

There is no build step. Open `index.html` in a browser to run it. The logic was checked with a Node `vm` harness against generated test data (three accounts, about 650 entries over three years, recurring items, transfers and reconciliations). Test data and scripts are not part of this repository.
