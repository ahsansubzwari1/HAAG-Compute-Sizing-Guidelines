# Recipe 3 — Classical ML and tabular work

The overlooked recipe. Not every problem needs a GPU, and CPU-only jobs start in minutes while GPU queues back up.

You're running gradient-boosted trees (XGBoost, LightGBM), random forests, SVMs, regressions, or any `scikit-learn` pipeline on tabular data.

---

## The numbers

### GPU VRAM — skip it

No GPU for >90% of these workloads. Unless your dataset is >50M rows, CPU versions of XGBoost/LightGBM match or beat GPU versions on ICE hardware.

### CPU cores — the real currency

Tree-based models scale nearly linearly to ~32 cores with `n_jobs=-1`.

| Problem size | Cores |
|---|---|
| < 1M rows | 8 |
| 1M–10M rows | 16–32 |
| 10M+ rows | 64 (AMD EPYC node) |

### System RAM — the binding constraint

**Pandas DataFrames use 2–5× the on-disk file size.** A 50 GB Parquet file balloons to 150–250 GB in memory.

| Dataset on disk | RAM | Notes |
|---|---|---|
| < 5 GB | 32 GB | |
| 5–50 GB | 128 GB | Consider `polars` or `duckdb` — 10× less memory than pandas |
| > 50 GB | 768 GB+ | Or chunk the data |

### Disk

Tabular data is small. Working directory almost always < 100 GB. Project storage is fine.

### Wall time

GBM on 1M rows, 1000 trees: minutes. 50-config Optuna sweep on 10M rows: a few hours. RandomForest on 10M rows, 32 cores: 30–60 min.

---

## Slurm template

No `--gres=gpu` line. Land on a CPU node with a short queue wait.

```bash
#SBATCH --cpus-per-task=32
#SBATCH --mem=128G
#SBATCH --time=04:00:00
```

---

## Does this fit on ICE?

| Your need | ICE hardware |
|---|---|
| ≤ 192 GB RAM, ≤ 24 cores | Dual Xeon Gold 6226 nodes (50+) — default, fast queue |
| ≤ 384 GB RAM | 12 nodes available |
| 64 cores, 512 GB RAM | AMD EPYC 7513/7452 (6 non-GPU nodes) |
| ≤ 768 GB RAM | 7 nodes |
| 192 cores, 1.5 TB RAM | 1 Xeon 6972P node — heavy hitter |
| > 1.5 TB RAM | **Escalate → [Part 4](../04-aws/)** (AWS `x2iedn` has up to 4 TB) |

---

## Common gotchas

> **This is a living document.** If you hit a resource issue while running a classical ML or tabular workload on ICE — and especially if you figure out the fix — please add it here.

---

