# llm-d — SGLang Support Map

A feature-by-feature map of llm-d capabilities and whether they work with the **SGLang** model server.

- **Sources:** `llm-d` guides (`~/project/ai-inference/llm-d/guides`), `llm-d-router` code (`~/project/ai-inference/llm-d-router`), and the upstream **SGLang engine** (`~/project/ai-inference/offloading-connectors/sglang`, `sgl-project/sglang`, `gateway-v0.3.1` branch).
- **Legend:** ✅ supported · 🔸 works via composition / metric only (no dedicated overlay) · ⚠️ works with a caveat · ❌ not supported for SGLang.
- **Key principle:** routing/scoring runs in the **router (EPP)** and is *engine-agnostic* — a scorer works with SGLang as long as the metric or event it needs is exposed. Backend-specific work lives in the **model-server overlays**, the **sidecar proxy**, and the **coordinator connectors**.

---

## 1. Legend of what "supported" means

| Layer | Where | SGLang hook |
|---|---|---|
| Model-server overlay | `guides/*/modelserver/**/sglang/` | Deploys `lmsysorg/sglang` with the right flags |
| Metrics → routing | `llm-d-router/pkg/epp/.../metrics/factories.go` | Built-in `sglang` engine config |
| Precise prefix events | `llm-d-router/pkg/kvevents/engineadapter/sglang_adapter.go` | Parses SGLang ZMQ KV events |
| P/D disaggregation | `llm-d-router/pkg/sidecar/proxy/connector_sglang.go`, `pkg/coordinator/connectors/kv/sglang.go` (`kv-sglang`) | Bootstrap-server protocol |

---

## 2. Routing scorers / filters (router, engine-agnostic)

| Scorer / plugin | Needs | SGLang | Notes |
|---|---|---|---|
| `queue-scorer`, `load-aware-scorer` | `sglang:num_queue_reqs` | ✅ | filled in `factories.go` |
| `running-requests-size-scorer` | `sglang:num_running_reqs` | ✅ | filled |
| `kv-cache-utilization-scorer` | `sglang:token_usage` | ✅ | filled |
| `prefix-cache-scorer` / `prefix-cache-affinity-filter` | EPP-side prefix hash index | ✅ | no engine metric needed |
| `precise-prefix-cache-scorer` | engine KV-block events (ZMQ) | ✅ | via `sglang_adapter.go`; needs `--page-size=64` |
| `latency-scorer` / SLO admitter | queue + KV-usage → predictor sidecar | ✅ | both metrics exist; assumes homogeneous pool |
| `active-request`, `no-hit-lru`, `session-affinity`, `token-load`, `context-length` | EPP state / headers | ✅ | engine-independent |
| **`lora-affinity-scorer`** | `ActiveModels`/`WaitingModels`/`MaxActiveModels` | ❌ | `LoRASpec: ""` for sglang → LoRA routing does not work |
| `mm-embeddings-cache-scorer` | multimodal embedding cache attr | ❌ | only wired for vLLM multimodal path |

---

## 3. Disaggregation & KV transfer

