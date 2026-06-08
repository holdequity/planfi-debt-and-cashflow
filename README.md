# debt-and-cashflow (Claude Agent Skill)

Pay down debt and route surplus cash optimally: avalanche/snowball debt payoff order, debt consolidation / balance-transfer break-even, mortgage prepay vs invest, refinance break-even, a next-best-dollar funding waterfall, and an optimal student-loan path (IDR / PSLF / refinance vs aggressive payoff). Thin orchestration over the planfi MCP.

- **Debt consolidation** — `analyze_debt_consolidation`: weighs rolling high-APR revolving (credit-card) debt into a personal loan or a 0% balance transfer (new rate + fees + intro period) against the status quo, holding the monthly budget constant; returns months-to-debt-free, total interest + fees, and the recommended option.
- **Student loans** — `analyze_student_loans`: compares standard, income-driven repayment (IDR), PSLF forgiveness, refinance, and aggressive payoff; returns total cost + payoff/forgiveness timeline per strategy and the recommended path.

It's a **thin orchestration layer** over the public **planfi MCP** (`https://ai.planfi.app/mcp`,
public, no auth) — all the math and financial logic live server-side. The skill itself bundles no
engine; it just gathers inputs and calls the tools.

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp
```

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-debt-and-cashflow
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/debt-and-cashflow ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/debt-and-cashflow .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-debt-and-cashflow
/plugin install debt-and-cashflow@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r debt-and-cashflow.zip debt-and-cashflow`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports — each tool returns a structured
  `assumed_defaults[]` array (`{ field, assumed_value, note }`) so you can correct any silent default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-debt-and-cashflow>.

## Use it in any MCP client (not just Claude Code)

This skill is Claude Code packaging — but the engine is a standard
[Model Context Protocol](https://modelcontextprotocol.io) server at
`https://ai.planfi.app/mcp` (Streamable HTTP, no auth). Connect it from any
MCP-capable agent and you get the same PlanFi tools directly. Every tool is
**self-orchestrating** (it reports its own assumed defaults and suggests the
next step), so it works well even without the skill wrapper.

Most clients take an `mcpServers` config block:

```json
{
  "mcpServers": {
    "planfi": { "type": "http", "url": "https://ai.planfi.app/mcp" }
  }
}
```

| Client | How to add it |
|--------|---------------|
| **Claude Code** | `claude mcp add --transport http planfi https://ai.planfi.app/mcp` |
| **Cursor** | add the block above to `~/.cursor/mcp.json` (field: `url`) |
| **Windsurf** | `~/.codeium/windsurf/mcp_config.json` (field: `serverUrl`) |
| **Cline / VS Code** | paste the block into the Cline MCP settings |
| **Claude Desktop & stdio-only clients** | bridge with `npx -y mcp-remote https://ai.planfi.app/mcp` |
| **ChatGPT (custom connectors / Deep Research)** | add a connector pointing at the MCP URL |
| **Custom / your own agent** | plain MCP Streamable HTTP — POST JSON-RPC `tools/list` / `tools/call` to the URL with `Accept: application/json, text/event-stream` |

Field names vary slightly by client and version — check your client's MCP docs;
the URL is always `https://ai.planfi.app/mcp`.

## License

MIT — see [LICENSE](./LICENSE).
