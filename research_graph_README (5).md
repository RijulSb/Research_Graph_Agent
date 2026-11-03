# ResearchGraph Agent

A single-notebook **KG-RAG research assistant**. It ingests research text (pasted, `.txt`, `.md`, `.pdf`), builds a constrained **Neo4j knowledge graph**, answers questions from **graph + source evidence**, and shows how it got there: citations, the Cypher used, an evidence subgraph, and an audit trail.

**Stack:** LangChain · Neo4j · `LLMGraphTransformer` · LangGraph ReAct · `all-MiniLM-L6-v2` (local embeddings) · NetworkX/Matplotlib

## Quick start

1. **Neo4j:** create a free [Aura](https://neo4j.com/cloud/aura-free/) instance (or run Neo4j 5.x Community locally).
2. **Python 3.10+**, then open `research_graph_agent.ipynb` and run Cell 1 (installs dependencies).
3. Create a `.env` next to the notebook:

   ```env
   NEO4J_URI=neo4j+s://xxxx.databases.neo4j.io
   NEO4J_USER=neo4j
   NEO4J_PASSWORD=...
   LLM_API_KEY=...              # any OpenAI-compatible provider with function calling
   # optional
   LLM_MODEL=gpt-4o-mini
   LLM_BASE_URL=                # e.g. another OpenAI-compatible endpoint
   NEO4J_READ_USER=             # read-only role for QA (self-hosted / paid Aura)
   NEO4J_READ_PASSWORD=
   DATA_DIR=data                # only folder ingestion may read files from
   ```
4. Run cells top to bottom. **Cell 10 Part A** (offline self-tests) needs no services; **Part B** runs the live demo.

## Usage

```python
run_agent({"action": "ingest", "source_id": "climate_report_2026",
           "title": "Climate Research Report", "file_path": "data/report.pdf"})

result = run_agent({"action": "ask",
                    "question": "Which organizations collaborated with MIT on climate research?"})
show_result(result)   # answer, citations, Cypher, tool trace, graph (saved to output/graph.png)
```

Ingestion returns `status, source_id, content_hash, chunks_created, entities_created, relationships_created, duplicate`. Questions return `request_id, answer, confidence, citations[], cypher, graph{nodes, relationships}, tool_trace` (plus `latency_ms`, and `graph.graph_detail` used for plotting).

## Notebook map

| Cell | Purpose |
|---|---|
| 1 | Install minimal dependencies |
| 2 | Config: secrets (env only), limits, fixed ontology |
| 3 | LLM, embedder, Neo4j writer/reader drivers, schema + vector index, `LLMGraphTransformer` |
| 4 | Security: hashing, sanitize, PII redaction, injection defenses, Cypher validator, path safety |
| 5 | Ingestion: read → sanitize → dedupe → chunk → embed → extract → canonicalize → write |
| 6 | Retrieval: `vector_search`, `graph_expand`, safe `graph_query`, `combine_evidence` |
| 7 | The 7 agent tools, per-request state, answer synthesis, audit |
| 8 | LangGraph ReAct agent and `run_agent` entry point |
| 9 | Subgraph visualization and `show_result` |
| 10 | Offline self-tests + live demo (3 plan test cases) |

## Design highlights

- **Fixed ontology:** 7 node types, 9 relationship types; the LLM cannot invent labels.
- **Canonical entities:** `MIT` / `M.I.T.` / `Massachusetts Institute of Technology` merge into one node with aliases retained.
- **Idempotent ingestion:** SHA-256 content hash + `MERGE`; one transaction per document (no partial graphs); `force_reindex=True` replaces a source cleanly.
- **Hybrid retrieval:** vector top-6 → one-hop graph expansion → optional Cypher; chunks found by both routes rank higher.
- **Evidence-only answers:** invented chunk IDs are dropped, uncited answers are refused, and the final answer is always the validated output of `synthesize_answer`. Confidence is a heuristic (mean top-3 cosine of cited chunks, +0.1 if graph-corroborated), not a calibrated probability.
- **Provider-portable:** structured output uses function calling, so any OpenAI-compatible endpoint with tool support works (`LLM_BASE_URL`).
- **Bounded agent:** max 6 tool iterations; on the cap or an agent error it falls back to answering from evidence already gathered.

## Security model

- Document text is treated as data: injection phrases are stripped before extraction, text is wrapped in `<untrusted_document>` tags (embedded delimiters neutralized), and the system prompt forbids following in-document instructions. Filtering is defense in depth, not the only layer.
- LLM-generated Cypher must pass an **allowlist** (`MATCH / OPTIONAL MATCH / WHERE / RETURN / ORDER BY / LIMIT`): no writes, `CALL`, `APOC`, `WITH`, comments, `;`, or string literals (all values are `$parameters`); `LIMIT ≤ 50` is mandatory. It then runs in a **READ-routed** transaction with a server-side timeout.
- Separate writer and reader connections. Use `NEO4J_READ_USER` for a true read-only role. Aura Free has a single user, so QA falls back to READ-routed transactions on the same user (the notebook says which mode is active).
- File ingestion is restricted to `DATA_DIR`, to `.txt/.md/.pdf`, with symlink/`..` resolution.
- Secrets come from the environment and are never printed. Optional PII redaction: set `REDACT_PII = True` in Cell 2.

## Testing performed

| Layer | Result |
|---|---|
| Static analysis (pyflakes) over all cells | clean |
| Offline self-tests (Cell 10 Part A, no services needed) | 32/32 |
| Full pipeline on in-memory fakes: real `LLMGraphTransformer`, real LangGraph agent, strict check that every Cypher statement's `$parameters` are supplied and brackets/quotes balance | 40/40 + 17 edge cases |
| **Diagnostic suite (90 checks)**: real `ChatOpenAI` client against a mock OpenAI-compatible server (real request/response serialization); provider-outage and malformed-model-output injection; 40+ hostile inputs; real PDF/encrypted/corrupt/blank/non-UTF-8/oversize/symlink files; chunker fuzzing (400 random docs); Cypher-validator fuzzing across every banned keyword, casing and position; 12 parallel requests (no context bleed); plotting edge cases; unreachable-Neo4j startup; a max-size (~100k char, 150-chunk) document; extraction-cache reuse | 90/90 |

**Not verifiable in the build sandbox** (no Neo4j server, real LLM key or HuggingFace access): Cypher *syntax* against a live Neo4j, the vector-index call (needs Neo4j >= 5.11), real embedding quality, and real LLM extraction/answer quality. Cell 10 Part B exercises these on your first run; failures are reported as error dicts with the cause, never crashes.

## Diagnostics changelog (bugs found and fixed)

| Issue | Fix |
|---|---|
| `langchain-openai` now defaults structured output to `json_schema`, which breaks `LLMGraphTransformer` on many OpenAI-compatible providers | Chat model subclass defaults to function calling everywhere |
| LLM-acronyms lost (`MIT` -> `Mit`) from the transformer's title-casing | Original spelling restored from the source text |
| Extraction failures hid their cause | First error surfaced (single line, secret-free) |
| `route_request` appeared twice in the trace | Recorded once |
| `request_id` accepted any type/length | Validated (`[A-Za-z0-9_.-]{1,64}`) else regenerated |
| Validator allowed unbounded paths (`[*]`) and `LIMIT $anything` | Both rejected |
| A *rejected* LLM query was reported as "the Cypher used" | Only validated, executed queries are reported |
| `None` structured output (model replied in prose) could raise | One retry, then a clean "insufficient evidence" |
| `visualize` crashed on null node names; long/non-Latin names swamped the plot | Null-safe, names truncated, glyph warnings silenced |
| Re-running Cell 3 leaked Neo4j drivers and left a stale cached agent | Drivers closed; agent rebuilt when the LLM changes |
| Schema-setup failure was reported as "connection failed" | Separate, specific message |
| Extraction cache ignored ontology changes; "index ..." questions misrouted as ingestion | Ontology hash in cache key; stricter intent rule |

## Known limitations

- Ingestion cost scales with chunk count (one extraction call per chunk; results are cached in `.cache/`). Limits: 100k chars and 200 chunks per document.
- `graph_expand` supports depth 1 only. Scanned PDFs need OCR first.
- `LLMGraphTransformer` lives in `langchain-experimental`, which is in maintenance mode; the notebook pins no versions, so consider pinning for production.
- Entity-merging is rule-based (spelling and acronym matching); it will not merge semantically different aliases such as nicknames.
- `DERIVED_FROM` links Entity → Chunk (a relationship cannot point at a node), and each relationship stores `chunk_ids` / `source_ids` properties for provenance.
