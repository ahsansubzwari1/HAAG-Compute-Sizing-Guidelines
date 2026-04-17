# Part 4 — AWS Reference

Most HAAG workloads fit on PACE ICE. This section is a brief reference for the cases that don't — specifically, what EC2 instances to use and how AWS storage works.

---

## EC2 instance families worth knowing

AWS has hundreds of instance types. Researchers only touch a few. Pick one of these families based on your workload from Part 2:

| Family | Hardware | Typical use | Workload match |
|---|---|---|---|
| `g5` | NVIDIA A10G, 24 GB | Cheap GPU for inference, small training | LoRA fine-tuning up to 13B, inference serving |
| `g6` / `g6e` | NVIDIA L4 / L40S, 24–48 GB | Newer, better for modern frameworks | Same as g5 but more VRAM |
| `p4d` / `p4de` | A100 40/80 GB, 8 GPUs per instance | Serious training with NVLink | Multi-GPU training of 13B–70B |
| `p5` / `p5en` | H100 / H200, 8 GPUs per instance | Highest tier, no queue | Equivalent to ICE's H-class — for when ICE queues are too long |
| `r6i` / `r7i` | Memory-optimized, up to 1 TB RAM | Tabular / pandas / DataFrames | Classical ML on large datasets |
| `x2iedn` | Extreme memory, up to 4 TB RAM | In-memory scientific workloads | What you use when 1.5 TB isn't enough |
| `c7i` / `c7a` | Compute-optimized, high CPU count | CPU-bound work | Feature extraction, Monte Carlo, bioinformatics |

**A few things this table doesn't show:**

- **Region matters.** `p5` instances aren't available in every region. Check availability before committing to an architecture.
- **SKU names drift.** Amazon adds and deprecates instance types constantly. Rather than memorize SKUs, learn the families (`g` = graphics/general GPU, `p` = high-end GPU, `r` = RAM-optimized, `c` = compute-optimized) and check current options on [the EC2 instance types page](https://aws.amazon.com/ec2/instance-types/) when you're ready to launch.
- **On-demand pricing is high.** An on-demand `p5.48xlarge` is around $100/hour. At that rate, a 48-hour fine-tune costs ~$4,800. Always check pricing before you launch, and consider whether a smaller instance running longer is cheaper.

---

## S3 for research data

Three storage services matter for research work on AWS. Learn the three, ignore the rest.

| Service | When to use | Cost shape |
|---|---|---|
| **S3** | Long-term dataset storage, checkpoint archival, shared data across collaborators | Pennies per GB per month |
| **EBS** | The scratch disk attached to your EC2 instance while it runs | ~$0.08/GB/month while attached |
| **FSx for Lustre** | Genuinely parallel I/O for big distributed training jobs | Expensive — only for when EBS can't keep up |

The pattern for most researchers: **upload dataset to S3 once, attach an EBS volume to your compute instance, read from S3 and stage to EBS if you need speed, write outputs back to S3, then terminate the EC2 instance.**

**The data transfer gotcha.** Moving data *into* AWS is free. Moving it *out* costs ~$0.09/GB — so 500 GB of trained model artifacts pulled back to your laptop costs $45. Moving data *within* an AWS region is free. Moving data *between* regions costs ~$0.02/GB. Practical rule: **pick one region, put your S3 bucket and your EC2 instances there, and don't move data around.** The free tier doesn't save you here — transfer-out is billed from byte one.

Compare this to ICE, where everything is on-campus and there's no metered transfer. That's one of the quiet advantages of staying on ICE when you can.

---

> ⚠️ **A note on cost.** AWS bills by the hour for compute and the GB-month for storage — an EC2 instance you forgot to terminate or an EBS volume you forgot to delete will keep billing until you notice. Set a monthly budget with email alerts in AWS Billing before you launch anything, and terminate resources as soon as you're done with them.
