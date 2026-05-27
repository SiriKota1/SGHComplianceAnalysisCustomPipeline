# Policy Compliance RAG Pipeline

This directory contains the full development history of a **Retrieval-Augmented Generation (RAG) pipeline** for automated JCI (Joint Commission International) ESS 7th Edition policy compliance checking. The notebooks progress from foundational vector database construction through re-ranker experimentation to a production-ready compliance analysis pipeline.

---

## Project Overview

The pipeline answers queries such as:
> *"Does this hospital transfer policy comply with JCI Standard ACC.1?"*

It retrieves the most relevant JCI standard chunks, re-ranks them for precision, and generates a structured compliance verdict using a local LLM.

**Core components across all notebooks:**
- **Document source**: JCI ESS 7th Edition (`ESS Standards_7th Ed .pdf`)
- **Chunking strategy**: Chapter-wise segmentation by JCI chapter codes (ACC, PCC, COP, etc.) with standard-level splitting
- **Vector store**: FAISS, one database per chapter
- **Embeddings**: HuggingFace sentence-transformers
- **Re-rankers**: CrossEncoder and/or Qwen3-Reranker-4B
- **LLM**: Mistral-7B-Instruct-v0.3 or Qwen2.5-7B-Instruct

---

## Notebooks Overview

| Notebook | Stage | Purpose |
|---|---|---|
| `VectorDB_v2.ipynb` | Vector DB — v2 | Baseline chapter-wise FAISS DB with simple flat chunking |
| `ChpWiseVectorDB_v3.ipynb` | Vector DB — v3 | Adds standard-code splitting and TOC skipping |
| `ChpWiseVectorDB_v5.ipynb` | Vector DB — v5 | Fixes page count bug; adds page total tracking |
| `ChpWiseDB_RE_RANKER_v9.ipynb` | Re-ranker — v9 | Adds CrossEncoder re-ranking on top of chapter-wise DBs |
| `ChpWiseDB_RE_RANKER_v12_agg_scores.ipynb` | Re-ranker — v12 | Multi-model evaluation with aggregate scores and CSV export |
| `MISTRAL_RE_RANKER_CLEAN.ipynb` | LLM Pipeline — Mistral | Dual re-ranker (CrossEncoder + Qwen) → Mistral answer generation |
| `QWEN_RE_RANKER_CLEAN.ipynb` | LLM Pipeline — Qwen | Same dual re-ranker pipeline with Qwen2.5 as the generator LLM |
| `MISTRAL_QWEN_policy_pipleline.ipynb` | LLM Pipeline — Combined | Dual re-ranker experiment comparing Mistral and Qwen generation |
| `policy_compliance_pipeline_v2.ipynb` | Compliance Pipeline — v2 | Full pipeline: policy PDF + JCI DB + routed retrieval + Mistral compliance analysis |
| `POLICY_PIPELINE_routed_retrieval.ipynb` | Compliance Pipeline — Routed | Removes policy DB; adds three-tier routed retrieval (exact code → chapter → global) |
| `Policy_Compliance_Final_Pipeline.ipynb` | **Final Pipeline** | Production-ready: routed retrieval, expanded keyword parser, per-chapter JSON output |

---

## Version Differences

### Vector DB Notebooks

#### `VectorDB_v2.ipynb`
The earliest version. Pages are grouped by JCI chapter code, but chunks are produced with a simple flat splitter (`RecursiveCharacterTextSplitter`) applied directly to raw page documents. No TOC removal, no standard-code-level splitting, no end-of-book cleanup. Standard detection (`ACC.1`, `AOP.2`) uses a basic regex applied to full text but is not used to structure the chunks.

**Key settings**: `chunk_size=1000`, `chunk_overlap=200`

#### `ChpWiseVectorDB_v3.ipynb`
Introduces **structured standard-level chunking**. Each chapter's pages are merged into a single string, the TOC section is skipped (everything before `Standards, Intents, and Measurable Elements`), and the text is then split by detected standard codes (`ACC.1`, `AOP .1`, `Requirement APR.1`). Metadata fields `chapter` and `standard_code` are attached to every chunk. End-of-book pages (glossary, appendix, index) are cleaned from the HRP chapter. Page count tracking has a minor bug (variable `total` not initialised before the loop, causing a potential `NameError`).

**Key settings**: `chunk_size=800`, `chunk_overlap=150`

