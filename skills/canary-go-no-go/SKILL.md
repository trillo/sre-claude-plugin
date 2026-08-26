---
description: Runbook for a canary rollout go/no-go decision in a Trillo Observability deployment — a new agent version (B) is running on some locations; decide whether to roll it out wider by comparing it to the current version (A) on normalized reliability, latency, cost, tokens, and quality. Use when the user asks "should we roll out version B" or "is the canary safe to expand".
---

# Runbook: canary go / no-go

Goal: decide **roll-out / hold / roll-back** for a new agent version (B) by
comparing it to the incumbent (A) over a short window on **normalized** metrics,
then file the decision. Reason from **structure, not payload** (see
`sre:sre-overview`).

> **Feature dependency (read first):** version A/B comparison is **Feature D**
> (Agent A/B / Version Comparison) — backing function `compareAgentVersions`.
> Until it is exposed as an agent tool, run the **partial** flow below (per-version baselines).
> When Feature D lands, this becomes a single `compareAgentVersions` call + verdict.

## The principle (applies either way)

Compare on **rate / per-execution** metrics, **never absolute totals** — a canary
B on a few locations takes far less traffic than A, so totals lie. Always read the
**per-version execution + location counts**, and treat a thin B sample as
**low-confidence**.

## Partial flow (available now)

1. Identify the two versions (A = current, B = canary) and the window (e.g. last
   24h). Confirm B's execution/location count is enough to be meaningful.
2. For each version, `getAgentPerformanceBaseline(agentId, metric = error_rate | latency)`
   and read any per-version `getTopPlatformFindings` (reliability/cost/latency) — did B
   introduce a **new failure signature** vs A?
3. Compare the **normalized** values you can get (error rate, latency). Flag that
   **cost/exec, tokens/exec, and composite quality** need Feature D for a proper
   side-by-side; state this limitation.
4. Draft `writeInvestigationReport` with a **provisional** verdict
   (roll-out / hold / roll-back) + the caveat that full normalized + quality
   comparison awaits Feature D. Low confidence if B's sample is thin.

## Full flow (when Feature D ships)

1. `compareAgentVersions(agentId, versionA, versionB, window)` → normalized error
   rate, P95, cost/exec, tokens/exec, and **composite quality** (with per-eval
   breakdown), plus per-version traffic + location counts and a materiality-based
   **rollout verdict** ("B 22% cheaper at equal quality → candidate to roll out").
2. Read the quality breakdown — quality-aware: cheaper-but-worse is **not** a pass.
3. File the report with the recorded decision (roll-out / hold / roll-back).

See `trillo-observability` Scheduled Feature Specs (Feature D) for the design
(normalized metrics, composite quality + breakdown, tunable materiality thresholds).
