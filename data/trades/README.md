# nad.fun trades and money flows

Every buy and sell of every [nad.fun](https://nad.fun) token on Monad, every Transfer of those tokens, and per-token money flows built from them. Like the rest of the dataset, everything here is decoded from public Monad blockchain events; nothing comes from nad.fun's website or API. Native MON movements behind the flows come from on-chain transactions, read through an explorer API.

This directory lives on the `trades` branch, which is `main` plus `data/trades/`. A job updates it about once an hour, separately from the `tokens` and `token_graduations` snapshots, and in its own commits. [`manifest.json`](manifest.json) says when it last ran and how far the data goes.

## Files

```
manifest.json                run time, watermark, last block covered, token count
index.json                   token → {updated_at, trades, transfers, flows}
volume_hourly.json           MON traded per UTC hour, all tokens
<token>/trades.csv           the token's trades, in chain order
<token>/transfers.csv        the token's ERC-20 Transfers, in chain order
<token>/flows.json           money flows (tokens with at least 10 trades)
```

Fetch one token straight from GitHub:

```
https://raw.githubusercontent.com/0x-ez/nadfun/trades/data/trades/<token>/flows.json
```

Formats follow the dataset's [conventions](../../docs/conventions.md): big integers are exact decimal strings, timestamps ISO 8601 UTC, addresses lowercase `0x` hex, NULL an empty field.

### trades.csv

`block_number, block_time, log_index, tx_hash, side (buy/sell), phase (curve/dex), trader, quote_token, quote_decimals, quote_amount, token_amount, reserve_quote, reserve_token, sqrt_price_x96, contract`

- `trader` is the transaction's sender. A bot, router or executor shows up as itself, not as the wallet it trades for. `transfers.csv` shows whose tokens actually moved.
- Amounts are in base units. The token has 18 decimals, and the quote token has `quote_decimals` (WMON: 18).
- `reserve_*` is the curve's virtual reserves or a pair's reserves after the trade; `sqrt_price_x96` is a Uniswap V3 pool's price.

### transfers.csv

`block_number, block_time, log_index, tx_hash, from, to, amount`

A token is covered from its first trade until about 7 days after its last one. If it trades again, coverage resumes.

### flows.json

Sankey-ready links in MON for four windows after the first trade: `300` (5 min), `3600` (1 h), `86400` (24 h) and `0` (all time). Each window has three graphs:

- `inflow`: source → `in:<cohort>` → `Buys`
- `outflow`: `Sells` → `out:<cohort>` → destination
- `combined`: source → `in:<cohort>` → `Token` → `out:<cohort>` → destination, with a balance node

Each graph is `{nodes: [{id, label, kind, cohort?}], links: [{source, target, value, wallets}]}`, where `source` and `target` index `nodes`.

How the flows are read:

- **Order.** Graduated tokens get their data first, then the rest from the newest launch back.
- **Wallets.** A trade belongs to the wallet whose token balance changed in its transaction. Tokens that only pass through a router or executor don't count.
- **Cohorts.** Each wallet gets one cohort for the token's whole history: `dev` (the creator, or a wallet it funded or sent tokens to), `insiders` (sold or held without buying), `bundlers`, `snipers` (first 25 blocks), `early buyers` (first 5 minutes), `executor` and `retail`.
- **Sources.** A wallet's MON inflows in the 24 hours before its first buy, plus proceeds from selling other nad.fun tokens. MON history is only looked up for each token's largest wallets, so "unknown" is often large.
- **Destinations.** A wallet's MON outflows and buys of other nad.fun tokens in the 24 hours after its first sell. The rest is "not moved".

Sources and destinations keep the top 8 per window, and the rest become "Other". Links under 0.5% of the window's larger side are dropped. `buys` and `sells` are exact totals.

### volume_hourly.json

`{format, generated_at, through_block, unit: "MON", start, volume}`: `volume[i]` is the MON traded in the hour `start + i` hours (UTC), buys and sells, over every token's WMON-quoted trades. Empty hours are 0.

Built by `services/flows` in the dyad repo, from `infra/clickhouse/queries/token_flows.sql`.
