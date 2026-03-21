# Autonomous Research Skill

Deep research on any topic — fully autonomous. Combines [Exa](https://exa.ai) semantic search, [Firecrawl](https://firecrawl.dev) web scraping, and [AlphaXiv](https://alphaxiv.org) paper analysis into a single research workflow that produces structured reports with citations.

Uses **subagent delegation** to keep the main context clean — all search/scrape output stays in subagent contexts, only compact findings return to the main model for synthesis.

Built on top of the excellent [superlight-exa-skill](https://github.com/edxeth/superlight-exa-skill) and [superlight-firecrawl-skill](https://github.com/edxeth/superlight-firecrawl-skill) by [@edxeth](https://github.com/edxeth).

## Supported Harnesses

| Harness | Subagent Tool | Status |
|---------|--------------|--------|
| **Claude Code** | `Agent` tool (built-in) | Full support |
| **Pi Agent** | `subagent` tool (via pi-interactive-subagents) | Full support |
| **OpenCode** | `Task` tool (built-in) | Full support |
| **Others** | Fallback: direct execution with reduced scope | Works, no context isolation |

## Install

Install all three skills (this skill + its two dependencies):

```bash
# 1. Install the search and scraping skills (by edxeth)
npx skills add edxeth/superlight-exa-skill
npx skills add edxeth/superlight-firecrawl-skill

# 2. Install this research skill
npx skills add FasalZein/autonomous-research-skill
```

After restarting your session, the `/research` command is available.

### Manual Install

If you prefer to install manually (or use a different coding harness):

```bash
# Clone into your skills directory
git clone https://github.com/FasalZein/autonomous-research-skill.git ~/.claude/skills/research

# Also install the dependencies
git clone https://github.com/edxeth/superlight-exa-skill.git ~/.claude/skills/exa
git clone https://github.com/edxeth/superlight-firecrawl-skill.git ~/.claude/skills/firecrawl
```

## Environment Variables

You need API keys for Exa and Firecrawl. AlphaXiv is free and requires no key.

```bash
# Add to your shell profile (~/.zshrc, ~/.bashrc, etc.)
export EXA_API_KEY="your-key"          # Get at: https://dashboard.exa.ai/api-keys
export FIRECRAWL_API_KEY="fc-your-key" # Get at: https://firecrawl.dev

# Both support comma-separated keys for automatic rotation on rate limits:
export EXA_API_KEY="key1,key2,key3"
export FIRECRAWL_API_KEY="fc-key1,fc-key2"
```

## Usage

```
/research <topic>
```

Examples:
```
/research the current state of Elliott Wave analysis algorithms
/research what's the landscape of AI code review tools in 2026
/research deep dive into WebSocket vs SSE for real-time data streaming
/research how does Raft consensus work
```

The skill runs fully autonomously — no stopping to ask questions mid-research.

## How It Works

### Architecture: Subagent-Delegated Research

The main model never runs search or scraping tools directly. It scopes the research, spawns subagents equipped with the exa/firecrawl/alphaxiv tools to do the actual searching and reading, then synthesizes their compact findings into the final report.

```
Main Model (director — scopes, delegates, synthesizes)
│
├─ Phase 1: Scope (main model, no tools)
│   Define core question, sub-questions, source types, depth
│
├─ Phase 2+3: Research subagents (parallel, tool-equipped)
│   ├─ Subagent A → searches + scrapes sub-question 1 → returns compact findings
│   ├─ Subagent B → searches + scrapes sub-question 2 → returns compact findings
│   └─ Subagent C → searches + scrapes sub-question 3 → returns compact findings
│   (all raw search/scrape output stays in subagent contexts — never enters main)
│
├─ Phase 4: Synthesize (main model — clean context, only compact findings)
│
└─ Phase 5: Verify claims (cross-check contradictions, recency, authority, gaps)
```

**Why subagents?** A typical research session generates 75K+ characters of raw search results and scraped pages. Without subagents, this fills the main context window, leaving no room for quality synthesis. With subagents, the main model only sees ~2K characters of structured findings per sub-question.

### Tools Available to Subagents

| Tool | Command | Best For |
|------|---------|----------|
| Exa search | `$EXA search "<query>" <n> [category]` | General discovery |
| Exa answer | `$EXA answer "<question>"` | Quick factual overview |
| Exa similar | `$EXA similar "<url>" <n>` | Trail-following from best source |
| Exa code | `$EXA code "<query>"` | Code and implementations |
| Firecrawl scrape | `$FIRECRAWL scrape "<url>"` | Full page content |
| Firecrawl search | `$FIRECRAWL search "<query>" <n>` | Web search + scrape combo |
| Firecrawl extract | `$FIRECRAWL extract "<url>" "<prompt>"` | Structured JSON extraction |
| Firecrawl map | `$FIRECRAWL map "<url>" <n>` | URL discovery on docs sites |
| AlphaXiv overview | `$ALPHAXIV overview "<paper-id>"` | Arxiv paper summaries |
| AlphaXiv search | `$ALPHAXIV search "<query>" <n>` | Academic paper search |

### Report Output

Every report includes:

| Section | Description |
|---------|-------------|
| **TL;DR** | <100 word executive summary |
| **Key Findings** | Per sub-question with inline citations `[1]`, `[2]` |
| **Landscape / Comparison** | Table comparing approaches, tools, or methods |
| **Contradictions & Disputes** | Explicit disagreements between sources (mandatory) |
| **Open Questions** | What remains unclear or contested |
| **Sources** | Numbered, authority-tagged (`official-docs`, `peer-reviewed`, `industry`, `blog`, `code`) |

## File Structure

```
autonomous-research-skill/
└── skills/
    └── research/
        ├── SKILL.md              # Research protocol with subagent delegation
        └── scripts/
            └── alphaxiv.sh       # AlphaXiv paper lookup (no API key needed)

Dependencies (installed separately):
├── edxeth/superlight-exa-skill       # Exa semantic search
│   └── scripts/exa.sh
└── edxeth/superlight-firecrawl-skill # Firecrawl web scraping
    └── scripts/firecrawl.sh
```

## OS Compatibility

All script paths use auto-detection with `~/.claude/skills/` fallback. Works on:
- macOS
- Linux
- Windows (WSL)

## Related Skills

- [superlight-exa-skill](https://github.com/edxeth/superlight-exa-skill) — Exa AI semantic web search (by [@edxeth](https://github.com/edxeth))
- [superlight-firecrawl-skill](https://github.com/edxeth/superlight-firecrawl-skill) — Firecrawl web scraping and crawling (by [@edxeth](https://github.com/edxeth))

## License

MIT
