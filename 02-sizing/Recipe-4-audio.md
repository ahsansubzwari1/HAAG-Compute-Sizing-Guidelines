# Recipe 4 — Audio and signal processing

You have audio (WAV, FLAC, MP3) and you're training a classifier, fine-tuning a speech model (Whisper, wav2vec2, HuBERT), or extracting features across a corpus.

Defining feature: **preprocessing is expensive, models are small.** Inverts the usual GPU-heavy recipe.

---

## The numbers

### GPU VRAM

| Model | Training VRAM |
|---|---|
| Small spectrogram CNN | 4–8 GB |
| wav2vec2-base, HuBERT-base | 12–20 GB |
| wav2vec2-large | 24–40 GB |
| Whisper-small / medium | 12–24 GB |
| Whisper-large-v3 | 40–80 GB (LoRA: ~20 GB) |

### CPU cores — the binding constraint

Loading MP3s, resampling, computing spectrograms, and augmentation (SpecAugment, noise injection) all happen on CPU.

- **Feature extraction (no GPU):** 32–64 cores
- **Training:** 16 cores per GPU
- **Whisper transcription at scale:** 12 cores per GPU (CPU-bound on decoding)

### System RAM

- Streaming: 48 GB
- Caching spectrograms in memory: 128 GB+

### Disk — the two-stage pattern

Raw audio is big (HAAG's bird_audio corpus is 508 GB). Mel-spectrograms of the same data are ~50–100 GB compressed.

**Don't compute spectrograms on every epoch.** Run a one-time feature extraction job (CPU-only, 64 cores) that converts audio to WebDataset or HDF5. Then train on the preprocessed features. A 6-hour preprocessing job saves 40+ hours of GPU time over a multi-epoch run.

### Wall time

- Feature extraction of 500 GB audio on 64 cores: 4–8 hours
- wav2vec2-base fine-tune for classification: 6–24 hours
- Whisper-large transcription on V100: ~10× real-time

---

## Slurm templates

**Stage 1 — feature extraction (one-time):**

```bash
#SBATCH --cpus-per-task=64
#SBATCH --mem=128G
#SBATCH --time=12:00:00
```

**Stage 2 — training:**

```bash
#SBATCH --gres=gpu:V100:1
#SBATCH --cpus-per-task=12
#SBATCH --mem=48G
#SBATCH --time=12:00:00
```

---

## Does this fit on ICE?

| Your need | ICE hardware |
|---|---|
| CPU-only feature extraction | AMD EPYC 64-core nodes (`ice-cpu`) |
| Small-to-medium training | V100 16/32 GB |
| Whisper-large fine-tuning | A100 40 GB or larger |
| Full speech model pretraining from scratch | **Escalate → [Part 4](../04-aws/)** |

---

## Common gotchas

> **This is a living document.** If you hit a GPU, VRAM, or resource-request issue while running an audio or signal processing workload on ICE — and especially if you figure out the fix — please add it here.

---