#### `ChpWiseVectorDB_v5.ipynb`
Identical to v3 in logic but fixes the page count bug by initialising `total = 0` before the loop. CrossEncoder is imported (in preparation for re-ranker work) but not yet used. This is the stable base used by all subsequent re-ranker and pipeline notebooks.

**Key settings**: `chunk_size=800`, `chunk_overlap=150`

---

### Re-ranker Notebooks

#### `ChpWiseDB_RE_RANKER_v9.ipynb`
First notebook to introduce **CrossEncoder re-ranking**. After FAISS retrieval returns the top-k chunks per chapter DB, a CrossEncoder (`cross-encoder/ms-marco-MiniLM-L-6-v2`) scores each (query, chunk) pair and reorders results. Evaluation uses a gold standard query set with hit-rate metrics (Hit@1, Hit@3, Hit@5). No Qwen re-ranker yet. Chunk parameters move to `chunk_size=800`, `chunk_overlap=150`.

#### `ChpWiseDB_RE_RANKER_v12_agg_scores.ipynb`
Significantly expanded evaluation framework. Key additions:
- **Three embedding models tested in parallel**: MiniLM, BGE-small, BGE-large
- **Two re-rankers tested in parallel**: CrossEncoder and Qwen3-Reranker-4B
- **Aggregate score** (`norm_score_aggregate`): running product of normalised scores across ranks, used to measure how well the re-ranker front-loads relevant results
- **Extended metrics**: MRR (Mean Reciprocal Rank) and NDCG@5 added alongside Hit@K
- **CSV exports per model**: `STANDARD_EVAL_{model}.csv`, `CHUNKS_{model}_{reranker}_top5/top10.csv`, `CHUNK_RELEVANCE_SUMMARY_*.csv`, `SUMMARY_ALL_MODELS.csv`
- Qwen scores use log-softmax over yes/no tokens (normalised 0–1)
- CrossEncoder scores use sigmoid of raw logit (normalised 0–1)
- Chunk parameters upgraded: `chunk_size=1000`, `chunk_overlap=250`

---

### LLM Generation Notebooks

#### `MISTRAL_RE_RANKER_CLEAN.ipynb`
Adds **Mistral-7B-Instruct-v0.3** as the answer-generation LLM on top of the dual re-ranker stack (CrossEncoder → Qwen). The top-5 re-ranked chunks each independently generate an answer; answers are compared to identify consensus. Uses `LLM_TOP_K=5` and `LLM_MAX_TOKENS=512`.

#### `QWEN_RE_RANKER_CLEAN.ipynb`
Structurally identical to `MISTRAL_RE_RANKER_CLEAN` but replaces the generator LLM with **Qwen2.5-7B-Instruct**. Intended for side-by-side quality comparison of Mistral vs. Qwen outputs on the same retrieval stack.

#### `MISTRAL_QWEN_policy_pipleline.ipynb`
A combined experiment notebook that runs both LLMs (Mistral and Qwen) in the same environment for direct comparison. Configuration, chunking, and retrieval are shared; only the `LLM_MODEL_ID` differs between runs.

---

### Compliance Pipeline Notebooks

#### `policy_compliance_pipeline_v2.ipynb`
First **end-to-end compliance pipeline**. Ingests two PDFs: the JCI ESS (the standard) and a hospital policy document (e.g. Transfer Policy). Both are chunked and stored in separate FAISS databases. A query parser classifies queries into `compliance_check`, `comparison`, or `clarification` and identifies mentioned JCI chapter codes. Retrieval queries both databases and Qwen re-ranks the merged results. Mistral generates a structured compliance verdict. Ten sections with a deliberate run order (Section 9 — LLM generation — is gated until retrieval accuracy is confirmed).

**Notable**: includes a `KNOWN_POLICIES` dictionary mapping policy keywords to display names.

#### `POLICY_PIPELINE_routed_retrieval.ipynb`
Removes the policy PDF database. Introduces **three-tier routed retrieval**:
1. **Tier 1 (Exact)**: directly looks up chunks whose `standard_code` metadata matches parsed standard codes from the query
2. **Tier 2 (Chapter)**: queries only the chapter-specific FAISS DB
3. **Tier 3 (Global)**: queries all chapter DBs if earlier tiers yield fewer than `MIN_CHUNKS=7` results

The query parser is extended to extract multiple standard codes from a single query. Compliance generation now compares answers generated from top-3, top-5, and top-7 chunks, saving all three to `COMPLIANCE_ANALYSIS.csv`. A `@dataclass` (`ParsedQuery`) structures parser output.

