# Load Test Plan — Vision Moderation Service

## 1. Tool Choice

**Tool: k6**

k6 is chosen because it supports scripted traffic shaping (ramp functions, custom VU logic) natively in JavaScript, has built-in p95/p99 threshold assertions that fail the test automatically, and outputs Prometheus-compatible metrics for side-by-side comparison with production dashboards. It handles mixed endpoint scenarios (sync + batch at different ratios) cleanly in a single script without requiring a separate orchestration layer.

---

## 2. Test Phases

| Phase | Duration | Target RPS | Notes |
|---|---|---|---|
| Warmup | 5 min | 30 RPS (10% of baseline) | Allow JIT, connection pools, Redis caches to stabilize. Metrics excluded from pass/fail. |
| Ramp | 10 min | 30 → 300 RPS (linear) | Gradual increase; catch saturation points before hitting target. |
| Sustained peak | 30 min | 300 RPS | Primary measurement window. All SLO thresholds evaluated here. |
| Spike | 5 min | 300 → 500 RPS (step) | Simulate 5-minute burst. Validate autoscaler response (≤ 90s scale-up). |
| Recovery | 5 min | 500 → 300 RPS (step down) | Confirm graceful scale-down; no error spike during shed. |
| Soak | 60 min | 300 RPS | Long-duration stability: detect memory leaks, connection exhaustion, gradual Redis degradation. |

**Total test duration: ~115 minutes**

---

## 3. Traffic Shape

**Endpoint split:**

| Endpoint | Share | RPS at peak | Concurrency model |
|---|---|---|---|
| `POST /v1/moderate` (sync) | 90% | 270 RPS | 1 VU per request; closed-loop with 10 ms think time |
| `POST /v1/moderate/batch` (batch) | 10% | 30 RPS | 1 VU sends batch of 8–16 images; open-loop |

**Payload size distribution (sync endpoint):**

| Size bucket | Share | Image dimensions |
|---|---|---|
| Small | 50% | 224×224 px, ~15 KB JPEG |
| Medium | 35% | 512×512 px, ~80 KB JPEG |
| Large | 15% | 1024×1024 px, ~300 KB JPEG |

Payloads are pre-generated and stored locally to avoid network variance from image generation during the test. Each VU cycles through a pool of 500 unique images to prevent caching artifacts.

**Concurrency model:** k6 open-arrival-rate executor for the sync endpoint (maintains target RPS regardless of response time), ensuring realistic load even during latency degradation. Batch endpoint uses ramping-vus executor.

---

## 4. Pass / Fail Criteria

The test **fails** if any of the following thresholds are breached during the sustained peak or soak phase:

| Metric | Threshold | Measured on |
|---|---|---|
| p95 end-to-end latency (sync) | ≤ 250 ms | Sustained peak window |
| p99 end-to-end latency (sync) | ≤ 500 ms | Sustained peak window |
| HTTP error rate (5xx + timeouts) | ≤ 0.5% | Sustained peak window |
| p95 latency during spike | ≤ 400 ms | Spike window (5 min) |
| Scale-up completion time | ≤ 90 s | Spike onset to 18 healthy replicas |
| Error rate during scale-up | ≤ 1% | First 90 s of spike |
| p95 latency (soak, last 30 min) | ≤ 300 ms | Soak phase tail |
| Memory per replica drift | ≤ +15% over 60 min | Soak phase |

k6 threshold configuration (excerpt):
```javascript
thresholds: {
  'http_req_duration{endpoint:sync}': ['p(95)<250', 'p(99)<500'],
  'http_req_failed':                  ['rate<0.005'],
}
```

---

## 5. Bottleneck Checklist

Inspect the following on **each replica** during sustained peak and soak phases:

**Compute:**
- [ ] GPU utilization % (nvidia-smi / CloudWatch GPU metrics) — expected 60–80%; >90% indicates under-provisioned replicas
- [ ] GPU memory usage — should stay below 12 GB of 16 GB T4 VRAM
- [ ] CPU utilization % — pre/post-processing is CPU-bound; >85% indicates CPU bottleneck even with GPU inference

**Memory:**
- [ ] Heap / RSS per replica — watch for upward drift during soak (memory leak signal)
- [ ] ONNX Runtime session memory — confirm model is loaded once per replica, not per request

**Downstream dependencies:**
- [ ] Redis p95 latency — baseline ~8 ms; if rising above 15 ms, investigate connection pool size or Redis CPU
- [ ] Redis connection pool utilization — pool exhaustion causes queuing before inference even starts
- [ ] Redis error rate — any connection refused or timeout increments error budget

**Application internals:**
- [ ] Inference queue depth per replica — if queue grows during sustained peak, per-replica throughput is lower than estimated
- [ ] Request queue wait time (time from receipt to inference start) — should be < 10 ms at 300 RPS baseline
- [ ] Batch endpoint queue depth — confirm batch requests do not spill into sync replica pool

**Infrastructure:**
- [ ] Network bandwidth in/out per replica — large image payloads; check for NIC saturation
- [ ] Autoscaler event log — confirm scale-up triggers within 60 s of spike onset, new replicas pass health checks within 90 s
- [ ] Load balancer connection errors / 502s — indicates replica restart or health-check failure during scale events
