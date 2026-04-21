# Chunking strategy

This document describes the **design and execution strategy** for the regulatory PDF chunking pipeline. Implementation lives in [`chunking/aml_chunking_pipeline.ipynb`](chunking/aml_chunking_pipeline.ipynb). For repository layout and how to run the notebook, see [`README.md`](README.md).

---

## 1. Problem statement

Dense AML and legal PDFs mix **hierarchy** (parts, sections, annexes), **tabular data**, **lists**, **boxed examples**, and **cross-references** to standards (e.g. FATF Recommendations). Naive fixed-size splitting destroys structure, splits tables mid-row, and produces chunks that are hard to filter or cite.

The chunker’s job is to produce **embedding-ready units** with **stable metadata**, **parent/child relationships** for two-stage retrieval, and **references** usable in compliance-oriented Q&A.

---

## 2. Design principles

| Principle | Rationale |
|-----------|-----------|
| **Hierarchy first** | Section paths and heading levels support navigation, filtering (“only Section 2”), and grounded answers. |
| **Atomic special blocks** | Tables, lists, and boxes stay intact when possible to avoid unusable partial tables or broken markdown. |
| **Parent / child chunks** | Children target retrieval size; parents hold broader context for the LLM after a hit. |
| **Reference surfacing** | Regex-based extraction of R.x, INR.x, Section x, Annex, Box, Table cues improves metadata and query routing. |
| **Deterministic provenance** | `chunk_hash`, page span, `ingested_at`, and `parser_version` support audit and reprocessing. |
| **Parallel CPU use** | Page parsing and table extraction are expensive; multi-core use cuts wall-clock time on large PDFs. |

---

## 3. Parser stack

### 3.1 Primary path (notebook default)

- **PyMuPDF (`fitz`)** — Fast, layout-aware text extraction per page; drives block geometry and reading order.
- **pdfplumber** — Table extraction per page, in parallel with text tasks where the pipeline merges results.

Tables and body text can overlap spatially; the assembly step **reconciles** overlaps so tables are not duplicated as raw prose and prose does not bisect table regions.

### 3.2 Tokenization

- **Preferred:** `tiktoken` encoding `cl100k_base` — aligns with common OpenAI-class embedding models for consistent token budgets.
- **Fallback:** If `tiktoken` cannot be initialized (e.g. air-gapped compliance environments), a **character-based approximation** (~4 characters per token) keeps the pipeline running with slightly less accurate size control.

### 3.3 Production upgrade path

- **Docling** (IBM) is noted in the notebook as a swap for PyMuPDF when **stronger document structure and table fidelity** are required; integration would replace or wrap the parse stage, not the chunk metadata model.

---

## 4. Internal representations

### 4.1 `Block` (pre-chunk)

Structural unit after PDF assembly, before token-based splitting:

- **Types:** `prose`, `heading`, `table`, `list`, `box`, `definition`, `footnote`, etc.
- **Fields:** text, page span, optional `heading_level`, `section_path`, and for tables `table_html` / `raw_rows`.

Blocks preserve **reading order** and **type** so chunking rules can differ by type (e.g. never split a table across two embed chunks).

### 4.2 `Chunk` (output record)

Final row written to JSONL:

- **Identity:** `chunk_id` (UUID), `doc_id`, `chunk_hash`.
- **Content:** `text`, `chunk_type`, `token_count`.
- **Structure:** `section_path`, `hierarchy_level`, `page_start` / `page_end`.
- **Graph:** `parent_chunk_id`, `child_chunk_ids` — tree for “retrieve child, expand parent.”
- **Semantics (lightweight):** `regulatory_references` from patterns; `topic_tags` reserved for a later LLM pass.
- **Provenance:** `doc_title`, `doc_version`, `doc_authority`, `ingested_at`, `parser_version`.

Child chunks may prefix text with **`[Context: section > subsection]`** so embeddings retain breadcrumb without losing the fine-grained span.

---

## 5. Pipeline stages (end-to-end)

### Stage A — Parallel PDF parsing

- Open the PDF once to get **page count**.
- Build tasks `(pdf_path, page_index)` for all pages.
- Run **PyMuPDF extraction** in a **process pool** (`ProcessPoolExecutor`): CPU-bound work, GIL-friendly for the C library.
- Run **pdfplumber table extraction** similarly in parallel.
- Merge per-page results into a single ordered structure for downstream assembly.

**Why processes for pages:** `pdfplumber` can hold the GIL during Python-heavy table work; processes isolate that cost.

### Stage B — Block assembly (sequential)

