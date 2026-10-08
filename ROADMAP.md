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
