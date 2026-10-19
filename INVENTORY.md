# INVENTORY.md — Homelab Observability Inventory

Generated 2026-10-19 ~15:55 UTC via Grafana MCP tools only.
Datasources used: Prometheus uid `cg08tpxzt30u8d`, Loki uid `ag1mgl00sdszkc`.

## Datasources

| name | type | uid | url | access | health |
|---|---|---|---|---|---|
| prometheus | prometheus | `cg08tpxzt30u8d` | http://10.254.1.16:9090 | proxy | OK — instant + range queries responding |
| loki | loki | `ag1mgl00sdszkc` | http://10.254.1.19:3100 | proxy | OK — log + metric queries responding |

Both healthy at time of check (no health-check API exposed via MCP; verified by live queries).

## Prometheus targets (instant `up`, 2026-10-19 ~15:51 UTC)

| job | instance | up | identity / notes |
|---|---|---|---|
| prometheus | localhost:9090 | 1 | self-scrape |
| snmp | snmp-exporter:9116 | 1 | exporter self-scrape only (device data arrives via `juniper`/`mikrotik` jobs through the exporter) |
| blackbox | 1.1.1.1 | 1 | Cloudflare DNS — ICMP + DNS-lookup module |
| blackbox | 8.8.8.8 | 1 | Google DNS — ICMP + DNS-lookup module |
| blackbox | 10.254.1.13 | 1 | internal, monitoring subnet (10.254.1.x); IP only, no hostname label |
| blackbox | 10.254.2.2 | 1 | internal, switch/office subnet; IP only, identity unknown from metrics |
| juniper | 10.254.2.1 | 1 | **EX2300-C-12P** "Determinist.local.retrop.co.uk" — 12×GbE PoE + 2×10G SFP |
| mikrotik | 10.254.2.3 | 1 | **"PrimeMover"** (sysName) |
| node | salvor | 1 | Proxmox VE host "Salvor", kernel 7.0.14-20-pve, ZFS pool present |
| node | gadfium | 1 | Ubuntu 24.04 desktop (KDE/Plasma), kernel 6.17.0-35-generic, AMD GPU (card1), AMD XDNA NPU |
| windows_exporter | 10.172.184.54:9182 | **0** | **SV-HV4** (labels: role=Hyper-V, site=Secondary) — DOWN for ≥24h (every sample in the last-24h window is 0) |

## Metric families per job

- **blackbox**: `probe_success`, `probe_duration_seconds`, `probe_icmp_duration_seconds`, `probe_dns_lookup_time_seconds`, `probe_icmp_reply_hop_limit`, `probe_ip_protocol` (all targets IP protocol 4 = IPv4/ICMP), `probe_ip_addr_hash`.
- **node** (both hosts, full node_exporter): CPU (`node_cpu_seconds_total`, load1/5/15, scaling freq, PSI pressure), memory (full `node_memory_*` incl. GPUActive/GPUReclaim), filesystem (`node_filesystem_size/free/avail_bytes`, `_info`, readonly, device_error), disk I/O (`node_disk_read/written_bytes_total`, io_time), network (`node_network_receive/transmit_bytes_total`, packets, drops, errors, carrier changes, `node_network_info`), **GPU** (`node_drm_card_info`, `node_drm_gpu_busy_percent`, `node_drm_memory_vram_used/size_bytes` — gadfium card1/amd only), hwmon (temps, voltages, power W), thermal zones, **ZFS ARC** (`node_zfs_arc_*` — salvor, c_max ≈ 3.35 GiB), TCP/UDP netstat (retransmits, CurrEstab, ListenDrops…), conntrack, processes, time/timex, ZFS + XFS counters, DMI/OS info.
- **juniper**: full ifTable (`ifHCInOctets`/`ifHCOutOctets`, `ifOperStatus`, `ifAdminStatus`, `ifInErrors`/`ifOutErrors`, `ifInDiscards`/`ifOutDiscards`, `ifAlias`, `ifHighSpeed`, `ifMtu`, `ifPhysAddress`) + **jnxOperating\*** (CPU, Memory, Temp, UpTime, red/yellow alarm state+count, DOM optics: `jnxDomCurrentRxLaserPower`, `jnxDomCurrentTxLaserOutputPower`, `jnxDomCurrentAlarms`, `jnxDomCurrentWarnings`) + sysDescr/sysUpTime.
- **mikrotik**: full ifTable (same family) + `hrProcessorLoad`, `hrStorage*` + **mtxr\*** (firmware version, interface stats incl. per-size RX buckets and driver rx/tx bytes/packets, link downs) + sysUpTime.
- **snmp**: exporter self-metrics only (`go_*`, `process_*`, `snmp_scrape_*`, `snmp_collection_duration_seconds`).
- **windows_exporter**: none currently (target down).

