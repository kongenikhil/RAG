# AML regulatory PDF — parsing & chunking for RAG

This repository contains a **Jupyter-based ingestion pipeline** that turns dense anti–money laundering (AML) and regulatory PDFs into **hierarchical, metadata-rich chunks** suitable for retrieval-augmented generation (RAG). It was developed around the FATF *Money Laundering National Risk Assessment Guidance* (2024) but is intended to generalize to other legal and policy PDFs.

The pipeline prioritizes **structure preservation** (sections, tables, lists, boxed content), **parent/child chunking** for two-stage retrieval, and **parallel CPU usage** during PDF parsing and chunk assembly.

## Repository layout

| Path | Description |
|------|-------------|
| [`chunking.md`](chunking.md) | **Detailed chunking strategy** — design principles, parser stack, pipeline stages, parallelism, QA, and extensions |
| [`chunking/`](chunking/) | **Notebook, example outputs, and artifacts** for the PDF → JSONL pipeline |
| [`chunking/aml_chunking_pipeline.ipynb`](chunking/aml_chunking_pipeline.ipynb) | Main notebook: parsing, chunking, JSONL export, sanity checks, plots, HTML report |
| [`chunking/aml_chunking_pipeline_executed.ipynb`](chunking/aml_chunking_pipeline_executed.ipynb) | Saved run with outputs embedded |
| [`chunking/fatf_aml_nra_chunks.jsonl`](chunking/fatf_aml_nra_chunks.jsonl) | Example output: one JSON object per line (doc `fatf_ml_nra_2024`) |
| [`chunking/chunk_report.html`](chunking/chunk_report.html) | Example HTML quality report |
| [`chunking/chunk_diagnostics.png`](chunking/chunk_diagnostics.png) | Example diagnostic charts |
| [`.gitignore`](.gitignore) | Ignores virtualenvs, secrets, vector DB dirs, etc. |

For **how and why** chunking works (blocks vs chunks, parent/child policy, PyMuPDF vs pdfplumber, token defaults), read **[`chunking.md`](chunking.md)**. The sections below stay focused on setup, schema, and running the notebook.

## What the pipeline does (summary)

1. **Parallel PDF parsing** — PyMuPDF for text/layout; pdfplumber for tables; merged into ordered blocks.
2. **Block assembly** — Sequential pass for global reading order and section paths.
3. **Parallel chunk building** — Token-budget parent/child chunks with overlap and metadata.
4. **JSONL export** — One UTF-8 JSON line per chunk.

**Parsing:** `tiktoken` `cl100k_base` when available; optional Docling for production-grade layout (see `chunking.md`).

**Orchestration:** `process_document(pdf_path, doc_meta, output_jsonl, ...)` in the notebook.

## JSONL schema

Each line in the output `.jsonl` file is one JSON object. Representative fields:

| Field | Description |
|-------|-------------|
| `chunk_id` | UUID for the chunk |
| `doc_id` | Stable document identifier (from metadata or derived from filename) |
| `text` | Chunk text (child chunks may include a `[Context: …]` prefix) |
| `chunk_type` | e.g. `prose`, `heading`, `table`, `list`, `box`, … |
| `section_path` | List of section titles forming a breadcrumb |
| `hierarchy_level` | Integer depth hint |
| `page_start`, `page_end` | Inclusive page range in the PDF |
| `parent_chunk_id`, `child_chunk_ids` | Links for hierarchical retrieval |
| `token_count` | Token count via tokenizer |
| `regulatory_references` | Extracted references (e.g. FATF, R.1) |
| `topic_tags` | Reserved for a later LLM enrichment step |
| `doc_title`, `doc_version`, `doc_authority` | Provenance |
| `ingested_at` | ISO-8601 timestamp (UTC) |
| `chunk_hash` | Short content hash |
| `parser_version` | Parser label (e.g. `pymupdf-1.x`) |

## Setup

### Requirements

- Python 3.10+ recommended (uses modern typing syntax in the notebook).
- A PDF file you are allowed to process (respect copyright and license terms).

### Install dependencies

```bash
python -m venv .venv
.venv\Scripts\activate   # Windows
pip install pymupdf pdfplumber tiktoken tqdm pandas matplotlib
```

Optional:

```bash
pip install docling
```

### Configure paths

Open [`chunking/aml_chunking_pipeline.ipynb`](chunking/aml_chunking_pipeline.ipynb) and set **`PDF_PATH`** to your FATF (or other) PDF. Defaults assume the PDF sits in **`chunking/`** next to the notebook; you can use an absolute path instead. **`OUTPUT_JSONL`** and diagnostic outputs are **relative to the notebook directory** (e.g. `fatf_aml_nra_chunks.jsonl`, `chunk_diagnostics.png`, `chunk_report.html`).

Example:

```python
PDF_PATH = r"C:\path\to\Money-Laundering-National-Risk-Assessment-Guidance-2024.pdf"
OUTPUT_JSONL = "fatf_aml_nra_chunks.jsonl"

DOC_META = {
    "doc_id": "fatf_ml_nra_2024",
    "title": "Money Laundering National Risk Assessment Guidance",
    "version": "November 2024",
    "authority": "FATF",
}
```

### Parallelism

Worker count defaults to roughly **CPU count minus one**. Override:

```text
CHUNKER_WORKERS=8
```

(Windows PowerShell: `$env:CHUNKER_WORKERS=8` before starting Jupyter.)

## Run

1. **Start Jupyter from the repo root** or open the notebook in VS Code / Cursor.
2. Ensure the kernel uses the environment where dependencies are installed.
3. Run [`chunking/aml_chunking_pipeline.ipynb`](chunking/aml_chunking_pipeline.ipynb) from top to bottom.

If your tool runs notebooks with the **working directory** set to the repo root, either `cd chunking` first or prefix output paths with `chunking/` so JSONL and reports land in [`chunking/`](chunking/).

### Outputs from the sanity-check section

- Pandas summaries and **sanity checks** (size bounds, orphaned headings, fractured tables).
- **`chunk_diagnostics.png`** — distribution plots.
- **`chunk_report.html`** — review of the first chunks plus check results.

## Roadmap (from the notebook)

1. Review `chunk_report.html` and spot-check random chunks.
2. Optional **metadata enrichment** with a local LLM (tags, entities, obligations).
3. **Embed and index** (e.g. OpenAI-class embeddings + Pinecone, pgvector, FAISS).
4. **Embedding QA** (e.g. UMAP in an observability tool).

See **[`chunking.md`](chunking.md)** for production hardening (Docling, tracing, data tests, incremental reprocessing).

## Legal and attribution

- **FATF publications** are subject to FATF copyright and terms of use. This repository’s tooling does not grant rights to redistribute official PDFs; obtain documents from [FATF](https://www.fatf-gafi.org/) and comply with their conditions.
- Example outputs under [`chunking/`](chunking/) are derived from such guidance for demonstration and RAG experiments only.

## License

Add a `LICENSE` file if you plan to open-source this repo; the repository currently contains no root-level license file.
