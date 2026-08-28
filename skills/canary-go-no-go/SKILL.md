---
description: Runbook for a canary rollout go/no-go decision in a Trillo Observability deployment — a new agent version (B) is running on some locations; decide whether to roll it out wider by comparing it to the current version (A) on normalized reliability, latency, cost, tokens, and quality. Use when the user asks "should we roll out version B" or "is the canary safe to expand".
---

# Runbook: canary go / no-go

Goal: decide **roll-out / hold / roll-back** for a new agent version (B) by
comparing it to the incumbent (A) over a specified window on **normalized** metrics,
then file the decision. Reason from **structure, not payload** (see
`sre:sre-overview`).

## The principle

Compare on **rate / per-execution** metrics, **never absolute totals** — a canary
B on a few locations takes far less traffic than A, so totals lie. Always evaluate
**per-version execution + location counts**, and treat a thin B sample as
**low-confidence**.

## Primary investigation flow (powered by `compareAgentVersions`)

1. **Identify versions & traffic:**
   - If the agent has multiple versions active, list available versions with `listVersionedAgents` (or use the versions specified by the user: A = current baseline, B = canary candidate).
   - Confirm version B has accumulated enough traffic across locations to form a statistically valid sample (e.g. >50 executions).
2. **Execute version comparison:**
   - Call `compareAgentVersions(agentId, versionA, versionB, window)` to obtain side-by-side normalized telemetry:
     - **Normalized Error Rate:** Has B introduced new error signatures or elevated failure rates?
     - **Latency Profile:** P50 / P95 / P99 comparison.
     - **Cost & Token Efficiency:** Cost per execution and input/output token density.
     - **Composite Quality:** Evaluation score delta with per-eval breakdown.
     - **Traffic & Location Distribution:** Execution counts and regional spread.
     - **Automated Rollout Recommendation:** Platform materiality assessment.
3. **Evaluate Quality-Aware Gate:**
   - Quality-aware constraint: cheaper or faster execution is **not** a pass if quality scores drop below acceptable thresholds.
4. **Draft Investigation Report & Verdict:**
   - Formulate the decision (**roll-out / hold / roll-back**):
     - **Roll-out:** B meets or beats A on quality, reliability, and cost/latency with adequate sample size.
     - **Hold:** B shows promise but sample size is thin, or non-critical anomalies need more soak time.
     - **Roll-back:** B shows statistically significant regressions in error rate, P95 latency, or quality.
   - Use `writeInvestigationReport` to publish the canary analysis to Trillo Observability.

## Fallback flow (per-version baseline comparison)

If comparing against an unversioned historical fleet or when `compareAgentVersions` data is unavailable:
1. Call `getAgentPerformanceBaseline(agentId, metric="error_rate")` and `getAgentPerformanceBaseline(agentId, metric="latency")`.
2. Check `getTopPlatformFindings` filtered to the canary window.
3. Draft report with explicit note that comparison relies on aggregate baseline deltas.
