# PSSDB — From Papers to a Standardized Dataset (Ingestion Pipeline)

The diagram and summary below describe how the ingestion component converts **PDF articles and supplements** into a **strict, merge‑ready CSV** and then **loads it into the database**.

## Block Diagram (S‑shaped, with LLM vs Code markers)
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

## Pipeline at a Glance (neutral, step‑by‑step)

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
