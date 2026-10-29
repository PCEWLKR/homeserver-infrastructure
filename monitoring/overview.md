# Monitoring Overview

**Baseline date:** 2026-10-01

## Demonstrated telemetry

Grafana dashboards demonstrate host and virtualization telemetry across the two physical systems:

- **Jellyfin host:** CPU and memory utilization, reachability status, and filesystem usage. The media filesystems include full or near-full volumes; this is expected for the archive and is not by itself an incident.
- **Proxmox:** resource allocation and current/history panels for the host and guests, including guest CPU, memory, and disk usage, plus storage utilization.

Sanitized examples: [Jellyfin host dashboard](../assets/jellyfin-monitoring-sanitized.png) and [Proxmox dashboard](../assets/proxmox-monitoring-sanitized.png). These screenshots demonstrate rendered dashboard telemetry, not alert delivery, sustained data retention, or monitoring availability.

## Known components

| Component | Known placement | Documented role |
|---|---|---|
| Prometheus PVE Exporter | Unprivileged LXC 102 | Exporter workload for Proxmox telemetry. |
| Prometheus | Proxmox-hosted LXC layer; exact guest unknown | Metrics collection component; exact targets and configuration are not documented. |
| Grafana | Proxmox-hosted LXC layer; exact guest unknown | Dashboarding; screenshots demonstrate host and virtualization telemetry. |
| Prometheus Node Exporter | Jellyfin physical host | Host telemetry endpoint. |
| Proxmox statistics and RRD services | Proxmox physical host | Platform-native statistics services. |

NFS availability is monitored through Prometheus and Grafana following a historical startup-dependency incident. The specific metric and dashboard implementation have not been recorded here.

## Scope and limitations

Exact scrape topology, component versions, retention, alert rules and thresholds, notification destinations, monitoring authentication, and backup/recovery arrangements are not documented in this public overview. Alerting or notification behavior has not been verified and is not claimed as a demonstrated capability. The location of Prometheus and Grafana within the LXC layer is also unknown.

The screenshots are point-in-time examples. They do not establish continuous monitoring coverage, alerting, service-level objectives, or automated remediation. Implementation and validation details are intentionally omitted from this public overview.
