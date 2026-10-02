# Proxmox Host

**Baseline:** 2026-10-01

## Role

The Getorli GT100 is the environment's standalone virtualization host. It provides separate lifecycle and privilege boundaries for long-running infrastructure and media-automation services while retaining QEMU support for workloads that require a complete guest operating system.

The host runs six active, intentionally unprivileged LXC guests and two QEMU VMs: an intermittently used video-game server and an inactive Windows Server lab. Bulk media remains on the separate Jellyfin host and is presented to Proxmox over NFS, keeping guest storage and large media storage in distinct tiers.

## Platform and hardware

| Item | Baseline state |
|---|---|
| Physical system | Getorli GT100 desktop-class system |
| Operating system | Debian GNU/Linux 13 |
| Virtualization platform | Proxmox VE 9 |
| Platform | Linux, x86-64 |
| CPU | AMD Ryzen 5 3550H, 4 cores / 8 threads |
| Hardware virtualization | AMD-V available |
| Memory | Approximately 12 GiB RAM |
| Swap | 8 GiB |
| Topology | Intentionally standalone Proxmox host |

The hardware footprint is sized for a compact collection of service containers and occasional VM use rather than clustered virtualization. Active Proxmox HA-related system services do not represent a multi-node HA design.

## Storage model

### Local system and guest storage

The host uses an approximately 477 GiB NVMe device. The layout includes:

- an approximately 94 GiB ext4 root filesystem, 86% used at baseline;
- an approximately 1 GiB EFI partition;
- an 8 GiB swap logical volume;
- an approximately 349 GiB LVM-thin pool;
- guest volumes for LXC IDs 102, 106, 108, 109, 111, and 112.

LVM-thin is the primary guest-storage model. No active ZFS pool was visible. An active ZFS event daemon alone does not indicate that guest storage uses ZFS.

The authoritative Proxmox storage configuration is not included here, so storage IDs, allocation policy, thin-pool thresholds, and QEMU disk placement are not documented.

### Shared media storage

Three read-write NFSv4.2 mounts originate from the Jellyfin physical host and serve media storage. At baseline they reported:

- two approximately 913 GiB exports at 74% utilization;
- one approximately 458 GiB export at 98% utilization.

High utilization of the media exports is expected because they hold archival media. Exact export names, paths, permissions, and per-LXC mount assignments are not documented.

This design avoids duplicating large media data on the Proxmox NVMe device, but makes the Jellyfin host and home LAN dependencies for shared-media availability. During a storm-related power outage, Proxmox started before the Jellyfin/NFS host and all three mounts entered a persistent blocking state. The Proxmox web interface and media-management workflows were disrupted for approximately 45–60 minutes.

The host mount configuration was changed to use `nofail` with a reduced timeout. Storage is mounted at the Proxmox host layer and selected paths are exposed to qBittorrent, Sonarr, Radarr, and SABnzbd. A deliberate test with the Jellyfin host powered off confirmed that Proxmox and its LXCs start in a degraded state rather than blocking. Media functions remain unavailable until NFS returns.

## Virtualization and workloads

### LXC service layer

All six LXC guests were running at baseline. Their unprivileged configuration and workload mapping are:

| Guest | Workload | Baseline state | Purpose |
|---|---|---|---|
| 102 | Prometheus PVE Exporter | Running | Proxmox telemetry export |
| 106 | qBittorrent | Running | Media download workload with Proton VPN egress |
| 108 | Radarr | Running | Media automation |
| 109 | Sonarr | Running | Media automation |
| 111 | Nginx Proxy Manager | Running | Reverse-proxy component; exact public-path role unknown |
| 112 | SABnzbd | Running | Download automation |

Each guest had a running per-container systemd unit, an `lxc-start` process, a bridge-attached virtual Ethernet interface, and a guest LVM volume. The separate LXCs create distinct service lifecycle and container privilege boundaries; guest configuration and effective isolation are outside this document.

Docker/containerd runs within the Proxmox LXC layer, but the responsible guest and nested-container purpose remain unresolved. Prometheus, Grafana, Nginx, Node.js, and Python also run in that layer without reliable guest attribution, so they are not assigned to specific guests here. Detailed Proxmox guest configuration is not included.

### QEMU virtual machines

| Guest | Role | Baseline state |
|---|---|---|
| 100 | Game server | Powered off at baseline; only VM currently in use |
| 103 | Windows Server lab | Inactive and not recently used |

