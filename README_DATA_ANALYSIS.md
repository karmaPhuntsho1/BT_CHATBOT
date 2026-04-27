# Data Analysis and Modeling Notes

This document describes the analytical methods and exploratory data analysis (EDA) used in this project, based on the current implementation in `pipeline/`, `chatbot/`, and `evaluation/`.

## 1) Analytical Methods Used

### A. Semantic Retrieval (Vector Search for RAG)
- **What it is:** Dense-vector retrieval over embedded text chunks stored in Chroma.
- **Why selected:** The task is question answering over heterogeneous telecom documents (Excel, JSON, CSV, PDF). Retrieval is more suitable than supervised label prediction because it directly grounds responses in source content.
- **Implementation:** `chatbot/rag_chain.py` + `pipeline/ingest.py`

**Key parameters**
- `embedding_model`: `all-MiniLM-L6-v2`
- `rag_retriever_k`: default `5`
- `rag_mmr_fetch_k`: default `24`
- `rag_use_mmr`: default `True` (MMR diversification)
- Chroma collection name and persist directory are configurable.

**Main assumptions**
- Semantically similar customer questions and knowledge chunks are close in embedding space.
- Retrieved top-k chunks contain sufficient evidence for response generation.
- Document chunk quality (cleaning and deduplication) strongly affects retrieval quality.

**Reliability/performance impact**
- MMR reduces redundant chunks and improves coverage diversity.
- Fixed `k` and `fetch_k` stabilize latency and retrieval behavior.
- Using a local embedding model improves reproducibility and avoids external API dependency.

---

### B. Rule-Based Fallback Classification (Routing)
- **What it is:** Binary routing outcome between `rag` and `fallback` based on retrieval/LLM success.
- **Why selected:** Support systems need safe degradation when retrieval/model calls fail or input is empty.
- **Implementation:** `chatbot/hybrid_bot.py`

**Key parameters**
- Empty input -> forced `fallback` with guidance message.
- Retrieval/generation exceptions -> `fallback`.
- Successful non-error RAG output -> `rag`.

**Main assumptions**
- A reliable answer requires successful retrieval + generation.
- If evidence or model output is unavailable, conservative fallback is safer than speculative response.

**Reliability/performance impact**
- Strongly improves robustness and user safety in failure scenarios.
- Prevents hallucination-like behavior under missing context.

---

### C. Heuristic Confidence Scoring
- **What it is:** Confidence derived from number of retrieved sources (`n`): 0, 0.4, 0.7, 1.0.
- **Why selected:** Lightweight interpretability signal without training/calibration overhead.
- **Implementation:** `_confidence_from_sources()` in `chatbot/rag_chain.py`

**Key parameters**
- `n >= 3 -> 1.0`
- `n == 2 -> 0.7`
- `n == 1 -> 0.4`
- `n == 0 -> 0.0`

**Main assumptions**
- More independent supporting chunks typically indicate better grounding.

**Reliability/performance impact**
- Provides transparent confidence approximation for logging and monitoring.
- Low computational cost.

---

### D. Content Deduplication via Hash Fingerprints
- **What it is:** Cross-source duplicate removal using normalized text + SHA256 fingerprints.
- **Why selected:** Multiple raw sources contain overlapping content; duplicates degrade retrieval quality and increase index noise.
- **Implementation:** `pipeline/deduplicate.py`, `pipeline/text_normalize.py`

**Key parameters**
- Source priority order:
  1. `bt_web`
  2. `structured_json`
  3. `excel_ticket`
  4. `csv_row`
  5. `pdf_page`
- Dedup text normalization: lowercase, punctuation-insensitive canonicalization.
- Minimum normalized length threshold (short items dropped).

**Main assumptions**
- Canonicalized text hash is an adequate duplicate proxy.
- Higher-priority sources are typically cleaner/more authoritative.

**Reliability/performance impact**
- Reduces repeated chunks, improving retrieval precision.
- Improves index efficiency and consistency.

---

