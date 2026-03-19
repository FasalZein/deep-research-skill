# Autonomous Research Skill

Deep research on any topic — fully autonomous. Combines [Exa](https://exa.ai) semantic search, [Firecrawl](https://firecrawl.dev) web scraping, and [AlphaXiv](https://alphaxiv.org) paper analysis into a single research workflow that produces structured reports with citations.

Built on top of the excellent [superlight-exa-skill](https://github.com/edxeth/superlight-exa-skill) and [superlight-firecrawl-skill](https://github.com/edxeth/superlight-firecrawl-skill) by [@edxeth](https://github.com/edxeth).

## Install

Install all three skills (this skill + its two dependencies):

```bash
# 1. Install the search and scraping skills (by edxeth)
npx skills add edxeth/superlight-exa-skill
npx skills add edxeth/superlight-firecrawl-skill

# 2. Install this research skill
npx skills add FasalZein/autonomous-research-skill
```

That's it. After restarting your Claude Code session, the `/research` command is available.

### Alternative: Manual Install

If you prefer to install manually (or use a different coding harness):

```bash
# Clone into your skills directory
git clone https://github.com/FasalZein/autonomous-research-skill.git ~/.claude/skills/research

# Also install the dependencies
git clone https://github.com/edxeth/superlight-exa-skill.git ~/.claude/skills/exa
git clone https://github.com/edxeth/superlight-firecrawl-skill.git ~/.claude/skills/firecrawl
```

For other harnesses (Cursor, Windsurf, Codex, etc.), clone into `~/.agents/skills/` instead.

## Environment Variables

You need API keys for Exa and Firecrawl. AlphaXiv is free and requires no key.

```bash
# Add to your shell profile (~/.zshrc, ~/.bashrc, etc.)
export EXA_API_KEY="your-key"          # Get one at: https://dashboard.exa.ai/api-keys
export FIRECRAWL_API_KEY="fc-your-key" # Get one at: https://firecrawl.dev

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
/research literature review on neural network approaches to time series forecasting
/research how does Raft consensus work
```

The skill runs fully autonomously — no stopping to ask questions mid-research. It searches, reads sources, follows trails, and delivers a complete report.

## How It Works

The research skill orchestrates a 5-phase pipeline:

### Phase 1: Scope
Defines the core question, 3-5 sub-questions, what source types are needed (papers, code, news, docs), and target depth (quick scan vs deep dive).

### Phase 2: Discover
Casts a wide net using multiple parallel search strategies:

| Strategy | Tool | Best For |
|----------|------|----------|
| Semantic search | `exa search` | General discovery |
| Category-filtered | `exa search` + category | Papers, GitHub, news, companies |
| AI answer | `exa answer` | Quick factual overview |
| Web search + scrape | `firecrawl search` | Content-rich results |
| Code search | `exa code` | Implementations, repos |

Runs at least 3 strategies per sub-question to avoid tunnel vision.

### Phase 3: Deep Read
Reads the best sources in full — this is where shallow research becomes deep:

| Source Type | Tool | Notes |
|-------------|------|-------|
| Web pages / articles | `firecrawl scrape` | Converts to clean markdown |
| Arxiv papers | `alphaxiv.sh overview` | AI-structured summaries (no API key needed) |
| Related sources | `exa similar` | **Mandatory** — follows the trail from the best source found |
| Structured data | `firecrawl extract` | Returns JSON from pages |
| Site maps | `firecrawl map` | Discovers all URLs before selective reading |

### Phase 4: Synthesize
Compiles findings into a structured report (not copy-paste — original synthesis):

```
# Research: [Core Question]
## TL;DR              → <100 words, direct, opinionated
## Key Findings       → Per sub-question, inline citations [1], [2]
## Landscape          → Comparison table
## Contradictions     → Explicit disagreements between sources
## Open Questions     → What's unclear or contested
## Sources            → Numbered, authority-tagged, with descriptions
```

### Phase 5: Verify
Cross-checks claims before delivery:
- **Contradiction check** — Sources that disagree are flagged
- **Recency check** — Outdated sources are noted
- **Authority check** — Each source tagged: `official-docs`, `peer-reviewed`, `industry`, `blog`, `forum`, `code`
- **Gap check** — Missing answers stated explicitly

## Report Output

Every report includes:

| Section | Description |
|---------|-------------|
| **TL;DR** | <100 word executive summary, direct and opinionated |
| **Key Findings** | Synthesized answers per sub-question with inline citations |
| **Landscape / Comparison** | Table comparing approaches, tools, or methods |
| **Contradictions & Disputes** | Explicit disagreements between sources (mandatory) |
| **Open Questions** | What remains unclear or contested |
| **Sources** | Numbered with authority tags and one-line descriptions |

Example source format:
```
[1] Title (URL) — [authority: peer-reviewed] — Key contribution from this source
[2] Title (URL) — [authority: industry] — What this source added
```

## Architecture

```
autonomous-research-skill/
└── skills/
    └── research/
        ├── SKILL.md              # The research protocol (what the agent follows)
        └── scripts/
            └── alphaxiv.sh       # AlphaXiv paper lookup (search + overview)

Dependencies (installed separately):
├── edxeth/superlight-exa-skill       # Exa semantic search
│   └── scripts/exa.sh               # search, answer, similar, code, contents
└── edxeth/superlight-firecrawl-skill # Firecrawl web scraping
    └── scripts/firecrawl.sh          # scrape, search, map, extract, crawl
```

## AlphaXiv Script

The included `alphaxiv.sh` provides structured AI summaries of arxiv papers:

```bash
# Get a structured overview of a paper
alphaxiv.sh overview 2301.12345
alphaxiv.sh overview https://arxiv.org/abs/2301.12345

# Search for research papers (delegates to Exa with "research paper" category)
alphaxiv.sh search "transformer attention mechanisms" 5
```

Accepts arxiv URLs, alphaxiv URLs, or raw paper IDs. Falls back from overview to full text automatically. No API key required.

## How It Was Built

This skill was optimized using the **autoresearch methodology** (inspired by [Karpathy's autonomous experimentation loops](https://github.com/karpathy)):

1. **Baseline** — Ran the skill 3x against diverse topics (Elliott Wave algorithms, AI code review tools, Raft consensus). Scored against binary evals.
2. **Eval tightening** — Initial evals were too easy (100% baseline). Replaced with harder evals that caught real weaknesses.
3. **Mutation** — Applied targeted changes to the skill prompt based on failure analysis.
4. **Validation** — Ran 3 more times against different topics (WebSocket vs SSE, time series forecasting, Elliott Wave repeat). Scored 100% on harder evals.

### 5 improvements from autoresearch:

| # | Improvement | Why It Mattered |
|---|-------------|-----------------|
| 1 | Mandatory AlphaXiv for arxiv papers | Agents silently skipped it and used firecrawl instead |
| 2 | Mandatory `exa similar` trail-following | Surfaces high-quality sources that search alone misses |
| 3 | Source authority tags | Makes source quality immediately visible in the report |
| 4 | Concise TL;DR (<100 words) | Long summaries are less useful; forces opinionated synthesis |
| 5 | Mandatory contradictions section | Prevents summarization-as-research; forces critical analysis |

## Related Skills

- [superlight-exa-skill](https://github.com/edxeth/superlight-exa-skill) — Exa AI semantic web search (by [@edxeth](https://github.com/edxeth))
- [superlight-firecrawl-skill](https://github.com/edxeth/superlight-firecrawl-skill) — Firecrawl web scraping and crawling (by [@edxeth](https://github.com/edxeth))
- [agent-browser](https://github.com/vercel-labs/agent-browser) — Browser automation for AI agents (by Vercel Labs)
- [skills](https://github.com/vercel-labs/skills) — Discover and install more skills (by Vercel Labs)

## License

MIT
