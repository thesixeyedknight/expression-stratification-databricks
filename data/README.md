# Data

Raw data files are **not committed to this repo**. Both source files are loaded into a Databricks Unity Catalog Volume at pipeline runtime — see [Reproducing access](#reproducing-access) below for how to obtain them.

## Sources

This pipeline draws from **two separate UCSC Xena hubs**. They are easy to conflate since both host TCGA-BRCA data, but they're distinct datasets with different processing histories — worth being explicit about which is which:

| File | Hub | Contents |
|---|---|---|
| `TCGA-BRCA.star_fpkm-uq.tsv` | **GDC hub** | Gene expression matrix, 1,226 samples |
| `BRCA_clinicalMatrix` | **legacy Pan-Cancer hub** | Clinical annotations, including PAM50 subtype calls |

The expression matrix and the clinical matrix are joined on sample barcode after ingest.

## Upstream preprocessing (already applied — do not re-transform)

The GDC hub expression values are already **log2(FPKM-UQ + 1)** transformed as delivered. This pipeline treats that transformation as given and does not re-normalize or re-log the expression values. Any further transformation in this pipeline (standardization, variance filtering) is applied on top of these values as-is.

## Sample composition

1,226 total samples, distinguished by TCGA barcode sample-type suffix:

| Sample type | Suffix | Count |
|---|---|---|
| Primary tumor | `-01` | 1,106 |
| Solid tissue normal | `-11` | 113 |
| Metastatic | `-06` | 7 |

Normal and metastatic samples are **intentionally retained** rather than filtered out — they serve as a built-in clustering validation check (e.g., whether normal-tissue samples separate from tumor samples in the unsupervised clustering, independent of PAM50 subtype).

## Gene filtering

Starting from ~60,000 genes in the raw expression matrix:

1. **Mean expression filter**: genes with mean expression ≥ 1.0 retained → 13,224 genes passed
2. **Variance ranking**: remaining genes ranked by variance across samples
3. **Top-N selection**: top 1,500 highest-variance genes retained for downstream analysis

Ensembl gene ID version suffixes (e.g., `.15` in `ENSG00000091831.15`) are stripped during preprocessing for consistency.

## Storage format

Filtered expression data is stored as a **narrow Delta schema** rather than a wide (1,500-column) table: one row per patient, with all gene values packed into a single `array<double>` column, plus a separate lookup table mapping `array_index → gene_id`. This avoids the Unity Catalog metadata registration issues that a wide-column table produces at this feature count (see top-level [README engineering notes](../README.md#engineering-notes) for details).

## Reproducing access

Both source files are publicly available from UCSC Xena:

- GDC hub: `TCGA-BRCA.star_fpkm-uq.tsv` — [https://xenabrowser.net](https://xenabrowser.net), GDC TCGA Breast Cancer cohort
- Legacy Pan-Cancer hub: `BRCA_clinicalMatrix` — [https://xenabrowser.net](https://xenabrowser.net), legacy TCGA Pan-Cancer cohort

Download both files and upload them to a Unity Catalog Volume (e.g., `/Volumes/<catalog>/<schema>/<volume>/`) before running `notebooks/patient_stratification.py`. The notebook reads from Volume paths configured near the top of the ingest stage — update those paths to match your own catalog/schema/volume names.
