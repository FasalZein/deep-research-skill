# Autonomous Research Skill

Deep research on any topic — fully autonomous. Combines [Exa](https://exa.ai) semantic search, [TinyFish](https://tinyfish.ai) web search/fetch, [Firecrawl](https://firecrawl.dev) Markdown scraping/crawling/extraction, and [AlphaXiv](https://alphaxiv.org) paper analysis into structured cited reports.

Uses **subagent delegation** to keep the main context clean: subagents do search/fetch/scrape work and return compact findings, while the main model scopes, verifies, and synthesizes.

Built on top of the excellent Exa, TinyFish, and Firecrawl superlight skills by [@edxeth](https://github.com/edxeth).

## Supported Harnesses

| Harness | Subagent Tool | Status |
|---------|--------------|--------|
| **Claude Code** | `Agent` tool | Full support |
| **Pi Agent** | `subagent` tool | Full support |
| **OpenCode** | `Task` tool | Full support |
| **Others** | Direct execution fallback | Works, no context isolation |

## Install

Install this skill and its search/fetch/scrape dependencies:

```bash
npx skills add edxeth/superlight-exa-skill
npx skills add edxeth/superlight-tinyfish-skill
npx skills add edxeth/superlight-firecrawl-skill
npx skills add FasalZein/autonomous-research-skill
```

After restarting your session, the `/research` command is available.

### Manual Install

```bash
git clone https://github.com/FasalZein/autonomous-research-skill.git ~/.claude/skills/research
git clone https://github.com/edxeth/superlight-exa-skill.git ~/.claude/skills/exa
git clone https://github.com/edxeth/superlight-tinyfish-skill.git ~/.claude/skills/tinyfish
git clone https://github.com/edxeth/superlight-firecrawl-skill.git ~/.claude/skills/firecrawl
```

## Environment Variables

You need API keys for Exa, TinyFish, and Firecrawl. AlphaXiv requires no key.

```bash
export EXA_API_KEY="your-key"              # https://dashboard.exa.ai/api-keys
export TINYFISH_API_KEY="your-key"         # https://agent.tinyfish.ai/api-keys
export FIRECRAWL_API_KEY="fc-your-key"     # https://firecrawl.dev

# All support comma-separated keys for rotation where the underlying skill supports it:
export EXA_API_KEY="key1,key2,key3"
export TINYFISH_API_KEY="key1,key2,key3"
export FIRECRAWL_API_KEY="fc-key1,fc-key2"
```

### Research Artifacts

For multi-angle research (2+ subagents), each subagent writes its findings to a temp directory to keep the parent context clean. By default this uses `$TMPDIR` (per-user temp on macOS, `/tmp` on Linux). The path is printed at the end of each research run.

To use a persistent location instead:

```bash
export RESEARCH_ARTIFACT_DIR="$HOME/research"
```

## Usage

```text
/research <topic>
```

Examples:

```text
/research the current state of Elliott Wave analysis algorithms
/research what's the landscape of AI code review tools in 2026
/research deep dive into WebSocket vs SSE for real-time data streaming
/research how does Raft consensus handle leader election
```

The skill runs autonomously: it scopes the question, launches subagents when useful, chains tools, verifies contradictions, and returns a cited report.

## How It Works

### Architecture: Subagent-Delegated Research

The main model is the research director. It scopes/refines the question, resolves tool paths once, spawns subagents for independent angles, and synthesizes only compact findings.

```text
Main Model (director)
├─ Scope/refine the question and source classes
├─ Preflight: resolve Exa/TinyFish/Firecrawl/AlphaXiv paths + API keys
├─ Subagents: search, fetch, scrape, crawl, deep-read sources
├─ Synthesis: merge findings, dedupe sources, weight authority/recency
└─ Verify: contradictions, gaps, and final cited report
```

Why subagents? Deep research creates lots of raw search and page content. Delegation keeps that raw output out of the main context so the final synthesis has room to think.

### Tool Chain

| Stage | Tool | Best For |
|------|------|----------|
| Search/discovery | Exa | Semantic discovery, authoritative sources, similar-page trails, code/paper search |
| Fetch/extract content | TinyFish | Broad web fetch, JS-rendered pages, batching URLs into clean Markdown |
| Markdown/crawl/map | Firecrawl | Known URL scraping, docs/site maps, recursive crawl, structured JSON extraction |
| Papers | AlphaXiv + Exa | Arxiv discovery and paper overviews |

Default flow: **Exa search → TinyFish fetch top URLs → Firecrawl map/scrape/crawl when needed → Exa similar trail → synthesize**.

### How Many Subagents?

Do not always split into three.

- **Focused question** → 1 subagent.
- **Multi-angle landscape** → 2-3 subagents, one per independent angle.
- **Broad survey** → 3-5 subagents, one per source class or thesis.
- **Unresolved contradiction** → 1 targeted gap-checker.

Split by independent angle, not by tool.

### Direct Execution Fallback

If a harness has no subagent tool, run the same chain in the main context with reduced scope: use 1-2 focused searches, fetch/scrape only the top 3-5 sources, summarize evidence as you go, and avoid pasting raw page output into the final report.

### Report Output

Every report includes:

| Section | Description |
|---------|-------------|
| **TL;DR** | Direct answer under 100 words |
| **Key Findings** | Synthesized themes with inline citations `[1]`, `[2]` |
| **Landscape / Comparison** | Optional table comparing approaches/tools/methods |
| **Contradictions & Disputes** | Mandatory conflicts/consensus section |
| **Open Questions** | Known gaps |
| **Sources** | Numbered, authority-tagged sources |

## File Structure

```text
autonomous-research-skill/
└── skills/
    └── research/
        ├── SKILL.md              # Complete research protocol
        └── scripts/
            └── alphaxiv.sh       # AlphaXiv paper lookup (no API key needed)

Dependencies (installed separately):
├── exa/scripts/exa.sh
├── tinyfish/scripts/tinyfish.sh
└── firecrawl/scripts/firecrawl.sh
```

## OS Compatibility

Script path guidance supports common skill locations:

- `~/.agents/skills/`
- `~/.claude/skills/`
- Pi agent skill installs where `PI_SKILL_DIR` is available

Works on macOS, Linux, and Windows via WSL.

## Related Skills

- Exa semantic search
- TinyFish web search/fetch
- Firecrawl scraping/crawling/extraction

## License

MIT
