---
name: research
description: "Autonomous deep research on any topic. Combines Exa semantic search, Firecrawl web scraping, and AlphaXiv paper analysis into a single research workflow. Use when: research this, find out about, what's the latest on, deep dive into, investigate, analyze the landscape of, compare approaches to, literature review, state of the art, how does X work. Produces a structured research report with citations."
---

# Autonomous Research

Deep research on any topic — fully autonomous, no hand-holding. Searches the web, scrapes primary sources, reads papers, and synthesizes findings into a structured report with citations.

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

## Protocol

### Phase 1: Scope the Research

Before searching, define the research clearly. Write these down:

1. **Core question** — One sentence. What exactly are we trying to answer?
2. **Sub-questions** — 3-5 specific angles that together answer the core question
3. **Source types needed** — Which of these matter for this topic:
   - Academic papers (arxiv, research)
   - Technical docs / specs
   - Industry analysis / news
   - Code repositories / implementations
   - Expert blogs / practitioner knowledge
   - Company/product pages
4. **Depth** — Quick scan (3-5 sources) or deep dive (10-20+ sources)

Do NOT skip scoping. Bad research starts with vague questions.

---

### Phase 2: Discovery — Cast a Wide Net

Run multiple searches in parallel to find candidate sources. Use different query strategies to avoid tunnel vision.

**Strategy 1: Direct semantic search**
```bash
/Users/tothemoon/.claude/skills/exa/scripts/exa.sh search "<core question rephrased as search query>" 10
```

**Strategy 2: Category-filtered search**
```bash
# For academic topics
/Users/tothemoon/.claude/skills/exa/scripts/exa.sh search "<query>" 10 "research paper"

# For code/implementations
/Users/tothemoon/.claude/skills/exa/scripts/exa.sh search "<query>" 10 "github"

# For industry news
/Users/tothemoon/.claude/skills/exa/scripts/exa.sh search "<query>" 10 "news"
```

**Strategy 3: AI-answered overview**
```bash
/Users/tothemoon/.claude/skills/exa/scripts/exa.sh answer "<core question>"
```

**Strategy 4: Web search + scrape combo**
```bash
/Users/tothemoon/.claude/skills/firecrawl/scripts/firecrawl.sh search "<query>" 5
```

**Strategy 5: Code-specific search**
```bash
/Users/tothemoon/.claude/skills/exa/scripts/exa.sh code "<technical query>"
```

**Run at least 3 of these strategies per sub-question.** Collect all URLs. Deduplicate. Rank by relevance.

---

### Phase 3: Deep Read — Extract Knowledge

Now read the best sources in full. This is where shallow research becomes deep research.

**For web pages and articles:**
```bash
/Users/tothemoon/.claude/skills/firecrawl/scripts/firecrawl.sh scrape "<url>"
```

**For arxiv papers — MANDATORY: use AlphaXiv script (do NOT scrape arxiv with firecrawl):**
```bash
/Users/tothemoon/.claude/skills/research/scripts/alphaxiv.sh overview "<paper-id-or-arxiv-url>"
```
Accepts any format: `2301.12345`, `https://arxiv.org/abs/2301.12345`, `https://arxiv.org/pdf/2301.12345`
Falls back automatically to full text if overview unavailable.

**To search for academic papers:**
```bash
/Users/tothemoon/.claude/skills/research/scripts/alphaxiv.sh search "<query>" 5
```

**MANDATORY — Follow the trail from your best source:**
After identifying your single most valuable source, ALWAYS run:
```bash
/Users/tothemoon/.claude/skills/exa/scripts/exa.sh similar "<url-of-best-source>" 5
```
This often surfaces the highest-quality sources that search alone misses. Do this at least once per research session.

**For extracting structured data from a page:**
```bash
/Users/tothemoon/.claude/skills/firecrawl/scripts/firecrawl.sh extract "<url>" "<what to extract>"
```

**For mapping an entire docs site before selective reading:**
```bash
/Users/tothemoon/.claude/skills/firecrawl/scripts/firecrawl.sh map "<docs-url>" 50
```

**Read at least 5 sources for a quick scan, 10+ for a deep dive.** Take notes as you go — write down key facts, quotes, and contradictions between sources.

---

### Phase 4: Synthesize — Build the Report

Compile everything into a structured research report. The report must be **original synthesis**, not a copy-paste of sources.

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
[MANDATORY section. At least one explicit disagreement, contradiction, or tension found between sources. If all sources agree, state that explicitly and explain why consensus exists.]

## Sources
[1] [Title](URL) — [authority: official-docs|peer-reviewed|industry|blog|forum|code] — [one-line description]
[2] [Title](URL) — [authority: ...] — [one-line description]
...
```

---

### Phase 5: Verify — Cross-Check Claims

Before delivering, verify critical claims:

1. **Contradiction check** — Did any sources disagree? Note it explicitly.
2. **Recency check** — Are any sources outdated? Flag if the field moves fast.
3. **Authority check** — Are sources authoritative (official docs, peer-reviewed, established practitioners) or random blog posts? Weight accordingly.
4. **Gap check** — Is there a sub-question you couldn't find good sources for? Say so explicitly rather than guessing.

---

## Autonomy Rules

1. **Never stop to ask permission between searches.** The user wants research, not a play-by-play. Run all searches, read all sources, then deliver the report.
2. **Follow the trail.** If source A references source B, go read source B. Good research is recursive.
3. **Diversify sources.** Don't rely on one search. Use exa AND firecrawl. Use semantic search AND category filters. Check papers AND blogs AND code.
4. **Prefer primary sources.** Official docs > blog summaries. Research papers > news articles about research papers. Code > descriptions of code.
5. **Note uncertainty.** If sources conflict or you can't verify a claim, say so. Never present uncertain information as fact.
6. **Stay focused.** Follow interesting tangents only if they serve the core question. Research is not browsing.
7. **No minimum source count theater.** Don't pad the report with weak sources just to hit a number. 5 excellent sources beat 15 mediocre ones.

---

## Error Handling

| Problem | Action |
|---------|--------|
| Exa returns no results | Rephrase query. Try broader terms. Try without category filter. |
| Firecrawl scrape fails | Try different URL format. Try the page's cached/archived version. Move on to next source. |
| AlphaXiv 404 | Try `/abs/` endpoint. If both fail, scrape the arxiv abstract page directly with firecrawl. |
| Rate limited (429) | Scripts auto-retry with key rotation. If all keys exhausted, wait and continue. |
| Topic too broad | Narrow the core question. Research one specific angle first, then expand. |
| Topic too niche | Broaden search terms. Try adjacent topics. Use `similar` to find related content from any relevant source you find. |
| Conflicting sources | Report the conflict explicitly. Note which sources are more authoritative and why. |

---

## Examples

### Example 1: Technical research
**User:** "Research the current state of Elliott Wave analysis algorithms"
- Scope: Academic + code + practitioner approaches
- Search: exa papers + github repos + exa answer for overview
- Deep read: Top 5 papers via alphaxiv, top 3 repos via firecrawl
- Output: Landscape report comparing approaches

### Example 2: Market research
**User:** "What's the landscape of AI code review tools?"
- Scope: Products + pricing + technical approaches
- Search: exa company search + firecrawl search + exa news
- Deep read: Product pages via firecrawl extract, reviews via scrape
- Output: Comparison table + analysis

### Example 3: How-does-it-work
**User:** "How does Raft consensus work?"
- Scope: Academic paper + implementations + explanations
- Search: exa paper search + exa answer + code search
- Deep read: Original Raft paper via alphaxiv, 2-3 explainer articles
- Output: Technical explanation with diagrams described

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
