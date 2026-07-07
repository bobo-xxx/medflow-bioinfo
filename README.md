# medflow-bioinfo

Agent-driven bioinformatics workflow framework. Agents execute multi-node analysis pipelines guided by protocol documents — no compiled code, no hardcoded DAGs.

## How It Works

1. **Protocol** — write a protocol `.md` describing the research question, datasets, analysis steps, and config
2. **Compile** — `/compile-workflow` translates the protocol into a `workflow.json` DAG
3. **Run** — `/run-workflow` executes the DAG, cloning node packages on demand and wiring file bindings between steps

## Project Structure

```
medflow-bioinfo/
├── protocols/          # Human-authored protocol .md files
├── nodes/              # Cloned node packages (git repos, one per node)
├── nodes-archive/      # Compressed node archives (.zip)
├── runs/               # Workflow execution outputs
├── manifest.yaml       # Node metadata — semantic types, subcommands, file discovery
└── registry.yaml       # Machine-readable node URLs + versions (no commit pins)
```

## Available Nodes

| Node | Subcommands | Purpose |
|------|------------|---------|
| `geo-microarray-processing` | `fetch` | Fetch GEO microarray data with 5-tier gene annotation fallback |
| `batch-correction` | `intersect` | Merge datasets via gene intersection with always-on ComBat + PCA QC |
| `univariate-filter` | `run` | Univariate tests (t-test, Wilcoxon, ANOVA, logistic, Cox, correlation) |
| `differential-analysis` | `run` | DEG analysis with auto-detection (DESeq2/limma/edgeR) |
| `diagnostic-model` | `train`, `valid`, `evaluate`, `visualize` | Multivariate logistic regression with LOOCV, ROC, calibration, DCA |
| `ml-feature-selection` | `sequential`, `parallel` | ML feature selection — RF, LASSO, consensus, convergence gate |
| `go-kegg-enrichment` | `enrich`, `merge`, `plot-bar`, `plot-bubble`, `plot-net`, `plot-emap` | GO + KEGG enrichment with clusterProfiler |

## Current Pipeline

### 7-Node Breast Cancer Diagnostic Pipeline

```
fetch1 (GSE25066) ──┐
                    ├── merge ── deg ──┬── enrich (GO+KEGG)
fetch2 (GSE20194) ──┘                  └── univariate-filter ── feature-selection ── diagnostic-model
```

**Research question:** ER+ vs ER- breast cancer — DEGs, pathway enrichment, ML feature selection, and multivariate diagnostic model.

- **Protocol:** `protocols/7-node-breast-cancer.md`
- **Groups:** `er_status` (P = positive, N = negative)
- **Data:** ~800 samples across GSE25066 + GSE20194 (GPL96 Affymetrix HG-U133A)

## Registry

- `registry.yaml` — machine-readable node URLs + versions (no commit pins; clones latest main at runtime)
- `manifest.yaml` — agent-oriented metadata: semantic types, subcommands, file discovery patterns, notes

## Language

English — all artifacts, commit messages, and agent communication.
