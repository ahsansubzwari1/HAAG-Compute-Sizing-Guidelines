# HAAG Compute & Storage Guide

A compute and storage education guide for **PACE ICE users** at Georgia Tech. Built for HAAG researchers — PhD students, postdocs, and faculty — who have working research code but limited HPC or cloud infrastructure experience.

---

## What this guide does

After reading this guide, you will be able to calculate exactly how much of each resource your research workload needs before you submit a job:

- **CPU cores** — for data loading, preprocessing, and CPU-bound computation
- **GPU VRAM** — for model weights, gradients, and activations during training or inference
- **System RAM** — for datasets, DataFrames, and working buffers
- **Storage** — for datasets, model caches, checkpoints, and intermediate files

The guide gives you a framework (the "four numbers"), a set of recipes for common workload types (LLM fine-tuning, computer vision, classical ML, audio), and a verdict table for each recipe that tells you which PACE ICE GPU actually fits your job.

---

## How the guide is structured

### [Part 2 — Sizing Your Workload](02-sizing/)

The heart of the guide. Introduces the four-number framework and walks through four recipes — LLM, computer vision, classical ML, audio — each ending with a "does this fit on ICE?" verdict table.

### [Part 3 — Choosing a GPU on ICE](03-pace-ice/)

A narrow, focused reference for picking a specific GPU out of ICE's eleven GPU types. Covers the reserved-node realities (H100/H200 capacity is tighter than it looks) and includes the **[LLM-GPU Sizing Calculator](tools/vram-calculator/)** for inference deployments.

### [Part 4 — AWS Reference](04-aws/)

Brief reference for when a workload can't fit on ICE. Covers EC2 instance families and AWS storage (S3, EBS, FSx Lustre), with a one-line cost warning at the end.

### [Tools — LLM-GPU Sizing Calculator](tools/vram-calculator/)

A browser-based calculator for estimating GPU memory requirements for LLM inference workloads. Single-file HTML, runs client-side, no install.

---

## Who should read this

- **New HAAG researchers** onboarding to PACE ICE who want to size their first real job
- **Returning researchers** picking a project back up and wanting a fast sanity-check on their resource requests
- **Project leads** reviewing job submissions from their team
- **Anyone** who has ever requested 8× H200 for a job that needed one V100

You do not need prior experience with Slurm, AWS, or HPC in general. You do need working research code.

---

## What this guide is not

- **Not a Slurm tutorial.** For Slurm mechanics, login procedures, and ICE-specific commands, see [guru-desh/Intro-To-PACE-ICE](https://github.com/guru-desh/Intro-To-PACE-ICE) and the PACE documentation linked throughout Part 3.
- **Not a job script validator.** For that, use Zach Wallace's [PACE Job Submission Validator](https://github.com/Human-Augment-Analytics/admin-high-performance-computing/tree/main/HPC_Job_Submission).
- **Not an AWS tutorial.** Part 4 is a narrow reference, not a getting-started guide.

---

## Related HAAG initiatives

This guide sits alongside other HAAG infrastructure work in the [admin-high-performance-computing](https://github.com/Human-Augment-Analytics/admin-high-performance-computing) repo:

- **[PACE Job Submission Validator](https://github.com/Human-Augment-Analytics/admin-high-performance-computing/tree/main/HPC_Job_Submission)** — catches common Slurm script errors before submission
- **[PACE Storage Audit](https://github.com/Human-Augment-Analytics/admin-high-performance-computing/tree/main/HPC_Storage_Audit)** — bi-weekly audits of HAAG shared storage
- **[HAAG HPC History](https://github.com/Human-Augment-Analytics/admin-high-performance-computing)** — documents the decisions and context behind HAAG's current infrastructure

---

## Living document

Every recipe in Part 2 has an empty "Common gotchas" section with a note inviting researchers to contribute real failures they've encountered and resolved. The goal is a HAAG-grown reference rather than generic internet advice.
