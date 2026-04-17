# LLM-GPU Sizing Calculator (ICE Port)

A single-file, browser-based tool for estimating GPU memory requirements when deploying large language models (LLMs) for inference. Input your model architecture, precision format, concurrency target, and context window — get a step-by-step memory breakdown and a recommendation matrix across PACE ICE's GPU inventory.

This is an ICE-specific port of the [standalone LLM-GPU Sizing Calculator](https://ahsansubzwari1.github.io/HAAG-Compute-Sizing-Guidelines/tools/vram-calculator/index.html). The math, layout, and tooltips are identical; the GPU library has been swapped for the hardware actually available on Georgia Tech's PACE ICE cluster.

**Part of the [HAAG Compute Guide](https://github.com/ahsansubzwari1/HAAG-Compute-Sizing-Guidelines/blob/main/README.md).**

---

## What it does

The calculator answers a specific question: given an LLM and a target deployment configuration (precision, concurrent users, context length), how much VRAM is required, and which ICE GPU configurations can support it?

It computes three components of total VRAM demand:

| Component | What it represents |
|---|---|
| **Model weights** | Parameters × bytes per precision format (BF16, FP8, INT4) |
| **KV cache** | Per-token memory × context length × concurrent users × utilization |
| **Serving overhead** | Framework buffers (vLLM, TensorRT-LLM), CUDA kernels, activation workspace |

It then maps the total against the ICE GPU library at 1×, 2×, 4×, 6×, 8×, and 16× GPU counts, with color-coded fit ratings (Comfortable / Tight / Insufficient) and a live headroom indicator when you select a specific GPU.

The 16× column is meaningful on ICE because the RTX 6000 Pro Blackwell nodes provide 16 GPUs per node — one of the few on-campus configurations where that column is actually achievable without multi-node setup.

---

## How the KV cache math works

The KV cache stores Key and Value vectors for every token in the active context window. The formula is:

```
Bytes per token = 2 × num_layers × num_KV_heads × head_dim × bytes_per_element
```

For a model like Llama 3.1 8B with GQA (8 KV heads), at FP16 KV cache:

```
Per token = 2 × 32 layers × 8 KV heads × 128 head_dim × 2 bytes (FP16)
          = 131,072 bytes ≈ 0.131 MB per token

Per session (8K context) = 0.131 MB × 8,192 tokens ≈ 1.07 GB (maximum)
At 50% average utilization = 0.54 GB/session × 3 users ≈ 1.6 GB KV cache
```

The calculator shows this breakdown step by step so users can audit the logic.

**Key concepts:**

- **GQA (Grouped Query Attention):** Most modern models share Key/Value heads across groups of Query heads, dramatically reducing KV cache size. The KV heads field captures this — not the full query head count.
- **Average vs. worst-case:** Sessions build context incrementally. The utilization slider accounts for the realistic average across concurrent users; the worst-case figure assumes every user simultaneously fills their entire context window.
- **MoE models:** For Mixture-of-Experts architectures (e.g., Mixtral, Nemotron 120B, gpt-oss 120B), only a fraction of parameters are active per forward pass, but *all* parameters must reside in VRAM. The full parameter count is used for weight memory estimation.

---

## ICE GPU library

The tool includes the GPUs available on PACE ICE, per [KB0042095](https://gatech.service-now.com/home?id=kb_article_view&sysparm_article=KB0042095):

| GPU | VRAM | Memory BW | MIG | NVLink | Note |
|---|---|---|---|---|---|
| V100 PCIe 16GB | 16 GB | 0.9 TB/s | No | No | Plentiful — shortest queue waits |
| Quadro RTX 6000 | 24 GB | 0.67 TB/s | No | No | Older hardware, quick queue |
| V100 PCIe 32GB | 32 GB | 0.9 TB/s | No | No | Good queue availability |
| A100 PCIe 40GB | 40 GB | 1.55 TB/s | Yes | No | Popular — sweet spot for 7B/13B |
| A40 48GB | 48 GB | 0.7 TB/s | No | No | |
| L40S 48GB | 48 GB | 0.86 TB/s | No | No | 8-GPU nodes |
| RTX 6000 Pro Blackwell 48GB | 48 GB | 1.6 TB/s | Yes | Limited | Newest (2026), 16 GPUs per node |
| MI210 64GB | 64 GB | 1.6 TB/s | No | No | AMD ROCm — verify framework compatibility |
| A100 PCIe 80GB | 80 GB | 1.94 TB/s | Yes | No | 70B QLoRA fits on one GPU |
| H100 SXM5 80GB | 80 GB | 3.35 TB/s | Yes | Yes | 14 of 20 nodes reserved for CoE/AI Makerspace |
| H200 SXM5 142GB | 142 GB | 4.8 TB/s | Yes | Yes | 12 of 18 nodes reserved for CoE/AI Makerspace |

Reservation notes reflect the operational reality: even though ICE has 20 H100 nodes and 18 H200 nodes, general-ICE users are routed to the unreserved capacity (6 of each), which is in high demand. If your job fits on an A100 80GB, ask for that instead — queue wait is often hours shorter.

---

## Model preset library

| Model | Params | Class | Notes |
|---|---|---|---|
| Nemotron-3-Super-120B-A12B | 120B | MoE | 8 KV heads (GQA) |
| gpt-oss-120b | 120B | MoE | 8 KV heads (GQA) |
| Llama 3.1 405B | 405B | Dense | 8 KV heads (GQA) |
| Llama 3.3 70B Instruct | 70B | Dense | 8 KV heads (GQA) |
| Qwen2.5-Coder-72B Instruct | 72B | Dense | 8 KV heads (GQA) |
| Mixtral 8×22B | 141B | MoE | 8 KV heads (GQA) |
| Qwen2.5-32B Instruct | 32B | Dense | 8 KV heads (GQA) |
| Llama 3.2 90B Vision | 90B | Dense | 8 KV heads (GQA) |
| Devstral-Small-2-24B | 24B | Dense | 8 KV heads (GQA) |
| Mistral-Small-22B | 22B | Dense | 8 KV heads (GQA) |
| Mixtral 8×7B | 46.7B | MoE | 8 KV heads (GQA) |
| Llama 3.1 8B Instruct | 8B | Dense | 8 KV heads (GQA) |
| Qwen2.5-7B Instruct | 7B | Dense | 4 KV heads (GQA) |
| Mistral 7B v0.3 | 7B | Dense | 8 KV heads (GQA) |

The larger models (e.g., 405B) will show "Insufficient" across most single-node ICE configurations. That's informative — the escalation path in [Part 4](../../04-aws/) covers what to do when ICE can't fit your workload.

For models not in the preset list, select **custom / manual entry** and fill in the architecture fields directly.

---

## Adding new GPUs or model presets

All GPU and model data is defined in the `<script>` block near the bottom of `index.html`.

**To add a GPU**, add an entry to the `GPUS` array:

```javascript
{ name:'H100 NVL', vram:94, bw:'3.35 TB/s', mig:'Yes', nvlink:'Yes', nodes:0, gpusPerNode:0, note:'' },
```

**To add a model preset**, add an entry to the `PRESETS` object and a matching `<option>` in the select element:

```javascript
// In PRESETS object:
mymodel: { layers:80, kvheads:8, headdim:128, params:70, label:'My Model 70B' },

// In the <select id="preset"> element:
<option value="mymodel">My Model 70B</option>
```

Fields for model presets:

| Field | Description |
|---|---|
| `layers` | Number of transformer layers |
| `kvheads` | Number of KV heads after GQA |
| `headdim` | Attention head dimension (usually 64 or 128) |
| `params` | Total parameter count in billions |

---

## Scope and limitations

- **Inference only.** This calculator estimates VRAM for *serving* an LLM. For *training* VRAM (fine-tuning, LoRA, QLoRA, full fine-tune), see the sizing tables in the [HAAG Compute Guide Part 2 — LLM recipe](../../02-sizing/part-2-llm.md).
- **Estimates only.** Results are engineering approximations based on published architecture specs and standard KV cache arithmetic. Actual VRAM usage varies by serving framework, batch size, sampling parameters, and quantization implementation.
- **Validate before committing.** Always profile with your actual serving stack (vLLM, TensorRT-LLM, TGI) with a short test run before reserving long GPU wall time.
- **Multi-GPU assumes tensor parallelism.** The 2×/4×/6×/8×/16× columns assume total VRAM is pooled via NVLink or equivalent. For GPUs without NVLink (V100 PCIe, A40, L40S, RTX 6000 series on ICE), multi-GPU serving requires pipeline parallelism, which has different efficiency characteristics.
- **Activation memory not included.** Forward pass activation buffers are small relative to weights and KV cache for typical batch sizes, but may be material for very large batches.

---

## Relationship to the upstream calculator

The upstream [LLM-GPU Sizing Calculator](https://ahsansubzwari1.github.io/llm-gpu-sizing-calculator) on GitHub Pages is a generic tool with a data-center GPU library (H200, H100, A100, L40, RTX 4090, etc.). This ICE port retains the same math and UI; only the GPU library differs. If you want to size for hardware not available on ICE (AWS EC2 instances, on-prem rigs), use the upstream calculator.

---

## License

MIT — free to use, modify, and redistribute.
