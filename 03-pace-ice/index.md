# Part 3 — Choosing a GPU on ICE

You've used [Part 2](../02-sizing/) to figure out how much VRAM, CPU, and RAM your workload needs. Now you need to pick an actual GPU to request. ICE has eleven different GPU types ranging from 16 GB V100s to 142 GB H200 SXMs, and the choice between them affects both how long you wait in the queue and how fast your job finishes once it starts.

This section tells you how to pick. Everything else about running on ICE — logging in, partitions, storage, job submission — is already well-documented:

- [Getting Started on ICE](https://gatech.service-now.com/home?id=kb_article_view&sysparm_article=KB0042102)
- [ICE Resources](https://gatech.service-now.com/home?id=kb_article_view&sysparm_article=KB0042095)
- [Open OnDemand Guide](https://gatech.service-now.com/home?id=kb_article_view&sysparm_article=KB0042133)
- [Guru Deshpande's Intro-To-PACE-ICE](https://github.com/guru-desh/Intro-To-PACE-ICE) — practical walkthrough


---

## The GPU landscape on ICE

Eleven GPU types, roughly in order of generation:

| GPU | VRAM | Notes |
|---|---|---|
| Quadro RTX 6000 | 24 GB | Older, fewer nodes — 2 nodes × 4 GPUs |
| V100 PCIe (16 GB) | 16 GB | 11 nodes × 2 GPUs. Plentiful, short queues. |
| V100 PCIe (32 GB) | 32 GB | 1 node × 1 GPU, 3 nodes × 4 GPUs |
| A40 | 48 GB | 2 nodes × 2 GPUs |
| MI210 | 64 GB | 2 nodes × 2 GPUs. **AMD ROCm** — check framework compatibility |
| A100 PCIe (40 GB) | 40 GB | 2 nodes × 2 GPUs |
| A100 PCIe (80 GB) | 80 GB | 2 nodes × 2 GPUs |
| L40S | 48 GB | 4 nodes × 8 GPUs — good for multi-GPU data parallel |
| H100 SXM5 | 80 GB | 20 nodes × 8 GPUs. **14 of 20 reserved for CoE/AI Makerspace.** |
| H200 SXM5 | 142 GB | 18 nodes × 8 GPUs. **12 of 18 reserved for CoE/AI Makerspace.** |
| RTX 6000 Pro Blackwell | 48 GB | 3 nodes × 16 GPUs — newest, 2026 addition |

---

## How to choose

Three questions in order. Stop at the first one that narrows you down.

**Question 1: Does your job fit on a 16 GB V100?** If yes, ask for one. Plentiful hardware, shortest queue wait. Most vision training, LoRA fine-tuning of small LLMs, inference for any model under 8B parameters, all classical ML with GPU acceleration — these all fit here.

**Question 2: If not 16 GB, what's the smallest tier that fits?** Work up the table. A 7B LoRA fine-tune is 20 GB → V100 32 GB. A 13B LoRA is 36 GB → A100 40 GB. A 13B full fine-tune or 70B QLoRA is ~80 GB → A100 80 GB. A 70B LoRA or anything requiring 100+ GB → H200 SXM.

**Question 3: Do you need multi-GPU?** If one GPU isn't enough and you can parallelize across multiple GPUs on the same node, the 8-GPU nodes (L40S, H100, H200) become relevant. Multi-*node* training is possible on ICE but not the primary use case — if you're at that scale, consider escalating to AWS ([Part 4](../04-aws/)) or ACCESS.

---

## Reserved nodes — the H100/H200 gotcha

14 of the 20 H100 nodes and 12 of the 18 H200 nodes are reserved for CoE students and AI Makerspace users. Jobs from general ICE users are auto-routed to the remaining capacity (6 H100 and 6 H200 nodes), which is in high demand.

**Practical implication:** if your job fits on an A100 80 GB, ask for that instead of an H200. You'll start hours earlier. The H200 is only worth the wait when you need 100+ GB VRAM on a single GPU, or when you specifically need the H200's bandwidth for multi-GPU training.

---

## Requesting a specific GPU

ICE's partitions auto-route based on your course, so **you should not specify a partition** in your Slurm script. You request the GPU type directly with `--gres`:

```bash
#SBATCH --gres=gpu:V100:1              # 1 V100 (any size)
#SBATCH --gres=gpu:V100-32GB:1         # 1 V100, 32 GB specifically
#SBATCH --gres=gpu:A100:2              # 2 A100s
#SBATCH --gres=gpu:H200:4              # 4 H200s
```

The exact GPU names accepted by `--gres` can drift as PACE adds hardware. If in doubt, run `sinfo -o "%P %G"` on an ICE login node to see current options.

---

## The calculator

Rather than working through the tables by hand, use the calculator at [`/tools/vram-calculator/`](../tools/vram-calculator/) or [standalone LLM-GPU Sizing Calculator](https://ahsansubzwari1.github.io/HAAG-Compute-Sizing-Guidelines/02-sizing/index.html) to get a specific recommendation for **LLM inference** workloads.

You tell it your model (or pick a preset — Llama, Qwen, Mistral, Mixtral, Nemotron, gpt-oss), precision, number of concurrent users, and context length. It returns:

1. **Total VRAM** (average and worst case), with step-by-step KV-cache math
2. **A fit matrix** across every ICE GPU at 1×, 2×, 4×, 6×, 8×, and 16× GPU counts — color-coded Comfortable / Tight / Insufficient
3. **Headroom visualization** for the GPU you click on

The calculator is inference-focused. For fine-tuning/training VRAM estimation, the sizing tables in [Part 2's LLM recipe](../02-sizing/part-2-llm.md) are the current reference; a training-mode calculator may be added later.

The calculator runs client-side — nothing leaves your browser.

---

## Common gotchas

> **This is a living document.** If you hit a GPU selection or request issue on ICE — and especially if you figure out the fix — please add it here.
