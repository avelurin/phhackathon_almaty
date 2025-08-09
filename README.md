# PSSDB — From Papers to a Standardized Dataset (Ingestion Pipeline)

The diagram and summary below describe how the ingestion component converts **PDF articles and supplements** into a **strict, merge‑ready CSV** and then **loads it into the database**.

## Block Diagram 
```mermaid
flowchart TB
  %% Styles
  classDef llm fill:#f3e8ff,stroke:#7e22ce,stroke-width:1.2px,rx:8,ry:8,color:#2e1065;
  classDef code fill:#eef,stroke:#6b7bb6,stroke-width:1.2px,rx:8,ry:8,color:#0f1a3b;
  classDef out fill:#fff7e6,stroke:#c08a2d,rx:6,ry:6,color:#5c3b00;
  linkStyle default stroke:#7f8fbf,stroke-width:1.2px

  %% Top row (left → right)
  subgraph TOP[ ]
    direction LR
    A["Profile (LLM)<br/>Extract species, assembly, metrics, populations<br/>→ paper_profile.json"]:::llm
    B["Parsing (CODE)<br/>Read tables and text; keep provenance<br/>→ raw_tables/, raw_text/, provenance.jsonl"]:::code
    C["Header Mapping (LLM)<br/>Map source headers to target schema<br/>→ table_mapping/"]:::llm
    D["Row Assembly (LLM)<br/>Produce schema-shaped rows<br/>→ intermediate_rows.jsonl"]:::llm
  end

  %% Bottom row (right → left)
  subgraph BOTTOM[ ]
    direction RL
    E["Normalization (CODE)<br/>Unify units and names; kb→bp; types<br/>→ normalized_rows.parquet"]:::code
    F["Gene Annotation (CODE)<br/>Intersect with GTF/GFF for the assembly<br/>→ annotated_rows.parquet"]:::code
    G["Metric Merge (CODE)<br/>Combine Fst, XP-EHH, iHS per signal<br/>→ merged_rows.parquet"]:::code
    H["QC (CODE)<br/>Validate coordinates, fields, presence rules<br/>→ out/qc_report.json"]:::code
    I["Export (CODE)<br/>Exact column order; schema-checked CSV<br/>→ out/result.csv"]:::out
    J["Database Load (CODE)<br/>Insert or append into PSSDB<br/>→ database or API"]:::code
  end

  %% Flow connections forming an S
  A --> B --> C --> D --> E
  E --> F --> G --> H --> I --> J

  %% Legend
  subgraph LEGEND[Legend]
    direction LR
    L1["LLM-driven"]:::llm
    L2["Deterministic code"]:::code
  end
```

## Pipeline at a Glance 

1) **Profile (context).** The article is analyzed once to capture the dataset contract: species and genome assembly, selection metrics and thresholds, comparison populations, and citation metadata. The result is saved as paper_profile.json and used to guide all subsequent steps.

2) **Parsing (mechanical intake).** All tables and text are extracted as-is from PDFs/DOCX/XLSX/TSV. A provenance trail records the origin of each row (file, sheet, page). No interpretation is performed at this stage.

3) **Header Mapping (bridge to the schema).** Source column names are mapped to the fixed PSSDB schema (for example, Chr → chrom, BP → snp_pos). Missing targets are listed explicitly to keep the process auditable and repeatable.

4) **Row Assembly (schema-shaped rows).** Parsed tables are converted into rows with the exact fields of the target schema. Coordinates are placed into (chrom, start, end, snp_pos, is_snp); populations follow the profile; supplement_id points to the original source. If a metric is used but has no numeric values, the corresponding presence flag is set to “used”.

5) **Normalization (comparability).** Units and names are standardized (for example, kb to bp, canonical metric and assembly names, population aliases), and strict data types are enforced. Rows from different files become directly comparable.

6) **Gene Annotation (biological context).** Using the specified assembly, genomic intervals are intersected with GTF/GFF to fill gene_symbol, gene_id, and gene_overlap_type (such as exon, intron, nearby).

7) **Metric Merge (one signal, many metrics).** Records that describe the same SNP or the same genomic window plus population pair are merged into a single row, bringing Fst, XP‑EHH, iHS, and other metrics together.

8) **QC (trustworthy output).** Consistency checks validate coordinate logic, population fields, presence flags versus numeric values, and mandatory metadata. Only a successful QC produces final artifacts.

9) **Export (merge‑ready CSV).** Data are written in the exact column order required by PSSDB and validated against the table schema. The deliverable is out/result.csv, reproducible and traceable to its sources.

10) **Database Load (ingestion into PSSDB).** The validated CSV is inserted or appended to the central database (or posted to an API endpoint), completing the ingestion cycle and making the data available for browsing and downstream analyses.


оформлен так, как у тебя оформлено представление для меня. 


ChatGPT said:
markdown
Copy
Edit
# Detailed Step Descriptions

## Step 1 — Profile (LLM)

**Purpose**  
Establish a machine-readable “contract” for the paper so downstream stages know the species, assembly, metrics, populations, and citation context.

**Inputs**
- `input/paper.pdf` (or equivalent)
- Optional: `input/overrides.yaml` (pre-known species/assembly)

**Outputs**
- `out/paper_profile.json` (strict JSON)
- `logs/profile.provenance.json` (tool versions, hashes, pages seen)