#### `Policy_Compliance_Final_Pipeline.ipynb` 
The production-ready notebook. Builds on routed retrieval with the following improvements:
- **Expanded task keyword lists**: `compliance_check`, `comparison`, and `clarification` categories now cover a broader vocabulary (e.g. `satisfy`, `fully align`, `compared with`, `what elements`, `what documentation`)
- **Chunk size increased** to `chunk_size=1500` to retain more context per standard
- **Refined system prompt**: the LLM is given strict evidence-anchoring rules, explicit uncertainty signalling requirements, and a formal audit-ready output style. Bullet points and tables are prohibited in the LLM output unless explicitly requested
- **Per-chapter JSON export**: answers are saved both as CSV (`COMPLIANCE_ANALYSIS_Baseline_{chapter}.csv`) and structured JSON (`COMPLIANCE_ANSWERS_Baseline_{chapter}.json`) for downstream use
- **Chunk loading decoupled**: retrieval chunks are pre-saved to `VALIDATION_CHUNKS_{chapter}.json` and loaded independently for answer generation, separating retrieval validation from LLM generation

---

## Pipeline Architecture (Final)

```
JCI ESS PDF
    └── Load pages → Group by chapter code → Clean (TOC, end-of-book)
        └── Standard-level chunks → FAISS DB per chapter
                                          │
Query ──► Parser (task type + standard codes + chapter codes)
                │
                └──► Tier 1: Exact code lookup
                     Tier 2: Chapter-scoped FAISS search     ──► Merge & dedup
                     Tier 3: Global FAISS search (fallback)
                                          │
                               Qwen3-Reranker-4B
                                          │
                        Top-3 / Top-5 / Top-7 chunks
                                          │
                           Mistral-7B-Instruct-v0.3
                                          │
                      Compliance verdict (CSV + JSON)
```

---

## Setup

### Requirements

Install dependencies with:

```bash
pip install -r requirements.txt
```

Key packages:

| Package | Purpose |
|---|---|
| `langchain`, `langchain-community`, `langchain-openai` | Document loading, splitting, vector store |
| `faiss-cpu` | FAISS vector store (CPU) |
| `sentence-transformers` | HuggingFace embedding models and CrossEncoder |
| `torch` | Required for Qwen re-ranker and LLM inference |
| `pypdf` | PDF loading |
| `pandas`, `openpyxl` | Evaluation output and CSV/Excel handling |
| `tiktoken` | Token counting for OpenAI-compatible models |

### Configuration

Before running any notebook, update the `JCI_PDF` path at the top of the config section to point to your local copy of the JCI ESS PDF:

```python
JCI_PDF = '/path/to/your/ESS Standards_7th Ed .pdf'
```




### First Run

The FAISS databases are built once and saved under `saved_dbs/{embedding_model}/{chapter}/`. Subsequent runs load from disk automatically. To rebuild, delete the `saved_dbs/` directory.

---

## JCI Chapter Codes

The pipeline supports all 16 JCI ESS chapters:

`APR` · `IPSG` · `ACC` · `PCC` · `AOP` · `COP` · `ASC` · `MMU` · `QPS` · `PCI` · `GLD` · `FMS` · `SQE` · `MOI` · `MPE` · `HRP`

---

## Outputs

| File | Produced by | Contents |
|---|---|---|
| `STANDARD_EVAL_{model}.csv` | v12 re-ranker | Per-query retrieval results and Hit@K/MRR/NDCG metrics |
| `CHUNKS_{model}_{reranker}_top5/top10.csv` | v12 re-ranker | Top retrieved chunks per query and re-ranker |
| `CHUNK_RELEVANCE_SUMMARY_*.csv` | v12 re-ranker | Aggregate score trends across embedding models |
| `SUMMARY_ALL_MODELS.csv` | v12 re-ranker | Cross-model metric summary |
| `VALIDATION_TOP1/3/5/7.csv` | Routed pipeline | Retrieval accuracy at different depths |
| `COMPLIANCE_ANALYSIS.csv` | Routed pipeline | Compliance verdicts (top-3, 5, 7) for all queries |
| `COMPLIANCE_ANALYSIS_Baseline_{chapter}.csv` | Final pipeline | Per-chapter compliance verdicts |
| `COMPLIANCE_ANSWERS_Baseline_{chapter}.json` | Final pipeline | Structured JSON answers for downstream use |

---

## Maintainer

Siri Kota
