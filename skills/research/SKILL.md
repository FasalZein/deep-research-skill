---
name: research
description: "Autonomous deep research on any topic. Combines Exa semantic search, TinyFish web fetch, Firecrawl Markdown scraping/crawling/extraction, and AlphaXiv paper analysis into structured cited reports. Use when: research this, find out about, what's the latest on, deep dive into, investigate, analyze the landscape of, compare approaches to, literature review, state of the art, how does X work. Produces a structured research report with citations."
---

# Autonomous Research

You are a research director. Refine the question, dispatch subagents, synthesize a cited report. Final heading: `# Research: [Core Question]`.

**Use** for multi-source research, literature reviews, landscapes, technical comparisons, source-backed answers.
**Skip** for single facts, one known URL, or library docs — use Exa, TinyFish, Firecrawl, or Context7 directly.

---

## 1. Preflight

Run once. Shell state does not persist between calls — save the printed paths as literal strings for all subsequent commands.

```bash
EXA="${PI_SKILL_DIR:+$(dirname "$PI_SKILL_DIR")/exa/scripts/exa.sh}"
TINYFISH="${PI_SKILL_DIR:+$(dirname "$PI_SKILL_DIR")/tinyfish/scripts/tinyfish.sh}"
FIRECRAWL="${PI_SKILL_DIR:+$(dirname "$PI_SKILL_DIR")/firecrawl/scripts/firecrawl.sh}"
ALPHAXIV="${PI_SKILL_DIR:-${skill_dir:-$HOME/.agents/skills/research}}/scripts/alphaxiv.sh"

for root in "$HOME/.agents/skills" "$HOME/.local/share/tia/pi-agent/skills" "$HOME/.claude/skills"; do
  [ -x "$EXA" ] || EXA="$root/exa/scripts/exa.sh"
  [ -x "$TINYFISH" ] || TINYFISH="$root/tinyfish/scripts/tinyfish.sh"
  [ -x "$FIRECRAWL" ] || FIRECRAWL="$root/firecrawl/scripts/firecrawl.sh"
  [ -x "$ALPHAXIV" ] || ALPHAXIV="$root/research/scripts/alphaxiv.sh"
done

RESEARCH_DIR="${RESEARCH_ARTIFACT_DIR:-${TMPDIR:-/tmp}/claude-research-$(date +%Y%m%d-%H%M%S)-$$}"
mkdir -p "$RESEARCH_DIR"

echo "EXA=$EXA"; echo "TINYFISH=$TINYFISH"; echo "FIRECRAWL=$FIRECRAWL"; echo "ALPHAXIV=$ALPHAXIV"
echo "RESEARCH_DIR=$RESEARCH_DIR"
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

Example: "AI code review tools" → 3 angles: (1) products and pricing — not technical internals, (2) technical approaches — not product features, (3) practitioner reception — not marketing claims.

**Do not proceed until you have a precise question and subagent split.**

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
  bash [exa-path] search "<query>" <num> [category]
  bash [exa-path] similar "<url>" <num>
  bash [tinyfish-path] fetch "<url1>" "<url2>" --format markdown
  bash [firecrawl-path] scrape "<url>" markdown
  bash [firecrawl-path] map "<url>" <limit>
  bash [firecrawl-path] extract "<url>" "<prompt>"
  bash [firecrawl-path] crawl "<url>" <limit> <depth>
  bash [alphaxiv-path] overview "<paper-id>"
  bash [alphaxiv-path] search "<query>" <num>
  bash [exa-path] code "<programming query>"

CHAIN (adapt order to your angle):
1. Search: 1-2 Exa searches, 5-10 results each
2. Fetch: TinyFish fetch top 3-5 URLs as markdown
3. If docs site: Firecrawl map then scrape relevant pages
4. If structured data needed: Firecrawl extract with schema
5. If arXiv paper found: AlphaXiv overview by ID
6. Trail (mandatory): Exa similar on best source

OUTPUT: Write to [research-dir]/[angle-slug].md:
## Findings: [angle]
### Answer — [5-10 bullets]
### Key Claims — [claim] — [Title](URL) — quote: "[15+ word verbatim quote from fetched content]"
### Contradictions — [conflicts or None]
### Sources — 1. [Title](URL) — [tag] — [YYYY-MM or Unknown] — [one-line]
  Tags: Primary | Academic | Official-docs | Vendor | News | Blog | Forum
### Gaps — [missing evidence or None]

QUALITY RULES:
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
| Fetch | **TinyFish** | Page content, JS-rendered pages, batch ≤10 URLs | Primary search (Exa is better) |
| Scrape/crawl | **Firecrawl** | Known-URL markdown, docs maps, recursive crawl | `extract` takes a prompt, returns structured JSON |
| Papers | **AlphaXiv** | arXiv/AlphaXiv paper overviews by ID | Non-arXiv URLs |

Default chain: **Exa search → TinyFish fetch → Firecrawl scrape/map when needed → Exa similar → synthesize**.

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
| TinyFish fails | Firecrawl scrape once |
| Firecrawl fails | TinyFish fetch once |
| Subagent thin | One gap-checker, not full retry |
| Provider vs source conflict | Trust source, report conflict |

## Hard rules

1. **Never ask permission** between searches. Scope → dispatch → synthesize → deliver.
2. **Similar trail is mandatory.** Every subagent runs `exa similar` on its best source.
3. **Primary sources first.** Official docs > blogs. Papers > news about papers.
4. **Admit uncertainty.** Conflicting sources → report the conflict, don't pick silently.
5. **Keep parent context clean.** Raw search/scrape output stays in subagent contexts or artifact files.