| Capability | Needs | SGLang | Notes |
|---|---|---|---|
| **P/D disaggregation** (incl. its KV transport) | SGLang `connector_sglang.go` + `kv-sglang` bootstrap-server (port 8998, prefill *pushes* KV) over `--disaggregation-transfer-backend=nixl` (also `mooncake`/`ascend`/`mori`). vLLM connectors (`nixlv2`/`shared-storage`/`mooncake`) don't carry SGLang KV | ⚠️ | **NVIDIA GPU only**; validated each release but **not in nightly E2E CI**; no AMD overlay. NIXL reclaim is **decode-initiated** (`SGLANG_DISAGGREGATION_WAITING_TIMEOUT` → ABORT → prefill frees); **prefill has no independent timeout**, so if decode dies without notifying (crash/partition) KV can strand until pod restart (open upstream TODO, `nixl/conn.py`) |
| **CPU offload — HBM→CPU KV tier (single-pod, L2)** | SGLang **HiCache CPU** (`--hicache-size`/`--hicache-ratio`, `--hicache-io-backend`, `--hicache-write-policy`). vLLM equivalent: **OffloadingConnector** (LMCache) | ✅ **wired** | Grows effective cache by spilling evicted KV blocks from GPU HBM to CPU RAM, pulled back on demand. Shipped for SGLang: `tiered-prefix-cache/modelserver/gpu/sglang/native/cpu/`; benchmark shipped. **This is intra-pod, distinct from the remote/L3 tier below** |
| **Remote / shared KV tier (cluster-wide, L3)** | vLLM: **OffloadingConnector** (± Mooncake store). SGLang: **HiCache L3** — filesystem (`--hicache-storage-backend file`, Lustre) or Mooncake (`--hicache-storage-backend mooncake`) | ✅ **partly wired (fs/Lustre)** | **UPDATED Jul 2026:** SGLang **filesystem L3 backed by GCP Managed Lustre** shipped — overlays `tiered-prefix-cache/modelserver/gpu/sglang/native/fs/{base,gke}/` ([#2093](https://github.com/llm-d/llm-d/pull/2093), part of [#1967](https://github.com/llm-d/llm-d/issues/1967)). **Mooncake** L3 shipped for **vLLM** (`gpu/vllm/mooncake-store/`, [#1826](https://github.com/llm-d/llm-d/pull/1826)) but **still pending for SGLang** ([#1980](https://github.com/llm-d/llm-d/issues/1980)) — no `sglang/.../mooncake` overlay yet |
| **P2P KV pull (cross-instance / prefix pull)** | vLLM: `offloading` connector (`--kv-connector=offloading`), decode leg pulls KV from the prefiller's **OffloadingConnector** endpoint (`remote_host`/`remote_port`) — this is the vLLM/LMCache OffloadingConnector, **not Mooncake**. SGLang has no equivalent P2P connector in llm-d | ❌ **(SGLang) — vLLM-only** | **The merged work is vLLM-only:** `offloading` P2P connector ([llm-d-router#1888](https://github.com/llm-d/llm-d-router/pull/1888)) + DP tier ([#2075](https://github.com/llm-d/llm-d-router/pull/2075)) wire the **vLLM OffloadingConnector**, which SGLang does not use. **P2P for SGLang is still open** — SGLang's nearest capability is the L3 *shared store* above (fs/Lustre wired, Mooncake pending), which is a cluster-shared tier, **not** a dedicated cross-instance P2P connector. No SGLang P2P connector exists (no tracked issue). Distinct from P/D transport (a *push* prefill→decode) |
| **Encode disaggregation (EPD / E/P/D / E/PD)** | `coordinator/connectors/ec/` + `steps/encode.go` + EPP disagg profile handler | ❌ | **Control plane** (the `steps/encode.go` orchestration) is engine-agnostic, **but the data plane is not**: llm-d's EC connectors move embeddings over **NIXL / shared-storage**, while SGLang's encoder only speaks **zmq / mooncake** (`ENCODER_TRANSFER_BACKEND_CHOICES = zmq_to_scheduler, zmq_to_tokenizer, mooncake` — **no NIXL**). → **transport mismatch, not just a missing overlay**: needs a new SGLang EC connector (or NIXL added upstream), plus a SGLang encode overlay. Also still experimental / vLLM-only |

---

## 4. Guide-by-guide feature map

| Guide / feature | What it does | SGLang | Evidence |
|---|---|---|---|
| **optimized-baseline** | Recommended default: prefix + load-aware routing | ✅ **native** | `modelserver/gpu/sglang`, `modelserver/amd/sglang`; README:12,139,325; benchmark shipped |
| **precise-prefix-cache-routing** | Exact per-pod KV routing via ZMQ events | ✅ **native** | `modelserver/gpu/sglang`; README:32 (`--page-size=64`); benchmark shipped |
| **pd-disaggregation** | Split prefill / decode pools | ⚠️ **native, GPU-only** | `modelserver/gpu/sglang` (base/coreweave/gke); README:42,162 "validated each release"; not in CI; no AMD |
| **tiered-prefix-cache** | HBM→CPU (L2) + filesystem/Lustre (L3) KV offload | ✅ **native (HiCache), L2 + L3-fs** | CPU: `modelserver/gpu/sglang/native/cpu/`. **L3 filesystem/Lustre added Jul 2026:** `native/fs/{base,gke}/` ([#2093](https://github.com/llm-d/llm-d/pull/2093)); benchmark shipped |
| **predicted-latency-routing** | XGBoost latency predictor routing | 🔸 partial | no sglang subdir; README:164 says use optimized-baseline sglang overlay; homogeneous-pool caveat |
| **flow-control** | Multi-tenant fair queuing / backpressure | 🔸 indirect | sits on optimized-baseline; no own overlay |
| **batch-gateway** | OpenAI Batch API | 🔸 indirect | sits on optimized-baseline |
| **asynchronous-processing** | Queue-based async requests | 🔸 indirect | sits on optimized-baseline |
| **multi-model-routing** | IPP, many models / LoRA behind one gateway | 🔸 indirect | per-pool follows optimized-baseline; **LoRA routing itself ❌ for sglang** |
| **workload-autoscaling** | SLO-aware autoscaling (EPP+KEDA / WVA) | 🔸 indirect | builds on optimized-baseline |
| **agentic-serving** | Long multi-turn agentic composition | ❌ | overlays vLLM/TPU only; no sglang dir |
| **multimodal-serving** (aggregation + e-disaggregation) | Text+image/video/audio, encode disagg | ❌ | all overlays vLLM only; encode overlay vLLM only |
| **no-kubernetes-deployment** | Run stack without k8s (file discovery) | ❌ | README explicitly "targets vLLM on NVIDIA GPUs" |
| **wide-ep-lws** | Large MoE (DeepSeek-R1) wide expert parallelism, LWS + DP-aware | ❌ | vLLM only (`vllm` + `vllm-deepseek-v4` overlays); uses **custom llm-d-built vLLM image**; no sglang overlay. SGLang wide-EP is engine-native → tracked in **[#2040](https://github.com/llm-d/llm-d/issues/2040)** |
| **rl** (verl) | RL integration doc | n/a | doc only; verl patches both vLLM & SGLang for metrics |
| **inference-scheduling** | (empty placeholder) | n/a | — |
| **recipes** | Shared building blocks | ✅ (provides sglang images: `gpu-sglang`, `amd-sglang`; P/D base) | not a feature guide |

---

## 5. SGLang ENGINE (upstream) capability vs llm-d wiring

**Critical distinction:** the SGLang *engine* natively supports nearly everything. Every gap below is a **llm-d integration gap (missing overlay / missing metric mapping / connector mismatch)** — NOT a SGLang limitation. Verified against `sgl-project/sglang`.

| Capability | SGLang engine native? | Flags / evidence | llm-d wired? |
|---|---|---|---|
| Wide-EP / large MoE (DeepSeek-R1/V3) | ✅ Yes | `--ep-size`, `--moe-a2a-backend deepep`, `--enable-dp-attention`, `--deepep-mode` | ❌ no overlay (vLLM-only wide-ep-lws) |
| P/D disaggregation | ✅ Yes | `--disaggregation-mode`, `--disaggregation-transfer-backend {mooncake,nixl,ascend,fake,mori}` | ✅ GPU only (NVIDIA) |
| HiCache offload | ✅ Yes | `--hicache-size/-write-policy/-io-backend/-storage-backend {file,mooncake,nixl,hf3fs,...}` | ✅ CPU (L2) **+ filesystem/Lustre L3** ([#2093](https://github.com/llm-d/llm-d/pull/2093)); Mooncake L3 pending |
| KV cache events (ZMQ) | ✅ Yes | `--kv-events-config` (BlockStored/BlockRemoved, default `tcp://*:5557`) | ✅ via `sglang_adapter.go` |
| **LoRA + LoRA metrics** | ✅ Yes | `--enable-lora`, `--lora-paths`; exports `sglang:lora_pool_slots_used/_total/_utilization` | ❌ router `LoRASpec:""` **and** metric names/semantics differ from `ActiveModels/WaitingModels/MaxActiveModels` |
| Multimodal (VLM) | ✅ Yes | `python/sglang/srt/multimodal/` | ❌ no SGLang multimodal overlay |
| **Encode disaggregation (EPD)** | ✅ Yes | `--encoder-only`, `--language-only`, `--encoder-transfer-backend {zmq_to_scheduler,mooncake}`, `encode_server.py` | 🔸 router EPD orchestration is engine-agnostic (NIXL-based), but SGLang uses zmq/mooncake encoder transfer → **transport mismatch**, and no SGLang encode overlay |

### Two discrepancies worth verifying

1. **`sglang:cache_config_info` may not exist.** The router's `sglang` engine config (`factories.go`) reads `sglang:cache_config_info` with `page_size`/`num_pages` labels, but this exact metric was **not found** in the SGLang build inspected (it exposes `sglang:full_token_usage`, `sglang:num_used_tokens`, `sglang:kv_available_tokens`, `sglang:cache_hit_rate`, etc. instead). If confirmed, block-size auto-detection for SGLang would be a no-op → verify against your deployed SGLang version.
2. **LoRA routing is a mapping gap, not an engine gap.** SGLang *does* export LoRA metrics now, but as `sglang:lora_pool_*` (pool slot utilization), which is not the per-adapter active/waiting model list the router's `lora-affinity-scorer` expects. Closing this needs a router metric mapping + scorer adaptation, not SGLang work.

---

## 6. What's missing for SGLang (gap list — all are llm-d wiring, engine supports them)

1. **LoRA routing** *(no tracking issue)* — no LoRA metric → `lora-affinity-scorer` and multi-model LoRA routing don't work.
2. **Multimodal serving (VLM)** *(no tracking issue)* — no SGLang overlay (guide's `aggregation` + `e-disaggregation` are vLLM-only). *Aggregated* just needs an overlay; *encode-disaggregated (E/PD)* also needs router work — SGLang's encoder transport (zmq/mooncake) ≠ the EC connectors' NIXL → **transport mismatch, not just a missing overlay**.
3. **Wide-EP / large MoE (DeepSeek)** — vLLM-only (`wide-ep-lws` ships only `vllm` + `vllm-deepseek-v4` overlays); no SGLang wide-EP overlay. SGLang supports wide-EP upstream (`--ep-size`, `--moe-a2a-backend deepep`, `--enable-dp-attention`), just not wired in llm-d → tracked in **[#2040](https://github.com/llm-d/llm-d/issues/2040)** ([sglang]: Update WideEP Guide).
4. **Storage-offload / shared remote KV tier** — *partly wired now (UPDATED Jul 2026).* vLLM uses the **OffloadingConnector**; SGLang's equivalent is **HiCache L3/L4 backed by a distributed store** (`--hicache-storage-backend {file/lustre, mooncake, lmcache, hf3fs, …}`) — a cluster-shared, pull-on-miss tier, engine-native. **SGLang filesystem/Lustre L3 is now shipped** (`native/fs/` overlays, [#2093](https://github.com/llm-d/llm-d/pull/2093), part of [#1967](https://github.com/llm-d/llm-d/issues/1967)). Mooncake L3 already shipped for **vLLM** (`gpu/vllm/mooncake-store/` + `helpers/mooncake-*`, [#1826](https://github.com/llm-d/llm-d/pull/1826)), but **not yet for SGLang** ([#1980](https://github.com/llm-d/llm-d/issues/1980)) — no `sglang/.../mooncake` overlay. L4 also pending (+ router [llm-d-router#1010](https://github.com/llm-d/llm-d-router/issues/1010), **closed**). The vLLM **P2P `offloading` connector is merged** ([llm-d-router#1888](https://github.com/llm-d/llm-d-router/pull/1888) + DP [#2075](https://github.com/llm-d/llm-d-router/pull/2075)) — but that is the **vLLM OffloadingConnector, not Mooncake and not SGLang**; **P2P for SGLang remains unimplemented**. Separately, the llm-d `mooncake` **P/D** connector is **merged** (router [#1193](https://github.com/llm-d/llm-d-router/pull/1193), sidecar [#1556](https://github.com/llm-d/llm-d/pull/1556)); tracking [#1524](https://github.com/llm-d/llm-d/issues/1524) still open.
5. **AMD P/D disaggregation** — SGLang P/D is NVIDIA-only.
6. **CI coverage** — SGLang P/D not in nightly E2E (→ **[#2045](https://github.com/llm-d/llm-d/issues/2045)** [sglang]: CI for all well lit paths); NIXL prefill-side KV-reclaim limitation.
7. **agentic-serving / no-kubernetes** guides — vLLM-only wiring.

---

## 6b. Tracking & roadmap (issues + owner-reported status, merged)

Single source of truth. Status is owner-reported (from the SGLang tracking issue); where it conflicts with a code-verified mark above, the verified mark wins. **gRPC** and **Observability** are roadmap items no gap row above tracks.

| Feature / gap | Status | Target | Issue / owner |
|---|---|---|---|
| Approximate prefix cache | ✅ Done | 0.8 | Rahul |
| Precise prefix-cache routing (KV events) | 🔄 In progress | 0.8 | @zdtsw |
| Tiered prefix cache **L1/L2** (CPU offload) | ✅ Done | 0.8 | Rahul |
| Latency predictor | ✅ Done | 0.8 | Rahul |
| Flow control | ✅ Done | 0.8 | — |
| Batch processing | ✅ Done | 0.8 | — |
| **Observability — SGLang in Grafana** 🆕 | ✅ Done | 0.8 | sudoalok |
| **gRPC support** 🆕 | 🔄 In progress | 1.0 | Ryan/Rahul |
| P/D disaggregation — **RDMA recipes** | 🔄 In progress (docs ✅ [#1641](https://github.com/llm-d/llm-d/issues/1641) closed Jul 21) | 0.9 | Rahul |
| P/D benchmarking (Prism) | ⬜ Not started | 0.9 | [#1752](https://github.com/llm-d/llm-d/issues/1752), [#1749](https://github.com/llm-d/llm-d/issues/1749) |
| Wide-EP / large-MoE — recipes + benchmarks | ⬜ Not started | 0.9 | [#2040](https://github.com/llm-d/llm-d/issues/2040) |
| Tiered prefix cache **L3/L4** (lmcache/lustre/mooncake) | 🔄 Partial — SGLang **fs/Lustre L3 ✅ merged** ([#2093](https://github.com/llm-d/llm-d/pull/2093)); SGLang **mooncake pending** (mooncake L3 exists for vLLM only, [#1826](https://github.com/llm-d/llm-d/pull/1826)) | 0.9 | Owner Yuchen; [#1967](https://github.com/llm-d/llm-d/issues/1967) (lustre, ✅), [#1980](https://github.com/llm-d/llm-d/issues/1980) (sglang mooncake); router [#1010](https://github.com/llm-d/llm-d-router/issues/1010) closed |
| **P2P KV pull — `offloading` connector** (vLLM only) 🆕 | ✅ Done for **vLLM**; ❌ **SGLang still open** (no tracked issue) | — | vLLM OffloadingConnector: router [#1888](https://github.com/llm-d/llm-d-router/pull/1888) + DP [#2075](https://github.com/llm-d/llm-d-router/pull/2075); **not Mooncake, not SGLang** |
| Multimodal **prefix-cache** (placeholder-guess algo) | ⬜ Not started | 1.0 | Rahul/Xiyue |
| Multimodal **encoder-cache** plugin | ⬜ Not started | 0.9 | Rahul, [#1952](https://github.com/llm-d/llm-d/issues/1952) |
| Mooncake as **P/D connector** | ✅ Merged (connector landed) | 0.9 | router [#1193](https://github.com/llm-d/llm-d-router/pull/1193), sidecar [#1556](https://github.com/llm-d/llm-d/pull/1556); tracking [#1524](https://github.com/llm-d/llm-d/issues/1524) open |
| CI across well-lit paths | ⬜ Not started | — | [#2045](https://github.com/llm-d/llm-d/issues/2045) |
| TPU support — recipes + benchmarks | ⬜ Not started | 1.0 | [#666](https://github.com/llm-d/llm-d/issues/666) |
| Workload autoscaling | ⬜ Not started (needs verification) | TBD | — |
| Approx prefix cache — Prism entry | ⬜ Not started | — | Rahul/Radhika |
| Production-readiness / Ascend NPU (question) | — | — | [#2060](https://github.com/llm-d/llm-d/issues/2060) |

---

## 7. One-line summary

*Last checked against GitHub: 2026-08-02.*

SGLang is a **first-class backend** for the core well-lit paths (optimized-baseline, precise-prefix, P/D on NVIDIA, tiered HiCache **incl. filesystem/Lustre L3**) and inherits all engine-agnostic routing. **Recently closed (Jul 2026):** Mooncake **P/D** connector merged ([#1193](https://github.com/llm-d/llm-d-router/pull/1193)/[#1556](https://github.com/llm-d/llm-d/pull/1556)), SGLang **fs/Lustre L3** offload shipped ([#2093](https://github.com/llm-d/llm-d/pull/2093)). The remaining gaps — **LoRA routing (no tracking issue), multimodal/encode-disagg (no tracking issue), wide-EP large-MoE ([#2040](https://github.com/llm-d/llm-d/issues/2040)), Mooncake L3 tier ([#1980](https://github.com/llm-d/llm-d/issues/1980)), P2P KV pull for SGLang (no tracked issue), AMD P/D, CI coverage ([#2045](https://github.com/llm-d/llm-d/issues/2045))** — are **all llm-d wiring gaps (missing overlays / metric mappings), not SGLang engine limitations**. **§6b** is the full tracking + roadmap table — 0.8 shipped; 0.9/1.0 in flight.