VM 100 has hosted Minecraft and Project Zomboid. Its current in-guest installation and service state are not documented. When enabled, its public service path uses an IONOS VPS and Tailscale; exact routing and reverse-proxy hops remain unresolved.

VM 103 supported historical Windows Server and domain-controller experimentation. It does not represent a current Active Directory service and is outside the repository's current service scope. No additional guests are documented.

## Networking role

The host has one active wired uplink attached to a primary Linux bridge on the private home LAN. Six LXC veth interfaces were attached to that bridge at baseline. A wireless interface was present but down.

A Tailscale interface and overlay routes were active. Tailscale participates in the public path for VM 100 when the game service is enabled and in broader environment connectivity, but peer policy and complete host-listener reachability are not established.

The intended exposure model is:

- Proxmox management and the LXC services do not have intentionally published inbound Internet paths;
- VM 100 has a public path through the IONOS VPS and Tailscale when enabled;
- qBittorrent in LXC 106 sends its external traffic through Proton VPN and uses the provider-forwarded port;
- NFS media traffic remains on the private LAN between the two physical hosts.

Exact IP addresses, domains, firewall rules, Tailscale ACLs, and VPN configuration are outside this document.

## Host services and management plane

The Proxmox host control plane includes the API/web proxy, core daemons, scheduler, statistics service, cluster filesystem, LXC services, firewall services, SPICE proxy, and QEMU event daemon. Supporting host services include SSH, Tailscale, chrony, Postfix, SMART monitoring, RPC/NFS components, RRD cache, and watchdog services.

These services establish that the host management plane is operating; they do not establish a cluster, active HA policy, configured QEMU inventory, or particular firewall behavior. The Proxmox web/API and management listeners are present, but the intended model is no published inbound Internet path.

## Monitoring

Monitoring on or beneath this host includes:

- the Prometheus PVE Exporter workload in LXC 102;
- `pve_exporter`, Prometheus, and Grafana processes in the LXC process tree;
- Proxmox statistics and RRD services on the physical host.

NFS availability is monitored through Prometheus and Grafana following the historical startup-dependency incident. The exact LXC placement of Prometheus and Grafana, metric source, scrape relationships, alert routing, retention, and dashboard ownership are not established. The presence of these processes and NFS coverage does not establish a fully documented monitoring pipeline.

## Backup strategy

A Proxmox backup job is configured to run daily at **04:00 local time**. It targets a dedicated partition on the Jellyfin physical host. Retention keeps:

- the latest seven backups;
- four weekly backups.

The target's internal mount path is intentionally omitted from this public-facing document. Because the target resides on the other physical host, backups are separated from the Proxmox NVMe device but still depend on the Jellyfin host and local network.

VM 100 was successfully recovered from backup as a controlled restore test, with the expected state recovered. Restore duration was not provided. Detailed job configuration, complete guest selection, routine execution history, integrity checks, and broader restoration history are not documented. The schedule, retention, and VM 100 test outcome are part of the established baseline.

## Dependencies

| Dependency | Why it matters |
|---|---|
| Jellyfin physical host | Supplies three NFS media mounts and hosts the Proxmox backup target |
| Private home LAN | Carries management, guest, NFS, and backup traffic |
| Local NVMe and LVM-thin pool | Holds the LXC guest volumes and host filesystem |
| Tailscale and IONOS VPS | Provide VM 100's public-service path when enabled |
| Proton VPN | Provides qBittorrent's external traffic path and forwarded port |

The Jellyfin host is the most significant cross-host dependency. Its loss would affect shared media access and availability of the Proxmox backup target, even if the Proxmox host and local guests remained operational.

## Operational considerations

- **Standalone operation:** there is no second Proxmox node for host-level failover. Installed HA-related services must not be interpreted as active clustered HA.
- **Root capacity:** the root filesystem was 86% used at baseline and warrants capacity trending, although no alert threshold or remediation policy was established.
- **Shared-storage dependency:** media-facing workloads depend on NFS service and network connectivity to the Jellyfin host. `nofail` and reduced timeout behavior allow Proxmox and its LXCs to start without NFS, but dependent media functions remain degraded until storage recovers.
- **Expected archive utilization:** the NFS mount at 98% is part of the accepted media-archive capacity model, not by itself an incident indicator.
- **Intermittent game workload:** VM 100 was intentionally off at baseline because of low current use.
- **Security boundaries:** unprivileged LXCs are intentional; guest configuration and effective isolation are outside this document.
- **Backup locality:** backups leave the Proxmox system disk but remain within the same two-host environment.