### Juniper ifAlias values (port names)
`Diziet`, `Macbook dock Eth`, `Prime Mover LACP member 1`, `Prime Mover LACP member 2`, `PrimeMover bond-determinist`, `Salvor Eth1`, `Salvor Eth2`, `WIFI-AP`

### MikroTik ifAlias values
`Bonded 10g link to Salvor`, `Bonded link to Determinist`, `office L2`

## Loki

Labels present: `app`, `facility`, `filename`, `host`, `job`, `level`, `service_name`, `unit` (+ derived `detected_level`).

| job (source) | host label | what ships |
|---|---|---|
| journal | salvor | Proxmox units: pvedaemon, pveproxy, pvescheduler, pvestatd, pveupdate, pvefw-logger, pveupdate, alloy-wrapper (Grafana Alloy itself), CRON, postfix/*, NetworkManager, systemd… |
| journal | gadfium | KDE/Plasma (kwin_x11, plasmashelf, ksmserver…), ollama, rustdesk, warp-svc, smartd, snapd, PackageKit, sshd… |
| syslog | router | OpenWrt: dnsmasq-dhcp (DHCP leases), kernel (WiFi/WPA — Ralink driver), dropbear/sshd, acme/run-acme, logread, chronyd |
| containers | *(no host label)* | Docker VM container stdout via Alloy file collector; 8 distinct container log files in last 24h (Grafana identifiable from content; others only by container-ID hash) |

### 24h log volume (count_over_time, sampled 15:53 UTC)
- salvor: **989** lines
- gadfium: **~34,298** lines
- router: **419** lines
- containers (unlabeled Docker VM): **~208,966** lines — dominant share

Levels seen: `debug`, `info`, `notice`, `warning`, `error`, `informational`. Error-level total last 24h: **76**.

## Topology (derived from ifAlias + sysName)

```
Salvor (Proxmox, ZFS) ──Eth1/Eth2──► PrimeMover (MikroTik, "Bonded 10g link to Salvor")
                                          │ "Bonded link to Determinist" / LACP member 1+2
                                          ▼
                              Determinist (Juniper EX2300-C-12P)
                              ports: Diziet, Macbook dock Eth, WIFI-AP
PrimeMover also has an "office L2" port.
Monitoring stack (Grafana/Prometheus/Loki/Alloy/blackbox/snmp exporters) runs on a Docker VM in 10.254.1.x (Prometheus at .16, Loki at .19; blackbox targets .13 and 10.254.2.2).
```

## Issues spotted during discovery

1. **windows_exporter SV-HV4 down ≥24h** — only target with up=0 (secondary Hyper-V host, site "Secondary"). No Windows metrics at all while down.
2. **Juniper EX2300 Routing Engine CPU sustained 41–88%** over last 6h (oscillating, never settles below ~40%). Unusual for an idle L2 switch; suggests churn (MAC/LACP activity or something on the RE). Needs attention.
3. **Router WiFi problems** (kernel/Ralink driver, repeated): `WpaEAPOLRetryAction() 4Way-MSG1 timeout with c0:dd:8a:4e:1e:d7` (one client repeatedly failing handshake), `IE_WLAN_EXTENSION: no handler for extension_id:137`, PMKID-cache errors (`pStaCfg is null`).
4. **gadfium kernel errors**: `amdxdna 0000:c7:00.1: [drm] *ERROR* amdxdna_drm_open: SVA bind device failed, ret -19` recurring (AMD Ryzen AI NPU driver failing to open), plus occasional `logitech-hidpp-device … protocol error 0x08`.
5. **Docker VM has no metrics** — not in node job; its container logs carry no `host` label, so they can't be attributed by host in Loki.
6. **No bind9 data anywhere** — no bind9/named metrics in Prometheus and no named/bind9 app in 24h of Loki. DNS observability is limited to dnsmasq (router) logs + blackbox `probe_dns_lookup_time_seconds`.
7. **OpenWrt router has no SNMP/node metrics** — logs only.
8. **MikroTik (10.254.2.3) and Juniper (10.254.2.1) are not blackbox-probed**; 10.254.2.2 is probed but has no SNMP job (identity unknown).
9. Blackbox internal targets are IP-only (no hostname label) — dashboards will show raw IPs.
