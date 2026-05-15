---
name: research
description: "Autonomous deep research director. Uses subagents by default, discovers sources with Exa, fetches/extracts with TinyFish, renders/crawls Markdown with Firecrawl, and uses AlphaXiv for papers."
---

# Autonomous Research

Original report name: **Autonomous Research**. Final report heading: `# Research: [Core Question]`.

Use this skill as a coordinator, not a monolith: refine the question, launch subagents, chain the right acquisition tools, preserve useful artifacts, then synthesize a cited report.

Pi skill frontmatter does not support dependencies/chains. Supported fields are `name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`, and `disable-model-invocation`; unknown fields are ignored. Put chaining rules in this body and verify tools at runtime.

If filing research, use `projects/<project>/research/` via `wiki research file|ingest ... --project <project>`. Use global `research/` only with `--global`. Never create `research/projects/<project>/...`.

## Use / skip

Use for deep research, literature reviews, market/product landscapes, technical comparisons, source-backed answers, or anything needing multiple sources.

Skip for single facts, one known URL, or library docs. Use Exa, TinyFish/Firecrawl, or Context7 directly instead.

## Tool chain

| Stage | Prefer | Purpose | Notes |
|---|---|---|---|
| Search/discovery | Exa | Find authoritative sources and trails | Search only; snippets are leads, not evidence. Use `similar` after the best source. |
| Fetch/extract content | TinyFish | Fetch pages, JS-rendered pages, batches up to 10 URLs | Default output is clean Markdown; good after search results. |
| Markdown/crawl/map | Firecrawl | Convert known URLs/sites to Markdown, map docs, crawl sections | Use `scrape <url> markdown`; use `extract` only for structured JSON. |
| Papers | AlphaXiv + Exa | Arxiv paper discovery/overview | Use for paper summaries, then fetch paper/abstract when claims matter. |

Default chain: **Exa search → TinyFish fetch → Firecrawl Markdown/crawl when needed → Exa similar trail → synthesize**.

### Chain patterns

- **General web:** Exa search 5-10 → TinyFish fetch top 3-5 → Exa similar on best source → synthesize.
- **Docs/specs:** Firecrawl map docs root → Firecrawl scrape selected pages as Markdown → TinyFish fetch if rendering is better.
- **Market/product:** Exa company search → TinyFish broad search/fetch → Firecrawl extract pricing/features JSON only after URLs are known.
- **News/current:** Exa news search → TinyFish search/fetch with current year → avoid relying on Exa answer except as overview.
- **Academic:** Exa research-paper search → AlphaXiv overview for arXiv/AlphaXiv IDs only → TinyFish/Firecrawl fetch non-arXiv abstracts or paper pages.
- **Code:** Exa code/GitHub search → fetch README/docs/releases with TinyFish or Firecrawl.

Provider snippets, `answer` summaries, and subagent prose are never final evidence. Evidence comes from fetched/scraped source content and exact quotes.

## Preflight

Resolve paths once in the parent and pass absolute paths to subagents.

```bash
# Seed from current skill dir when available, then try common install roots.
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

test -x "$EXA" && echo "exa: OK" || echo "exa: MISSING"
test -x "$TINYFISH" && echo "tinyfish: OK" || echo "tinyfish: MISSING"
test -x "$FIRECRAWL" && echo "firecrawl: OK" || echo "firecrawl: MISSING"
test -x "$ALPHAXIV" && echo "alphaxiv: OK" || echo "alphaxiv: MISSING"
```

Also check env vars: `EXA_API_KEY`, `TINYFISH_API_KEY`, `FIRECRAWL_API_KEY`. If required tools/keys are missing, stop and tell the user exactly what to install/export.

## Scope before search

Do this in the parent, without tools. Better scope beats more queries.

1. Rewrite vague asks into one precise core question.
2. Decide source classes: official docs/specs, academic, code, company/product, practitioner, news.
3. Decide depth: quick scan (3-5 good sources) or deep dive (10-20+ sources).
4. Split only into independent angles; if all angles would search the same query, use one subagent.

Examples:
- "Research WebTransport" → "What is WebTransport's browser support, production adoption, and tradeoff vs WebSocket/SSE for real-time apps as of 2026?"
- "Look into RAG" → "Which RAG chunking, embedding, reranking, and retrieval patterns currently improve recall/latency, and what evidence supports them?"

## Subagents are mandatory by default

All target harnesses support delegation:

| Harness | Mechanism |
|---|---|
| Pi Agent | `subagent` |
| Claude Code | `Agent` |
| OpenCode | `Task` |

Use subagents because they improve source diversity, reduce anchoring, keep raw acquisition output out of the parent context, and allow parallel angles.

Only skip subagents for trivial single-source lookups or if the harness truly lacks delegation. If there is no subagent tool, run the same chain in the parent with reduced scope: 1-2 searches, fetch/scrape only the top 3-5 sources, and avoid dumping raw page output into the final answer.

### Split by angle, not by tool

