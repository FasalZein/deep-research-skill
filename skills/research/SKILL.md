---
name: research
description: "Autonomous deep research on any topic. Combines Exa semantic search, TinyFish web fetch, Firecrawl Markdown scraping/crawling/extraction, and AlphaXiv paper analysis into structured cited reports. Use when: research this, find out about, what's the latest on, deep dive into, investigate, analyze the landscape of, compare approaches to, literature review, state of the art, how does X work. Produces a structured research report with citations."
---

# Autonomous Research

You are a research director. Refine the question, dispatch subagents, synthesize a cited report. Final heading: `# Research: [Core Question]`.

**Use** for multi-source research, literature reviews, landscapes, technical comparisons, source-backed answers.
**Skip** for single facts, one known URL, or library docs — use Exa, TinyFish, Firecrawl, or Context7 directly.

---

## 1. Preflight — resolve tools only

Run once. Shell state does not persist between calls — save the printed paths as literal strings for all subsequent commands.

The skill directory is provided by your harness. Look for "Base directory for this skill:" in your system prompt (Claude Code), or use `$PI_SKILL_DIR` (Pi Agent). Pass the correct value as SKILL_ROOT below:

```bash
# Detect skill root: PI_SKILL_DIR (Pi) > base-directory from system prompt > search common roots
SKILL_ROOT="${PI_SKILL_DIR:-}"
if [ -z "$SKILL_ROOT" ]; then
  for root in "$HOME/.agents/skills/research" "$HOME/.claude/skills/research" "$HOME/.local/share/tia/pi-agent/skills/research"; do
    [ -f "$root/SKILL.md" ] && SKILL_ROOT="$root" && break
  done
fi
SKILLS_DIR="$(dirname "$SKILL_ROOT")"

EXA="$SKILLS_DIR/exa/scripts/exa.sh"
TINYFISH="$SKILLS_DIR/tinyfish/scripts/tinyfish.sh"
FIRECRAWL="$SKILLS_DIR/firecrawl/scripts/firecrawl.sh"
ALPHAXIV="$SKILL_ROOT/scripts/alphaxiv.sh"

echo "SKILLS_DIR=$SKILLS_DIR"
echo "EXA=$EXA"; echo "TINYFISH=$TINYFISH"; echo "FIRECRAWL=$FIRECRAWL"; echo "ALPHAXIV=$ALPHAXIV"
test -x "$EXA" && echo "exa: OK" || echo "exa: MISSING"
test -x "$TINYFISH" && echo "tinyfish: OK" || echo "tinyfish: MISSING"
test -x "$FIRECRAWL" && echo "firecrawl: OK" || echo "firecrawl: MISSING"
test -x "$ALPHAXIV" && echo "alphaxiv: OK" || echo "alphaxiv: MISSING"
[ -n "$EXA_API_KEY" ] && echo "EXA_API_KEY: set" || echo "EXA_API_KEY: MISSING"
[ -n "$TINYFISH_API_KEY" ] && echo "TINYFISH_API_KEY: set" || echo "TINYFISH_API_KEY: MISSING"
[ -n "$FIRECRAWL_API_KEY" ] && echo "FIRECRAWL_API_KEY: set" || echo "FIRECRAWL_API_KEY: MISSING"
```

**Stop if any tool or key is MISSING.** Tell the user exactly what to install/export. Do not dispatch subagents.

## 2. Scope — no tools, just thinking

1. Rewrite the user's ask into one precise core question.
2. Decide source classes: official-docs, academic, code, company/product, practitioner, news.
3. Decide depth: quick scan (3-5 sources) or deep dive (10-20+ sources).
4. Split into independent angles. For each angle, state what IS and IS NOT in scope to prevent overlap.
5. **Create the research directory** — derive a short kebab-case slug from the core question (max 40 chars), then run:

```bash
SLUG="<topic-slug>"  # e.g. "context-engineering-llm", "ai-code-review-tools"
RESEARCH_DIR="${RESEARCH_ARTIFACT_DIR:-/tmp/research-${SLUG}-$(date +%Y%m%d-%H%M%S)}"
mkdir -p "$RESEARCH_DIR"
echo "RESEARCH_DIR=$RESEARCH_DIR"
```

Example: "AI code review tools" → 3 angles: (1) products and pricing — not technical internals, (2) technical approaches — not product features, (3) practitioner reception — not marketing claims.

**Do not proceed until you have a precise question, subagent split, and RESEARCH_DIR.**

## 3. Dispatch subagents

Subagents are mandatory for 2+ angles. Use whatever subagent/agent/task tool the harness provides. Launch independent subagents in parallel when the harness supports it. If no subagent tool exists, run the chain yourself with reduced scope: 1-2 searches, top 3-5 sources, no raw page dumps.

Split by **angle**, not by tool:
- Focused question → 1 subagent (return in-context is fine)
- Multi-angle → 2-3 subagents (write to `$RESEARCH_DIR`)
- Broad survey → 3-5 subagents (write to `$RESEARCH_DIR`)
- Contradiction → 1 targeted gap-checker after first synthesis

### Subagent prompt contract

