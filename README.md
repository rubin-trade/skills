<div align="center">

# Rubin Agent Skills

**Official agent skills for [Rubin](https://rubin.trade) — perpetual futures DEX on Rubin Chain.**

[Docs](https://docs.rubin.trade) · [Testnet](https://testnet.rubin.trade) · [MCP endpoint](https://mcp.testnet.rubin.trade/mcp)

</div>

Teach your AI agent (Claude Code, Cursor, Codex, any [Agent Skills](https://agentskills.io)-compatible
host) to trade on Rubin the right way: through the official MCP server, with both EVM `0x…`
and cosmos `rit1…` address forms, correct order semantics, and the testnet faucet.

## Install

```bash
npx skills add rubin-trade/skills
```

Or straight from the docs site, which publishes an
[Agent Skills Discovery](https://github.com/cloudflare/agent-skills-discovery-rfc) manifest:

```bash
npx skills add https://docs.rubin.trade
```

The `skills` CLI needs **Node 20.12+** (it imports `styleText` from `node:util`). On Node 18
it fails with `SyntaxError: The requested module 'node:util' does not provide an export
named 'styleText'` — run it under a newer Node, e.g. `nvm exec 22 npx skills add
rubin-trade/skills`, rather than changing your default Node.

Claude Code plugin (with updates):

```bash
claude plugin marketplace add rubin-trade/skills
claude plugin install rubin-trade@rubin-trade
```

Manual: copy `skills/rubin-trade/` into your agent's skills directory
(e.g. `~/.claude/skills/`).

## Skills

| Skill | What it does |
|---|---|
| [`rubin-trade`](skills/rubin-trade/SKILL.md) | Connect the Rubin MCP server and trade perps: market data, portfolio, orders with TP/SL brackets, funding, fees, leaderboard, testnet faucet. Steers the agent away from raw REST calls and onto the scoped, chain-enforced MCP tools. |

## Why MCP and not raw REST?

The MCP server signs transactions with a **scoped on-chain key** (place/cancel only — it
physically cannot withdraw or transfer funds), verifies fills instead of trusting broadcast
codes, and knows the funding model (wallet vs subaccount, gas reserve). Raw Indexer REST
calls give you none of that. Read-only market data over REST is fine:
`https://indexer.testnet.rubin.trade/v4` (Swagger at `/docs`).

## Connect the MCP server

- **claude.ai / Claude Desktop / ChatGPT** — add `https://mcp.testnet.rubin.trade/mcp` as a
  custom connector; OAuth opens the Rubin web app to authorize a scoped trading key.
- **Claude Code** — `claude mcp add --transport http rubin https://mcp.testnet.rubin.trade/mcp`
- **Headless/API** — static Bearer credential from the web app's Trading Keys dialog.

Details: [skills/rubin-trade/references/connect.md](skills/rubin-trade/references/connect.md)

## License

[MIT](LICENSE)
