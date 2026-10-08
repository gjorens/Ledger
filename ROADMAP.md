# myLedger ideas (not built yet)

## Multiple real ledgers (e.g. personal and business)
Today there is effectively one real ledger (`ledger.json`) plus the test data file (`ledger-test.json`).
To support several:
1. **Start a new, empty ledger** asks for a name and saves as `ledger-<name>.json` (instead of always `ledger.json`).
2. Show the ledger's name in the page header / title so it is always clear which one is open.
3. Update the Automator script so each `ledger-<name>.json` is kept as its own file with its own dated backups (today every `ledger*.json` goes to `ledger.json` and would overwrite another real ledger).
4. Archive files would carry the ledger name (`ledger-<name>-archive-YYYY-MM.json`).
5. Folder mode: look for the chosen ledger's file name rather than the fixed standard name.
