# Connecting to the Rubin MCP server

| Network | MCP endpoint | Web app | Indexer REST | EVM RPC |
|---|---|---|---|---|
| Testnet | `https://mcp.testnet.rubin.trade/mcp` | https://testnet.rubin.trade | `https://indexer.testnet.rubin.trade/v4` | `https://evm-rpc.testnet.rubin.trade` |
| Mainnet | `https://mcp.mainnet.rubin.trade/mcp` | https://rubin.trade | `https://indexer.mainnet.rubin.trade/v4` | `https://evm-rpc.rubin.trade` |

The network is pinned per deployment — it is **not** part of the credential. Docs:
https://docs.rubin.trade

## Option A — OAuth (claude.ai, Claude Desktop, ChatGPT, Claude Code)

Hosts that support MCP OAuth discover it automatically
(`/.well-known/oauth-protected-resource`, `/.well-known/oauth-authorization-server`,
dynamic client registration). Flow:

1. Add the MCP URL as a custom connector / remote MCP server.
2. The consent page redirects to the Rubin web app's authorize page.
3. The user connects a wallet and approves — the web app issues a **scoped trading key**
   for that account and completes the flow. The private key travels in a POST body,
   never in a URL, and never enters the model context.

Claude Code:

```bash
claude mcp add --transport http rubin https://mcp.testnet.rubin.trade/mcp
```

then run `/mcp` to authenticate.

## Option B — static Bearer credential (headless, API, Cursor, mcp-remote)

The web app's Trading Keys dialog issues a credential — `base64(JSON)` of:

```json
{ "tradingPrivateKey": "<hex, omit for read-only>", "masterAddress": "rit1…" }
```

Send it as `Authorization: Bearer <base64>` on MCP initialize. Examples:

- Anthropic Messages API: `mcp_servers: [{ url, authorization_token }]`
- Cursor / VS Code `mcp.json`: `"headers": { "Authorization": "Bearer <base64>" }`
- Any stdio host: `npx mcp-remote https://mcp.testnet.rubin.trade/mcp --header "Authorization: Bearer <base64>"`

Omit `tradingPrivateKey` for a read-only session (market data + account reads).

## What the key can and cannot do

The trading key is registered on-chain (x/accountplus authenticator) with scope:
**place order / cancel order / batch cancel, subaccount 0 only**. It physically
**cannot withdraw or transfer funds** — that boundary is enforced by the chain, not the
server. One authorization = one account; to trade from another wallet, authorize the
connector again from that wallet (a new authorization = a new MCP key).

## EVM signatures

Accounts are dual-form: the same 20 bytes as `0x…` (EVM) and `rit1…` (bech32). Trading
keys can be authorized by an **EVM wallet signature** — the web app uses an EIP-712
typed message (`RubinTransaction:ApproveAgent`) and registers an
`EthAddressSignatureVerification` authenticator, so MetaMask-style wallets work without
any chain-native tooling. Agents should always report both address forms (both are returned
by `whoami`).

## Troubleshooting

- **Tool call returns AUTH error** — the session has no credential: complete OAuth or
  supply the Bearer header. This is the only case where re-authorizing helps.
- **Low/zero balance** — never an auth problem. `get_funding_status` →
  `deposit_to_subaccount`.
- **Tool list looks outdated** — the host cached an older catalog; reconnect the server.
- **Testnet funds** — deposit flow in the web app (`whoami.depositUrl`) includes the
  faucet; scripted: `POST https://faucet.testnet.rubin.trade/faucet/tokens
  {"address":"rit1… or 0x…"}` (either address form works — both map to one account and
  one rate-limit counter).
