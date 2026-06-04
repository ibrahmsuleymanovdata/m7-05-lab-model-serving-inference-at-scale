# Capacity Plan — Vision Moderation Service

## 1. Latency Budget Breakdown

Target: **p95 ≤ 250 ms** (synchronous endpoint, 90% of traffic)

| Stage | Budget (ms) | Notes |
|---|---|---|
| Network in | 5 | TCP receive, TLS decryption |
| Auth + routing | 3 | JWT verify, load-balancer routing |
| Payload parse | 5 | JSON decode + image decode |
| Feature lookup (Redis) | 10 | p95 ~8 ms + 2 ms network to Redis |
| Pre-processing | 10 | Resize, normalize — CPU-bound |
| Model inference | 25 | GPU T4 median ~22 ms, +3 ms buffer |
| Post-processing | 5 | Score threshold, label mapping |
| Serialization | 2 | JSON encode response |
| Network out | 5 | TCP send |
| **Headroom** | **180** | Covers GC pauses, queue wait, cold-start jitter |
| **Total** | **250** | |

> **Headroom rationale:** The sum of deterministic stages is 70 ms. The remaining 180 ms acts as a buffer for p95 tail effects: connection pool contention, Redis retries, ONNX Runtime warm-up jitter, and OS scheduling noise. At p95 we expect actual headroom consumption to be ~80–120 ms, leaving a real safety margin of ~60–100 ms.

---

## 2. CPU vs GPU Decision

**Decision: GPU (T4)**

| | CPU (c5.2xlarge, 8 vCPU) | GPU (g4dn.xlarge, 1× T4) |
|---|---|---|
| Inference latency | ~75 ms median | ~22 ms median |
| Concurrent requests / replica | ~8 (8 cores, 1 req/core) | ~20 (batching + GPU parallelism) |
| Throughput / replica (RPS) | ~10 RPS | ~35 RPS |
| On-demand price (us-east-1) | ~$0.34/hr | ~$0.526/hr |
| Monthly cost / replica | ~$245 | ~$379 |

*Prices from AWS on-demand list (approximate, within 30%): [aws.amazon.com/ec2/pricing](https://aws.amazon.com/ec2/pricing/on-demand/)*

**Justification:** At 300 RPS sustained, CPU needs ~30 replicas (~$7,350/mo) while GPU needs ~9 replicas (~$3,411/mo). GPU fits inside the $4,000/mo budget; CPU does not. Additionally, GPU inference at 22 ms leaves a far more comfortable latency headroom vs the 250 ms budget than CPU at 75 ms, reducing p95 tail risk. The 180 MB ONNX model fits comfortably in T4's 16 GB VRAM.

---

## 3. Replica Sizing

| Scenario | Target RPS | Per-replica throughput | Raw replicas needed | +30% headroom | Strategy | Monthly cost estimate |
|---|---|---|---|---|---|---|
| Sustained | 300 RPS | 35 RPS | 9 | **12 replicas** | Warm baseline | 12 × $379 = **$4,548/mo** |
| Spike | 500 RPS | 35 RPS | 15 | **18 replicas** | Autoscaling | 18 × $379 = **$6,822/mo** |

**Spike strategy: Autoscaling**

- Baseline: 12 warm replicas (covers sustained 300 RPS with 30% headroom).
- Autoscaler triggers at 70% CPU/GPU utilization, scales up to 18 replicas.
- Scale-up time target: ≤ 90 seconds (pre-warmed AMI with model cached on disk).
- 5-minute spike window is sufficient for autoscaler to respond and stabilize.
- Spike cost is transient; monthly bill stays near baseline (~$4,548/mo) since spikes are short.

> **Budget note:** Sustained cost of $4,548/mo slightly exceeds the $4,000 compute budget. Mitigation options: use Spot Instances for ~4 of the 12 baseline replicas (60–70% discount), or negotiate a 1-year Reserved Instance for baseline nodes. Spot-mixed baseline can bring cost to ~$3,700/mo.

---

## 4. Batching Decision

**Decision: Enable dynamic batching on the batch endpoint only (10% of traffic). Do NOT enable on the synchronous endpoint.**

The synchronous endpoint carries a strict p95 ≤ 250 ms SLO. Dynamic batching introduces a configurable wait window during which the server holds a request until a full batch accumulates. Even a 20 ms wait window would consume ~28% of the entire latency budget before inference even begins, making it incompatible with the synchronous SLO. Single-request inference on GPU at ~22 ms already fits the budget comfortably, so there is no throughput reason to batch on the sync path.

The batch endpoint (partner uploads, 10% of traffic) has no stated latency SLA — partners are aggregating uploads asynchronously. Here, dynamic batching with **max batch size = 16** and **max wait window = 50 ms** is appropriate. A batch of 16 images on a T4 GPU takes approximately 60–80 ms of inference time, well within any reasonable async SLO. Batching increases GPU utilization on the batch path from ~30% to ~85%, reducing the number of batch-path replicas needed and lowering cost. The batch endpoint should run on a dedicated replica pool (or at least a separate queue) to prevent batch wait-window delays from bleeding into the synchronous request queue.
