# myLedger ideas (not built yet)

## Multiple real ledgers (e.g. personal and business)
Today there is effectively one real ledger (`ledger.json`) plus the test data file (`ledger-test.json`).
To support several:
1. **Start a new, empty ledger** asks for a name and saves as `ledger-<name>.json` (instead of always `ledger.json`).
2. Show the ledger's name in the page header / title so it is always clear which one is open.
3. Update the Automator script so each `ledger-<name>.json` is kept as its own file with its own dated backups (today every `ledger*.json` goes to `ledger.json` and would overwrite another real ledger).
4. Archive files would carry the ledger name (`ledger-<name>-archive-YYYY-MM.json`).
5. Folder mode: look for the chosen ledger's file name rather than the fixed standard name.

## Investments (portfolio tracking, then prices, exchange rates, dividends)

### Owner's requirements (Oct 19 2026)
**Hierarchy:** Institution > Portfolio > Account > Holding. For example, at BMO there may be several portfolios, each with accounts of different types (RRSP, RESP, RRIF, TFSA, FHSA, cash, margin and more). Tax treatment is irrelevant to the app, so the account type is only a label. An account holds one or more products (stocks, bonds, HISA funds, ETFs and so on) plus cash. Each product is listed on a US or Canadian exchange and has a currency, USD or CAD; cash can be in either currency.

**Main function 1: daily portfolio record.** The user enters the portfolio value each day, plus any deposits or withdrawals. The app keeps the history of daily values and shows a daily table like the owner's spreadsheet (screenshot of the 2026 sheet):
- Date and weekday; **Value (Net)** = the value entered; **Change (Net)** = today's value minus the previous value minus deposits (plus withdrawals), so deposits and withdrawals do not count as performance; **Deposits (-Withdrawals)** column.
- **Weekly summary row** (Saturday): week's net change, **% Week (Net)**, **YTD (Net)** in dollars, **% YTD (Gross)**, **% YTD (Net)**. Sunday is a shaded spacer row.
- **Average Daily (Net)** = YTD net change divided by the number of **trading days** so far (not the number of entries), so a missed entry does not distort the average. Trading days are Monday to Friday minus market holidays; the US and Canadian holidays differ.
- **Projected Year End Value** = today's value plus the average daily change times the remaining trading days (about 252 at the start of the year; to confirm against the sheet), using the same trading-day calendar.
- **TSX %** column: the index's YTD % alongside, for comparison (needs index data, see below).
- Gross includes deposits and withdrawals; net excludes them. Example from the sheet: 976,170 on Jan 1, a 335,000 deposit on Jan 2 and a value of 1,318,102 gives a net change of 6,932, a gross YTD of 35.03% and a net YTD of 0.71%.
- **Year-to-date** starts from a January 1 value the user gives initially, and is **reconciled each following January 1** (the closing value of the old year becomes the opening value of the new one, with the option to correct it against the year-end statement).

