# PROGRESS — Overnight observability pass (2026-10-19)

Phase: **Phase 2 — Dashboards** (discovery done, INVENTORY.md written)

## Done
- [x] Phase 1 discovery via MCP only. Key facts in INVENTORY.md:
  - Datasources: prometheus `cg08tpxzt30u8d` (10.254.1.16), loki `ag1mgl00sdszkc` (10.254.1.19). Both healthy.
  - Targets: 11 up, 1 down (windows_exporter SV-HV4 10.172.184.54:9182, down ≥24h).
  - Devices: Juniper EX2300-C-12P "Determinist" (10.254.2.1), MikroTik "PrimeMover" (10.254.2.3), Proxmox "salvor", Ubuntu desktop "gadfium" (AMD GPU + XDNA NPU), OpenWrt "router" (logs only), Docker VM (container logs, no host label, no metrics).
  - Loki: hosts salvor/gadfium/router + unlabeled containers; ~245k lines/24h total; 76 error-level lines.
- [x] INVENTORY.md written.

## Decisions
- Blackbox has exactly 4 targets (1.1.1.1, 8.8.8.8, 10.254.1.13, 10.254.2.2), all ICMP+DNS-lookup, all healthy. (Earlier misread of `up` output — 10.254.2.1 belongs to the juniper job.)
- **bind9 dashboard: no bind9/named metrics or logs exist** → will build "DNS & Resolver Health" from blackbox DNS-lookup metrics + dnsmasq router logs instead; documenting the gap rather than shipping empty panels.
- **Docker part of "Hosts and Docker": no cAdvisor/container metrics in Prometheus** → dashboard covers hosts (CPU/GPU/mem/disk/net/ZFS); container section skipped with note.
- Loki MCP tool flattens grouped metric results into one unlabeled stream → for per-group values during verification, run scalar queries per group with narrow time ranges. Dashboards themselves render fine in Grafana.
- Dashboard plan (all verified-query-first):
  1. `network-reachability` — blackbox success/latency/DNS lookup/hop limit, target variable.
  2. `switch-router-interfaces` — SNMP traffic/errors/up-down per device+port, device & interface variables, device health (RE CPU, temp, memory), DOM optics.
  3. `hosts-docker` — CPU, load, memory, disk usage/IO, network, GPU (gadfium), temps, ZFS ARC (salvor), uptime; host variable.
  4. `dns-resolver-health` — probe_dns_lookup_time per target + dnsmasq activity/errors from router logs.
  5. `logs-overview` — volume by host/app, level distribution, error rate, top error lines, log explorer.
  6. `wifi-router-deep-dive` (creative) — WPA handshake timeouts per client MAC, DHCP churn, kernel error rate on the OpenWrt router; all from Loki.

## Problems / Blocked
- (none yet)

## Next action
1. search_folders for "Agent Lab - Overnight" → create if missing.
2. Verify dashboard-1 queries (probe_success/duration/dns/hop-limit per target) via MCP, build `dashboards/network-reachability.json`, upload with update_dashboard, commit.
