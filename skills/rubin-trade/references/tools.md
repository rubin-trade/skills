# Rubin MCP — tool & prompt reference

Server: `https://mcp.testnet.rubin.trade/mcp` (testnet) / `https://mcp.mainnet.rubin.trade/mcp`
(mainnet). Streamable HTTP. 33 tools, 12 prompts (server v0.2.0+). If the host's tool list
is missing recent tools, reconnect — hosts cache the tool catalog.

## Session & chain

| Tool | Kind | Purpose |
|---|---|---|
| `whoami` | read | Call first. Master account in **both** forms (cosmos `rit1…` + EVM `0x…`, same 20 bytes), subaccount, trade vs read-only mode, on-chain scope (`canWithdraw:false`, `canTransfer:false`), operator limits (`maxOrderUsdc`, `allowedMarkets`), `depositUrl`, `webUrl`, `serverVersion`. |
| `get_block_height` | read | Latest chain height (needed for SHORT_TERM `goodTilBlock` math). |

## Market data

| Tool | Kind | Purpose |
|---|---|---|
| `list_markets` | read | All perp markets + oracle prices. Use for ticker discovery. |
| `get_market` | read | One market's params and stats. |
| `get_orderbook` | read | Live bids/asks, optional depth. |
| `get_candles` | read | OHLCV at a resolution. |
| `get_candles_multi` | read | Candles for several markets/timeframes in one call. |
| `get_news` | read | Curated crypto/markets news feed (ru/en); filter by category/channels/query/sinceHours. No importance field — the agent judges impact. |

## Account reads

| Tool | Kind | Purpose |
|---|---|---|
| `get_balance` | read | Wallet + subaccount balances, includes funding summary. |
| `get_equity` | read | Account equity. |
| `get_positions` | read | Open positions. |
| `get_open_orders` | read | Resting orders. |
| `get_portfolio` | read | Positions + open orders + margin risk in **one** call — the robust answer to "what's open?". |
| `get_position_risk` | read | `maintenanceMarginBufferUsd` + estimated per-position liquidation price (estimate — cross-margin). |
| `get_fills` | read | Trade fills. |
| `get_pnl` | read | PnL history. |

## Orders (writes)

| Tool | Purpose |
|---|---|
| `place_limit_order` | Default **GTT long-term**, broadcast-commit. SHORT_TERM opt-in (expires ≤ ~20 blocks). Returns verified `confirmation.outcome` — `code:0` ≠ filled. `postOnly` may default to true (operator config). |
| `place_market_order` | Executes as IOC limit capped at oracle ± `slippageBps` mirrored by side (BUY caps above oracle, SELL below; default 500 bps). Unfilled → retry with larger `slippageBps`. |
| `open_position` | Market entry sized by `size` **or** `notionalUsd` (exactly one), optional reduce-only `stopLossPrice`/`takeProfitPrice` bracket in the same call. |
| `close_position` | Reduce-only market close, side auto-flipped, `percent` (default 100). |
| `close_all_positions` | Flatten everything (one reduce-only market order per market, sequential). Confirm with the user first. |
| `place_stop_loss` / `place_take_profit` | Reduce-only conditional market orders; side must be the **closing** side; execute as IOC limit at trigger ± `slippageBps`. |
| `cancel_order` | By `clientId`; SHORT_TERM needs `goodTilBlock`, LONG_TERM needs `goodTilTimeSeconds`. |
| `cancel_all_orders` | All open orders in a market; check `confirmation.remainingOpen`, retry if > 0. |
| `batch_cancel` | Short-term only, by market + clientIds. |

Rate limit for stateful (GTT) orders: **2/block, 20/100 blocks**.

## Funds (within the account)

| Tool | Kind | Purpose |
|---|---|---|
| `get_funding_status` | read | Wallet USDC vs subaccount collateral, `depositableUsdc`, `suggestedAction`. |
| `deposit_to_subaccount` | write | Wallet → trading subaccount (default: everything above the $0.95 gas reserve). Internal only. |
| `top_up_gas` | write | Subaccount → wallet, only to restore the $0.95 gas reserve when the wallet ≤ $0.55. Cannot send anywhere else. |

## Testnet funds (faucet)

There is **no faucet MCP tool** — on testnet, free funds come from outside the MCP session:

1. Preferred: the user opens the deposit flow in the web app (`whoami.depositUrl`) — it
   includes the testnet faucet for the connected account.
2. Scripted (REST, testnet only, ~1 request/hour per address per denom; accepts either
   address form from `whoami` — `rit1…` or `0x…`, both share one rate-limit counter):

```
POST https://faucet.testnet.rubin.trade/faucet/tokens        {"address":"rit1… or 0x…"}   # test USDC
POST https://faucet.testnet.rubin.trade/faucet/native-token  {"address":"rit1… or 0x…"}   # gas token
```

Faucet funds land in the **wallet** (bank balance), not the trading subaccount. The full
flow is: faucet → wallet → `deposit_to_subaccount` → trade. If balance still shows 0
after the faucet, run `get_funding_status` — it tells you what to do next. On mainnet
there is no faucet; deposits come through the bridge (see https://docs.rubin.trade).

## Profile & programs

| Tool | Purpose |
|---|---|
| `get_leaderboard` | PnL leaderboard per time span, includes own row. |
| `get_my_rank` | "N of M" across ONE_DAY…ALL_TIME. |
| `get_fee_tier` | Current fee tier, 30d volume, staking discount, next tier + full table. |
| `get_referral_program` | Referral code + link, affiliate status/tier/earnings. |

## Prompts (guided flows)

`check_portfolio` · `scan_markets(focus)` · `protect_positions` · `enter_trade(market,
direction, sizeUsd)` · `flatten_all` · `assess_situation(focus)` · `my_rank` · `my_fees` ·
`referral_program` · `fund_account` · `my_account` · `latest_news(query)`

Notable behaviors baked into the prompts:

- `scan_markets` / `assess_situation` — analysis only, never place orders.
- `protect_positions` — proposes SL/TP brackets, places only on approval.
- `flatten_all` — shows what will close, confirms, retries unfilled with larger slippage.
- `fund_account` — `get_funding_status` → `deposit_to_subaccount`; never suggests re-auth.
- `my_account` — reports both address forms and what the session cannot do.
