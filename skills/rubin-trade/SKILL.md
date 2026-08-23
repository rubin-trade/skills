---
name: rubin-trade
description: >
  Trade perpetual futures on Rubin (rubin.trade), the perps DEX on the RITBIT chain — market
  data, portfolio, orders with TP/SL brackets, funding, fees, leaderboard, referrals, testnet
  faucet. Connects the agent to the official Rubin MCP server instead of raw REST calls.
  Use when the user mentions Rubin, rubin.trade, RITBIT, ritbit chain, wants an AI agent to
  trade or monitor perpetuals, check positions/PnL/balance, get testnet funds, or asks how to
  connect the Rubin MCP connector.
license: MIT
metadata:
  author: rubin-trade
  version: "1.1.1"
---

# Trading on Rubin (rubin.trade)

Rubin is a decentralized perpetual futures exchange on the RITBIT chain (a Cosmos SDK
chain with EVM support). Full documentation: **https://docs.rubin.trade**

## Rule 1 — use the MCP server, not raw REST

All trading and account work goes through the official MCP server. Do **not** hand-roll
requests to the Indexer REST API or the faucet when the MCP connection is available — the
MCP server signs transactions with a scoped on-chain key, verifies fills, and enforces
safety limits that raw REST cannot.

| Network | MCP endpoint (Streamable HTTP) | Web app |
|---|---|---|
| Testnet | `https://mcp.testnet.rubin.trade/mcp` | https://testnet.rubin.trade |
| Mainnet | `https://mcp.mainnet.rubin.trade/mcp` | https://rubin.trade |

If the MCP server is **not** connected yet, help the user connect it — see
[references/connect.md](references/connect.md). Quick versions:

- **claude.ai / Claude Desktop / ChatGPT**: add a custom connector with the MCP URL above;
  OAuth flow opens the Rubin web app, the user picks a wallet and approves a scoped
  trading key. No keys are ever typed into the chat.
- **Claude Code**: `claude mcp add --transport http rubin https://mcp.testnet.rubin.trade/mcp`
  (then authenticate via `/mcp`).
- **Headless / API**: pass `Authorization: Bearer <base64 credential>` obtained from the
  web app's Trading Keys dialog.

## First calls

1. `whoami` — always call first. Returns the bound account, mode (trade vs read-only),
   on-chain scope, operator limits, deposit URL, server version.
2. `list_markets` — discover tickers before quoting or trading anything.

The session is bound to **one** account, fixed at authorization time. It cannot create
wallets, import or reveal mnemonics/private keys, or switch accounts. To trade from
another wallet the user re-authorizes the connector from that wallet in the web app.
Never invent an address, mnemonic, or key.

## Addresses: always both forms, EVM included

Every RITBIT account has two representations of the **same 20 bytes**:

- EVM form: `0x…` (hex)
- Cosmos form: `rit1…` (bech32)

When telling the user their address, **always give both forms** (both come from `whoami`).
Deposits from EVM wallets, block explorers, and EVM tooling use the `0x` form;
chain-native tooling uses `rit1`. When passing an address to the Indexer REST API or the
testnet faucet, send the `rit1…` form — converting `0x` → `rit1` is a mechanical bech32
re-encode of the same bytes: `toBech32('rit', fromHex(hex.slice(2)))`.

The chain accepts EVM (EIP-712) signatures: trading keys can be authorized from an EVM
wallet signature (`RubinTransaction:ApproveAgent`), and an EVM JSON-RPC endpoint is
available (`https://evm-rpc.rubin.trade`, testnet: `https://evm-rpc.testnet.rubin.trade`).

## Trading semantics (what trips agents up)

- **Prefer GTT limit orders.** Stateful order rate limit: **2 per block, 20 per 100
  blocks** — pace placements.
- A "market" order executes as an **IOC limit capped at oracle ± `slippageBps`**
  (default 500), bound mirrored by side. If the confirmation says `unfilled` or
  `partially_filled`, **retry with a larger `slippageBps`** — never conclude "can't close".
- Trust `confirmation.outcome` (`filled / partially_filled / resting / unfilled /
  pending`), not broadcast `code: 0`. `pending` = indexer lag; re-check shortly.
- A filled order leaves **no open order** — it becomes a position. To answer "what's
  open?", call `get_portfolio` (positions + orders + margin risk in one call).
- Liquidation prices from `get_position_risk` are estimates (cross-margin); the hard
  number is `maintenanceMarginBufferUsd`.
- Bulk irreversible actions (`close_all_positions`) — confirm with the user first.

## Funds: wallet vs subaccount

USDC lands in the **wallet** (bank balance); only **subaccount** collateral backs trading.
"I deposited but balance is 0" → `get_funding_status`, then `deposit_to_subaccount`.
Always leave **$0.95** in the wallet for gas (orders themselves are fee-free); if the
wallet drops to ≤ $0.55, `top_up_gas` restores the reserve. A low balance is **never**
an auth problem — do not suggest re-authorizing.

## Testnet funds (faucet)

Preferred: the user opens the deposit flow in the web app (`whoami.depositUrl`) — it
includes the testnet faucet for the connected account. For scripts/CI there is a public
REST faucet (testnet only, rate-limited ~1 request/hour per address per denom). Send the
`rit1…` form of the address:

```
POST https://faucet.testnet.rubin.trade/faucet/tokens        {"address":"rit1…"}   # test USDC
POST https://faucet.testnet.rubin.trade/faucet/native-token  {"address":"rit1…"}   # gas token
```

## Read-only REST fallback

Public market data only, when MCP is unavailable. Base URLs:
`https://indexer.testnet.rubin.trade/v4` / `https://indexer.mainnet.rubin.trade/v4`
(Swagger UI at `/docs`). Anything involving the user's account or orders still requires
the MCP connection. Full API reference: https://docs.rubin.trade

## Going deeper

- [references/tools.md](references/tools.md) — all 33 MCP tools + 12 guided prompts.
- [references/connect.md](references/connect.md) — connection matrix, credential format,
  OAuth flow, key scopes, EVM agent keys.
