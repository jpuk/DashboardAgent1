# Project: Overnight observability pass on John's homelab

You are working alone and unattended overnight. Nobody will answer questions.
Make reasonable decisions, record them in PROGRESS.md, and keep going.

## Tools
- Your Grafana tools are named `mcp__MCP_DOCKER__*`. They connect a few seconds after the session starts. If you don't see them, call `WaitForMcpServers` once, then use them.
- Never look for Grafana by other routes (curl, hostnames, hosts files, config files). The MCP tools are the only way in.

## Context
The homelab runs a Docker Compose monitoring stack: Grafana, Prometheus, Loki
(logs via Grafana Alloy), blackbox_exporter and snmp_exporter. The network
includes Proxmox (host "Salvor"), Linux hosts, Juniper and MikroTik switches,
OpenWrt routers, a bind9 DNS server, and a Docker VM. You have Grafana MCP
tools to inspect datasources, metrics, labels and logs, and to create dashboards.

## Hard rules — never break these
- Create and update dashboards ONLY inside a Grafana folder named
  "Agent Lab - Overnight". Create the folder if missing.
- Never modify or delete any dashboard, folder, datasource, alert rule,
  silence, incident, annotation or plugin outside that folder.
- Never create alert rules, silences or incidents. Propose alerts in
  ALERTS.md instead.
- Keep Prometheus queries reasonable: ranges of 24h or less, no unbounded
  regex across all series.
- Work only inside this directory on disk.

## Phase 1 — Discovery (write INVENTORY.md)
List datasources and check their health. For Prometheus: every job and
instance, which exporters are present, key metric families per job. For Loki:
label names and values, which hosts/services ship logs. Note anything that
looks broken: targets down, stale series, datasources failing health checks.
- Call `list_datasources` with no filter first, and record every datasource's name, type and uid.
- Always pass the uid of a datasource of the matching type: the Loki uid to Loki tools, the Prometheus uid to Prometheus tools.

## Phase 2 — Dashboards
Build what the data actually supports, in this priority order. Skip any that
has no data and say why in PROGRESS.md.
1. Network reachability — blackbox probe success, latency and DNS lookups
   per target, with a target variable, other useful metrics.
2. Switch and router interfaces — SNMP traffic, errors and up/down per
   device and port, with device and interface variables
3. Hosts and Docker — CPU, GPU, memory, disk and network per instance
4. DNS — bind9 health from metrics and/or logs
5. Logs overview — log volume by host/service, top error patterns from Loki
6. Be creative and details and inovate useful new ideas

For every panel: run its query through the MCP tools first and confirm it
returns data. Never ship a panel you haven't verified. Use template
variables rather than hard-coded hosts. Save each dashboard's JSON to
`dashboards/<name>.json` as well as to Grafana.
- Panels for rare events (errors, warnings, security events) can be legitimately empty. Never broaden a filter just to make a panel return data. An empty panel because nothing went wrong is correct; fix only syntax errors and wrong label or metric names.

To create a dashboard, call the `update_dashboard` tool. It creates a new
dashboard when the JSON has no existing uid. Pass the dashboard JSON, the
folder UID of "Agent Lab - Overnight" (from `create_folder` or
`search_folders`), and overwrite: true for later updates. Do not write
dashboards only to disk. Saving to `dashboards/` is in addition to uploading.

## Phase 3 — Report
Write FINDINGS.md: what the stack covers, gaps in coverage (hosts or devices
with no metrics or logs), anything that looks wrong right now, notable error
patterns from Loki, and the top 5 improvements you'd make. Write ALERTS.md
with proposed alert rules (PromQL/LogQL, threshold, reasoning). Do not
create the alerts themselves.

## Working method
- Read PROGRESS.md first and continue from it. Keep it current: phase, what's
  done, decisions, problems, and the exact next action. Assume your memory
  will be wiped at any moment.
- Commit to git after each dashboard and each document.
- If something fails three times, log it under "Blocked" and move on.

## Definition of done
INVENTORY.md, FINDINGS.md and ALERTS.md written, every dashboard that the data
supports is built and verified, everything committed. Then write
"STATUS: COMPLETE" as the first line of PROGRESS.md and stop.
