# AI-for-industrial-knowledge-intelligence

A RAG + knowledge graph system built for the ET AI Hackathon 2026 (Problem
8: Industrial Knowledge Intelligence). Answers questions over industrial
safety documents — regulations, maintenance logs, incident reports,
permits — by combining vector search with a relationship graph, routed
through a small multi-agent LangGraph setup. No single framework does the
whole thing; each piece was added because the previous approach genuinely
couldn't answer a question the next one could.

This repo is the build log — each phase as its own notebook, in the order
they were actually built and debugged, plus a consolidated notebook that
runs the whole pipeline end to end.

## What's in here

1. **Document ingestion** — PDFs parsed to Markdown (`pymupdf4llm`), chunked
   by section header where the document has one, embedded with
   `sentence-transformers`, stored in ChromaDB.
2. **A knowledge graph** (`networkx`) linking Equipment, Hazard, Regulation,
   Permit, Incident, Maintenance Log, and Person Role nodes — built from
   the synthetic operational records plus a hand-written hazard ontology.
3. **A LangGraph orchestrator** that classifies each query (regulatory /
   operational / relational / unsupported), resolves any equipment/hazard
   entity mentioned even indirectly, and routes to the right specialist
   agent(s) — merging results when more than one applies.
4. **A citation-grounding check** on every generated answer — a second LLM
   call verifies each cited source actually supports the claim made
   against it, and the system says `INFO NOT IN CONTEXT` rather than
   guessing when it doesn't have the answer.
5. **A small fine-tuned entity extractor** (Qwen2.5-1.5B + LoRA) that pulls
   structured entities (equipment ID, date, hazard mentions, etc.) out of
   raw document text — validated on a document it never saw during
   training.

## Working of the pipeline

1. **Ingest** — new document dropped in → hashed → if changed, parsed,
   chunked, embedded. If unchanged, skipped entirely (nothing gets
   re-processed for free).
2. Query comes in → classified into one or more intents, entities resolved
   deterministically first (substring/metadata match), LLM only used as a
   fallback if that finds nothing.
3. Routed to specialist(s):
   - **Regulatory agent** — vector search over regulatory docs only
   - **Operational agent** — vector search over maintenance/incident/permit
     records, filtered by resolved equipment if one was found
   - **Relational agent** — walks the knowledge graph (e.g. Equipment →
     Hazard ← Regulation) *and* pulls vector chunks, merges both into one
     grounded answer
4. Every generated answer is citation-checked before being returned. If
   more than one specialist fired, their answers get merged and re-checked
   as a single grounded answer.
5. Every query result is cached (query, routing decision, entities,
   answer) so repeat questions cost zero LLM calls.

## Notebooks, in build order

| Notebook | What it covers |
|---|---|
| `manual_implementation.ipynb` | Phase 1 — chunking + retrieval validated by hand, no agent loop, no tools. Read this first if you want to see the pieces before they're wrapped in automation. |
| `phase2.ipynb` | Ingestion pipeline + tools + single-agent RAG loop |
| `phase3.ipynb` | Knowledge graph construction + hybrid (vector + graph) reasoning |
| `phase4.ipynb` | LangGraph multi-agent orchestrator — intent routing, entity resolution, specialist agents |
| `phase5.ipynb` | Interface + evaluation — precision/recall against a held-out set, graph completeness metrics |
| `entity_extractor.ipynb` | LoRA fine-tuning of a small model (Qwen2.5-1.5B) for structured entity extraction |
| `industrial_rag_pipeline.ipynb` | Everything above, consolidated into one runnable pipeline |

## Tools / components

- `ingest_regulatory_pdf` / `ingest_synthetic_json` — the two ingestion paths
- `retrieve_chunks` — vector search, optionally filtered by document type
- `query_knowledge_graph` / `get_regulations_for_equipment` — graph lookups
- `verify_citations` — the grounding safety net
- `entity_extractor.ipynb` — the fine-tuned extraction model, built to
  eventually replace the hand-written equipment/hazard dictionary

## What's real vs. what's still manual (being upfront about this)

- Document ingestion and retrieval genuinely generalize to new PDFs —
  tested on documents with completely different structure (numbered legal
  clauses vs. flowing procedural text) and both handled correctly.
- The knowledge graph's equipment→hazard→regulation mapping is currently a
  hand-written dictionary, not auto-populated from new documents by
  default. `entity_extractor.ipynb` is built and validated to close this
  gap; wiring it into the live graph-building path is a separate,
  in-progress patch, not done in every notebook here.
- RAG answer generation runs on Gemini; the fine-tuned model is scoped to
  entity extraction only, not the core Q&A.

## Setup

Everything's built and run in Google Colab against Google Drive (PDFs,
vector store, graph, and query cache all persisted there between runs).
For the exact walkthrough of how each phase was built and debugged, run
the notebooks in the order listed above.

`requirements.txt` is out of date against what's actually imported in these
notebooks — it's missing `langgraph`, `networkx`, `peft`, `transformers`,
`bitsandbytes`, and `pymupdf4llm`, and still lists `unstructured`,
`deepeval`, and `langsmith`, none of which ended up used. Worth
regenerating from the actual notebook imports before anyone else tries to
set this up from a fresh clone.
