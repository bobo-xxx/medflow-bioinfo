# Protocol: 6-Node Breast Cancer DEG Pipeline

**Planner Agent:** manual-test | **Confidence:** high | **Decision:** proceed

---

## research_question

Compare estrogen receptor positive (ER+) vs estrogen receptor negative (ER-) breast cancer
tumors to identify differentially expressed genes. ER status is the most established molecular
classifier in breast cancer — ER+ and ER- tumors have distinct transcriptional programs,
prognosis, and treatment responses. Grouping by `er_status` column (P = positive, N = negative).

**Group column:** `er_status`
**Group labels:** `P` (ER-positive) vs `N` (ER-negative)
**Datasets:** GSE25066 + GSE20194 (breast cancer, GPL96 platform)

**Per-dataset group columns:**
| Dataset | Group column | Values | Notes |
|---------|-------------|--------|-------|
| GSE25066 | `er_status_ihc` | P, N | ER status by immunohistochemistry |
| GSE20194 | `er_status` | P, N | ER status |

Both columns represent the same biology. Each fetch extracts its dataset's column into a standardized `sample_group.csv` for downstream.

---

## config

```yaml
fetch1:
  subcommand: fetch
  gse_id: GSE25066
  rename: er_status_ihc:group
  proxy: http://127.0.0.1:2999

fetch2:
  subcommand: fetch
  gse_id: GSE20194
  rename: er_status:group
  proxy: http://127.0.0.1:2999

merge:
  subcommand: intersect

deg:
  subcommand: run
  method: limma
  logfc_cutoff: 0.5

enrich:
  subcommand: enrich
  tax-id: "9606"

univariate-filter:
  subcommand: run
  outcome: group
  outcome_type: binary
  method: logistic
  p_adjust: BH
  sig_basis: padj
  sig_threshold: 0.1

feature-selection:
  subcommand: sequential
  control-label: N
```

---

## 5. analysis_pipeline

### Workflow Diagram

```mermaid
flowchart TD
    A[fetch1] --> C[merge]
    B[fetch2] --> C[merge]
    C --> D[deg]
    D --> E[univariate-filter]
    E --> F[feature-selection]
    D --> G[enrich]
```

### Detailed Steps

| Step | Method | Tool/R Package | Input | Output | Quality Gate | Fallback |
|------|--------|----------------|-------|--------|--------------|----------|
| fetch1 | GEO data retrieval | geo-microarray-processing | — | probe + gene expression + metadata | — | — |
| fetch2 | GEO data retrieval | geo-microarray-processing | — | probe + gene expression + metadata | — | — |
| merge | Gene intersection + ComBat | batch-correction | gene expression matrices | shared expression + metadata + PCA plots | — | — |
| deg | Differential expression (ER+ vs ER-) | differential-analysis | shared expression, sample group map | DEGs (all genes), volcano, heatmap | — | — |
| univariate-filter | Univariate logistic regression on DEGs | univariate-filter | DEG gene list, shared expression, sample group map | significant genes (padj ≤ 0.1) | outcome must contain exactly P/N | — |
| feature-selection | RF → LASSO sequential | ml-feature-selection | univariate significant genes, expression, sample group map | selected genes (LASSO nonzero), importance table, CV plots | ≥ 2 genes selected | — |
| enrich | GO/KEGG enrichment of DEGs | go-kegg-enrichment | DEG gene list | enrichment tables, bar/bubble plots | — | — |

---

## 9. quality_gates_and_veto_rules

(no gates — first multi-node test)

---
