# SageMaker Endpoint Quotas - eu-west-1 (Ireland)

**Account:** 753916465480
**Region:** eu-west-1 (Ireland)
**Last Updated:** 2026-03-17
**Total Instances Across Active Endpoints:** 20

> **Note:** Pricing shown is approximate SageMaker Real-Time Inference on-demand pricing for eu-west-1.
> Quotas may differ from eu-north-1. Verify current values in the Service Quotas console.
> Actual prices may vary. Check [AWS SageMaker Pricing](https://aws.amazon.com/sagemaker/pricing/) for current rates.

---

## GPU Instances (Required for vLLM)

These are the **only instances that support vLLM** inference with the DJL-LMI container.

| Instance Type | Quota | Price/Hour | vCPUs | RAM | GPU | GPU Model | GPU Memory | GPU Arch | Notes |
|---------------|-------|------------|-------|-----|-----|-----------|------------|----------|-------|
| **ml.g4dn.xlarge** | 1 | ~$0.74 | 4 | 16 GB | 1 | NVIDIA T4 Tensor Core | 16 GB GDDR6 | Turing (SM75) | **Recommended** - Best for distilgpt2, small models |
| **ml.g4dn.2xlarge** | 1 | ~$1.05 | 8 | 32 GB | 1 | NVIDIA T4 Tensor Core | 16 GB GDDR6 | Turing (SM75) | More CPU/RAM for larger models |

### NVIDIA T4 GPU Specifications

| Specification | Value |
|---------------|-------|
| Architecture | Turing (TU104) |
| CUDA Cores | 2,560 |
| Tensor Cores | 320 |
| GPU Memory | 16 GB GDDR6 |
| Memory Bandwidth | 320 GB/s |
| FP32 Performance | 8.1 TFLOPS |
| FP16 Performance | 65 TFLOPS (with Tensor Cores) |
| INT8 Performance | 130 TOPS (with Tensor Cores) |
| TDP | 70W |
| Compute Capability | 7.5 |

### GPU Instances with Zero Quota (Request Increase if Needed)

| Instance Type | Quota | Price/Hour | vCPUs | RAM | GPU | GPU Model | GPU Memory |
|---------------|-------|------------|-------|-----|-----|-----------|------------|
| ml.g4dn.4xlarge | 0 | ~$1.60 | 16 | 64 GB | 1 | NVIDIA T4 | 16 GB |
| ml.g4dn.8xlarge | 0 | ~$2.90 | 32 | 128 GB | 1 | NVIDIA T4 | 16 GB |
| ml.g4dn.12xlarge | 0 | ~$5.20 | 48 | 192 GB | 4 | NVIDIA T4 | 64 GB (4x16) |
| ml.g4dn.16xlarge | 0 | ~$5.80 | 64 | 256 GB | 1 | NVIDIA T4 | 16 GB |
| ml.g5.xlarge | 0 | ~$1.41 | 4 | 16 GB | 1 | NVIDIA A10G | 24 GB |
| ml.g5.2xlarge | 0 | ~$1.58 | 8 | 32 GB | 1 | NVIDIA A10G | 24 GB |
| ml.g5.4xlarge | 0 | ~$2.12 | 16 | 64 GB | 1 | NVIDIA A10G | 24 GB |
| ml.g5.8xlarge | 0 | ~$3.18 | 32 | 128 GB | 1 | NVIDIA A10G | 24 GB |
| ml.g5.12xlarge | 0 | ~$7.09 | 48 | 192 GB | 4 | NVIDIA A10G | 96 GB (4x24) |
| ml.g5.16xlarge | 0 | ~$5.36 | 64 | 256 GB | 1 | NVIDIA A10G | 24 GB |
| ml.g5.24xlarge | 0 | ~$10.63 | 96 | 384 GB | 4 | NVIDIA A10G | 96 GB (4x24) |
| ml.g5.48xlarge | 0 | ~$21.26 | 192 | 768 GB | 8 | NVIDIA A10G | 192 GB (8x24) |
| ml.g6.xlarge | 0 | ~$0.98 | 4 | 16 GB | 1 | NVIDIA L4 | 24 GB |
| ml.g6.2xlarge | 0 | ~$1.23 | 8 | 32 GB | 1 | NVIDIA L4 | 24 GB |
| ml.g6e.xlarge | 0 | ~$1.86 | 4 | 32 GB | 1 | NVIDIA L40S | 48 GB |
| ml.g6e.2xlarge | 0 | ~$2.30 | 8 | 64 GB | 1 | NVIDIA L40S | 48 GB |

---

## CPU Instances - General Purpose (M-Series)

Best for balanced workloads with moderate compute and memory requirements.

| Instance Type | Quota | Price/Hour | vCPUs | RAM | Storage | Network | Notes |
|---------------|-------|------------|-------|-----|---------|---------|-------|
| ml.m5.large | 4 | ~$0.12 | 2 | 8 GB | EBS | Up to 10 Gbps | |
| ml.m5.xlarge | 2 | ~$0.23 | 4 | 16 GB | EBS | Up to 10 Gbps | |
| ml.m5.2xlarge | 1 | ~$0.46 | 8 | 32 GB | EBS | Up to 10 Gbps | |
| ml.m5d.large | 4 | ~$0.14 | 2 | 8 GB | 1x75 NVMe SSD | Up to 10 Gbps | Local NVMe |
| ml.m5d.xlarge | 2 | ~$0.27 | 4 | 16 GB | 1x150 NVMe SSD | Up to 10 Gbps | Local NVMe |
| ml.m5d.2xlarge | 1 | ~$0.54 | 8 | 32 GB | 1x300 NVMe SSD | Up to 10 Gbps | Local NVMe |
| ml.m6g.large | 4 | ~$0.10 | 2 | 8 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.m6g.xlarge | 2 | ~$0.20 | 4 | 16 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.m6g.2xlarge | 1 | ~$0.39 | 8 | 32 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.m6gd.large | 4 | ~$0.11 | 2 | 8 GB | 1x118 NVMe SSD | Up to 10 Gbps | Graviton2 + NVMe |
| ml.m6gd.xlarge | 2 | ~$0.23 | 4 | 16 GB | 1x237 NVMe SSD | Up to 10 Gbps | Graviton2 + NVMe |
| ml.m6gd.2xlarge | 1 | ~$0.45 | 8 | 32 GB | 1x474 NVMe SSD | Up to 10 Gbps | Graviton2 + NVMe |

---

## CPU Instances - Compute Optimized (C-Series)

Best for compute-intensive workloads requiring high CPU performance.

| Instance Type | Quota | Price/Hour | vCPUs | RAM | Storage | Network | Notes |
|---------------|-------|------------|-------|-----|---------|---------|-------|
| ml.c5.large | 4 | ~$0.10 | 2 | 4 GB | EBS | Up to 10 Gbps | |
| ml.c5.xlarge | 2 | ~$0.20 | 4 | 8 GB | EBS | Up to 10 Gbps | |
| ml.c5.2xlarge | 1 | ~$0.40 | 8 | 16 GB | EBS | Up to 10 Gbps | |
| ml.c5.4xlarge | 1 | ~$0.80 | 16 | 32 GB | EBS | Up to 10 Gbps | |
| ml.c5d.large | 4 | ~$0.11 | 2 | 4 GB | 1x50 NVMe SSD | Up to 10 Gbps | Local NVMe |
| ml.c5d.xlarge | 2 | ~$0.23 | 4 | 8 GB | 1x100 NVMe SSD | Up to 10 Gbps | Local NVMe |
| ml.c5d.2xlarge | 1 | ~$0.46 | 8 | 16 GB | 1x200 NVMe SSD | Up to 10 Gbps | Local NVMe |
| ml.c6g.large | 4 | ~$0.09 | 2 | 4 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.c6g.xlarge | 2 | ~$0.17 | 4 | 8 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.c6g.2xlarge | 1 | ~$0.34 | 8 | 16 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.c6g.4xlarge | 1 | ~$0.68 | 16 | 32 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.c6gd.large | 4 | ~$0.10 | 2 | 4 GB | 1x118 NVMe SSD | Up to 10 Gbps | Graviton2 + NVMe |
| ml.c6gd.xlarge | 2 | ~$0.19 | 4 | 8 GB | 1x237 NVMe SSD | Up to 10 Gbps | Graviton2 + NVMe |
| ml.c6gd.2xlarge | 1 | ~$0.39 | 8 | 16 GB | 1x474 NVMe SSD | Up to 10 Gbps | Graviton2 + NVMe |
| ml.c6gn.large | 4 | ~$0.11 | 2 | 4 GB | EBS | Up to 25 Gbps | Graviton2 + Enhanced Network |
| ml.c6gn.xlarge | 2 | ~$0.22 | 4 | 8 GB | EBS | Up to 25 Gbps | Graviton2 + Enhanced Network |
| ml.c6gn.2xlarge | 1 | ~$0.43 | 8 | 16 GB | EBS | Up to 25 Gbps | Graviton2 + Enhanced Network |
| ml.c7g.large | 4 | ~$0.09 | 2 | 4 GB | EBS | Up to 12.5 Gbps | Graviton3 (ARM) |
| ml.c7g.xlarge | 2 | ~$0.18 | 4 | 8 GB | EBS | Up to 12.5 Gbps | Graviton3 (ARM) |
| ml.c7g.2xlarge | 1 | ~$0.36 | 8 | 16 GB | EBS | Up to 15 Gbps | Graviton3 (ARM) |
| ml.c7g.4xlarge | 1 | ~$0.73 | 16 | 32 GB | EBS | Up to 15 Gbps | Graviton3 (ARM) |

---

## CPU Instances - Memory Optimized (R-Series)

Best for memory-intensive workloads, large datasets, or models requiring significant RAM.

| Instance Type | Quota | Price/Hour | vCPUs | RAM | Storage | Network | Notes |
|---------------|-------|------------|-------|-----|---------|---------|-------|
| ml.r5.large | 1 | ~$0.15 | 2 | 16 GB | EBS | Up to 10 Gbps | |
| ml.r5.xlarge | 1 | ~$0.30 | 4 | 32 GB | EBS | Up to 10 Gbps | |
| ml.r5d.large | 1 | ~$0.17 | 2 | 16 GB | 1x75 NVMe SSD | Up to 10 Gbps | Local NVMe |
| ml.r5d.xlarge | 1 | ~$0.35 | 4 | 32 GB | 1x150 NVMe SSD | Up to 10 Gbps | Local NVMe |
| ml.r6g.large | 1 | ~$0.13 | 2 | 16 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.r6g.xlarge | 1 | ~$0.25 | 4 | 32 GB | EBS | Up to 10 Gbps | Graviton2 (ARM) |
| ml.r6gd.large | 1 | ~$0.14 | 2 | 16 GB | 1x118 NVMe SSD | Up to 10 Gbps | Graviton2 + NVMe |
| ml.r6gd.xlarge | 1 | ~$0.29 | 4 | 32 GB | 1x237 NVMe SSD | Up to 10 Gbps | Graviton2 + NVMe |

---

## Burstable Instances (T-Series)

Best for development, testing, and low-traffic workloads. Uses CPU credits for bursting.

| Instance Type | Quota | Price/Hour | vCPUs | RAM | Baseline CPU | Network | Notes |
|---------------|-------|------------|-------|-----|--------------|---------|-------|
| ml.t2.medium | 6 | ~$0.06 | 2 | 4 GB | 20% | Low to Moderate | Burstable, cost-effective |
| ml.t2.large | 6 | ~$0.11 | 2 | 8 GB | 30% | Low to Moderate | Burstable |
| ml.t2.xlarge | 6 | ~$0.22 | 4 | 16 GB | 40% | Moderate | Burstable |
| ml.t2.2xlarge | 6 | ~$0.45 | 8 | 32 GB | 40% | Moderate | Burstable |

---

## Cost Comparison Summary

### GPU Instances (Available)

| Instance | Hourly | Daily (24h) | Monthly (720h) | Best For |
|----------|--------|-------------|----------------|----------|
| ml.g4dn.xlarge | ~$0.74 | ~$17.76 | ~$533 | Small models (distilgpt2, GPT-2) |
| ml.g4dn.2xlarge | ~$1.05 | ~$25.20 | ~$756 | Medium models, more preprocessing |

### CPU Instances (Cheapest Available)

| Instance | Hourly | Daily (24h) | Monthly (720h) | Best For |
|----------|--------|-------------|----------------|----------|
| ml.t2.medium | ~$0.06 | ~$1.44 | ~$43 | Dev/test, minimal inference |
| ml.c6g.large | ~$0.09 | ~$2.16 | ~$65 | Lightweight CPU inference |
| ml.m6g.large | ~$0.10 | ~$2.40 | ~$72 | Balanced workloads |

---

## Important Notes

### vLLM Compatibility

- **GPU Required**: vLLM with DJL-LMI container requires NVIDIA GPU instances (g4dn, g5, g6 series)
- **ARM Not Supported**: Graviton-based instances (m6g, c6g, c7g, r6g) are NOT compatible with the vLLM DJL-LMI container
- **Minimum GPU Memory**: Most LLMs require at least 8GB GPU memory; T4's 16GB is sufficient for models up to ~7B parameters in fp16

### Requesting Quota Increases

1. Go to **AWS Console** → **Service Quotas** → **Amazon SageMaker**
2. Search for the instance type (e.g., "ml.g5.xlarge for endpoint usage")
3. Click **Request quota increase**
4. Specify the new limit and provide a use case justification
5. Typical approval time: 1-3 business days

### Cost Optimization Tips

- **Spot Instances**: Not available for SageMaker Real-Time Inference
- **Savings Plans**: Up to 64% discount with 1-year or 3-year commitment
- **Auto-scaling**: Configure endpoint auto-scaling to scale to 0 during idle periods
- **Delete When Unused**: SageMaker endpoints charge continuously while running

---

## References

- [AWS SageMaker Pricing](https://aws.amazon.com/sagemaker/pricing/)
- [AWS Service Quotas Console](https://console.aws.amazon.com/servicequotas/)
- [SageMaker Instance Types](https://aws.amazon.com/sagemaker/pricing/)
- [NVIDIA T4 Specifications](https://www.nvidia.com/en-us/data-center/tesla-t4/)
- [Vantage Instance Pricing](https://instances.vantage.sh/)
