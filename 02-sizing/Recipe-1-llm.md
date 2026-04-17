# Recipe 1 — LLM fine-tuning and inference

You have a transformer-based model (Llama, Mistral, Qwen, Gemma, DeepSeek) between 1B and 70B parameters, and you want to fine-tune it or run inference. You're using Hugging Face `transformers`, usually with `peft` for LoRA/QLoRA.

---

## The numbers

### GPU VRAM — the binding constraint

Three regimes cover almost everything:

| Regime | Rule of thumb | 7B | 13B | 70B |
|---|---|---|---|---|
| **Inference (FP16)** | ~2 GB per 1B params | 14 GB | 26 GB | 140 GB |
| **LoRA fine-tuning** | ~2–3× inference | 20 GB | 36 GB | 160 GB |
| **QLoRA fine-tuning** | ~¼ model + overhead | 10–12 GB | 18–24 GB | ~46 GB |

Long context (8K+), large batches, and full fine-tuning all push these up significantly. Start conservative and measure.


### CPU cores

- **Single GPU:** 8–12 cores, `dataloader_num_workers=8`
- **Multi-GPU:** 8–12 cores *per GPU*
- **Inference only:** 4–6 cores

Undersized CPU starves the GPU. This is the #1 reason GPU utilization sits at 30%.

### System RAM

- **Dataset < 5 GB on disk:** 32 GB
- **5–50 GB:** 64 GB
- **> 50 GB:** 128 GB, and stream (`datasets.load_dataset(..., streaming=True)`)

Add 16 GB if you're using QLoRA with `bitsandbytes` — the 4-bit loading routines are RAM-hungry.

### Disk

Three things take space, and people forget at least one:

1. **Base model cache** — 7B ≈ 14 GB, 70B ≈ 140 GB
2. **Tokenized dataset** — roughly 2× the raw text size
3. **Checkpoints** — LoRA adapters are tiny (100–400 MB); full fine-tune checkpoints are model-sized

Pattern that works on ICE: base model and dataset in **project storage**, checkpoints written to **scratch**, final adapter copied back to project.

### Wall time

On a single A100-80GB or H100:

| Model | Method | 100K samples |
|---|---|---|
| 7B | QLoRA | 8–16 hours |
| 7B | LoRA | 12–24 hours |
| 13B | QLoRA | 16–32 hours |
| 13B | LoRA | 24–48 hours |
| 70B | QLoRA | 2–4 days |

~2× slower on V100, ~2× faster on H200 with Unsloth.

---

## Worked example — QLoRA fine-tune Llama 3 8B on 50K instruction pairs

| Number | Value | Why |
|---|---|---|
| GPU VRAM | ~10 GB | 4-bit base (4 GB) + adapters/gradients/activations (6 GB) |
| CPU cores | 8 | Single GPU, moderate augmentation |
| System RAM | 48 GB | Small dataset + QLoRA overhead |
| Disk | 20 GB | Base model cache + tokenized data + adapters |
| Wall time | 12 hours | ~6–10 hours expected, 20% headroom |

```bash
#SBATCH --gres=gpu:V100:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=48G
#SBATCH --time=12:00:00
```

---

## Does this fit on ICE?

| Your VRAM need | ICE hardware | Notes |
|---|---|---|
| ≤ 16 GB | V100 16 GB | Plentiful — short queue waits |
| ≤ 32 GB | V100 32 GB | Good for 7B LoRA or 13B QLoRA |
| ≤ 48 GB | A40, L40S, RTX 6000 Pro Blackwell | 13B LoRA |
| ≤ 80 GB | A100 80 GB | 70B QLoRA single-GPU |
| ≤ 142 GB | H200 142 GB SXM | Best on campus; 70B LoRA fits. Tight queue — 12 of 18 nodes reserved for CoE/AI Makerspace. |
| > 142 GB or > 8 GPUs | **Escalate → [Part 4](../04-aws/)** | |

---

## Common gotchas

> **This is a living document.** If you hit a GPU, VRAM, or resource-request issue while running an LLM workload on ICE — and especially if you figure out the fix — please add it here.

---



