---
description: Runbook for confirming behavioral drift in a Trillo Observability deployment — a slow-moving degradation over time (rising hallucination/eval-fail rate, latency creep, declining quality) that no single trace makes obvious. Use when the user suspects an agent is "getting worse over time" or a model silently changed.
---

# Runbook: drift confirmation

Goal: confirm (or rule out) **gradual drift** in an agent's behavior over time and
attribute it — a silent **provider** model update vs. **input/data** shift — then
file a report. Reason from **structure, not payload** (see `sre:sre-overview`).

> **Feature dependency (read first):** continuous drift detection is **Feature E**
> (Behavioral Drift Detection). SRE read investigations must query existing drift findings
> via `getTopPlatformFindings(findingType = 'drift')` or `getImpactedAgentFindings` — **never call the `detectBehavioralDrift` write sweeper directly**.
> Until Feature E scheduled sweepers are fully active, run the **partial** flow below (manual baseline comparison).

## Partial flow (available now)

1. `getTopPlatformFindings` filtered to `findingType = reliability` or
   `tokenEfficiency` over a **longer window** → any slow trends the sweepers
   surfaced.
2. `getAgentPerformanceBaseline(agentId, metric = <error_rate | latency | tokens>)` →
   compare the **recent** value to the **baseline**. A steady worsening (not a
   single spike) is the drift signal.
3. **Cause hint:** if the degradation **steps at a model boundary** (the agent's
   `response_model` changed), suspect a **provider update**; a gradual slope
   without a model change suggests **input/data drift**.
4. Draft and `writeInvestigationReport`: state it is a **manual/partial** drift
   check (Feature E not yet available), the metric + baseline delta, the suspected
   cause, and recommend enabling Feature E for continuous detection. Confidence
   should reflect the partial evidence.

## Full flow (when Feature E sweepers are active)

1. `getTopPlatformFindings` (filtered to `findingType = 'drift'`) or `getImpactedAgentFindings`
   (scope: agent / agent+version / model / tool) → ranked DRIFT
   findings with the drifting metric, magnitude, direction, window.
2. Open the drift detail: recent-vs-baseline distribution/trend; provider-vs-input
   attribution.
3. `getAgentPerformanceBaseline` to corroborate; representative executions across the
   window for evidence via `getExecutionDetails`.
4. File the report referencing the DRIFT `findingId` using `writeInvestigationReport`.

See `trillo-observability` Scheduled Feature Specs (Feature E) for the design.