- Focused question → 1 subagent that searches + fetches + reports.
- Multi-angle landscape → 2-3 subagents, e.g. products / technical approach / practitioner reception.
- Broad survey → 3-5 subagents, each covering an independent source class or thesis.
- Unresolved contradiction → 1 targeted gap-checker after first synthesis.

Launch independent children together when the harness allows it. The parent must not redo delegated search; it should synthesize returned findings.

### Subagent roles

| Role | Job | Return |
|---|---|---|
| Discovery researcher | Search with Exa/TinyFish and follow similar trails | Ranked sources and rejected weak sources |
| Deep reader | TinyFish fetch / Firecrawl Markdown scrape | Claims with exact quotes and source excerpts |
| Specialist | One angle: docs, academic, market, code, news, practitioner | Compact angle findings |
| Gap checker | Resolve one contradiction/missing source class | Targeted answer and confidence update |

## Protocol

1. **Scope:** core question, independent sub-questions, source classes, depth, artifact/filing target.
2. **Plan chain:** choose search/fetch/Markdown/extraction tools per source class.
3. **Dispatch subagents:** one focused prompt per independent angle; pass resolved tool paths and exact commands.
4. **Deep read:** require top 3-5 authoritative URLs per angle, fetched/scraped content, and exact quotes.
5. **Synthesize:** merge duplicate sources, weight primary/recent sources, compare contradictions, spawn one gap-checker if needed.
6. **File:** final cited report plus correct wiki path when filing is requested.

## Artifacts

For substantial deep research, preserve a small evidence trail instead of only a chat answer:

- Subagents return compact findings to the parent; they may also write a Markdown handoff when the harness supports artifacts.
- Artifact contents: refined question, angle, commands run, ranked sources, exact quotes, contradictions, gaps. Do not dump full scraped pages unless explicitly requested.
- Parent final report should cite sources and optionally link artifact paths. If the user requested wiki filing, ingest/file only the final report unless they ask for raw evidence notes too.

## Subagent prompt template

```text
You are one research subagent. Do not write the final report.

CORE QUESTION: [overall question]
YOUR ANGLE: [narrow independent angle]
PATHS: Exa=[resolved absolute path], TinyFish=[resolved absolute path], Firecrawl=[resolved absolute path], AlphaXiv=[resolved absolute path]
ARTIFACT: [optional artifact path or "none"]

CHAIN:
1. Search with: [resolved Exa path] search "[query]" 5 [category]
2. Fetch top URLs with: [resolved TinyFish path] fetch "<url1>" "<url2>" --format markdown
3. For docs/sites or cleaner Markdown: [resolved Firecrawl path] map "<root>" 50, then [resolved Firecrawl path] scrape "<url>" markdown
4. For structured product/data fields only: [resolved Firecrawl path] extract "<url>" "<schema/prompt>"
5. For arXiv/AlphaXiv papers only: [resolved AlphaXiv path] overview "<paper-id-or-url>"; for other papers fetch abstract/paper pages with TinyFish or Firecrawl
6. Trail: [resolved Exa path] similar "<best-url>" 5

RETURN ONLY:
## Findings: [angle]
### Answer
- [5-10 compact bullets]
### Key Claims
- [claim] — [Title](URL) — quote: "[exact quote from fetched/scraped content]"
### Contradictions
- [conflict or None found]
### Sources Ranked
1. [Title](URL) — [official-docs|peer-reviewed|industry|blog|forum|code] — [why it matters]
### Gaps
- [missing evidence or None]
### Follow-up
- [one targeted query or None]
### Artifact
- [path written or None]
```

## Final report structure

```markdown
# Research: [Core Question]

## TL;DR
[Direct answer under 100 words.]

## Key Findings
### [Theme]
[Synthesis with citations like [1], [2].]

## Landscape / Comparison
[Optional table.]

## Contradictions & Disputes
[Mandatory: conflicts, or why sources agree.]

## Open Questions
[Known gaps.]

## Sources
[1] [Title](URL) — [authority] — [one-line description]
```

## Error handling

| Problem | Action |
|---|---|
| Preflight script/key missing | Stop before spawning subagents; tell user which skill/key is missing. |
| Search returns weak/no results | Rephrase once, broaden terms, try another source class. |
| TinyFish fetch fails | Try Firecrawl scrape once. |
| Firecrawl scrape fails | Try TinyFish fetch once. |
| Firecrawl extract returns JSON but you need prose | Use Firecrawl scrape `markdown` instead. |
| Provider summary conflicts with source content | Trust source content; report conflict. |
| Subagent result is thin | Spawn one targeted gap-checker, not a full retry. |

## Token budget

Keep this skill compact. Target under ~3k loaded tokens. Put command details in sibling skills (`exa`, `tinyfish`, `firecrawl`) and include only chain decisions, prompt contracts, report shape, and hard rules here.

## Quality bar

A good report answers the question, cites source content, uses primary sources, flags disagreements, admits gaps, avoids snippet-based claims, keeps acquisition output out of parent context, preserves useful evidence artifacts for deep work, and files under the right project/global research path.
