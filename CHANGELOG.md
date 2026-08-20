# Changelog

## 0.4.0 — rename to the SRE plugin

- Renamed the plugin from `tao` to **`sre`** (audience-accurate: SRE / on-call
  engineers, not execs). Repo → `sre-claude-plugin`; MCP server key `tao` → `sre`;
  OAuth client `tao-claude-code` → `sre-claude-code`; env `TAO_MCP_URL` →
  `SRE_MCP_URL`; skill `tao-overview` → `sre-overview` (prefix `sre:`).
- Generalized wording from "Trillo Agent Observability (TAO)" to **Trillo
  Observability** — the plugin serves agent *and* infrastructure observability
  apps (login/app resolved per deployment by appName).

## 0.3.0 — user guide; internal docs relocated

- Added `docs/user-guide.md` — the human-facing SRE guide (what it is, install +
  auth, a worked first investigation, the runbook catalog, reading the report,
  troubleshooting, and an admin appendix).
- Moved the internal design docs out of this public repo into
  **trillo-observability**: the tool manifest (tool→function mapping, allow-list,
  tool gaps) and the app-team investigation-report write-path handoff. README and
  the `sre-overview` skill now point at the user guide; Claude discovers the live
  tool list from the MCP server at runtime.

## 0.2.0 — investigation runbooks

- Added runbooks: `cost-spike-investigation`, `latency-regression-drilldown`
  (grounded in the exposed tools, with tool-gap notes), `drift-confirmation`
  (partial until Feature E), `canary-go-no-go` (partial until Feature D).
- `sre-overview` lists all runbooks; `docs/tool-manifest.md` documents the tool
  gaps (deep cost/latency functions to expose) + feature dependencies.

## 0.1.0 — initial scaffold

- Plugin shell mirroring `trillo-claude-plugin`: `.mcp.json` (Trillo AOS MCP,
  `sre-claude-code` OAuth client), `.claude-plugin/plugin.json` +
  `marketplace.json`.
- `docs/tool-manifest.md` — curated investigation tool set mapped to the real Trillo Observability
  app functions (`invocationMode: agent_tool`), the structure-not-payload posture,
  and the single `write_investigation_report` (→ `AiAnalysis`).
- Skills: `sre-overview` (session guide) and `reliability-incident-triage` (first
  runbook).
