---
name: research
description: "Autonomous deep research on any topic. Combines Exa semantic search, Firecrawl web scraping, and AlphaXiv paper analysis into a single research workflow. Use when: research this, find out about, what's the latest on, deep dive into, investigate, analyze the landscape of, compare approaches to, literature review, state of the art, how does X work. Produces a structured research report with citations."
---

# Autonomous Research

Deep research on any topic — fully autonomous. Delegates search and scraping to subagents so the main context stays clean for synthesis.

## When to Use

**Use this skill** when the user needs:
- Deep research on any topic (technical, business, academic, market)
- Literature reviews or state-of-the-art analysis
- Technology landscape comparisons
- Understanding how something works with authoritative sources
- Fact-checked answers backed by primary sources
- Investigation that requires reading multiple sources and synthesizing

**Do NOT use** for:
- Simple factual questions (just use exa answer)
- Scraping a single known URL (just use firecrawl)
- Looking up library docs (just use context7)

---

## Setup

### Dependencies

This skill requires two sibling skills for search and scraping. Install them first:
```bash
npx skills add edxeth/superlight-exa-skill
npx skills add edxeth/superlight-firecrawl-skill
```

### Environment Variables

Set these in your shell profile (`~/.bashrc`, `~/.zshrc`, or equivalent):
```bash
export EXA_API_KEY="your-key"              # Get at: https://dashboard.exa.ai/api-keys
export FIRECRAWL_API_KEY="your-key"        # Get at: https://firecrawl.dev/
# Both support comma-separated keys for rotation: "key1,key2,key3"
```

### Tool Paths

All script paths use shell variables so the skill works on **any machine** regardless of OS or install location.

**Resolve these paths at the start of every research session:**
```bash
# Auto-detect skill install directory (works on macOS, Linux, WSL)
RESEARCH_DIR="$(cd "$(dirname "$(readlink -f "${BASH_SOURCE[0]}" 2>/dev/null || echo "${BASH_SOURCE[0]}")")" && pwd)"
SKILLS_DIR="$(dirname "$RESEARCH_DIR")"
EXA="$SKILLS_DIR/exa/scripts/exa.sh"
FIRECRAWL="$SKILLS_DIR/firecrawl/scripts/firecrawl.sh"
ALPHAXIV="$RESEARCH_DIR/scripts/alphaxiv.sh"
```

If auto-detect fails, fall back to the standard install location:
```
~/.claude/skills/exa/scripts/exa.sh
~/.claude/skills/firecrawl/scripts/firecrawl.sh
~/.claude/skills/research/scripts/alphaxiv.sh
```

---

## Architecture: Subagent-Delegated Research

The main model **never runs search or scraping tools directly**. Instead, it scopes the research, then delegates all tool-heavy work to subagents. Each subagent has its own context window — when it finishes, only its compact summary enters the main context. All raw search results and scraped page content stay in the subagent's context and are discarded.

This keeps the main context clean for synthesis, regardless of how many sources are consulted.

```
Main Model (director — scopes, delegates, synthesizes)
│
├─ Phase 1: Scope the research (main model, no tools needed)
│
├─ Phase 2+3: Research subagents (parallel, tool-equipped)
│   ├─ Subagent A → searches + scrapes sub-question 1 → returns findings
│   ├─ Subagent B → searches + scrapes sub-question 2 → returns findings
│   └─ Subagent C → searches + scrapes sub-question 3 → returns findings
│   (all raw tool output lives and dies in subagent contexts)
│
├─ Phase 4: Synthesize report (main model — works from compact findings only)
│
└─ Phase 5: Verify claims (main model, or one verification subagent)
```

### Spawning Research Subagents

Your harness determines how to spawn subagents. Use whichever tool your environment provides:

**Claude Code:**
```
Agent(prompt="<research task>", model="sonnet")
```

**Pi Agent:**
```
subagent(name="research-1", task="<research task>", agent="scout", tools="read,bash")
```

**OpenCode:**
```
Task(prompt="<research task>")
```

**If your harness has no subagent tool:** Fall back to running tools directly in the main context. Use fewer searches (5 instead of 10 per strategy) and scrape only top 3-5 sources to manage context size.

---

## Protocol

### Phase 1: Scope the Research

The main model does this directly — it's cheap and requires no tools.

