# long-term-rental-analyzer (Claude Agent Skill)

Long-term-rental after-tax analyzer: 27.5-yr depreciation shelter, ~25% recapture at sale, 1031 like-kind deferral, and a Schedule E P&L (rent, vacancy, opex, NOI). Thin orchestration over the planfi MCP.

### Ask Claude things like…

- "Analyze a $500k long-term rental — year-1 Schedule E P&L and depreciation shelter."
- "How much tax do I owe when I sell my rental after 10 years?" (§1250 recapture + LTCG + NIIT)
- "Can I deduct this year's rental loss, or is it suspended?" (§469 / `analyze_passive_losses`)
- "Should I do a cost-segregation study to front-load depreciation?" (`analyze_cost_segregation`)
- "Can I do a 1031 exchange to defer the gain on my rental sale — what are the 45/180-day deadlines, the boot, my carryover basis, and is deferring worth it vs paying the tax now?" (`analyze_1031_exchange`)

It's a **thin orchestration layer** over the public **planfi MCP** (`https://ai.planfi.app/mcp/free`,
public, no auth) — all the math and financial logic live server-side. The skill itself bundles no
engine; it just gathers inputs and calls the tools.

### MCP bootstrap (one time)

If the planfi tools aren't connected yet, run:

```
claude mcp add --transport http planfi https://ai.planfi.app/mcp/free
```

> **Try free, then add your key.** The command above adds the **free** connector — `https://ai.planfi.app/mcp/free` (no key needed). Once you create an API key, add a **new** connector with the MCP url — `https://ai.planfi.app/mcp` — and authorize it with your key.

On **claude.ai**: Settings → Connectors → add a custom connector pointing at
`https://ai.planfi.app/mcp/free` (no auth). The skill also reminds you to do this if the tools are
missing when you invoke it.

## Install

### Quickest — skills.sh CLI (recommended)

```
npx skills add holdequity/planfi-long-term-rental-analyzer
```

### Claude Code — copy the folder (zero prerequisites)

User-level (available in every project):

```
cp -r skills/long-term-rental-analyzer ~/.claude/skills/
```

Or project-level (only this repo/project):

```
mkdir -p .claude/skills && cp -r skills/long-term-rental-analyzer .claude/skills/
```

Restart Claude Code (or start a new session). The skill auto-loads by its description.

### Claude Code — as a plugin (shareable one-liner)

```
/plugin marketplace add holdequity/planfi-long-term-rental-analyzer
/plugin install long-term-rental-analyzer@planfi
```

### claude.ai — upload as a skill

1. Zip the folder: `cd skills && zip -r long-term-rental-analyzer.zip long-term-rental-analyzer`
2. In claude.ai go to Settings → Capabilities → Skills and upload the zip.

## Notes & honest caveats

- All decimals are fractions (24% → `0.24`); all figures are today's (real, inflation-adjusted)
  dollars; tax brackets/limits are approximate ~2026 values.
- The skill surfaces every assumption the server reports (`disclosures.key_assumptions`) so you can
  correct any silent default.
- Not financial advice. Planning estimates only.

See `SKILL.md` for the full instructions, exact tool params, and output format.
Source + issues: <https://github.com/holdequity/planfi-long-term-rental-analyzer>.

## Use it in any MCP client (not just Claude Code)

This skill is Claude Code packaging — but the engine is a standard
[Model Context Protocol](https://modelcontextprotocol.io) server at
`https://ai.planfi.app/mcp/free` (Streamable HTTP, no auth). Connect it from any
MCP-capable agent and you get the same PlanFi tools directly. Every tool is
**self-orchestrating** (it reports its own assumed defaults and suggests the
next step), so it works well even without the skill wrapper.

Most clients take an `mcpServers` config block:

```json
{
  "mcpServers": {
    "planfi": { "type": "http", "url": "https://ai.planfi.app/mcp/free" }
  }
}
```

| Client | How to add it |
|--------|---------------|
| **Claude Code** | `claude mcp add --transport http planfi https://ai.planfi.app/mcp/free` |
| **Cursor** | add the block above to `~/.cursor/mcp.json` (field: `url`) |
| **Windsurf** | `~/.codeium/windsurf/mcp_config.json` (field: `serverUrl`) |
| **Cline / VS Code** | paste the block into the Cline MCP settings |
| **Claude Desktop & stdio-only clients** | bridge with `npx -y mcp-remote https://ai.planfi.app/mcp/free` |
| **ChatGPT (custom connectors / Deep Research)** | add a connector pointing at the MCP URL |
| **Custom / your own agent** | plain MCP Streamable HTTP — POST JSON-RPC `tools/list` / `tools/call` to the URL with `Accept: application/json, text/event-stream` |

Field names vary slightly by client and version — check your client's MCP docs;
the URL is always `https://ai.planfi.app/mcp/free`.

## License

MIT — see [LICENSE](./LICENSE).
