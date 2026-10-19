# ALERTS — Proposed alert rules (NOT created)

Per the task constraints, **no alert rules were created**. These are proposals with verified expressions, thresholds and reasoning. "Firing now?" is evaluated against live data at 2026-10-19 ~19:45Z.

Prometheus datasource UID: `cg08tpxzt30u8d` · Loki datasource UID: `ag1mgl00sdszkc`

---

## 1. Target down (scrape failure) — **critical**

- **Type:** Prometheus
- **Expr:** `up == 0`
- **For:** 5m · **Severity:** critical
- **Labels:** `{severity="critical"}`; annotations: `{{ $labels.job }}/{{ $labels.instance }} has been unscrapeable for >5m`
- **Reasoning:** The most basic availability guarantee. A dead exporter means an entire host is invisible — exactly what happened with SV-HV4, which sat dark ≥24h with no alert to say so.
- **Firing now?** **YES** — `windows_exporter` on `10.172.184.54:9182` (SV-HV4).

## 2. Blackbox probe failing — **warning → critical**

- **Type:** Prometheus
- **Expr:** `avg_over_time(probe_success[10m]) < 1`
- **For:** 10m · **Severity:** warning (escalate to critical if `< 0.5`)
- **Reasoning:** Catches intermittent target unreachability that a single failed scrape would miss. All 4 targets are currently 100% — this is pure headroom protection, e.g. for the router IP 10.254.2.2 or the external resolvers.
- **Firing now?** No (all probes succeeding).

## 3. Juniper Routing Engine CPU sustained high — **warning**

- **Type:** Prometheus
- **Expr:** `max by (instance) (jnxOperatingCPU{jnxOperatingDescr="Routing Engine 0"}) > 70`
- **For:** 15m · **Severity:** warning
- **Reasoning:** An idle L2 switch RE should sit in single digits. The Determinist was oscillating **41–88%** over 6h — sustained >70% for 15m is the line between "noisy polling" and "something is wrong on the box". The `for:` window avoids firing on momentary SNMP-poll spikes.
- **Firing now?** Likely, intermittently (values observed up to 88%; a 15m-sustained >70% window would have triggered during the high phases).

## 4. Host temperature high — **warning**

- **Type:** Prometheus
- **Expr:** `(max by (instance) (node_hwmon_temp_celsius) or max by (instance) (node_thermal_zone_temp)) > 80`
- **For:** 10m · **Severity:** warning
- **Reasoning:** gadfium was at **89–91°C with ~2% CPU load** — well above a safe idle band and a throttling risk under real workloads. `or` covers hosts that expose only thermal zones. 10m avoids transient spike noise from short bursts.
- **Firing now?** **YES** on gadfium (max sensor 89–91°C).

## 5. Disk usage high — **warning**

- **Type:** Prometheus
- **Expr:** `(node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes > 0.9` with `fstype=~"ext4|xfs|zfs"` (exclude tmp/overlay/squashfs via label filters)
- **For:** 30m · **Severity:** warning
- **Reasoning:** Headroom alert, not a current incident: gadfium `/home` is at **77%** and ZFS pools on salvor grow over time. 90% + 30m gives room for legitimate churn while catching the slow-fill failure mode before it becomes an outage.
- **Firing now?** No (highest observed ≈ 77–80%).

## 6. Memory pressure — **warning**

- **Type:** Prometheus
- **Expr:** `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1`
- **For:** 15m · **Severity:** warning
- **Reasoning:** gadfium runs at ~76% used; the desktop also hosts Docker workloads. Available-memory (not "free") is the right signal on Linux. 15m tolerates normal working-set swings.
- **Firing now?** No (~24% available).

## 7. WPA handshake timeout burst (router) — **warning** · LogQL

- **Type:** Loki
- **Expr:** `sum(count_over_time({host="router"} |~ "4Way-MSG[0-9]+ timeout" [30m])) > 5`
- **For:** 15m · **Severity:** warning
- **Reasoning:** A client that can't complete the WPA 4-way handshake keeps retrying and spams the log. Baseline is ~1–5/24h (client `c0:dd:8a:4e:1e:d7`); >5 in 30m means an actively flapping or misconfigured client worth identifying.
- **Firing now?** No (≈1 in the last 6h).

## 8. Router kernel error rate — **warning** · LogQL

- **Type:** Loki
- **Expr:** `sum(count_over_time({host="router", app="kernel", level="error"}[1h])) > 10`
- **For:** 30m · **Severity:** warning
- **Reasoning:** Kernel-level errors on this router are dominated by Ralink WiFi driver events (≈13 in 24h). A sustained >10/hour rate signals a hardware/driver problem, not the usual trickle. Scoped to `app="kernel"` so the acme.sh debug noise (rule #9 context) can't mask or trigger it.
- **Firing now?** No (~0.5/hour).

## 9. ACME renewal failure (router) — **warning** · LogQL

- **Type:** Loki
- **Expr:** `sum(count_over_time({host="router", app="run-acme"} |~ "(?i)(renewal.*fail|fail.*renew|unable to|certificate.*expired)" [24h])) > 0`
- **For:** 5m · **Severity:** warning
- **Reasoning:** The router renews `inet-router.local.retrop.co.uk` via acme.sh. **Caveat (verified):** acme.sh currently runs with `--debug` and its *normal* output is logged at error priority (~32 lines per renewal), so any alert on raw error count would be pure noise — this rule targets actual failure phrasing instead. Fix the log level first (FINDINGS.md improvement #5) and this becomes a clean "cert renewal broke" alert; consider also alerting on *absence* of a successful-renewal line every ~60d via Prometheus if acme.sh exposes one.
- **Firing now?** No (renewals succeeded in the observed window).

## 10. DNS lookup slow during probe — **warning**

- **Type:** Prometheus
- **Expr:** `probe_dns_lookup_time_seconds > 1`
- **For:** 5m · **Severity:** warning
- **Reasoning:** Resolver degradation detector. Values are currently sub-millisecond (targets are IP-only, see FINDINGS gap #6) — the rule is armed for when hostname targets are added, at which point a >1s lookup means the resolver is struggling.
- **Firing now?** No.

---

### Suggested contact-point routing (for when these are created)

- critical (#1): immediate push/phone
- warning (#2–#10): daily digest unless they repeat, then escalate after 2 consecutive fires

Nothing in this file has been applied to Grafana — creating the rules is a deliberate follow-up step.