1. **Core question** — One sentence. What exactly are we trying to answer?
2. **Sub-questions** — 3-5 specific angles that together answer the core question
3. **Source types needed** — Which matter for this topic:
   - Academic papers (arxiv, research)
   - Technical docs / specs
   - Industry analysis / news
   - Code repositories / implementations
   - Expert blogs / practitioner knowledge
   - Company/product pages
4. **Depth** — Quick scan (3-5 sources) or deep dive (10-20+ sources)

Do NOT skip scoping. Bad research starts with vague questions.

---

### Phase 2+3: Research via Subagents

For each sub-question, spawn a subagent with this prompt template. Run them **in parallel** when possible.

**Subagent prompt template:**

```
You are a research agent. Your job is to search for and extract information on ONE specific question, then return a compact summary. You have access to bash tools for web search and scraping.

QUESTION: [sub-question from Phase 1]
SOURCE TYPES TO PRIORITIZE: [from Phase 1 scoping]

TOOLS AVAILABLE (use via bash):
- Exa search:    $EXA search "<query>" <numResults> [category]
                  Categories: company, research paper, news, pdf, github, tweet
- Exa answer:    $EXA answer "<question>"
- Exa similar:   $EXA similar "<url>" <numResults>
- Exa code:      $EXA code "<query>"
- Firecrawl:     $FIRECRAWL scrape "<url>"
- Firecrawl:     $FIRECRAWL search "<query>" <limit>
- Firecrawl:     $FIRECRAWL extract "<url>" "<what to extract>"
- Firecrawl:     $FIRECRAWL map "<docs-url>" <limit>
- AlphaXiv:      $ALPHAXIV overview "<paper-id-or-arxiv-url>"
- AlphaXiv:      $ALPHAXIV search "<query>" <numResults>

INSTRUCTIONS:
1. Run at least 3 different search strategies (exa search, exa with category filter, firecrawl search, exa answer, exa code — pick what fits the question)
2. From all results, pick the top 3-5 most relevant URLs
3. Scrape/read those sources using firecrawl scrape (or alphaxiv for arxiv papers — NEVER scrape arxiv with firecrawl)
4. After reading your best source, run: $EXA similar "<best-source-url>" 5 — to find related high-quality sources search may have missed
5. Extract key claims, data points, and quotes WITH their source URLs

RETURN FORMAT (this is all the main model will see — be thorough but compact):
---
## Findings: [sub-question]

### Key Claims
- [Claim 1] — source: [Title](URL)
- [Claim 2] — source: [Title](URL)
- [Claim 3] — source: [Title](URL)

### Contradictions Found
- [Source A says X, but Source B says Y]

### Best Sources (ranked)
1. [Title](URL) — [authority: official-docs|peer-reviewed|industry|blog|forum|code] — [why it's good]
2. [Title](URL) — [authority] — [why it's good]
3. [Title](URL) — [authority] — [why it's good]

### Gaps
- [What you couldn't find or verify]
---
```

**Adjust the subagent prompt based on source type:**
- For academic topics: emphasize `$ALPHAXIV search` and `$EXA search ... "research paper"`
- For code/implementations: emphasize `$EXA code` and `$EXA search ... "github"`
- For market/industry: emphasize `$FIRECRAWL search`, `$EXA search ... "company"`, `$EXA search ... "news"`
- For docs/specs: use `$FIRECRAWL map` to discover pages, then `$FIRECRAWL scrape` selectively

---

### Phase 4: Synthesize — Build the Report

The main model now has compact findings from each subagent — no raw search results, no full page scrapes. Synthesize them into the final report.

**Report structure:**

```markdown
# Research: [Core Question]

## TL;DR
[Executive summary answering the core question. MUST be under 100 words. Be direct and opinionated, not hedging.]

## Key Findings

### [Sub-question 1]
[Synthesized answer with inline citations like [1], [2]]

### [Sub-question 2]
[Synthesized answer]

### [Sub-question 3]
[Synthesized answer]

## Landscape / Comparison
[If applicable — table or structured comparison of approaches, tools, methods]

## Open Questions
[What remains unclear or contested across sources]

## Contradictions & Disputes
[MANDATORY section. Collate contradictions from all subagent findings. If all sources agree, state that explicitly and explain why consensus exists.]

## Sources
[1] [Title](URL) — [authority: official-docs|peer-reviewed|industry|blog|forum|code] — [one-line description]
[2] [Title](URL) — [authority: ...] — [one-line description]
...
```

---

### Phase 5: Verify — Cross-Check Claims

Before delivering, verify critical claims:

