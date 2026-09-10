# Trillo SRE Copilot (Claude Code plugin)

Investigate a **Trillo Observability** deployment without leaving Claude Code.
Install the plugin, sign in to your Trillo AOS account, and Claude becomes an
**SRE investigation copilot**: "why did fleet X degrade at 2am?" becomes a
conversation.

Trillo Observability applications discover, monitor, govern, and optimize a fleet
— of AI agents or AI infrastructure — from its OpenTelemetry data. This plugin
exposes the deployment's **investigation tools** and ships the **runbooks** that
teach Claude how to investigate the fleet.

## What it does

The plugin registers one MCP server (**`sre`**, on Trillo AOS) and ships
investigation skills. From a Claude Code session you can:

- **Triage incidents** — find findings, read a failure cluster's blast-radius /
  spread, classify **code vs deployment vs dependency**, see impacted agents.
- **Drill the evidence** — execution span skeletons, correlated logs, dependency
  topology, baselines, location + executive health.
- **File a report** — one write: `writeInvestigationReport`, which appears in the
  Trillo Observability product UI alongside the platform's own agent analyses.

It is **read-only against the fleet** (the single exception is the investigation
report) and follows a **structure-not-payload** rule — tools return incident
*shape* (findings, clusters, topology, span skeletons), never raw prompts or model
outputs — so nothing sensitive leaves your boundary.

The `sre:sre-overview` skill walks Claude through the flow; see the
**[User Guide](docs/user-guide.md)** for the full walkthrough.

## Documentation

- **[User Guide](docs/user-guide.md)** — install & authenticate, a worked first
  investigation, the runbook catalog, reading the report, and troubleshooting
  (plus an admin appendix for enabling the plugin).
- **Runbooks** — the investigation skills under [`skills/`](skills/):
  `reliability-incident-triage`, `cost-spike-investigation`,
  `latency-regression-drilldown`, `drift-confirmation`, `canary-go-no-go`.

## Relationship to the authoring plugin

`trillo-claude-plugin` is for **authoring** Trillo AOS apps (Trillo AI / Designer,
read-write). This plugin is for **investigating** a deployed Trillo Observability app (Trillo AOS
runtime, read-only + report). Different audience, different tools; installed
independently.

## Prerequisites

- A **Trillo AOS account** with access to the Trillo Observability deployment, and an RBAC role
  (admin / auditor / user / owner) that scopes what you can see.
- Claude Code (recent version with plugin support).

## Install

Install the plugin (runbooks/skills), then register the `sre` MCP server for your
observability app — the app rides the OAuth **client id** (`sre-claude-code-<appName>`),
registered via the CLI so your shell fills in the name:

```bash
claude plugin marketplace add trillo/sre-claude-plugin
claude plugin install sre

export SRE_APP_NAME="Neoclouds_Observability"                 # your app's name
export SRE_MCP_URL="https://aos.trillo.ai/api/v2.0/mcp"       # or your deployment's URL
claude mcp add --transport http \
  --client-id "sre-claude-code-$SRE_APP_NAME" \
  --callback-port 8090 \
  sre "$SRE_MCP_URL"
```

Then authenticate: `/mcp` → **sre** → authenticate.

**Why the CLI and not the plugin's `.mcp.json`:** Claude Code does not expand
`${VAR}` inside the `oauth` block of `.mcp.json`, so the target app can't be set by
an env var there. Registering via the CLI lets your shell expand it into a literal
client id (a public identifier, not a secret). See the
[User Guide](docs/user-guide.md) §4.

## Configuration

- `SRE_APP_NAME` — the Trillo Observability app to inspect; forms the client id
  `sre-claude-code-$SRE_APP_NAME` (app names differ per customer/version).
- `SRE_MCP_URL` — the Trillo AOS MCP endpoint (e.g. `https://aos.trillo.ai/api/v2.0/mcp`,
  or your dev / self-hosted URL).

## Status

v0.5.0. The tool surface and all six skills (`sre-overview` + five runbooks) are
aligned directly with the live Trillo Observability server functions (`camelCase`). Two runbooks (`drift-confirmation`,
`canary-go-no-go`) run a partial flow until their platform features ship; the
report write-path lights up once the Trillo Observability app implements it.
