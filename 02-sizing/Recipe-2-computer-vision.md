# Recipe 2 — Computer vision training

The HAAG workhorse. If you're training on images — animal detection, lizard classification, iNaturalist, LiDAR — this is your recipe.

---

## The numbers

### GPU VRAM

Vision models are smaller than LLMs. **The knob that drives VRAM is input resolution × batch size**, not parameter count. Doubling resolution quadruples activation memory.

| Model family | Training VRAM (batch 32) |
|---|---|
| ResNet-50, EfficientNet, YOLO | 6–16 GB |
| ViT-Base, Swin-Base | 12–20 GB |
| ViT-Large, ConvNeXt-Large | 20–32 GB |
| SAM, DINOv2-G, billion-param VLMs | 40–80 GB |

### CPU cores — where vision jobs live or die

Image preprocessing (JPEG decode, resize, augmentation) is CPU-heavy.

- **Single GPU:** 12–16 cores, `num_workers=12`
- **Multi-GPU:** 12 cores *per GPU*
- Heavy augmentation pipelines: add 4 cores per GPU

### System RAM

- Single GPU: 32 GB
- Multi-GPU (≤4): 64 GB
- 8-GPU: 128 GB

Vision datasets stream from disk — they're not held in RAM.

### Disk — the sleeper constraint

This is where ICE storage planning matters most. Reading 500K JPEGs one at a time from shared storage is brutally slow. Two patterns:

1. **Stage to local scratch.** Copy the dataset to the compute node's NVMe at job start. Every GPU node has at least 512 GB scratch; most have 1.9 TB+. Turns 30-min epochs into 5-min epochs.
2. **Pack the dataset.** WebDataset `.tar` shards or HDF5 *before* training. One 50 GB archive reads faster than 500K files.

### Wall time

| Model / Dataset | V100 | H100 |
|---|---|---|
| ResNet-50 on 100K images, 50 epochs | 12–24 hrs | 3–6 hrs |
| ResNet-50 on ImageNet-scale, 90 epochs | ~2 days | ~10 hrs |
| Segmentation on 10K high-res images | 8–16 hrs | 2–4 hrs |

---

## Worked example — ResNet-50 on 389 GB animal detection dataset

| Number | Value | Why |
|---|---|---|
| GPU VRAM | ~14 GB | batch 64, 512×512 |
| CPU cores | 16 | Heavy detection augmentation |
| System RAM | 48 GB | |
| Disk | 389 GB dataset | Fits on NVMe 1.9 TB nodes; stage to scratch |
| Wall time | 72 hours | Submit chained jobs with per-epoch checkpoints |

```bash
#SBATCH --gres=gpu:V100:1
#SBATCH --cpus-per-task=16
#SBATCH --mem=48G
#SBATCH --time=72:00:00
```

---

## Does this fit on ICE?

| Your VRAM need | ICE hardware | Notes |
|---|---|---|
| ≤ 16 GB | V100 16 GB | ResNet, YOLO, ViT-Base |
| ≤ 32 GB | V100 32 GB | Most moderate-resolution training |
| ≤ 48 GB | A100 40 GB, A40, L40S | ViT-Large, high-res segmentation |
| ≤ 80 GB | A100 80 GB | Billion-parameter vision transformers |
| Multi-GPU | 2/4-GPU nodes, 8× L40S | Use `torchrun`, not `DataParallel` |
| 3D volumes, video VLM pretraining, >8 GPUs | **Escalate → [Part 4](../04-aws/)** | |

---

## Common gotchas

> **This is a living document.** If you hit a GPU, VRAM, or resource-request issue while running a computer vision workload on ICE — and especially if you figure out the fix — please add it here.

---

