---
description: Runbook for investigating a latency regression or slowness in a Trillo Observability deployment. Walks from "it got slow" to the bottleneck component (model / tool / retrieval / orchestration) and whether it's a regression vs. baseline. Use when the user reports high latency, a P95/P99 spike, or a slow agent.
---

# Runbook: latency-regression drill-down

Goal: locate the **latency bottleneck** and decide whether it's a **regression**
(a step change vs. baseline) or steady-state, then file a report. Reason from
**structure, not payload** (see `sre:sre-overview`).

## Step 1 — Find the latency finding

1. `getTopPlatformFindings` filtered to `findingType = latency`, `status = open`, time
   window → the latency findings/spikes.
2. Pick the one matching the user's report; note `findingId` and the affected
   agent/app.

## Step 2 — Is it a regression?

3. `getAgentPerformanceBaseline(agentId, metric = latency)` (P95/P99) → is the current
   value a **step change** vs. baseline? A step aligned to an **agent version
   boundary** points at a change in the agent (prompt/model/logic).

## Step 3 — Locate the bottleneck

4. Take a representative slow execution → `getExecutionDetails(executionId)` → the
   **span skeleton**; identify which span **category** dominates the duration:
   `MODEL` (LLM call), `TOOL` (external call), `RETRIEVAL` (vector store), or
   orchestration/other.
5. If a **TOOL/RETRIEVAL** span dominates → `getDependencyTopology` /
   `getAgentDependencyTree(agentId)` to name the slow tool → external system
   and see who else depends on it (shared-dependency blast radius).
6. `getCorrelatedLogsAndEvents(traceId|executionId)` → confirm timeouts/retries on the
   slow dependency.

## Step 4 — Conclude & file

7. Synthesize: **where the latency is** (which category/dependency), **regression
   or not** (baseline delta, version alignment), **impacted agents** if it's a
   shared dependency, **recommended action**, **confidence**. Facts vs inference.
8. Draft and `writeInvestigationReport` (reference the `findingId`); confirm
   before filing.

## Tool gaps (raise with the app team if deeper drill is needed)

The detailed **latency-breakdown** and **regression-analysis** functions are not
currently exposed as `agent_tool` (they run as scheduled sweepers): `analyzeLatency`
(the network/model/tool/orchestration decomposition + percentile trends) and
`analyzePerformanceRegression`. This runbook derives the bottleneck from the
execution **span skeleton** + baseline instead. To get the platform's own 4-bucket
breakdown / regression verdict inline, ask the app team to expose those functions
as `agent_tool`.
