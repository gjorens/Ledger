# myLedger version history

The version number and build time are shown next to the app name at the top of the page. Newest first.

Data file format: `version: 4` since v4.0. Files from a newer format than the app understands are refused; older files are upgraded when opened.

## 4.47 (2026-10-09)
- Recurring items are sorted (Description A-Z by default) with a **Sort by** choice: Description A-Z, Next date, or Amount (largest first). Saved in the file.

## 4.46 (2026-10-09)
- Recurring items: items whose last date has passed, and items you stopped, move to a collapsible **Finished and stopped items** group, so the active list stays short.
- **Restore** a stopped item (it is projected again). Edit a finished item to give it a new last date and continue it.
- **Remove** an item from the file for good, or **Clean up** all finished and stopped items at once. Only items with no entries in the file are removed; items that still have posted entries stay in the group (after Archive old data they can be removed too).

## 4.45 (2026-10-09)
- **Change one occurrence only**: the pencil on a projected row now opens "Edit this occurrence" with a choice: *My estimate (this month only)*, for budgeting something you do not know exactly yet (shows as an estimate, other months keep the regular estimate), or *Confirmed by the biller* (shows in bold). "Back to regular estimate" removes the change. To change every following month, use the calendar icon in Recurring items.

## 4.44 (2026-10-08)
- Each category now has a **type**: Money in, Money out or Either, set with a drop-down on every row in Settings > Categories (saved in the file). The "Is that right?" amount check follows it.
- Default types are guessed from the name; names containing "tax" (such as Income Tax) are money out. Renaming a category keeps its type; deleting removes it.

## 4.43 (2026-10-08)
- The last date of a repeating item can no longer be set before its first date: the date picker disables earlier days and the form will not save (new item and Edit recurring item).

## 4.42 (2026-10-08)
- **Edit recurring item** (pencil in Recurring items): correct a mistake in the original entry: description, category, estimated amount, last date, and (until anything has been posted, skipped or confirmed) the first date and how often it repeats; for transfers the From / To accounts and direction. The old pencil, which schedules a new estimate from a date, is now the calendar icon.

## 4.41 (2026-10-08)
- **Add another** now resets "Repeats" to "Does not repeat" (and clears the last date), so a repeating item is not repeated by accident.
- Reasonableness check on amounts: an expense category with money coming in, or an income category (Income, Salary, Refund, Interest, ...) with money going out, asks "Is that right?" before saving. Applies to new items and to edits that change the amount or category. "Other" and "Gifts" are never questioned; transfers are not checked.

## 4.40 (2026-10-08)
- Amount boxes accept zero: typing 0 gives 0.00 (for example a zero opening balance on a new account). Backspacing a zero amount clears the box.

## 4.39 (2026-10-07)
- Transfers can now be edited fully: the edit dialog has **From / To account** drop-downs with a swap button. Changing an account moves that side of the transfer; swapping reverses the direction. The description updates to "From -> To", both accounts are recalculated, and a leg that moves accounts is unreconciled. Currency mismatch asks to confirm.

## 4.38 (2026-10-07)
- Reports: categories are sorted A-Z (Uncategorized last), and the category column stays in place when scrolling sideways.

## 4.37 (2026-10-07)
- **Start a new, empty ledger** (Settings > File location): clears the open data (test or real), removes the TEST DATA stripe and opens "Create your first account". The previously open file is not touched; the new ledger saves as ledger.json.
- Safety: saving a new ledger into a folder that already contains ledger.json now asks before replacing it.

## 4.36 (2026-10-07)
- Confirmations and alerts are now myLedger's own dialogs (titled "myLedger", with action-specific buttons such as Delete / Undo reconciliation, red for destructive ones) instead of the browser's "gjorens.github.io says" boxes.

## 4.35 (2026-10-07)
- The app now checks the file every 45 seconds while open (as well as when you return to it). A newer version saved elsewhere loads automatically if you have no unsaved changes; otherwise a banner offers **Load newer file** or **Keep my version**.
- Brave / Safari (no direct file link): opening a file that is older than the version already loaded asks for confirmation.

## 4.34 (2026-10-08)
- Search popup: **Show items exceeding threshold** lists every real and projected item after which the account balance is below its warning level (below zero when no level is set). Respects This account / All accounts.
- Negative amounts are shown in red (search results, recurring items list, scheduled changes, archive viewer).

## 4.33 (2026-10-08)
- The "Projected from recurring items" separator is now a bold full-width blue bar.
- **Today** button (left of the search button) scrolls the list back to the latest entry as of today and flashes it.

## 4.32 (2026-10-08)
- Currency shown in front of balances: `US$` for US dollars (and `€`, `£`, `MX$`, `A$`...; plain `$` for CAD), so dollars are never confused.
- **Search** popup (magnifier next to New item): matches description, category, amount, date and account, for this account or all; a target button scrolls the ledger to the entry (switching account if needed) and flashes it; Back to ledger returns without scrolling. Projected items are included for the current account.
- Month separator bars between months in the transaction list.

## 4.31 (2026-10-08)
- **Archive old data** (Settings): moves old reconciled (and deleted) entries into `ledger-archive-YYYY-MM.json` (test data: `ledger-test-archive-...`). Balances, running balances and projections stay exactly the same; the main file keeps a short list of its archives. A read-only **archive viewer** (account picker, search, running balances) opens one or several archive files.
- New colour scheme: calm neutral "tonal" buttons, blue only for primary actions, soft red for destructive; icon buttons are lighter (green check for Post). Style reference: Material Design 3 button hierarchy (filled / tonal / outlined).
- Thin border around every section on the main page.
- Category lists are sorted A-Z by default (Settings still offers your own order or most used).
- **Balance graph** button is the same size as its neighbours.
- iPhone: the date and amount boxes in the Edit dialog no longer overlap.

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
