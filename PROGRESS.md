# PROGRESS — Overnight observability pass (2026-10-19)

STATUS: COMPLETE

Phase: **Done** — all three phases finished; final commit made. If a future session reads this: the build is complete, see IMPROVE.md for how to make one small verified improvement per session.

## Done
- [x] Phase 1 discovery (MCP only) → INVENTORY.md committed (2ffb7a3).
- [x] Phase 2 — six dashboards in folder "Agent Lab - Overnight" (`ag1pi52uocn40c`), every panel query verified via MCP before upload; JSON saved to `dashboards/`:
  1. `agent-lab-reachability` — blackbox success/latency/DNS/hop-limit, target var (2ffb7a3)
  2. `agent-lab-switches` — SNMP ifTable + device health, Juniper & MikroTik, device/interface vars (b8784e1)
  3. `agent-lab-hosts` — CPU/mem/disk/net/GPU/temp/ZFS per host; Docker part absent by design (no cAdvisor) (c4a99d1)
  4. `agent-lab-dns` — blackbox DNS-lookup metrics + dnsmasq router logs; bind9 gap documented (7d95905)
  5. `agent-lab-logs` — volume by host/app, level distribution, top error sources, log viewer (3330a6a)
  6. `agent-lab-wifi` (creative) — Ralink WPA timeouts / PMKID / unknown-IE / DHCP from router syslog (1dfb885)
- [x] Phase 3 — FINDINGS.md + ALERTS.md written and committed.

## Decisions
- Blackbox has exactly 4 targets (1.1.1.1, 8.8.8.8, 10.254.1.13, 10.254.2.2), all healthy.
- No bind9/named in the stack → DNS dashboard built from probe_dns_lookup_time + dnsmasq logs; gap documented (FINDINGS #4).
- No cAdvisor/container metrics → hosts dashboard is host-only; gap documented (FINDINGS #1).
- Loki MCP tool flattens grouped metric results into unlabeled streams and once returned a wrong grouped value (per-MAC WPA count 135 vs ground truth 1) → verification uses scalar queries + raw-line cross-checks; the WiFi dashboard deliberately ships no per-MAC aggregation panel, only verified queries.
- Dashboard JSON uploads: keep payloads ≤~9KB or split into create + `update_dashboard` patch ops (one upload truncated mid-panel and needed a `$.panels/-` append fix).

## Problems / Blocked
- None unresolved. (Transient MCP gateway timeouts early in the pass recovered on retry; not 3-strike territory.)

## Key findings (full detail in FINDINGS.md)
- SV-HV4 windows_exporter down ≥24h (would fire alert #1 today).
- Juniper Determinist RE CPU 41–88% sustained-ish — abnormal for idle L2.
- gadfium 89–91°C at ~2% load; /home 77%; amdxdna NPU `SVA bind device failed, ret -19` recurring (22 err lines/24h).
- Router: flaky WiFi client c0:dd:8a:4e:1e:d7 (4Way-MSG1 timeouts); acme.sh `--debug` spamming ~32 fake error-level lines per renewal.
- 24h error split (exact): router 45, gadfium 22, salvor 6, containers 0.

## Next action (for a future session)
Per IMPROVE.md: pick ONE small improvement, verify it, commit. Candidates ranked in FINDINGS.md top-5 (host label on container streams is the cheapest; cAdvisor is the highest value).