```
You are one research subagent. Do not write the final report.

CORE QUESTION: [overall question]
YOUR ANGLE: [narrow angle — what IS and IS NOT in scope]

TOOLS (invoke via the Bash tool with these literal paths):
  bash [exa-path] search "<query>" <num> [category] [--start-date ISO] [--type deep-reasoning]
  bash [exa-path] similar "<url>" <num>
  bash [exa-path] contents "<url>" [--highlights] [--summary]
  bash [exa-path] code "<programming query>"
  bash [tinyfish-path] fetch "<url1>" "<url2>" --format markdown
  bash [firecrawl-path] scrape "<url>" markdown ['{"waitFor":3000}']
  bash [firecrawl-path] batch-scrape '["<url1>","<url2>"]' markdown
  bash [firecrawl-path] map "<url>" <limit>
  bash [firecrawl-path] extract "<url>" "<prompt>"
  bash [firecrawl-path] crawl "<url>" <limit> <depth>
  bash [alphaxiv-path] overview "<paper-id>"
  bash [alphaxiv-path] search "<query>" <num>

CHAIN (adapt order to your angle):
1. Search: 1-2 Exa searches, 5-10 results each. For "latest/current/2026" queries, add --start-date. Exa returns highlights and published dates by default.
2. Triage (mandatory): Read the highlights and dates from search results. Pick the top 3-5 most relevant URLs based on highlight quality and recency. Do NOT fetch every search result — only fetch URLs whose highlights show direct evidence for your angle.
3. Fetch: Use TinyFish fetch for the triaged URLs — it returns 5-7x more content than Firecrawl scrape. It also returns author and published_date metadata.
4. Firecrawl is for site maps (map), batch-scrape (5+ URLs from one domain after mapping), recursive crawls (crawl), structured extraction (extract), and PDFs. Use batch-scrape when you need many pages from a docs site.
5. If structured data needed: Firecrawl extract with prompt
6. If arXiv paper found: AlphaXiv overview by ID
7. Trail (mandatory): Exa similar on best source

OUTPUT: Write to [research-dir]/[angle-slug].md:
## Findings: [angle]
### Answer — [5-10 bullets]
### Key Claims — [claim] — [Title](URL) — quote: "[15+ word verbatim quote from fetched content]"
### Contradictions — [conflicts or None]
### Sources — 1. [Title](URL) — [tag] — [YYYY-MM or Unknown] — [one-line]
  Tags: Primary | Academic | Official-docs | Vendor | News | Blog | Forum
### Gaps — [missing evidence or None]

QUALITY RULES:
- Triage before fetching. Use highlights to decide relevance — never bulk-fetch all search results.
- Only cite URLs you actually fetched/scraped. Never cite raw search-result URLs.
- For "current/latest/2026" queries, include the year in search queries and prefer sources <18 months old.
- For code/framework angles, use exa code. For academic angles, use alphaxiv search.
```

## 4. Synthesize

Do not proceed until all subagents have returned.

1. Read each file in `$RESEARCH_DIR/` (one per angle).
2. Merge findings, dedupe sources by URL, weight by authority (Primary > Academic > Official-docs > Vendor > Blog > Forum) and recency. If two angles cite the same source with different takeaways, include both.
3. Compare contradictions across angles — this becomes the Contradictions section.
4. If a critical gap remains (a core sub-question with zero evidence), spawn one gap-checker. Do not retry the same search.
5. Write the final report. Print `Research artifacts: $RESEARCH_DIR` at the end.

## Tool chain

| Stage | Tool | Use for | Not for |
|---|---|---|---|
| Search | **Exa** | Semantic discovery, similar trails, code/paper search | Fetching full content |
| Fetch | **TinyFish** | All page fetching — returns full content, handles JS, batches ≤10 URLs | Primary search (Exa is better) |
| Map/crawl/extract | **Firecrawl** | Site maps, batch-scrape, recursive crawls, structured JSON extraction, PDFs | Single-page fetching (TinyFish returns more content for HTML) |
| Papers | **AlphaXiv** | arXiv/AlphaXiv paper overviews by ID | Non-arXiv URLs |

Default chain: **Exa search → TinyFish fetch → Firecrawl map/crawl when needed → Exa similar → synthesize**.

Provider snippets and `answer` summaries are never evidence. Evidence = fetched/scraped source content with exact quotes.

## Report format

```markdown
# Research: [Core Question]

## TL;DR
[Under 100 words. End: Confidence: High (sources agree, primary evidence) | Mixed (conflicts or thin coverage) | Low (few sources, mostly secondary) — [reason].]

## Key Findings
### [Theme]
[Synthesis with inline citations [1], [2].]

## Landscape / Comparison
[Optional table.]

## Contradictions & Disputes
[Mandatory. Never omit.]

## Open Questions
[Gaps and what would resolve them.]

## Sources
[1] [Title](URL) — [authority-tag] — [date] — [one-line]
```

Target 1000-3000 words depending on depth.

## Error recovery

| Problem | Do this |
|---|---|
| Search returns nothing | Rephrase once, broaden, try another source class |
| TinyFish fails | Firecrawl scrape as fallback |
| Firecrawl fails | TinyFish fetch as fallback |
| Subagent thin | One gap-checker, not full retry |
| Provider vs source conflict | Trust source, report conflict |

## Hard rules

1. **Never ask permission** between searches. Scope → dispatch → synthesize → deliver.
2. **Similar trail is mandatory.** Every subagent runs `exa similar` on its best source.
3. **TinyFish for fetching single pages.** TinyFish returns the most content for HTML pages. Use Firecrawl batch-scrape when you need 5+ URLs from one site, or for PDFs.
4. **Primary sources first.** Official docs > blogs. Papers > news about papers.
5. **Admit uncertainty.** Conflicting sources → report the conflict, don't pick silently.
6. **Keep parent context clean.** Raw search/scrape output stays in subagent contexts or artifact files.
