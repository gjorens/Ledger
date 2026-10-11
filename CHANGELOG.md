# myLedger version history

The version number and build time are shown next to the app name at the top of the page. Newest first.

Data file format: `version: 5` since v4.67 (4 before; v4 files are upgraded when opened). Files from a newer format than the app understands are refused; older files are upgraded when opened.

## 4.68 (2026-10-11)
- **Appearance setting.** Settings > Display > Appearance: **Default** (follow the device's light or dark setting, as before), **Light** or **Dark**. The choice is remembered on that device only (not in the ledger file), so the iPad can be dark while the Mac is light. It is applied before the page draws, so there is no flash of the wrong colours.

## 4.67 (2026-10-11)
- **Autos page built.** Pick **Autos** in the page drop-down. Add vehicles (name, plate, VIN, kilometres or miles, odometer now, your yearly distance estimate) and keep a **service history** for each.
- **Service record form:** date, vehicle, mileage, services performed (tick boxes from a list kept A-Z), performed by (remembers earlier shops), cost and notes. A box lets you add a service that is not in the list; it is added to the list and ticked. The form shows the **total cost of service to date** for the vehicle, including the record being entered. The page also shows total, this year and number of visits.
- **Due next, by service:** each vehicle has its own schedule per service item, every N months, every N km/miles, or both (whichever comes first), for example brake inspection every 12 months and brake pads every 60,000 km. Each item shows when it was last done and when it is next due, marked **Due soon** (within 45 days) or **Overdue**, or "No record yet".
- **Projected mileage:** starts from your yearly estimate and switches to the distance you actually drive once readings and service records span about three months (the last two years are used). **Odometer reading** adds a reading without a service. Distance-based due dates use this estimate and are labelled estimated.
- Service records can be edited and deleted (soft delete; shown and restorable when Settings > Display shows deleted entries). A vehicle can be removed; its records stay in the file.
- Data format 5 adds `vehicles`, `serviceItems` and `services`. The test file now includes two sample vehicles with service history.

## 4.66 (2026-10-11)
- **Pages framework.** The "ACCOUNTS" label at the top of the main box is now a drop-down: **Accounts** (the existing screen, unchanged), **Investments**, **Loans and mortgages** and **Autos**. The choice is remembered. The three new pages are placeholders for now: each lists what it is planned to hold, and Investments and Loans also list your real accounts of those types (Investment; Loan and Line of credit) with an "Open in Accounts" button. The + and pencil buttons show only on the Accounts page.
- Investment and loan accounts remain ordinary accounts in the same list, so transfers between them and your other accounts already work with + New item > Transfer.

## 4.65 (2026-10-10)
- Edit / change-amount / account dialogs on iPhone and iPad no longer flash the date picker or keyboard for a moment when they open: nothing is focused until you tap a field.

## 4.64 (2026-10-10)
- iPhone and iPad: opening Edit recurring item, Edit entry or Edit account no longer pops the keyboard up automatically, and while the keyboard is showing every dialog moves to the top of the visible area and scrolls, so the Save and Cancel buttons can always be reached. In forms the buttons also stay pinned at the bottom of the dialog.

## 4.63 (2026-10-10)
- New item > Repeats has a new choice, **One time, estimated amount**: a single future item that shows in the list as a projected "est." row. When the real amount is known, press Edit on the row to confirm it, or Post it when it happens; it can also be skipped, like any projected item. It appears in Recurring items as "One time on <date>". (It only shows once its date is inside the Look ahead period.)
- Balance graph on iPhone and iPad: tracing the line no longer starts text selection or the long-press menu; the read-out still follows your finger.

## 4.62 (2026-10-10)
- Settings > About has a **Reload app** button: it saves, clears the app's cached copy and reloads, so the home-screen app on iPhone and iPad (which has no refresh button) picks up a new version without closing it. Unsaved changes are kept in the browser draft.

## 4.61 (2026-10-10)
- The Balance graph button is now an **expand** icon (a double-ended diagonal arrow) at the top right of the small balance graph beside the lowest-balance figure, instead of a text button in the Transactions row.

## 4.60 (2026-10-10)
- iPhone only: the item list no longer shows the Category column, dates drop the year (the month bars already show it, e.g. "Oct 14"), cell padding is tighter, and the Edit / Post / Delete buttons stack so the whole list fits the screen with Amount and Balance visible and no sideways scrolling. iPad and Mac are unchanged.

## 4.59 (2026-10-10)
- Balance graph: the window now uses the full width of the app, and the graph is drawn at that width (taller on wide screens). Move the mouse over it (or touch and drag on iPhone and iPad) to see the date and balance at that point, plus the balance a year earlier. Weekly points; projected dates are marked.

## 4.58 (2026-10-10)
- Cleaner main page. The Open file buttons are gone from the front page. The top row now has the app name with a short version number (v4.58), the Save button, a new **file icon** and the gear. The full version with its build time is in Settings > About.
- The **file icon** opens a File window holding everything file-related that used to be in Settings: Open a file, Open any file, choose folder or single file, Start a new ledger, Archive old data and View an archive. Settings keeps Display, Categories, About and the User guide. The banner and Save button work as before.
- Accounts: Add and Edit are now a **+** and a **pencil** button. Account tabs are all the same size (room for about 12 characters; longer names are shortened with "...", and hovering or long-pressing shows the full name) and line up in a grid.

## 4.57 (2026-10-10)
- Post this item: the dialog now fits the screen on iPhone and iPad (the amount box and its ± button were running off the edge, with Cancel and Post crowding it). The cause was the row above the transactions (Today, search, New item, Look ahead, Balance graph), which was wider than a phone screen and made every dialog wider than the screen; it now wraps onto two lines. Field labels in all item forms also have a little more space above their boxes.

## 4.56 (2026-10-10)
- Skipped occurrences of a recurring item can be put back: each skipped date is listed under the item ("Skipped: Nov 14, 2026") with a restore button. Skips from months you have since archived are not offered.

## 4.55 (2026-10-09)
- Recurring items list: a scheduled transfer no longer shows the two account names twice ("Chequing → CIBC Mastercard · transfer Chequing → CIBC Mastercard"); it now reads "Chequing → CIBC Mastercard · transfer · est. 5,000.00 · Monthly from ...".

## 4.54 (2026-10-09)
- New repeat choice **Monthly (except Dec)**: every month but December (for example a payment that is skipped over the holidays). Available when adding an item, converting an entry, and editing a repeating item.

## 4.53 (2026-10-09)
- An entry you have already made can now become a repeating item: open Edit on it and choose **Repeats** (and an optional last date). The entry stays as the first occurrence and the following ones are projected. Works for transfers too.

## 4.52 (2026-10-09)
- Settings window now uses the full width of the screen, has an X button at the top (stays visible while scrolling), and opening a section closes the others.

## 4.51 (2026-10-09)
- Credit card, line of credit and loan accounts: payments and credits (the negative amounts as the bank shows them) now appear in red, in the transaction list, projected items, search results, recurring items, archive viewer and Edit lists. Charges stay black.

## 4.50 (2026-10-09)
- The last two browser prompt boxes (adding a new category from the New item / Edit forms, and renaming a category) are now myLedger dialogs titled "myLedger", with a text box and Enter to confirm. No browser "<site> says" boxes remain.

## 4.49 (2026-10-09)
- Larger checkbox for the Show deleted entries setting.

## 4.48 (2026-10-09)
- Deleted entries are now hidden from the transaction list and search by default. **Settings > Display > Show deleted entries** shows them again (struck through, with the restore button). Saved in the file; balances are unaffected either way.

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
