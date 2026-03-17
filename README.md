# pandas + NumPy Practice

Daily practice notebooks
## Structure

Each notebook (`pandas_practice_N.ipynb`) follows the same four-level format:

- **Level 1 — Basics**: DateTime, MultiIndex, indexing
- **Level 2 — Transformations**: reshaping, string ops, custom logic
- **Level 3 — Aggregation**: merging, groupby, pivot tables
- **Level 4 — Real-world**: end-to-end pipelines combining everything

Notebooks build on each other — familiar functions get leaner instructions over time, and 1–2 new functions are introduced per notebook.

## Topics covered

`groupby` · `merge` · `concat` · `melt` · `stack` / `unstack` · `pivot_table` · `MultiIndex` · `pd.cut` · `transform` · `apply` · `rank` · `.str` accessor · DateTime (`.dt`) · `np.where` · `np.dot` · `np.corrcoef` · `np.argsort` · rolling windows · and more

## 13-Week Syllabus

Notebooks 1–11 covered foundational basics. The structured syllabus begins at notebook 12.

Start date: ~2026-03-10 · Daily practice (7 days/week) · 91 sessions total

| Week | Notebooks | New Concept |
|------|-----------|-------------|
| 1 | 12 | MultiIndex — create, index, filter |
| 2 | 13–19 | `stack()` / `unstack()` / `.xs()` |
| 3 | 20–26 | `.pipe()` for method chaining |
| 4 | 27–33 | `category` dtype + memory usage |
| 5 | 34–40 | Vectorization vs `.apply()` — performance |
| 6 | 41–47 | Regex: `.str.extract()`, `.str.contains()` |
| 7 | 48–54 | Advanced time series — `DateOffset`, freq-aware `shift()`, timezone |
| 8 | 55–61 | `pd.crosstab()`, `wide_to_long()` |
| 9 | 62–68 | Named aggregations — `pd.NamedAgg`, dict agg |
| 10 | 69–75 | Advanced merges — `merge_asof()`, `merge_ordered()` |
| 11 | 76–82 | Window functions — `expanding()`, `ewm()` |
| 12 | 83–89 | Full messy data cleaning pipeline |
| 13 | 90–96 | Capstone — end-to-end real-world project |

## Setup

```bash
uv sync
```

Then open any notebook in VS Code or JupyterLab.
