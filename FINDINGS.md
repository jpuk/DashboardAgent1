# FINDINGS — Overnight observability pass (2026-10-19, ~19:45Z)

All data gathered exclusively via the Grafana MCP tools (Prometheus `cg08tpxzt30u8d` @ 10.254.1.16, Loki `ag1mgl00sdszkc` @ 10.254.1.19). Full asset detail in [INVENTORY.md](INVENTORY.md); dashboards in the **"Agent Lab - Overnight"** Grafana folder (JSON copies in `dashboards/`).

## Coverage

| Area | Status |
|---|---|
| Prometheus targets | 11 total: **10 up, 1 down** (windows_exporter on SV-HV4) |
| Hosts (CPU/mem/disk/net/temp/GPU/ZFS) | ✅ salvor, gadfium via node_exporter — dashboard `agent-lab-hosts` |
| Switches/router interfaces (SNMP) | ✅ Juniper "Determinist" + MikroTik "PrimeMover" — dashboard `agent-lab-switches` |
| Network reachability (blackbox) | ✅ 4 targets (1.1.1.1, 8.8.8.8, 10.254.1.13, 10.254.2.2), all healthy — dashboard `agent-lab-reachability` |
| DNS / resolver health | ⚠️ Partial — no bind9 exists; built from blackbox DNS-lookup metrics + dnsmasq router logs — dashboard `agent-lab-dns` |
| Logs overview | ✅ Loki: journal (salvor, gadfium), syslog (router), containers (Docker VM) — dashboard `agent-lab-logs` |
| WiFi / router deep dive (creative) | ✅ Ralink WPA/PMKID/DHCP events from router syslog — dashboard `agent-lab-wifi` |

Six dashboards built, **every panel query verified against live data before upload**, template variables used throughout (no hard-coded hosts).

## Gaps in monitoring

1. **No container/VM metrics.** No cAdvisor or Proxmox exporter anywhere. The Docker VM and its containers are only visible via logs; "Hosts and Docker" is effectively hosts-only (documented on the dashboard).
2. **Container log streams have no `host` label** — they can't be attributed to a host, and the host variable in dashboards doesn't filter them.
3. **OpenWrt router: logs only.** No SNMP/metrics for router CPU, memory, disk or temperature.
4. **No bind9 anywhere.** The task assumed bind9; actual DNS is dnsmasq on the router plus blackbox probes. If a dedicated resolver was expected, it's not instrumented (or doesn't exist).
5. **SV-HV4 (10.172.184.54) is dark.** windows_exporter down ≥24h — that Windows Hyper-V host has zero current telemetry.
6. **Blackbox targets are IP-only**, so `probe_dns_lookup_time_seconds` measures trivially-fast resolution and does **not** exercise the local resolver with real hostnames. Switch/router management IPs (10.254.2.1 / 10.254.2.3) are not directly probed — only SNMP scrape up/down covers them.

## What is wrong right now

1. **windows_exporter on SV-HV4 down ≥24h** (`up == 0` since before this pass started). Either the VM/host is off or the exporter service died.
2. **Juniper "Determinist" Routing Engine CPU oscillating 41–88%** over 6h (saw values like 41%, 88% within minutes). An idle L2 switch RE should sit near single digits — sustained load suggests heavy SNMP polling, a runaway process, or hardware. Worth a look on the console.
3. **gadfium running hot: max temp 89–91°C at ~2% CPU load** (node_hwmon/thermal zones). Idle-adjacent temperatures that high are a throttling/lifespan risk — check cooling/fan curve. Also memory 76% used and /home at 77%.
4. **Flaky WiFi client `c0:dd:8a:4e:1e:d7`** — repeated `4Way-MSG1 timeout` events (WPA handshake failures) on the OpenWrt router, including a burst of ~5 within minutes around 13:41Z. Identify the device; re-provision credentials or update its firmware.
5. **gadfium amdxdna NPU failing to open**: recurring `SVA bind device failed, ret -19` (plus a `logitech-hidpp` error 0x08). Driver/kernel mismatch on the AMD XDNA stack — either update or blacklist the module if the NPU isn't used.

Healthy: all switch port error/discard rates are **zero**; Juniper red/yellow alarms off; ZFS ARC hit ratio high (salvor); blackbox probes 100% success; router DHCP/DNS serving normally (3 DHCPOFFERs in 6h).

## Notable Loki error patterns (last 24h, exact counts)

| Host | Error lines | What they are |
|---|---|---|
| **router** | **45** | ~32 are **acme.sh debug output tagged at error priority** (one `--renew -d inet-router.local.retrop.co.uk --debug` burst — normal renewal chatter, not a failure); ~13 genuine Ralink WiFi driver events: 4Way-MSG1 timeouts (client c0:dd:8a:4e:1e:d7), `IE_WLAN_EXTENSION: no handler for extension_id:137` (a client offering an unknown vendor IE), PMKID cache deletes incl. the `pStaCfg is null` driver-bug signature |
| **gadfium** | **22** | amdxdna NPU `SVA bind device failed, ret -19` (recurring) + logitech-hidpp 0x08 |
| **salvor** | **6** | Low-volume kernel noise |
| containers | **0** | — |

Two labeling problems inflate the "error" signal:
- **acme.sh runs with `--debug` and its stdout lands at error priority** → every renewal spams ~32 fake errors. Remove `--debug` (or raise the log level) so real ACME failures stand out.
- The **Ralink driver logs routine WPA association events at warning/error priority**, which inflates the warning series on the Logs Overview dashboard (noted on both dashboards).

## Top-5 improvements

1. **Fix/restore SV-HV4 monitoring** — bring windows_exporter back and add the `up == 0` alert (ALERTS.md #1) so a dark host pages instead of sitting unnoticed for 24h+.
2. **Add container/VM metrics + label the streams** — cAdvisor on the Docker VM (and/or the Proxmox exporter on salvor), and add the `host` label to the container log stream in Alloy. This closes the biggest blind spot: the machine running the whole monitoring stack is itself unmonitored for resource use.
3. **Investigate Juniper RE CPU 41–88%** — check SNMP poll frequency vs. switch capability, NTP, and `show system processes` on the console; add the sustained-CPU alert (ALERTS.md #3).
4. **Cool gadfium down** — 89–91°C at ~2% load is abnormal; inspect fan curve/airflow before it throttles under real load. Also resolve or silence the amdxdna NPU failures so they stop generating 22 error lines/day.
5. **Fix the router log hygiene** — drop `--debug` from acme.sh (32 fake errors per renewal) and deal with client `c0:dd:8a:4e:1e:d7` (WPA handshake timeouts). Then the LogQL alerts in ALERTS.md #7–#9 become trustworthy.

Bonus: add at least one **hostname** to the blackbox targets so DNS resolution is actually tested end-to-end, and consider probing 10.254.2.1/10.254.2.3 directly.