- Walk merged page material in **document order**.
- Infer or carry **section_path** from headings and document conventions.
- Classify fragments into **Block** types; attach tables as atomic blocks with HTML/rows when available.
- **Deduplicate** table vs. text overlap so content appears once, consistently.

This stage is **sequential** to guarantee a single global reading order and stable section stacks.

### Stage C — Chunk building (parallel where safe)

- From blocks, apply **token-budget rules** to form parent-sized and child-sized segments.
- **Preserve** certain types (e.g. tables, boxes) as single chunks when under ceilings.
- **Split** long prose with overlap between adjacent chunks.
- Assign **parent/child links**: large “parent” spans link to smaller “child” retrieval units.
- Run chunk construction in a **thread pool** where appropriate — lighter coordination, tokenizer releases GIL on encode.

### Stage D — Post-processing

- Drop or flag chunks below a **minimum token** threshold (noise or parse errors).
- Enforce a **hard maximum** token count to reduce embedder truncation surprises.
- **Repair parent/child links** after filtering: remove pointers to dropped IDs so the graph stays consistent.

### Stage E — Serialization

- Write **JSONL**: one UTF-8 JSON object per line, `ensure_ascii=False` for proper Unicode in regulatory text.

Orchestration entrypoint: **`process_document(pdf_path, doc_meta, output_jsonl, ...)`**.

---

## 6. Size and overlap policy (defaults)

Values are **constants in the notebook** and should be tuned per corpus and embedding model.

| Parameter | Typical role |
|-----------|----------------|
| Child target (~400 tokens) | Retrieval granularity for vector search. |
| Child overlap (~15%) | Reduces boundary cuts through sentences and definitions. |
| Parent target (~1500 tokens) | Context window for generation after retrieving a child. |
| Parent overlap (~10%) | Smooths parent boundaries for multi-hop questions. |
| Minimum tokens (~50) | Flags suspiciously small fragments. |
| Maximum tokens (~800) | Hard cap per chunk for embedding models with strict limits. |

**Regulatory PDFs** often have long uninterrupted articles; the overlap + parent layer avoids losing coherence at splits.

---

## 7. Parallelism and environment

- **`CHUNKER_WORKERS`** — Optional environment variable; defaults to about **CPU count − 1** to leave a core for the OS.
- Tuning: I/O-bound vs. CPU-bound balance; on **very large corpora**, scale **out** by running one process per PDF at the orchestration layer (not shown in the single-notebook demo).

---

## 8. Quality assurance

After export, the notebook supports:

- **DataFrame summary** — Counts by `chunk_type`, token distribution, page coverage, top extracted references.
- **Sanity checks** (suitable for CI-style gates):
  - Too many **undersized** chunks (possible parse errors).
  - Any **oversized** chunks (truncation risk).
  - **Orphan headings** (heading-only fragments).
  - **Fractured tables** (markdown tables missing a valid header separator row).
- **Visual diagnostics** — Histograms and section charts saved as `chunk_diagnostics.png`.
- **HTML report** — Human review of the first N chunks with metadata and check summary (`chunk_report.html`).

---

## 9. What this chunker does not do (by design)

- **LLM-based segmentation** — Boundaries are structural + token rules, not model-predicted (keeps runs reproducible and cheap).
- **Full OCR** — Assumes selectable text; scanned-only PDFs need a separate OCR stage upstream.
- **Legal interpretation** — `regulatory_references` are pattern-based hints, not compliance judgments.

---

## 10. Hardening and extensions

Aligned with the notebook “next steps”:

| Direction | Purpose |
|-----------|---------|
| **Docling** | Richer layout and table structure for difficult PDFs. |
| **Langfuse / logging** | Trace `process_document` in production. |
| **Great Expectations** | Encode sanity checks as data tests on each ingest. |
| **Incremental reprocessing** | Hash pages or PDF revision to skip unchanged work. |
| **LLM metadata pass** | Fill `topic_tags`, obligations, entities per chunk. |
| **Embed + index** | Vectors in pgvector, Pinecone, FAISS, etc.; optional parent expansion at query time. |

---

## 11. Outputs in `chunking/`

When you run the notebook with working directory `chunking/`, artifacts land alongside the notebook:

| File | Description |
|------|-------------|
| `fatf_aml_nra_chunks.jsonl` | Example chunk export (FATF ML NRA guidance). |
| `chunk_report.html` | Review report from the export cell. |
| `chunk_diagnostics.png` | Plots from the diagnostics cell. |

Place your **source PDF** in `chunking/` or set `PDF_PATH` to an absolute path before running.
