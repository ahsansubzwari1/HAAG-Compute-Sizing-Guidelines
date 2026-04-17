# Part 2 — Sizing Your Workload

Before you request a node, answer one question: **what does my workload actually need?** Over-request and you wait in the queue forever. Under-request and your job crashes at hour seventeen.

This part is a set of recipes. Read the framework below. Jump to the recipe that matches your workload.

---

## The four numbers

Every resource request comes down to four numbers. Plus wall time.

| Number | What it is | When it bites |
|---|---|---|
| **CPU cores** | Parallel non-GPU work: data loading, preprocessing, I/O | Undersized CPU = GPU at 30% utilization |
| **System RAM** | Regular memory that holds datasets, buffers, DataFrames | Too little = OOM kill with a cryptic error |
| **GPU VRAM** | GPU memory for weights, gradients, activations | The binding constraint for most ML work |
| **Disk** | Dataset size + working files + checkpoints | Forgotten until the job fails at checkpoint-save time |
| **Wall time** | How long the job runs | Too little = killed mid-epoch; too much = deprioritized in queue |

---

## Two rules before you start

**Profile small first.** Don't compute resources from first principles. Run on a single batch or 1K samples in an Open OnDemand interactive session and measure what it actually uses.

**Request actual usage + ~20% headroom, not the biggest node you can get.** Requesting 8× H200 for a job that needs one V100 is worse than requesting too little — you wait longer in the queue and block other researchers. Target 70–90% utilization across all four numbers.

---

## Recipes

- [**Recipe 1 — LLM fine-tuning and inference**](./part-2-llm.md)
- [**Recipe 2 — Computer vision training**](./part-2-computer-vision.md)
- [**Recipe 3 — Classical ML and tabular work**](./part-2-classical-ml.md)
- [**Recipe 4 — Audio and signal processing**](./part-2-audio.md)

Every recipe ends with a "does this fit on ICE?" table. If yes, go to [Part 3](../03-pace-ice/). If no, go to [Part 4](../04-aws/).
