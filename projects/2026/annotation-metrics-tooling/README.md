# Annotation Metrics Tooling for the Ensembl Assembly/Annotation Tracking App

**Contributor:** Shrinidhi Jamuna Natarajan  
**Mentors:** Anna Lazar, Leanne Haggerty, Simarpreet Bhurji   
**Organisation:** EMBL-EBI  
**Programme:** Google Summer of Code 2026  

---

## Project summary

The Ensembl genebuild pipeline produces rich annotation quality metrics for thousands of genomes, all stored in the `gsoc_registry` MySQL database. This project builds two Python modules on top of that database to turn raw metrics into something annotators can actually act on.

---

## What was built

### Module 1: Per-genome annotation quality reports

Takes a GCA accession and generates a quality report in HTML, CSV, TXT and PNG format. The report includes:

- Protein and assembly BUSCO scores with composition breakdown
- 9 AGAT-derived stats from the `new_metrics` table (coding genes, total transcripts, transcripts per gene, average CDS length, average coding intron length, single exon coding genes, longest coding gene, average coding exon length, non-coding genes)
- Protein vs assembly BUSCO differential flagging (Excellent / Acceptable / Investigate / Problematic)
- Missing metrics classification by genebuild lifecycle stage
- Clade Outlier Analysis section (populated by Module 2)
- Side-by-side comparison view for multiple annotations per GCA

### Module 2: Clade comparative analysis and outlier detection

Answers the question: is this annotation quality unusual for its clade? It:

- Loads all live genomes from the registry and assigns clades via `taxonomy_service` and `clade_settings.json`
- Runs PCA across 11 annotation quality metrics per clade
- Uses MAD-based outlier detection (threshold 8.0) to flag genomes that stand out
- Feeds outlier results back into Module 1 HTML reports
- Generates a clade summary dashboard: a single HTML file with 18 switchable clade tabs, BUSCO distribution charts, and sortable genome tables

---

## Stats

- 3,736 live genomes across 18 clades
- 203 unit tests, all passing
- pylint 10/10, mypy clean, black formatted throughout
- Python 3.9 compatible (required for Codon HPC at EBI)

---

## Code

All code lives in the Ensembl genebuild metadata repository:

- **Branch:** [shrinidhi/gsoc2026-annotation-metrics](https://github.com/Shrinidhi-JN/ensembl-genes-metadata/tree/shrinidhi/gsoc2026-annotation-metrics)
- **Pull request:** [#36 — GSoC 2026: Annotation metrics reporting and clade analysis](https://github.com/Ensembl/ensembl-genes-metadata/pull/36)
- **Module 1:** `metadata_app/backend/app/services/gsoc/module1/`
- **Module 2:** `metadata_app/backend/app/services/gsoc/module2/`

---

## What's left / future work

- FastAPI endpoints to expose reports directly from the metadata web app
- Batch SLURM report generator for entire clades or releases
- Further MAD threshold tuning as more data is added to the registry
- Release-aware diff module to track annotation quality changes across Ensembl releases

---

## Challenges and learnings

- `species.clade` is NULL for all rows in the registry — clade assignment had to go through `taxonomy_service` and `clade_settings.json` rather than the DB column
- pandas had to be pinned to 2.3.3 since pandas 3.0+ requires Python 3.11+ which Codon runs at 3.9
- MAD outlier detection required threshold tuning — the default 3.5 produced 40-50% false positive rates in tight clades like mammalia
- Working inside a large production codebase and adding new modules cleanly without breaking existing functionality
