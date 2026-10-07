# myLedger version history

The version number and build time are shown next to the app name at the top of the page. Newest first.

Data file format: `version: 4` since v4.0. Files from a newer format than the app understands are refused; older files are upgraded when opened.

## 4.30 (2026-10-08)
- Transfers: standard description "From account → To account" (read-only, follows account renames; older "Transfer to X" / "Payment from X" wording is converted when a file is opened).
- Transfer amounts are positive only; a double-arrow button between the From and To boxes swaps the direction. The Edit dialog for a transfer has no description or sign.

## 4.29 (2026-10-08)
- Row buttons are icons with tooltips: edit (pencil), delete (trash), post (check), skip, unpost, stop, remove; also in the recurring list, reconciliation and category settings.
- **Undelete:** a deleted entry shows a round-arrow button that brings it back (a transfer is restored on both sides).

## 4.28 (2026-10-07)
- Account tabs are rectangular with a darker border and a light-blue fill; the selected tab is strong blue.
- Buttons are colored (teal for everyday actions, blue for primary, red for destructive), with separate dark-mode colors.

## 4.27
- Red warning ring on an account tab is much wider.
- The "Lowest projected" box gets a red border when the account crosses its warning level.
- With no warning level set, an ordinary account is flagged when it is projected to go below zero (no pop-ups, just the ring and border).
- Text in the summary box is centered both ways.

## 4.26
- Balance box removed from the Accounts section (it could be misleading before reconciliation); "Lowest projected" takes its place and the mini graph is larger.
- **+ New item** moved to the left of Look ahead. **Add account** / **Edit account** stacked at the right.

## 4.25
- Very small balance graph in the Accounts section (solid = so far, dashed = projected, grey = a year earlier). The Balance graph button still opens the full popup.

## 4.24
- Currency for each account (CAD, USD, EUR, GBP, MXN and others). Symbols on balances and warnings; confirmation for transfers between currencies; warning on mixed-currency reports. No conversion.

## 4.23
- Posting an estimated item uses the fixed-decimal amount box (and the account's sign convention) instead of a plain prompt.

## 4.22
- Open file on a computer no longer filters by file type (the filter greyed out .json files). The filter stays on iPhone/iPad.

## 4.21
- Test-data status is stored in the file (`settings.testData`), so it follows the file on any device or name. A test-named file without the flag asks once and is then marked.

## 4.20
- Removed the Data set switch. Test data is recognised from the file that is open; orange stripe and TEST DATA tag show while it is open.

## 4.19
- Save file keeps the name the file was opened with (`ledger-test.json` stays `ledger-test.json`; Safari/Brave "(1)" suffixes are dropped).

## 4.18
- Folder lookup tolerates look-alike file names (for example `ledger-test (1).json`); **Open a file...** added to Settings; clearer messages when no file is found.

## 4.17
- Settings: edit the category list (add, rename, delete, merge) and choose the dropdown order (your own order, A-Z, or most used).

## 4.0 to 4.16 (before 2026-10-07; grouped)
- Settings gear: file location (choose or create a folder), About, User guide; "Download" wording changed to "Save file".
- Test data set and switch (later replaced in 4.20/4.21).
- iPhone/iPad: roomier rows, Web Share save without the stray `text` file, Unpost for a mistaken Post, broadened file picker with an "Open any file" fallback.
- Home-screen/PWA support: manifest, service worker, icons, Mac installer script, README.
- Balance graph popup (12 months ending at the look-ahead date vs the year earlier); accounts shown in a box.
- Credit cards, lines of credit and loans shown the way the bank shows them (owing positive); entry and labels follow the account type.
- Low-balance warning level per account with pop-up and red ring.
- Description renamed with the account; recurring transfers between accounts; one **New item** popup for everything, also used for missed items in reconciliation.
- Fixed-decimal amount entry (type digits only); tighter spacing.
- Balances re-anchor to the latest reconciled statement balance.
- Multiple accounts and transfers.
- Reports by category and by category and month; category dropdown with autocomplete of earlier descriptions.
- Monthly reconciliation per account.
- Recurring items with estimates, confirmed amounts, Post, Skip, Stop and Change amount; look-ahead projections.
- Transaction dates, sorting and running balance; soft delete.
- Autosave to a chosen file in Chrome/Edge/Brave (with the browser's file feature enabled); manual save elsewhere; data kept as JSON in iCloud Drive or any folder.