**Responsibilities (checklist)**
- [ ] Extract plain text from the PDF (layout-aware; OCR if scanned).  
- [ ] Identify **species** and **common name**.  
- [ ] Identify **genome assembly** exactly as used for selection scans.  
- [ ] List **selection metrics** (e.g., Fst, XP-EHH, iHS, nSL, XP-CLR, H12, H2/H1, ω, π, Tajima’s D).  
- [ ] Capture **thresholds/windowing** if stated (e.g., 100 kb windows, |iHS| > 2.58).  
- [ ] Capture **populations** and their **aliases** (e.g., NeC, NeS, NeT).  
- [ ] Record **citation metadata** (DOI, PMID, title, URL).  
- [ ] Store **notes** for any ambiguity (do not guess).  

**LLM contract**
- Output must be **strict JSON** matching the profile schema (no free text).  
- Unknown/unstated fields must remain **empty**; no inference.  
- Ambiguity may be summarized in a `notes` field; facts must not be invented.

**Shape (illustrative)**
```json
{
  "species": "Bos taurus",
  "common_name": "cattle (Nelore breed)",
  "assembly": "UMD3.1.1",
  "doi": "10.1186/s12711-018-0381-2",
  "pmid": "",
  "article_title": "...",
  "article_url": "https://doi.org/...",
  "metrics": [
    {
      "name": "Fst",
      "threshold": "top 1% per 100-kb window",
      "windowing": "100 kb, 75 kb overlap",
      "note": ""
    },
    {
      "name": "XP_EHH",
      "threshold": "mean > 2.58",
      "windowing": "100 kb",
      "note": ""
    }
  ],
  "populations": [
    {
      "name": "Nelore Control",
      "aliases": ["NeC"],
      "note": ""
    },
    {
      "name": "Nelore Selection",
      "aliases": ["NeS"],
      "note": ""
    }
  ],
  "notes": "..."
}

**Validation**
- Validate against `jsonschema` (required keys, types).  
- Cross-checks:
  - Assemblies mentioned in text vs. the one used for scans.  
  - Population aliases also appear in tables/figures.  
  - Threshold strings are parsable when present.  
- On failure: generate a compact error report and re-prompt the same LLM to **fix only the listed issues**.

**Provenance & caching**
- Save PDF hash, OCR flag, extractor version, and page ranges in `logs/profile.provenance.json`.  
- If hashes and tool versions match, the step is skipped (cache hit).

**Failure modes to watch**
- Multiple assemblies mentioned → keep the one **explicitly** used for scans; note alternatives in `notes`.  
- Metrics cited in methods but not used in results → note this; downstream may set `*_presence="used"` where appropriate.

---

## Step 2 — Parsing (CODE)

**Purpose**  
Create an **exact, lossless snapshot** of all source tables and text, with complete lineage and without any interpretation or unit conversion.

**Inputs**
- All files in `input/` (PDF, DOCX, XLSX/CSV/TSV, supplementary PDFs)

**Outputs**
- `raw_tables/<table_id>.parquet` (one per file/sheet/table)
- `raw_text/article.txt` (PDF text dump; OCR-backed if needed)
- `provenance.jsonl` (one line per extracted table with full lineage)

**Extraction rules**
- **PDF**
  - Text: pdfplumber / PyMuPDF; if scanned → OCR (pytesseract) first.  
  - Tables: camelot / tabula; choose lattice/stream per page; one table → one artifact.  
- **DOCX**: python-docx tables; capture captions such as “Table S3” if present.  
- **XLSX/CSV/TSV**: openpyxl / pandas; one artifact per sheet/file.

**What is preserved**
- Original **headers** unchanged (no renaming).  
- Original **cell strings** (no kb→bp, no Chr parsing, no numeric coercion).  
- Thousands/decimal separators as in source; encodings normalized to UTF-8.  
- Multi-row headers flattened conservatively (e.g., join with a space) while the **raw header text** is kept in metadata.

**What is recorded for each table**
- `source_file`, `sheet_or_table`, detected `supplement_id` (e.g., “Table S4”).  
- PDF page range, extraction mode (lattice/stream/docx/xlsx), and tool versions.  
- `content_sha1` over the normalized tabular content (deduplication).  
- Optional `extraction_note` (e.g., “low-contrast grid; OCR applied”).

**Produced files (example)**
    raw_tables/
      S3_lattice_p12-13.parquet
      S4_stream_p18.parquet
      supp_table1_sheetA.parquet
    raw_text/
      article.txt
    provenance.jsonl

**Deduplication & skips**
- If two artifacts share the same `content_sha1`, keep one and add the other origins to its provenance.  
- Empty or header-only tables are skipped with a logged reason.

**Provenance line (illustrative)**
    {
      "table_id": "S4_stream_p18",
      "source_file": "MOESM1.docx",
      "sheet_or_table": "Table S4",
      "supplement_id": "Table S4",
      "pages": "18",
      "parser": "tabula:stream",
      "ocr": false,
      "content_sha1": "sha1:...",
      "extraction_note": "ok"
    }

**Idempotency & cache**
- If inputs, tool versions, and configs are unchanged, the step is a no-op.  
- Any change invalidates only the affected artifacts.

**Failure modes to watch**
- Non-UTF-8 encodings → normalize to UTF-8, log the original.  
- Merged cells / multi-line text → store visible text; keep raw in metadata if truncated.  
- Inconsistent sheet naming → still produce a stable `table_id` and record the original label in provenance.

**Intentional non-goals of this step**
- No header mapping to the target schema.  
- No coordinate parsing or unit conversion.  
- No metric presence flags.  
(Handled in later steps: Header Mapping, Row Assembly, Normalization.)