### E. Methods Not Used (Important Clarification)
The current project does **not** implement classical supervised/unsupervised ML methods such as:
- Linear/logistic regression
- Decision-tree/ensemble classification models
- Clustering (K-Means, DBSCAN, etc.)

This is expected: the system is primarily a retrieval-augmented assistant pipeline, not a predictive tabular ML model.

## 2) Hyperparameter Tuning and Model Selection

### What is currently done
- Parameter values are configurable in `config/settings.py` (and environment variables), e.g.:
  - `rag_retriever_k`
  - `rag_mmr_fetch_k`
  - `rag_use_mmr`
  - `ollama_temperature`
  - model choices (`ollama_model`, `embedding_model`)
- `evaluation/evaluate.py` runs built-in and domain test suites and reports pass rates/method distribution.

### What is **not** currently implemented
- Automated search strategies such as:
  - Random search
  - Grid search
  - Bayesian optimization
- Cross-validation workflows for hyperparameter selection

### Current selection criteria
- Operational behavior under domain tests (`PASS/FAIL`)
- Routing stability (`rag` vs `fallback` behavior)
- Practical quality judgment (groundedness, naturalness, and robustness)

### Reliability/performance implication
- Manual configuration + deterministic evaluation keeps pipeline simple and maintainable.
- Lack of systematic hyperparameter search means there is room to further optimize quality/latency trade-offs.

## 3) Exploratory Data Analysis (EDA) Performed in This Project

The project performs practical data profiling and quality checks during ingestion and evaluation.

### A. Ingestion-Level EDA / Data Profiling
Implemented in `pipeline/ingest.py` and `pipeline/verify_kb.py`:
- Counts loaded records/pages per source file (`file_counts`)
- Total raw document count before deduplication
- Deduplication stats:
  - `input`
  - `output`
  - `dropped_duplicate`
  - `dropped_too_short`
- Indexed document count written to Chroma
- Manifest generation in `database/ingest_manifest.json` for run traceability

**Why selected**
- These checks quickly validate data coverage and ingestion quality before deploying the assistant.

**Assumptions**
- File-level and aggregate counts are sufficient to detect major ingestion regressions.

**Reliability/performance impact**
- Early detection of missing/broken sources improves downstream response reliability.

### B. Runtime/Evaluation EDA
Implemented in `evaluation/evaluate.py`:
- Built-in and domain-specific test pass rates
- Method usage distribution for a run (`method_counts`)
- SQLite operational metrics:
  - total conversations
  - number of feedback rows
  - average feedback rating
  - fallback rate

**Why selected**
- Provides a compact operational picture of routing quality and user feedback trends.

**Assumptions**
- Evaluation cases are representative of expected telecom support queries.
- Fallback rate is a useful proxy for retrieval/model coverage.

**Reliability/performance impact**
- Monitoring these metrics supports iterative quality improvements and safer deployment decisions.

## 4) Tools and Libraries Used

### Language/runtime
- Python

### Data handling and parsing
- `pandas` (Excel/CSV parsing and row handling)
- `openpyxl` (Excel support through pandas)
- `pypdf` + `PyPDFLoader` (PDF extraction)
- `json` / standard library IO utilities

### Retrieval and LLM stack
- `langchain-*` packages
- `langchain-chroma` / `chromadb` (vector store and retrieval)
- `sentence-transformers` backend via `HuggingFaceEmbeddings`
- `langchain-ollama` / local Ollama model for generation

### API, config, and testing
- `fastapi`, `uvicorn`
- `pydantic-settings`
- `pytest`
- `sqlite3` (conversation/feedback logging and analytics)

## 5) Practical Summary

- The project is a **RAG-focused analytical system** with robust ingestion, deduplication, retrieval, and evaluation loops.
- Reliability comes mainly from:
  - conservative fallback routing
  - source cleaning/deduplication
  - manifest-based ingestion validation
  - domain test execution and runtime metric tracking
- Performance/quality trade-offs are currently controlled by configurable retrieval and generation parameters, evaluated through scripted test runs rather than automated hyperparameter optimization.