**Main function 2: graphs.** Performance of the portfolio against previous years (one line per year, from Jan 1) and against stock exchange indexes (TSX, S&P 500 and so on). (Owner's second main function not yet described.)

**Main function 2: performance of individual instruments.** Performance means the change in the instrument's value **plus any dividends or distributions paid** (total return). The user sees at a glance how each instrument is doing, alone and compared with the other instruments in the portfolio (for example a ranked table of total return % for the year, with price return and dividend return shown separately, a small trend line per row, and colour for best and worst).

**Dividend projection (owner's spreadsheet, "Total Portfolio" sheet).** One row per holding, with cash, GIC and HISA lines above them:
- Columns: currency, **shares**, **yearly dividend per share**, **annualized income** (shares x yearly dividend, converted to CAD; e.g. 6,260 x 0.888 = 5,558.88), **total value in CAD**, **% yield** (annualized income / value), **dividend payments per year** (12 monthly, 4 quarterly) and the **payment each time** (annual income / payments per year).
- A **Jan to Dec grid** showing what is expected in each month. Monthly payers appear every month; quarterly payers appear only in their own months (for example ENB in Mar/Jun/Sep/Dec, BMO in Feb/May/Aug/Nov, ZWU in Jan/Apr/Jul/Oct), so the monthly totals differ (6,403.65 / 6,016.64 / 9,348.20 in the sample). Totals row for value, annual income and portfolio yield (87,073.97 and 6.34% in the sample).
- Cash can be in CAD and USD. USD holdings and USD dividends are converted to CAD with an exchange rate (the sample implies about 1.40), so the app needs a CAD/USD rate (fetched, with manual override).
- The projection is "based on their past dividends": the yearly dividend per share and the payment months come from the instrument's dividend history (entered by the user as each dividend arrives, or fetched where a source exists), and can be overridden by hand.

### Decisions (owner, Oct 19 2026)
- **Entry is one portfolio balance per day, in CAD.** No per-account or per-holding entry, no currency conversion in the daily record. (Holdings, USD and CAD products, and prices stay in the later add-ons.)
- **Index closes are fetched by the app** (TSX and S&P 500 at least), best effort: it does not have to be 100% reliable; missing days can be typed in by hand. Open point: a browser can only call a data source that allows cross-origin requests, which many free sources do not, so a source must be tested from the real app before the design is fixed (a key-based free API is the likely fallback; only the index symbol is sent, never any balance).
- **One file:** investments data goes into the same ledger file (data format v6).
- **Transfers between investment accounts and bank accounts:** the existing transfer feature is extended so an investment account can be the From or To account. A transfer in or out creates the matching **deposit / withdrawal** on that day for the portfolio, so the bank side and the portfolio side stay in step and the net performance excludes it.
- **Trading-day calendar:** weekdays minus holidays for the TSX and the NYSE/Nasdaq, worked out by rule inside the app (New Year, Family Day, Good Friday, Victoria Day, Canada Day, Civic Holiday, Labour Day, Thanksgiving, Christmas, Boxing Day for Canada; New Year, MLK, Presidents, Good Friday, Memorial, Juneteenth, July 4, Labour, Thanksgiving, Christmas for the US). A day counts as a trading day when at least one of the two markets is open (a setting could change this). No data source is needed for holidays.

### Decisions on holdings (owner, Oct 19 2026)
- **Calculated totals with overrides:** the portfolio total is calculated from holdings x prices (plus cash, GIC, HISA), and the user can override it, and record deposits, withdrawals and other deviations by hand.
- **No trade entry required:** the user does a **true-up of the number of shares per account** (a periodic count per account, as on the owner's "S-RRIF" sheet), not a record of each buy and sell. Per account the sheet has: cash in CAD and USD, GICs (up to four, with rates), a HISA with a rate per currency, then one row per symbol with shares and value in CAD or USD, converted to a CAD total. Between true-ups the share counts carry forward; the difference between two true-ups, together with deposits and withdrawals, explains the change.
- **Several institutions and portfolios** are normal (a typical user spreads money across a few banks and investment firms), so the hierarchy stays Institution > Portfolio > Account, each account with its own true-up and a rolled-up total.
- **Status (v4.77):** the hierarchy (institution > portfolio > account), the daily value and deposit/withdrawal record, the daily table with weekly summaries, the year-comparison chart and the trading-day calendar are built. Alpha Vantage returned a Toronto ETF price (ZEB.TRT) in the owner's test, so a free key is the likely price source (about 25 requests a day, one per second); the app would need to refresh a few symbols a day, with manual entry as fallback. Done since: import of the owner's yearly CSV sheets (v4.78). Next: holdings and the true-up, prices, index columns, dividend projection, transfers.
- **Price test result (Oct 19, Brave on Mac, with a free Alpha Vantage key):** end-of-day quotes came back for ZEB, HUTS, TXF, ZWU, ENB (Toronto, symbol suffix .TRT), JEPQ and SPY (US), and XIU.TRT, all dated the last trading day. Yahoo and Stooq are blocked for browsers. Exchange rates work from the Bank of Canada and Frankfurter. Index levels are not tested: SPY and XIU are ETF proxies for the S&P 500 and TSX (price only, so dividends are excluded). Free key limits: about 1 request per second and 25 per day, so a full refresh of about 20 holdings plus 2 indexes uses most of a day's allowance. Not yet tested: dividend history, price history, other devices.
- **Price availability:** to be tested from the real app. `price-test.html` (a one-page test) tries Yahoo, Stooq, Frankfurter, Bank of Canada and optional keyed sources for Canadian ETFs, US ETFs, the TSX and S&P indexes and USD/CAD, and reports which work in the browser. Results decide the design.

### Design questions still open
- **Total value:** with holdings entered (shares, price) the portfolio total can be calculated from them, instead of typed daily. Does the owner still want to type the daily total, or should holdings x prices replace it (and the typed value remain as an override)?
- Cost / start-of-year reference per holding for the return calculation (Jan 1 price for year-to-date, or purchase price), and how buys, sells and reinvested distributions are recorded.
- Prices for each instrument carry the same free-source browser-access question as the index closes (symbols with exchange suffix; Canadian ETFs are often missing on free sources).
- Which Institution > Portfolio > Account levels carry the daily balance if the user has several portfolios (one balance series per portfolio, with a combined total)?
- Whether Investment accounts also stay in the Accounts tabs (see the section pages note).

### Later add-ons (from the earlier notes)
Goal: transfers can use investment accounts (buy / sell / contribute / withdraw), and investment accounts show a market value.
- **Holdings** per investment account: symbol (with exchange suffix for TSX etc.), shares, optional cost.
- **Refresh prices** button (no constant polling). Free provider with the user's own API key stored in Settings, never in the code on GitHub. Delayed or end-of-day prices only; real-time streaming is a paid service. Only symbols are sent to the provider, never balances or transactions.
- **Exchange rates**: free daily reference rates (e.g. ECB-based, such as Frankfurter; test browser access first and check the currency list). Used to suggest the rate on transfers between accounts in different currencies (the user can type the real amounts the bank used), and for an optional "all accounts in home currency" total.
- **Dividends / distributions**: fetch history where a free source has it; project the next payments from the pattern (e.g. monthly, about $X per share) as projected income like recurring items; the user enters the actual amount when it lands, and a matching deposit confirms the projected item. Upcoming declared amounts and Canadian ETF schedules are mostly paid or issuer-website only, so treat them as forecasts.
- Transfers: allow investment accounts as From/To; decide whether a transfer stores a rate or share count.

## Extra security (app lock)
Goal: a phone or iPad that is passed around or left in public should not reveal financial data, similar to WhatsApp's app lock.
- **Lock screen** when the app opens or returns from the background, and after an idle timeout (user setting: immediately / 1 min / 5 min / 15 min). Data is hidden behind the lock and not rendered until unlocked.
- **Unlock** with a PIN or passphrase (stored only as a salted hash), and where the device supports it Face ID / Touch ID via WebAuthn (platform authenticator). Limited attempts, with a growing delay after failures.
- **Privacy screen**: blur / hide balances when the app is in the app switcher, and an optional "hide amounts" toggle for showing the app to others.
- **Encrypt the data file at rest** (optional): passphrase-based encryption (AES-GCM, key derived with PBKDF2 or Argon2) so the JSON in iCloud and the browser's saved draft are unreadable without the passphrase. A forgotten passphrase means the data cannot be recovered, so backups and a clear warning are needed. The Automator script and archive files would need to handle encrypted files (they only move files, so it should be fine).
- Note: a screen lock alone stops a casual onlooker; only encryption protects the file and the browser's stored draft if someone gets at the device storage.
- **Risk assessment (owner, Oct 2026):** the most sensitive stored item is the account numbers (about as exposed as a number on a paper cheque). Balances and history could help someone impersonate or target the owner. No online banking passwords, PINs or SIN / SSN are stored, which keeps the worst-case impact moderate. So: lock screen first (cheap, covers the real scenario of a passed-around device); encryption second and optional.
- **Lost passphrase mitigation (for the encryption item):** envelope encryption (random data key wrapped by both the passphrase and a generated recovery key); show the recovery key once and force a "saved it" confirmation; store it offline; keep an unencrypted export; clear "no recovery" warning; optional Shamir split between trusted people; Face ID / Keychain only as a convenience.
- **Why encrypt in the app:** disk encryption (FileVault, BitLocker) has no known intentional back door, but recovery keys are often escrowed (Microsoft account for BitLocker, optional iCloud for FileVault), and iCloud files are readable by Apple unless Advanced Data Protection is on. In-app encryption means the iCloud copy is ciphertext and only the owner holds the key. Details in DESIGN.md section 13.

## Native iOS app (Capacitor wrapper)
Goal: a real installable iOS app that keeps the existing web code, fixing the limits of a Home Screen web app.
- **Capacitor** (free, open source, from the Ionic team) generates an Xcode project containing a native app shell with the existing `index.html` running inside a web view; a bridge lets the JavaScript call native features. Compiled in Xcode, it installs like any app (own icon, no browser chrome). Can also target Android from the same project.
- **What it fixes**: direct read/write of the iCloud Drive folder (no Share-sheet save, real autosave; today the phone chooses the save folder and the app can only ask the owner to confirm it, see DESIGN.md Storing and saving), reliable local storage (a Home Screen web app can lose its data after long disuse), Face ID / passcode app lock (ties in with the "Extra security" item), background / privacy-screen behaviour.
- **Work involved**: replace the open/save code with native file plugins (keep the web path for the website version); native lock screen; iOS layout checks (safe areas, keyboard); app icon and launch screen; keep one codebase so the website and the app share `index.html`.
- **Requirements**: a Mac with Xcode; an Apple Developer account (about US$99/year; check current pricing) to install on devices long-term and to use TestFlight. App Store publishing needs Apple review, a privacy policy and screenshots, but is not required for personal use (install via Xcode or TestFlight).
- **Limits of this environment**: Claude can write the project files and the native file / lock code but cannot build or test on iOS here; builds and device testing are done by the user in Xcode, feeding results back.
- **Alternative (not recommended unless the app outgrows the web code)**: full native rewrite in Swift / SwiftUI: best feel, widgets and iCloud sync, but everything is rewritten and two apps would need maintaining.

## Section pages (framework built in v4.66; pages are placeholders)

A second tab row above the account tabs for sections beyond individual accounts, for example **Accounts**, **Net worth** (all accounts and liabilities summed, with a history chart), **Loans** (balance, rate, payoff date), **Investments** (see above) and **Automobiles** (cost, odometer, service and fuel log). Each section would reuse the same file, categories and reports. Sensible first step: Net worth, because it needs no new data. The others need new record types and would be added one at a time.

## Export and import accounts (idea)

Export one or more accounts, with their entries, recurring items, reconciliations and the categories they use, to a file that can be imported into another ledger file. Uses: splitting one file into several (see multiple real ledgers), moving an account to a new year's file, sharing one account with someone else, or recovering an account from a backup. Points to decide: new account ids on import so they never clash; what to do with transfers whose other leg is in an account that is not exported (turn them into ordinary entries or keep the link and warn); merging categories by name.

## TFSA account type with contribution tracking (idea, part of Investments)

A **TFSA** (Tax-Free Savings Account, Canada) account type under investment accounts. It tracks **contributions** separately from growth: contributions and withdrawals by calendar year, the annual limit for each year, the lifetime contribution room used and remaining, and a warning before an over-contribution. Withdrawals are added back to room on 1 January of the following year, so the tracker needs to record when a withdrawal happened. The yearly limits come from the Canada Revenue Agency and are entered by the user or kept in a small table, and the app should say that it is not tax advice and that the person should check their CRA My Account for official room. The same design would suit an RRSP (deduction limit) or FHSA later.

Status: the drop-down and page registry exist. **Autos is built (v4.67)**: vehicles, service history, per-service schedules and projected mileage. Investments and Loans and mortgages are still placeholders. Open design points: whether investment and loan accounts should disappear from the Accounts page tabs once their own page exists (they must stay in transfer lists either way); a **Mortgage** account type (a liability, with rate, term and renewal).

## Autos follow-ups (ideas)
- **Reminders:** a web page cannot send email or SMS by itself (no server; a push service or a paid email/SMS provider would be needed, and the data would leave the device). Practical options: export a due service as a calendar event (an `.ics` file, so the phone's Calendar gives the alert), or a native-app local notification once the iOS app exists. A "mailto:" or "sms:" link can only prefill a message that the owner sends.
- Fuel purchases and fuel economy; insurance, registration and licence renewal dates; yearly cost per vehicle; invoice notes or photos; link a service record to the ledger entry that paid it; show more than the next occurrence of each service.

## Safer saving on phones: further ideas
- **Dated file names option** for the share-sheet path (for example `ledger-2026-10-11-2010.json` each time) so nothing can ever be replaced by mistake, at the cost of clutter; the newest file would be the one to open.
- Apply the same "did you save it?" confirmation to the archive export, which also uses the share sheet.
- Show on open which device and time last saved the file (`savedAt` plus a device name), to make a stale copy easier to spot.

## Dividend lookup test (2026-10-19)
Alpha Vantage DIVIDENDS from the browser: works for US symbols (JEPQ: 53 payments with dates and amounts); returns empty data for Canadian `.TRT` symbols (HDIV, ENB, TXF, ZEB). So free automatic lookup covers US-listed positions only. Canadian dividends stay forecast from broker yield % and editable months/amounts; a way to record or import actual dividends received (from broker activity) is the likely next step.

## Price history test (2026-10-19)
Alpha Vantage TIME_SERIES_WEEKLY_ADJUSTED returned an empty object for every symbol, including US JEPQ, so it is not available on the free key. Total return per instrument (price change + dividends over a period, divided by the starting price) will be built from the owner's own records: price snapshots saved with every holdings import, year-start prices entered or imported once, and dividends recorded.

## Shelved (2026-10-19): per-instrument total return
Owner has no automated source for dividends received or year-start prices, so manual entry would be required. Idea shelved until the requirements can be matched to these limits. Nothing to build now. v4.82 (holdings true-up, forecast dividends) stays as delivered.

## Survivor Data tab (placeholder since v4.85, sections in v4.88)
Locked with the PIN. Planned sections: Day-to-day money (paying utility bills, transferring money, investment upkeep); Home and vehicles (car maintenance, house maintenance); People and papers (key contacts, insurance and pension, where documents and the will are kept, wishes and instructions). Not built: no data is stored yet. Passwords themselves should not be kept here, only where they are kept.
