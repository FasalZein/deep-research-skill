# Autonomous Research Skill

Deep research on any topic — fully autonomous. Combines [Exa](https://exa.ai) semantic search, [Firecrawl](https://firecrawl.dev) web scraping, and [AlphaXiv](https://alphaxiv.org) paper analysis into a single research workflow that produces structured reports with citations.

## Install

```bash
npx skills add edxeth/autonomous-research-skill
```

## What It Does

Give it any research question and it will:

1. **Scope** — Define sub-questions and source types needed
2. **Discover** — Run 3-5 parallel search strategies (semantic, category-filtered, code, news)
3. **Deep Read** — Scrape primary sources, fetch arxiv papers via AlphaXiv, follow trails with `exa similar`
4. **Synthesize** — Produce a structured report with inline citations, comparison tables, and explicit contradictions
5. **Verify** — Cross-check claims, flag authority levels, note gaps

## Usage

```
/research
```

Then describe what you want researched:
- "Research the current state of Elliott Wave analysis algorithms"
- "What's the landscape of AI code review tools in 2026?"
- "Deep dive into WebSocket vs SSE for real-time data streaming"
- "Literature review on neural network approaches to time series forecasting"

## Report Output

Every report includes:

| Section | Description |
|---------|-------------|
| **TL;DR** | <100 word executive summary, direct and opinionated |
| **Key Findings** | Synthesized answers per sub-question with inline citations |
| **Landscape / Comparison** | Table comparing approaches, tools, or methods |
| **Contradictions & Disputes** | Explicit disagreements between sources |
| **Open Questions** | What remains unclear or contested |
| **Sources** | Numbered with authority tags (peer-reviewed, official-docs, industry, blog, code) |

## Requirements

This skill uses bash scripts from these companion skills for search and scraping:

- **Exa** — [`edxeth/superlight-exa-skill`](https://github.com/edxeth/superlight-exa-skill) — Semantic web search
- **Firecrawl** — [`edxeth/superlight-firecrawl-skill`](https://github.com/edxeth/superlight-firecrawl-skill) — Web scraping and crawling

Install them:
```bash
npx skills add edxeth/superlight-exa-skill
npx skills add edxeth/superlight-firecrawl-skill
```

### Environment Variables

```bash
export EXA_API_KEY="your-key"          # https://dashboard.exa.ai/api-keys
export FIRECRAWL_API_KEY="fc-your-key" # https://firecrawl.dev
```

AlphaXiv requires no API key.

## How It Was Built

This skill was developed using the [autoresearch methodology](https://github.com/karpathy) — autonomous experimentation loops that:

1. Establish a baseline by running the skill against diverse test inputs
2. Score outputs against binary evals (pass/fail, no scales)
3. Mutate the prompt to fix failures
4. Keep mutations that improve scores, discard the rest

The autoresearch loop identified 5 key improvements over the initial version:
- Mandatory AlphaXiv for arxiv papers (was silently skipped)
- Mandatory trail-following via `exa similar` (surfaces sources search alone misses)
- Source authority tags on every citation
- Concise TL;DR (<100 words, opinionated)
- Mandatory contradictions section (forces critical analysis over summarization)

## License

MIT
