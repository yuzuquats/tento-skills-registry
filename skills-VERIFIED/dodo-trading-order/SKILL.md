---
name: dodo-trading-order
description: Use Dodo's pre-vetted dodo-trading.order command to inspect the paper trading account, query tradable assets, create market-order previews, and submit only approved Alpaca paper previews. Use when a Dodo agent needs account state, asset availability, trade previews, or paper order submission through the company command bridge.
---

# Dodo Trading Order

Use `./dispatch dodo-trading.order ...` for Dodo trading account and order
operations. The command is the only approved path from company agents to
`tento-trading`; do not call broker APIs or host paths directly.

## Backend Modes

- `alpaca-paper`: account, assets, quotes, and submit go through Alpaca paper
  trading; previews and order records are also stored in the local
  `tento-trading` SQLite ledger.
- `no-alpaca-sqlite`: fully local paper account. Use it for dry runs when
  Alpaca credentials are unavailable.
- `alpaca-live`: not allowed for Dodo company-agent workflows.

Check the active mode first:

```bash
./dispatch dodo-trading.order account
```

If the response provider is not `alpaca-paper`, treat the workflow as local
simulation unless the task explicitly says the bridge was reconfigured.

## Workflow

1. Inspect account state before planning a trade:

   ```bash
   ./dispatch dodo-trading.order account
   ```

2. Confirm the asset is available and tradable:

   ```bash
   ./dispatch dodo-trading.order asset AAPL
   ```

3. Create a preview. This does not submit an order:

   ```bash
   ./dispatch dodo-trading.order preview --side buy --symbol AAPL --qty 1 --type market --time-in-force day
   ```

4. Put the preview ID, estimated price, estimated notional, thesis link, risk
   plan, and approver in the trade journal.

5. Submit only after the Head of Trading & Risk has approved the exact preview:

   ```bash
   ./dispatch dodo-trading.order submit --preview-id <preview-id>
   ```

## Rules

- Preview before every submit.
- Submit only the exact preview that was reviewed.
- Never use compatibility aliases (`buy`, `sell`, `trade`) for final approval;
  they are preview conveniences only.
- Never change provider, endpoint, credentials, or DB path from inside a company
  task.
- Never use or request `alpaca-live`.
- Treat Alpaca paper submissions as broker-shaped tests, not real capital
  allocation, custody movement, or compliance approval.
- If a command fails, capture the command, provider, sanitized error, and
  preview ID if present. Do not request or print API secrets.

## Useful Commands

```bash
./dispatch dodo-trading.order assets --status active --asset-class us_equity
./dispatch dodo-trading.order asset <symbol>
./dispatch dodo-trading.order preview --side <buy|sell> --symbol <symbol> --qty <quantity> --type market --time-in-force day
./dispatch dodo-trading.order submit --preview-id <preview-id>
```
