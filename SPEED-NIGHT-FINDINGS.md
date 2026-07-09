# GLM-5.2 Speed Night — Findings (2026-07-09)

**Question:** is GLM-5.2 decode on 4× DGX Spark at its physics limit (28.8 tok/s), or is there attackable overhead?
**Answer: NOT physics. 63% of every decode step is attackable overhead.** But most of it is unreachable with config flags — collecting it requires custom communication engineering.

---

## The headline measurement: step-time decomposition (k=0 test)

Running with speculation OFF makes tok/s = engine steps/sec directly:

| | value |
|---|---|
| k=0 single-stream | 14.9 tok/s → **67ms per raw engine step** |
| physics floor (weights @ 273GB/s, ~6GB/node/step) | **~25ms (37%)** |
| **overhead (NCCL latency, kernel launch, sync)** | **~42ms (63%)** |

Second finding from the same test: **MTP speculation is doing 2× of the work** (14.9 → ~29 tok/s). And comparing step times k=4 vs k=0 shows each drafter pass costs ~14ms — vs ~2ms of actual compute — so **even the drafter is overhead-bound.** Any per-step overhead reduction compounds across target + 4 draft passes.

**Ceiling math:** overhead fully eliminated → k=4 ≈ 50 tok/s theoretical. Realistic partial win → mid-30s.

## Experiment results (all vs baseline 26.5–31.1 c1 / 60.8 c6, k=4, 512-tok temp-0 prose)

| config | c1 | c6 agg | verdict |
|---|---|---|---|
| baseline (k=4, FLASHMLA 200K) | 26.5–31.1 | 60.8 | reference |
| k=0 (no spec) | 14.9 | 41.8 | decomposition probe — not a serving config |
| NCCL_PROTO=LL | 26.3–28.5 | 61.9 | **neutral** — NCCL already auto-selects LL for small messages |
| **fuse_gemm_comms** | 28.4–29.7 | **63.0** | **small real win on aggregate (+2 c6); c1 within noise. KEPT — it's free** |
| expert-parallel | — | — | **FLEET-KILLER: OOM'd all 4 nodes into swap-death; required physical power-cycle.** EP's MoE layout blows the <1GB-free memory budget. Do not retry without full memory retune (lower gmu, smaller KV). |
| k=5 | not run | — | skipped after the EP incident; theory says marginal (position-5 acceptance ~0.45 vs +1 draft pass/step). Optional future cycle. |

**Shipped config after the night: k=4 + fuse_gemm_comms** (`~/glm-5.2-gb10/speednight-fuse.sh`).

## The RDMA-allreduce verdict (the strategic question)

**BUILD-WORTHY — the data now justifies it.** Reasoning:
- The 42ms/step overhead is real and measured, not hypothesized.
- Config-level attacks are exhausted: NCCL flags neutral (auto-tuned already), fuse pass collects only ~1–2ms, EP structurally infeasible on this memory budget.
- The remaining overhead lives in per-call NCCL/RoCE latency across ~156 tiny allreduces per step. lukealonso's b12x proved the same attack works on PCIe (single-box); nobody has built the RoCE-fabric equivalent for Spark clusters.
- Prize: 5–10 tok/s single-stream (28.8 → mid-30s), compounding through the drafter. It would be the defining community contribution for every multi-Spark owner.
- Cost: weeks-class kernel/verbs engineering. Next step if pursued: profile with torch-profiler to get exact per-allreduce latency, then prototype a one-shot RC-verbs allreduce for the 24KB decode message size.

## Operational lessons
- **EP incident:** untested memory-layout flags on nodes running <1GB free can swap-wedge the entire fleet beyond SSH recovery. Rule: any experiment that changes weight/KV layout gets a reduced gmu first boot. (Cost: one fleet power-cycle, no data loss.)
- Staged monitors with 2.5-min pings (Tony's cadence preference) worked well — kept visibility through every cycle including the outage.

## Scoreboard vs community (for context)
Our 28.8–31 median remains the fastest known sustained single-stream for this stack on 4 nodes; Zatz 640K: 19.6–25.7 (peaks 33–37); Cosmic DCP2 328K: 20–36. Everyone's k=4 now. The differences between rigs are measurement-basis (prose vs synthetic) more than config.
