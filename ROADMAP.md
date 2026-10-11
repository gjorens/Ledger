# myLedger ideas (not built yet)

## Multiple real ledgers (e.g. personal and business)
Today there is effectively one real ledger (`ledger.json`) plus the test data file (`ledger-test.json`).
To support several:
1. **Start a new, empty ledger** asks for a name and saves as `ledger-<name>.json` (instead of always `ledger.json`).
2. Show the ledger's name in the page header / title so it is always clear which one is open.
3. Update the Automator script so each `ledger-<name>.json` is kept as its own file with its own dated backups (today every `ledger*.json` goes to `ledger.json` and would overwrite another real ledger).
4. Archive files would carry the ledger name (`ledger-<name>-archive-YYYY-MM.json`).
5. Folder mode: look for the chosen ledger's file name rather than the fixed standard name.

## Investments (prices, exchange rates, dividends)
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

## Native iOS app (Capacitor wrapper)
Goal: a real installable iOS app that keeps the existing web code, fixing the limits of a Home Screen web app.
- **Capacitor** (free, open source, from the Ionic team) generates an Xcode project containing a native app shell with the existing `index.html` running inside a web view; a bridge lets the JavaScript call native features. Compiled in Xcode, it installs like any app (own icon, no browser chrome). Can also target Android from the same project.
- **What it fixes**: direct read/write of the iCloud Drive folder (no Share-sheet save, real autosave), reliable local storage (a Home Screen web app can lose its data after long disuse), Face ID / passcode app lock (ties in with the "Extra security" item), background / privacy-screen behaviour.
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

Status: the drop-down and page registry exist; the Investments, Loans and mortgages and Autos pages are placeholders. Open design points: whether investment and loan accounts should disappear from the Accounts page tabs once their own page exists (they must stay in transfer lists either way); a **Mortgage** account type (a liability, with rate, term and renewal); and, for Autos, vehicle and service-record types in the file (data version 5).