1. **Contradiction check** — Did any subagents report conflicting information? Note it explicitly.
2. **Recency check** — Are any sources outdated? Flag if the field moves fast.
3. **Authority check** — Are sources authoritative (official docs, peer-reviewed, established practitioners) or random blog posts? Weight accordingly.
4. **Gap check** — Did any subagent report gaps? Is there a sub-question with weak coverage? Say so explicitly rather than guessing.

If gaps are critical, spawn one more targeted subagent to fill them.

---

## Autonomy Rules

1. **Never stop to ask permission between searches.** The user wants research, not a play-by-play. Scope, dispatch subagents, synthesize, deliver.
2. **Follow the trail.** Subagents must run `$EXA similar` on their best source. This is mandatory, not optional.
3. **Diversify sources.** Each subagent should use multiple search strategies. Don't rely on one tool — use exa AND firecrawl.
4. **Prefer primary sources.** Official docs > blog summaries. Research papers > news articles about research papers. Code > descriptions of code.
5. **Note uncertainty.** If sources conflict or you can't verify a claim, say so. Never present uncertain information as fact.
6. **Stay focused.** Follow interesting tangents only if they serve the core question. Research is not browsing.
7. **No minimum source count theater.** Don't pad the report with weak sources just to hit a number. 5 excellent sources beat 15 mediocre ones.

---

## Error Handling

| Problem | Action |
|---------|--------|
| Subagent tool not available | Fall back to running tools directly. Use fewer searches to manage context. |
| Exa returns no results | Rephrase query. Try broader terms. Try without category filter. |
| Firecrawl scrape fails | Try different URL format. Try the page's cached/archived version. Move on to next source. |
| AlphaXiv 404 | Try `/abs/` endpoint. If both fail, scrape the arxiv abstract page directly with firecrawl. |
| Rate limited (429) | Scripts auto-retry with key rotation. If all keys exhausted, wait and continue. |
| Topic too broad | Narrow the core question. Research one specific angle first, then expand. |
| Topic too niche | Broaden search terms. Try adjacent topics. Use `similar` to find related content from any relevant source you find. |
| Conflicting sources | Report the conflict explicitly. Note which sources are more authoritative and why. |
| Subagent returns thin results | Spawn a follow-up subagent with a rephrased question or broader search terms. |

---

## Examples

### Example 1: Technical research
**User:** "Research the current state of Elliott Wave analysis algorithms"
- Scope: 3 sub-questions (academic approaches, code implementations, practitioner tools)
- Subagent A: `$EXA search "Elliott Wave" "research paper"` + `$ALPHAXIV search` → academic findings
- Subagent B: `$EXA code "Elliott Wave algorithm"` + `$FIRECRAWL scrape` top repos → implementation findings
- Subagent C: `$EXA search "Elliott Wave trading software"` + `$FIRECRAWL search` → practitioner findings
- Main model: Synthesize into landscape report comparing approaches

### Example 2: Market research
**User:** "What's the landscape of AI code review tools?"
- Scope: 3 sub-questions (products & pricing, technical approaches, user reception)
- Subagent A: `$EXA search ... "company"` + `$FIRECRAWL extract` pricing → product findings
- Subagent B: `$EXA search` technical blogs + `$FIRECRAWL scrape` → technical findings
- Subagent C: `$EXA search ... "news"` + `$FIRECRAWL search` reviews → reception findings
- Main model: Synthesize into comparison table + analysis

### Example 3: How-does-it-work
**User:** "How does Raft consensus work?"
- Scope: 3 sub-questions (core algorithm, implementations, trade-offs vs alternatives)
- Subagent A: `$ALPHAXIV overview "raft-paper-id"` + `$EXA answer` → core algorithm
- Subagent B: `$EXA code "raft consensus"` + `$FIRECRAWL scrape` → implementations
- Subagent C: `$EXA search "raft vs paxos vs pbft"` → comparisons
- Main model: Synthesize into technical explanation

---

## The Test

A good research report:

1. **Answers the question** — The TL;DR directly addresses what was asked
2. **Cites every claim** — No unsourced assertions
3. **Uses primary sources** — Not just summaries of summaries
4. **Notes disagreements** — Sources that conflict are flagged, not hidden
5. **Admits gaps** — Unknown things are stated as unknown
6. **Is original synthesis** — Not copy-paste from any single source
7. **Was autonomous** — Did not stop to ask the user mid-research
8. **Kept context clean** — Used subagents so the main model had room to think
