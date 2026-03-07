# Benchmarking and Performance Evaluation of Inference Routing Extensions in kgateway

## Table of Contents

- [1. Project Overview](#1-project-overview)
- [2. Background and Context](#2-background-and-context)
- [3. Test Setup Architecture](#3-test-setup-architecture)
- [4. Scope: GSoC Core vs Stretch Goals](#4-scope-gsoc-core-vs-stretch-goals)
- [5. Benchmark Configurations](#5-benchmark-configurations)
- [6. Tools Selection and Analysis](#6-tools-selection-and-analysis)
- [7. Implementation Plan](#7-implementation-plan)
- [8. Metrics and Data Collection](#8-metrics-and-data-collection)
- [9. CI/CD Automation](#9-cicd-automation)
- [10. Documentation Plan](#10-documentation-plan)
- [11. Timeline and Milestones](#11-timeline-and-milestones)
- [12. TODO Checklist](#12-todo-checklist)

---

## 1. Project Overview

### Problem Statement

kgateway provides inference routing capabilities via the [Gateway API Inference Extension](https://github.com/kubernetes-sigs/gateway-api-inference-extension) (GIE), enabling model-aware routing of self-hosted GenAI models through Envoy's `ext_proc` filter. There is currently **no standardized or reproducible way** to evaluate the performance impact of these extensions in kgateway:

- No published benchmarks comparing **kgateway + inference extension** vs **plain k8s Service**
- No reproducible test environment script for users to validate results
- No periodic CI job for inference perf regression testing
- No integration of perf regressions into the release process

### Goals

1. Publish benchmarks (**TPOT, ITL, TTFT, E2E latency**) comparing kgateway inference routing vs plain k8s service
2. Build **reproducible environment setup scripts** so users can validate results themselves
3. Implement a **periodic CI job** (nightly/weekly) against the `main` branch
4. Integrate benchmarks into the **release process** to gate on regressions
5. Document methodology and interpretation

### Relationship to Existing Work

> [!IMPORTANT]
> This is complementary to [design/11210](file:///home/shubham/Code/Personal/kgateway/design/11210-kgateway-load-testing-framework.md) (control-plane scale testing). This proposal targets **data-plane inference performance** — token latency, throughput, and routing overhead — following the same methodology as the [GIE upstream benchmark](https://gateway-api-inference-extension.sigs.k8s.io/performance/benchmark/).

---

## 2. Background and Context

### What is the Gateway API Inference Extension (GIE)?

| Component | Description |
|-----------|-------------|
| **InferencePool** | K8s CRD defining a pool of model-serving backends |
| **Endpoint Picker (EPP)** | ext_proc server selecting optimal backends via KV-cache/LoRA/queue-depth awareness |
| **Body Based Router (BBR)** | Optional ext_proc server parsing request bodies for model-name routing |
| **InferenceModel** | CRD mapping client model names to backend models/LoRA adapters |

### How kgateway Integrates GIE

```mermaid
flowchart LR
    subgraph "Phase 1: Policy -> IR"
        CRD["InferencePool CRD"] --> PIR["PolicyIR"]
    end
    subgraph "Phase 2: Route -> IR"
        HR["HTTPRoute"] --> GIR["Gateway IR"]
        PIR -.->|"targetRef attachment"| GIR
    end
    subgraph "Phase 3: IR -> xDS"
        GIR --> xDS["Envoy xDS Config<br/>(routes + ext_proc filter)"]
    end
    xDS --> Envoy["Envoy Proxy"]
```

---

## 3. Test Setup Architecture

This diagram shows how all components connect during a benchmark run. It mirrors the GIE upstream approach with kgateway-specific additions.

```mermaid
flowchart TB
    subgraph "Load Generation"
        IP["inference-perf<br/>(Kubernetes Job)"]
        LPG["Latency Profile Generator<br/>(regression testing)"]
    end

    subgraph "Kubernetes Cluster (GKE / Kind)"

        subgraph "Gateway Layer"
            GW["kgateway Controller"]
            ENV["Envoy Proxy<br/>(data plane)"]
            GW -.->|"xDS config"| ENV
        end

        subgraph "Inference Extension Layer"
            EPP["Endpoint Picker<br/>(ext_proc server)"]
            BBR["Body Based Router<br/>(optional)"]
        end

        subgraph "Target A: Inference Gateway"
            IGW_RT["HTTPRoute → InferencePool"]
        end

        subgraph "Target B: Baseline"
            K8S_SVC["HTTPRoute → k8s Service<br/>(round-robin LB)"]
        end

        subgraph "Model Servers  ×8-10 replicas"
            MS1["vLLM / llm-d-inference-sim #1"]
            MS2["vLLM / llm-d-inference-sim #2"]
            MSN["vLLM / llm-d-inference-sim #N"]
        end

        subgraph "Observability"
            PROM["Prometheus"]
            CA["cAdvisor"]
        end
    end

    subgraph "Result Storage"
        LOCAL["Local (pod filesystem)"]
        GCS["Google Cloud Storage"]
        S3["AWS S3"]
    end

    IP -->|"POST /v1/chat/completions<br/>(Scenario A)"| IGW_RT
    IP -->|"POST /v1/chat/completions<br/>(Scenario B)"| K8S_SVC
    LPG -->|"regression load"| IGW_RT

    IGW_RT --> ENV
    ENV <-->|"ext_proc gRPC"| EPP
    ENV <-->|"ext_proc gRPC (opt)"| BBR
    ENV --> MS1 & MS2 & MSN

    K8S_SVC --> MS1 & MS2 & MSN

    EPP -.->|"metrics scrape"| MS1 & MS2 & MSN
    PROM -.->|"scrape"| ENV & EPP & CA

    IP -->|"results JSON"| LOCAL
    LOCAL -.->|"optional upload"| GCS & S3
```

> [!NOTE]
> **What we are NOT including:** Jupyter notebook analysis, HuggingFace dataset downloads, or Looker Studio dashboards. These are part of the upstream GIE project's internal tooling. Our reporting will use Go-generated Markdown + JSON summaries, consistent with kgateway's existing test infrastructure.

---

## 4. Scope: GSoC Core vs Stretch Goals

| # | GSoC Core (6 Weeks) | Description |
|---|---------------------|-------------|
| 1 | Reproducible setup scripts | One-command env setup using `llm-d-inference-sim` (no GPU) |
| 2 | `inference-perf` integration | Configure and wire up for kgateway-specific scenarios |
| 3 | IGW vs k8s service comparison | Side-by-side benchmark with TPOT/ITL/TTFT metrics |
| 4 | LPG regression testing | Latency Profile Generator for CI regression detection |
| 5 | 4 upstream test configs | Prefix Cache Aware, Decode Heavy, Prefill Heavy, Multi-LoRA |
| 6 | Result storage (local + GCS/S3) | Store JSON results; upload to cloud on release runs |
| 7 | Periodic CI job (nightly) | GitHub Actions workflow against `main` branch |
| 8 | Markdown + JSON reporting | Go-generated report; no Jupyter or external dashboards |
| 9 | Documentation | Methodology, how to reproduce, result interpretation |

| # | Stretch Goal | Blocker |
|---|-------------|---------|
| S1 | Real GPU env (B100 / H100 80GB, 8-10 replicas) | Cloud GPU budget / OSS credits |
| S2 | GKE-based e2e workflows (like upstream) | GPU env (S1) |
| S3 | Published benchmark page on kgateway website | Website infra access |
| S4 | Release process regression gate | Maintainer sign-off on thresholds |
| S5 | GPU utilization metrics | `inference-perf` roadmap |

---

## 5. Benchmark Configurations

We adopt the same four configurations as the upstream GIE benchmark, tailored for kgateway.

### 5.1. Standard Comparison (IGW vs k8s Service)

The foundational benchmark — identical to what the upstream GIE project publishes:

| Scenario | Target | Routing |
|----------|--------|---------|
| **Scenario A** | kgateway + InferencePool (IGW) | EPP-aware scheduling |
| **Scenario B** | kgateway + k8s LoadBalancer Service | Plain round-robin |

Both target the same pool of model server pods. Multiple `inference-perf` instances run simultaneously for a fair comparison.

### 5.2. Prefix Cache Aware

Tests EPP's KV-cache-hit-aware scheduling. EPP preferentially routes requests with shared prompt prefixes to pods that have the prefix cached, reducing TTFT.

- **Workload:** Requests with shared prompt prefixes (multi-turn conversations)
- **What we measure:** TTFT improvement from cache hits vs random routing
- **Key metric:** TTFT delta between IGW (cache-aware) and k8s service (random)

### 5.3. Decode Heavy

Sustained token generation workload — long output sequences relative to input. Stresses the EPP's queue-depth awareness as pods build up large generation queues.

- **Workload:** Short input (~50 tokens), long output (~500+ tokens)
- **What we measure:** TPOT and ITL under sustained output load
- **Key metric:** Throughput (tokens/sec) and tail TPOT at high concurrency

### 5.4. Prefill Heavy

Computationally expensive initial token generation — long input, short output. Stresses the EPP's awareness of prefill phase saturation.

- **Workload:** Long input (~2000 tokens), short output (~50 tokens)
- **What we measure:** TTFT under heavy prefill load
- **Key metric:** TTFT p95/p99 at increasing concurrency

### 5.5. Multi-LoRA

Multiple LoRA adapter variants served from the same pool. EPP routes based on which pods have the requested adapter loaded.

- **Workload:** Mixed traffic across 2-4 different model names (each mapping to a LoRA adapter)
- **What we measure:** Routing accuracy (requests land on correct adapter) and overhead of LoRA-aware scheduling
- **Key metric:** Error rate + per-adapter TPOT

### 5.6. Load Profiles

```mermaid
graph LR
    subgraph "Standard Ramp"
        L1["Warm-up<br/>(low RPS, 60s)"] --> L2["Sustained<br/>(target RPS, 300s)"]
        L2 --> L3["Saturation<br/>(increasing until error, 120s)"]
        L3 --> L4["Cool-down<br/>(60s)"]
    end
```

`inference-perf` natively supports Gaussian, fixed, and min-max input/output token distributions. We use its built-in load profiles rather than custom implementations.

---

## 6. Tools Selection and Analysis

### 6.1. Primary Benchmark Tool: `inference-perf`

> [!TIP]
> **Primary tool: [`kubernetes-sigs/inference-perf`](https://github.com/kubernetes-sigs/inference-perf)** — the official `wg-serving` GenAI benchmarking standard, already used by GIE's upstream benchmark.

| Why `inference-perf` | Detail |
|----------|--------|
| Official upstream tool | Used by GIE published benchmarks; ensures apples-to-apples comparison |
| LLM-aware metrics natively | TPOT, ITL, TTFT built-in — no custom scripting needed |
| Supports our 4 workload configs | Decode heavy, prefill heavy, prefix cache, multi-LoRA all supported |
| Runs as a K8s Job | Inside-cluster traffic — no external network bias |
| Multiple model server backends | vLLM, SGLang, TGI, llm-d, Inference Gateway |
| Configurable load patterns | Burst, constant rate, scale to saturation |

**Why not k6?** k6 is excellent for HTTP API benchmarking but has no native concept of token distributions, LLM datasets, or LLM-specific metrics. We would reinvent what `inference-perf` already provides.

### 6.2. Regression Tool: Latency Profile Generator (LPG)

The **Latency Profile Generator** is a separate upstream tool used specifically for **regression testing** — it generates a controlled latency profile to detect performance degradation between kgateway versions.

| Feature | Description |
|---------|-------------|
| **Purpose** | Detect regressions (not measure peak performance) |
| **Use in CI** | Run on every PR against `main`; fast, deterministic |
| **Output** | Pass/fail against stored latency profiles |
| **Complement** | Works alongside `inference-perf` — perf benchmarks measure absolutes; LPG catches regressions |

### 6.3. Model Server: `llm-d-inference-sim`

| Option | Use case | Pros | Cons |
|--------|----------|------|------|
| **`llm-d-inference-sim`** (core) | CI / local dev (no GPU) | GPU-free, realistic vLLM metrics, LoRA sim, physics-based latency | External image dependency |
| **vLLM (H100 80GB) ×8-10** (stretch) | GPU env / release validation | Most realistic | Requires cloud GPU budget |
| GIE's simple vLLM sim | Quick integration test | Minimal dep | No real EPP signal quality |

> 8-10 replicas recommended by the upstream team for meaningful EPP routing decisions (EPP needs enough pods to have scheduling choices).

### 6.4. Storage Options

| Storage | When to Use |
|---------|-------------|
| **Local** (default) | Local dev / PR-level runs; results lost when pod terminates |
| **Google Cloud Storage (GCS)** | Nightly / release runs; persistent, queryable history |
| **AWS S3** | Alternative to GCS for AWS-hosted CI |

For GSoC core we start with **local + GitHub Actions artifact upload**. GCS/S3 integration is implemented as part of the CI workflow for release runs.

---

## 7. Implementation Plan

### 7.1. Directory Structure

```
test/
  benchmark/
    README.md
    Makefile
    setup/
      cluster-kind.sh             # Reproducible Kind cluster (no GPU)
      cluster-gke.sh              # GKE cluster with GPU nodes (stretch)
      install-kgateway.sh         # kgateway + inference extension enabled
      install-inference-perf.sh   # inference-perf + LPG install
    manifests/
      gateway-igw.yaml            # Gateway + HTTPRoute -> InferencePool (Scenario A)
      gateway-k8s-svc.yaml        # Gateway + HTTPRoute -> Service (Scenario B)
      inferencepool.yaml          # InferencePool + InferenceModel
      epp.yaml                    # EPP Deployment + Service
      llm-d-sim.yaml              # llm-d-inference-sim Deployment (×3, no-GPU)
      prometheus.yaml             # Prometheus stack
    inference-perf/
      config-standard.yaml        # Basic IGW vs k8s service
      config-prefix-cache.yaml    # Prefix cache aware workload
      config-decode-heavy.yaml    # Decode heavy workload
      config-prefill-heavy.yaml   # Prefill heavy workload
      config-multilora.yaml       # Multi-LoRA workload
    lpg/
      config-regression.yaml      # LPG regression test config
      profiles/
        baseline-profile.json     # Committed latency profile for regression detection
    harness/
      benchmark_test.go           # Go test entry points
      setup.go                    # Cluster + resource setup
      teardown.go                 # Cleanup
      results.go                  # Results collection (inference-perf JSON + Prometheus)
      report.go                   # Markdown + JSON report generation
      storage.go                  # Upload results to GCS / S3
    results/
      baseline/
        benchmark-results.json    # Committed baseline for GSoC-era comparison
    docs/
      methodology.md
      interpreting-results.md
      reproducing.md
      best-practices.md
```

### 7.2. Benchmark Execution Flow

```mermaid
flowchart LR
    subgraph "1. Setup"
        S1["Create cluster<br/>(Kind or GKE)"]
        S2["Install kgateway<br/>+ GIE CRDs"]
        S3["Deploy llm-d-sim<br/>×3 (or vLLM ×8-10)"]
        S4["Deploy EPP"]
        S5["Create Gateways<br/>A (IGW) + B (k8s svc)"]
        S1 --> S2 --> S3 --> S4 --> S5
    end

    subgraph "2. Benchmark"
        B1["Run inference-perf<br/>→ IGW (A)"]
        B2["Run inference-perf<br/>→ k8s svc (B)"]
        B3["Run LPG<br/>(regression check)"]
        B1 & B2 & B3
    end

    subgraph "3. Collect"
        C1["Parse inference-perf JSON"]
        C2["Query Prometheus<br/>(resource overhead)"]
        C3["Compute deltas<br/>(A vs B)"]
        C1 --> C3
        C2 --> C3
    end

    subgraph "4. Report + Store"
        R1["Markdown report"]
        R2["JSON summary"]
        R3["Upload to GCS/S3<br/>(nightly/release)"]
        C3 --> R1 & R2 --> R3
    end

    S5 --> B1 & B2 & B3
    B1 & B2 & B3 --> C1 & C2
```

---

## 8. Metrics and Data Collection

### 8.1. Primary LLM Metrics (from `inference-perf`)

| Metric | Full Name | What it measures |
|--------|-----------|-----------------|
| **TPOT** | Time Per Output Token | Avg time between generated tokens — EPP scheduling cost |
| **ITL** | Inter-Token Latency | Latency between consecutive streaming tokens |
| **TTFT** | Time to First Token | Time to first response token — includes EPP round-trip + prefill |
| **E2E latency** | End-to-end (p50/p95/p99) | Total request duration |
| **Throughput** | Tokens/sec + Requests/sec | System capacity |
| **Error rate** | Failed request % | Stability under load |

### 8.2. Resource Overhead Metrics (from Prometheus + cAdvisor)

| Metric | Target Components |
|--------|------------------|
| CPU milliseconds | Envoy proxy, EPP, BBR, kgateway controller |
| Memory bytes | Envoy proxy, EPP, BBR |
| IGW vs k8s svc delta | Overhead attributable to inference routing |

### 8.3. Envoy ext_proc Metrics

| Metric | What it reveals |
|--------|----------------|
| `envoy_ext_proc_streams_started` | Total EPP calls |
| `envoy_ext_proc_streams_msgs_sent` | gRPC message volume |
| `envoy_http_downstream_rq_time_bucket` | End-to-end Envoy routing latency |
| `envoy_cluster_upstream_rq_time_bucket` | Upstream (model server) response latency |

### 8.4. Report Format

```json
{
  "timestamp": "2026-04-01T00:00:00Z",
  "kgateway_version": "v2.3.0",
  "config": "prefix-cache-aware",
  "model_server": "llm-d-inference-sim",
  "replicas": 3,
  "results": {
    "igw": { "tpot_ms": 12.4, "itl_ms": 11.8, "ttft_ms": 45.2, "e2e_p99_ms": 3100, "throughput_tps": 810 },
    "k8s_svc": { "tpot_ms": 12.1, "itl_ms": 11.5, "ttft_ms": 28.3, "e2e_p99_ms": 2950, "throughput_tps": 840 }
  },
  "overhead": { "ttft_delta_ms": 16.9, "epp_cpu_millicores": 145, "epp_memory_mb": 92 },
  "regression": { "lpg_passed": true, "baseline_ttft_ms": 44.1, "delta_pct": 2.5 }
}
```

---

## 9. CI/CD Automation

### 9.1. CI Workflow

```mermaid
flowchart TD
    subgraph "Triggers"
        NT["Nightly Schedule"] & RL["Release Tag"] & WD["Manual Dispatch"] & PR["PR + 'benchmark' label"]
    end

    subgraph "GitHub Actions Job"
        SC["Setup cluster + kgateway"]
        DS["Deploy llm-d-sim + EPP"]
        LR["Run LPG (regression)"]
        BR["Run inference-perf<br/>(all 4 configs × 2 scenarios)"]
        CO["Collect + compare results"]
        UP["Upload artifacts<br/>(JSON + Markdown)"]
        GU["Upload to GCS/S3<br/>(nightly + release only)"]
    end

    subgraph "Result"
        TH{"Regression<br/>detected?"}
        PASS["✓ PASS"]
        FAIL["✗ FAIL"]
    end

    NT & RL & WD & PR --> SC --> DS --> LR & BR --> CO --> UP --> TH
    CO -.->|"nightly/release"| GU
    TH -->|"No"| PASS
    TH -->|"Yes"| FAIL
```

### 9.2. Workflow Files

| File | Trigger | Description |
|------|---------|-------------|
| `.github/workflows/benchmark-nightly.yaml` | Nightly cron | Full benchmark suite against `main` |
| `.github/workflows/benchmark-release.yaml` | Release tag | Full suite + GCS upload |
| `.github/workflows/benchmark-regression.yaml` | PR (label-gated) | LPG regression test only (fast) |

### 9.3. Makefile Targets

```makefile
.PHONY: benchmark
benchmark:           ## Run full benchmark suite (IGW vs k8s-svc, all configs)
benchmark-setup:     ## Set up benchmark environment (cluster + dependencies)
benchmark-igw:       ## Run inference gateway scenarios only
benchmark-k8s-svc:   ## Run k8s service baseline only
benchmark-regression:## Run LPG regression test only
benchmark-report:    ## Generate report from latest results
benchmark-compare:   ## Compare latest vs committed baseline
```

---

## 10. Documentation Plan

| Document | Location | Description |
|----------|----------|-------------|
| **README** | `test/benchmark/README.md` | Quick start: run in one command |
| **Methodology** | `test/benchmark/docs/methodology.md` | Config rationale, assumptions, what we don't test |
| **Interpreting Results** | `test/benchmark/docs/interpreting-results.md` | What TPOT/ITL/TTFT mean, how to read the JSON report |
| **Reproducing Benchmarks** | `test/benchmark/docs/reproducing.md` | Step-by-step guide for users to validate results |
| **Best Practices** | `test/benchmark/docs/best-practices.md` | EPP tuning, InferencePool sizing, scaling tips |
| **Design Document** | `design/XXXX-inference-routing-benchmarks.md` | Formal EP following kgateway EP template |

---

## 11. Timeline and Milestones

```mermaid
gantt
    title Inference Routing Benchmark Framework — 6 Weeks
    dateFormat YYYY-MM-DD
    axisFormat %b %d

    section Week 1: Foundation
    Design doc + maintainer review     :des,   2026-05-01, 5d
    Cluster setup scripts              :setup, 2026-05-03, 4d

    section Week 2: Core Integration
    inference-perf + standard config   :ip,    2026-05-08, 5d
    llm-d-sim deployment + EPP wiring  :sim,   2026-05-10, 3d

    section Week 3: All 4 Configurations
    Prefix cache + decode/prefill heavy :cfg,  2026-05-15, 4d
    Multi-LoRA configuration           :lora,  2026-05-18, 3d

    section Week 4: LPG + Harness
    LPG regression setup               :lpg,   2026-05-22, 3d
    Go test harness + results/reports  :har,   2026-05-22, 5d

    section Week 5: CI + Storage
    GitHub Actions workflows           :ci,    2026-05-29, 4d
    GCS/S3 result upload               :store, 2026-06-01, 3d

    section Week 6: Docs + Polish
    Documentation                      :doc,   2026-06-05, 4d
    Final PR + review                  :rev,   2026-06-08, 3d
```

| Week | Milestone | Deliverables |
|------|-----------|-------------|
| **1** | Foundation | Design doc approved, cluster setup scripts working |
| **2** | Core integration | `inference-perf` running standard IGW vs k8s-svc comparison |
| **3** | All configs | Prefix cache, decode heavy, prefill heavy, multi-LoRA scenarios |
| **4** | Regression + harness | LPG regression test, Go harness, Markdown/JSON reports |
| **5** | CI + storage | Nightly workflow, GCS/S3 upload, artifact storage |
| **6** | Polish | Full docs, final PR |

---

## 12. TODO Checklist

### Phase 1: Foundation (Week 1)
- [ ] Write `design/XXXX-inference-routing-benchmarks.md` following EP template
- [ ] Get design doc reviewed by maintainers
- [ ] Write `test/benchmark/setup/cluster-kind.sh`
- [ ] Write `test/benchmark/setup/install-kgateway.sh` (inference extension enabled)
- [ ] Write `test/benchmark/setup/install-inference-perf.sh`
- [ ] Create base Kubernetes manifests (Gateway, HTTPRoute, InferencePool, InferenceModel)

### Phase 2: Core Integration (Week 2)
- [ ] Deploy `llm-d-inference-sim` × 3 replicas with vLLM-compatible metrics
- [ ] Deploy EPP targeting `llm-d-sim` pods
- [ ] Write `inference-perf/config-standard.yaml` (IGW vs k8s-svc)
- [ ] Validate end-to-end: requests → Envoy → EPP → llm-d-sim
- [ ] Capture initial TPOT/ITL/TTFT output from `inference-perf`

### Phase 3: All 4 Configurations (Week 3)
- [ ] Write `inference-perf/config-prefix-cache.yaml`
- [ ] Write `inference-perf/config-decode-heavy.yaml`
- [ ] Write `inference-perf/config-prefill-heavy.yaml`
- [ ] Write `inference-perf/config-multilora.yaml` (2-4 LoRA adapters)
- [ ] Validate all 4 configs produce valid result JSON

### Phase 4: LPG + Harness (Week 4)
- [ ] Set up Latency Profile Generator (`lpg/config-regression.yaml`)
- [ ] Commit initial latency baseline profile (`lpg/profiles/baseline-profile.json`)
- [ ] Implement Go test harness (`setup.go`, `teardown.go`, `benchmark_test.go`)
- [ ] Implement results collector (`results.go`) — parse inference-perf JSON + Prometheus
- [ ] Implement Markdown + JSON report generator (`report.go`)
- [ ] Commit benchmark baseline (`results/baseline/benchmark-results.json`)

### Phase 5: CI + Storage (Week 5)
- [ ] Create `.github/workflows/benchmark-nightly.yaml`
- [ ] Create `.github/workflows/benchmark-regression.yaml` (PR-level LPG only)
- [ ] Implement GCS upload (`storage.go`) for nightly + release runs
- [ ] Add Makefile targets to root `Makefile`
- [ ] Test full nightly workflow end-to-end

### Phase 6: Documentation (Week 6)
- [ ] Write `test/benchmark/README.md`
- [ ] Write `test/benchmark/docs/methodology.md`
- [ ] Write `test/benchmark/docs/interpreting-results.md`
- [ ] Write `test/benchmark/docs/reproducing.md`
- [ ] Write `test/benchmark/docs/best-practices.md`
- [ ] Final PR review + merge

### Stretch Goals (Post-GSoC)
- [ ] **S1**: Provision B100/H100 80GB GPU env (GKE) with 8-10 vLLM replicas
- [ ] **S2**: Add `cluster-gke.sh` + `.github/workflows/benchmark-e2e-gke.yaml`
- [ ] **S3**: Publish benchmark results page on kgateway website
- [ ] **S4**: Gate releases on inference perf regressions
- [ ] **S5**: Cross-reference with llm-d benchmark results

---

> [!IMPORTANT]
> **Open questions for maintainers:**
> 1. Is there existing cloud budget / OSS GPU credits for running H100/B100 benchmarks (Stretch Goal S1)?
> 2. Should the design document follow the existing EP template at `design/`?
> 3. Is `llm-d-inference-sim` an acceptable dependency, or should we use the simpler GIE-provided vLLM stub?
> 4. For CI regression gating (Stretch Goal S4), what is an acceptable latency regression threshold (e.g., p99 TTFT increase of >10%)?
> 5. Where should long-term benchmark results be stored — GCS bucket maintained by the project, or GitHub Actions artifacts only?
