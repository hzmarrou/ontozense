# Ontozense

Auto-generate domain ontologies from a curated knowledge base.

Ontozense takes authoritative domain documents, governance spreadsheets, database schemas, and production code, and produces a rich data dictionary with provenance for every claim — the starting point for domain experts to review in their existing Excel workflow instead of building from a blank page.

## The four-source pipeline

Rather than trying to extract everything from a single source, Ontozense combines four complementary sources, each contributing only the fields it can defensibly produce:

| Source | Input | Provides | Method |
|---|---|---|---|
| **A — Authoritative domain documents** | PDF, DOCX, MD, HTML — regulations, internal policies, academic papers, vendor specs | Concepts, relationships, definitions, citations | OntoGPT + SPIRES (peer-reviewed), with a raw-output bypass to recover items lost in SPIRES recursion, plus a regex-based definitions second pass |
| **B — Governance / data quality spreadsheets** | Canonical CSV (14 columns) | Critical-data flags, mandatory/optional flags, the six DQ dimensions (completeness, accuracy, uniqueness, timeliness, consistency, validity) | Schema-aware structured parser, no LLM — currently a locked-API scaffold, implementation pending |
| **C — Database schemas** | PostgreSQL live introspection or Django model files | Entities, properties, types, FK relationships, enum values, NOT NULL constraints | Direct introspection via `psycopg2` / AST parsing of Django models — no LLM |
| **D — Production code** | Python, SQL | Computational rules, thresholds, state transitions, classification logic, constraints | Deterministic parsing first (stdlib `ast`, `sqlglot`) then LLM labelling against the parsed symbol table — methodology: AI-RBX (*Leveraging Generative AI for Extracting Business Requirements*), 93% expert agreement at 3.4M LoC. Currently the deterministic layer is implemented; LLM labelling and the symbol-table validator are pending. |

A **router** classifies each incoming file by extension and content sniffing, dispatching it to the right source. Multi-source routing is supported — a markdown developer guide with both prose and code blocks goes to both A and D.

A **fusion layer** (next milestone) combines the four sources into a single rich data dictionary, with per-field provenance, conflict detection, and confidence-aware merge rules. The rich dictionary is the **output** of fusion, not the extraction target of any single source.

## Quick start

```bash
pip install -e ".[dev]"

# Source A — authoritative domain documents
ontozense extract-a path/to/document.md --output dd.xlsx --json dd.json

# Router — classify an incoming file
ontozense ingest path/to/unknown/file.md --dry-run
ontozense ingest path/to/knowledge-base/ --auto --auto-threshold 0.9
```

The `extract-a` command writes an Excel file with two sheets (`Concepts` and `Relationships`) plus a JSON sidecar that carries per-field confidence and provenance. Concepts below the review threshold (default 0.7) are flagged for human attention rather than silently dropped.

## Confidence scoring

Every extracted field gets a confidence score in `[0.0, 1.0]`. The rubric is **field-aware** — different field categories use different rules:

- **NARRATIVE fields** (definition, term_definition) — `0.95` verbatim substring of the source / `0.75` ≥70% significant word overlap / `0.55` ≥40% overlap / `0.35` minimal / `0.00` empty
- **CITATION fields** — `0.95` verbatim / `0.70` matches citation regex pattern / `0.40` non-empty but unrecognised / `0.00` empty
- **RELATIONSHIP TRIPLES** — `0.95` both endpoints verbatim / `0.625` one endpoint / `0.30` neither
- **ENUM fields** — `0.85` valid enum / `0.30` non-empty not in valid set / `0.00` empty
- **STRUCTURED-SOURCE fields** (from Source B or C) — `0.95` uniformly, since there's no LLM judgement and no hallucination risk

Name-only concepts are explicitly penalised: a concept with a verbatim name but no definition scores ~`0.475` overall (not `0.95`), flagging it for review.

## Honest failure modes

Per the project's operating principle, the system refuses to silently succeed with bad output:

- **Exit code 2** — zero elements extracted. No Excel is written; the CLI prints likely causes (OntoGPT subprocess failure, wrong template, document lacks structured content).
- **Exit code 3** — all extracted elements (concepts + relationships) have confidence below 50%. Excel is written anyway so the user can inspect it, but scripts can detect the low-quality exit via the non-zero code.
- **Router auto-dispatch** only fires when the routing confidence exceeds 0.9. Lower-confidence decisions are listed as skipped so a human can review them.

## Repository layout

```
src/ontozense/
├── cli.py                          # Typer CLI — extract-a, ingest, extract, refine, export, ...
├── log.py                          # Per-domain append-only log.md writer
├── router/                         # File router (extension rules + content sniffing)
├── extractors/
│   ├── domain_doc_extractor.py     # Source A — domain documents via OntoGPT
│   ├── definitions_extractor.py    # Source A second pass — regex definitions
│   ├── governance_extractor.py     # Source B — canonical CSV (scaffold)
│   ├── django_schema.py            # Source C — Django model AST parser
│   ├── pg_schema.py                # Source C — PostgreSQL introspection
│   ├── code_extractor.py           # Source D — Python AST + SQL via sqlglot
│   └── ontogpt_extractor.py        # OntoGPT subprocess wrapper
├── exporters/                      # Excel + Playground JSON outputs
├── core/                           # OWL ontology manager + schema refiner
└── templates/                      # LinkML templates for OntoGPT
```

## Status

The four-source scaffolding is complete. Source A is production-shaped and produces rich output on real regulatory documents (Basel D403: 30+ concepts, 20+ relationships). Sources C and D have deterministic layers in place. Source B has a locked API scaffold. The fusion layer is the next milestone.

## Note on design documentation

Internal docstrings reference files like `docs/PLAYBOOK.md`, `docs/AI-RBX.pdf`, and `docs/SPIRES.md`. These design documents are maintained in a separate private location and are not included in this repository. The code is self-contained and runs without them.

## Methodology citations

- **SPIRES** — Caufield et al. 2024, *Bioinformatics*. Zero-shot structured extraction from LLMs via LinkML schemas.
- **AI-RBX** — *Leveraging Generative AI for Extracting Business Requirements*. Deterministic parsing → LLM labelling → symbol-table validator pipeline for code-based rule extraction.

## License

Copyright © Hicham Zmarrou. All rights reserved.
