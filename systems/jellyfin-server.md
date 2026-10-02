# Jellyfin Media Server

**Baseline:** 2026-10-01

## Role

The ASUS GL702VM is the dedicated media and storage node. It runs Jellyfin directly on the operating system, provides attached media capacity, exports shared media to Proxmox over NFS, and hosts a small Docker layer for supporting media applications.

The system also serves as a cross-host infrastructure dependency: it holds the dedicated Proxmox backup partition and provides all three NFS media mounts consumed by the Proxmox host. Jellyfin is one of the environment's two intentionally public-facing service paths; other services on this host have no intentionally published inbound path.

This role combines media serving, storage, supporting applications, and backup targets on one physical system. The design makes effective use of available hardware but concentrates several dependencies on the media node.

## Platform and hardware

| Item | Baseline state |
|---|---|
| Physical system | ASUS GL702VM laptop |
| Operating system | Ubuntu Server 24.04 LTS |
| Platform | Linux, x86-64 |
| Virtualization detection | None |
| CPU | Intel Core i7-6700HQ, 4 cores / 8 threads |
| Memory | Approximately 11 GiB RAM |
| Swap | 4 GiB |
| GPU | NVIDIA GeForce GTX 1060 Mobile, 6 GB |
| NVIDIA driver | Proprietary driver installed |

The host runs Jellyfin natively rather than as a VM or Docker container. An NVIDIA persistence service was active, but active Jellyfin hardware transcoding was not established and is not assumed from GPU presence.

## Storage model

The host combines system storage, application and staging capacity, portable storage, and a multi-disk media archive.

### Filesystems

| Storage tier | Approximate capacity | Filesystem / layout | Baseline utilization |
|---|---:|---|---:|
| System HDD | 931 GB device | EFI, separate ext4 `/boot`, and LVM-backed ext4 root | Root filesystem 74% used |
| Internal SSD | 120 GB | ext4 | 1% used |
| Portable SSD | 466 GB | ext4 | 98% used |
| External HDD | 1.8 TB | ext4 | 38% used |
| External HDD | 5.5 TB | ext4 | 96% used |
| External HDD | 3.6 TB | ext4 | 54% used |
| External HDD | 3.6 TB | ext4 | 100% used |

The root filesystem is approximately 913 GB within an approximately 928 GB LVM-backed root volume. At baseline, it contained approximately 639 GB before exclusions; approximately 578 GB was in the explicitly excluded completed-downloads directory.

A dedicated, separately mounted SSD of approximately 125 GB is configured as the Timeshift snapshot target. The relationship between this device and the approximately 120 GB internal SSD measurement is not established. Exact device identifiers, serial numbers, UUIDs, and mount paths are intentionally omitted.

### Media archive

The attached high-capacity filesystems hold the media archive. High utilization is expected, including the filesystem that was full at baseline. The portable SSD also currently contains media, so its 98% utilization is expected; it is planned for later reuse as additional backup storage.

The media archive currently has no backup or replica because of budget constraints. This is an accepted risk because the content is considered replaceable, although reacquisition after failure would require time and effort. During a historical disk failure, non-corrupted files were salvaged through an Ubuntu Server live environment and copied to a rescue drive; this was a partial salvage method, not proof of complete archive recoverability. Disk health and redundancy are not established.

### NFS service

The host provides three media-storage exports to the Proxmox physical host over NFSv4.2. The corresponding Proxmox mounts are read-write. This allows Proxmox-hosted media and download automation to operate against shared storage without placing the archive on the Proxmox NVMe device.

Exact export definitions, client permissions, and directory paths are not documented. During a storm-related outage, Proxmox started before this host and entered a persistent NFS mount failure. Proxmox mount handling was remediated separately, while automatic restart behavior and Wake-on-LAN procedures were enabled for this host. NFS availability is monitored through Prometheus and Grafana; the detailed monitoring configuration is not documented.

### Backup storage role

A separate partition on this host is the target for a Proxmox backup job configured to run daily at 04:00 local time and retain the latest seven backups plus four weekly backups. The partition's local mount path is omitted from this public-facing document.

Hosting this target on the media node separates backups from the Proxmox system disk, but it does not create off-site protection and remains dependent on the health and availability of this physical host.

## Native workloads

| Service | Role |
|---|---|
| Jellyfin | Primary media-serving application; active systemd service |
| NFS server and RPC components | Export media storage to Proxmox |
| Docker Engine and containerd | Run supporting media applications |
| Prometheus Node Exporter | Expose host telemetry; scrape relationships unknown |
| Tailscale | Connect the host to the overlay used by the public ingress path |
| OpenSSH | Administrative access; publication details intentionally omitted |
| NVIDIA persistence daemon | Maintain NVIDIA driver state; Jellyfin GPU use not established |
| Timeshift | RSYNC-mode system snapshots; seven daily snapshots configured on a dedicated approximately 125 GB SSD |

Jellyfin runs under a dedicated service account. Its version, application configuration, library-to-filesystem mapping, authentication settings, and active transcoding configuration are not documented.

## Docker workloads

Three Docker runtime instances, bridge interfaces, veth interfaces, and network namespaces support the application layer. Container names, images, versions, health, volume mappings, and network settings are not available in this documentation.

| Application | Placement and role |
|---|---|
| Seerr | Docker workload used as the environment's media-request aggregator; a Docker-published listener was present |
| Prowlarr | Docker workload for indexer management; detailed container metadata is unavailable |
| Bazarr | Docker workload for subtitle management; detailed container metadata is unavailable |

The three runtime instances align with these applications, although exact process-to-container mappings for Prowlarr and Bazarr remain unavailable. API integrations and request or automation flows among these applications and Proxmox-hosted services are not documented.

## Networking role

The host uses one active wired interface on the private home LAN. The default route uses the LAN gateway. A Wi-Fi interface was present but down at baseline.

A Tailscale interface was active. Public Jellyfin access uses a domain-based entry point, an IONOS VPS, and the Tailscale overlay. During a historical 12-hour IONOS outage, about 10 of the environment's 20+ users temporarily accessed Jellyfin through a restricted, sanitized Tailscale path that bypassed the unavailable VPS route. Connectivity returned promptly for those users after they enabled the VPN; other users remained affected until the normal path recovered. Exact peer policy, reverse-proxy sequence, VPS software, DNS records, tunnel configuration, and Nginx Proxy Manager role remain unresolved or intentionally omitted.

The host also provides:

- private-LAN NFS connectivity to the Proxmox host;
- Docker bridge networking for three supporting application workloads;
- outbound connectivity for normal host and application operations;
- no intentionally published inbound paths for other host or Docker services.

The last point describes intended exposure, not established firewall behavior. Exact IP addresses, domains, firewall rules, Docker publication configuration, and Tailscale ACLs are not documented.

## Monitoring

Prometheus Node Exporter runs directly on this host. This establishes a host telemetry endpoint, but its collector, scrape interval, authentication, retention, dashboards, and alert routing are not documented.

Prometheus and Grafana processes run within the Proxmox-hosted LXC layer, but their exact placement and relationship to this Node Exporter are unresolved. No complete monitoring flow is asserted.

Storage utilization is an important operational signal because several archive filesystems are intentionally near capacity. No alert thresholds or capacity policy were established.

## Backup strategy

### Jellyfin host protection

Timeshift is configured in RSYNC mode for system snapshots. Daily snapshots are enabled with retention of seven daily snapshots. The target is a dedicated, separately mounted SSD of approximately 125 GB; its identifier and exact mount path are intentionally omitted.

The large media HDD filesystems are outside snapshot scope. A completed-downloads directory containing approximately 578 GB of replaceable media at baseline is explicitly excluded. The root filesystem contained approximately 639 GB before exclusions, with the overwhelming majority attributable to that directory.

An initial manual Timeshift snapshot was created on 2026-10-01 and reported a size of approximately 52 GB.

Timeshift was successfully used after a historical root-filesystem failure caused by improper administrative commands. It restored the operating system and important Jellyfin configuration and user files. Total service downtime was approximately two hours, and important data was preserved. The isolated restore-operation duration and a complete file manifest were not recorded.

Seerr, Bazarr, and Prowlarr configuration files were copied manually during recovery and are now included in Timeshift scope. Whether the current snapshot contains them is not established, and restoration of the current Docker configuration has not been separately tested.

### Proxmox backup target

The dedicated backup partition on this host is the target for the Proxmox job scheduled at 04:00 local time each day. Proxmox retention keeps the latest seven backups and four weekly backups.

The Proxmox backup target and Timeshift snapshot SSD reduce dependence on the Proxmox system disk, but both remain within the same local two-host environment. Whether they share any lower-level storage failure domain is not established. No off-site backup relationship exists.

### Media protection

The large media HDD filesystems are outside Timeshift scope, and the completed-downloads directory is explicitly excluded. The media archive currently exists only on the high-capacity filesystems. This is a budget-driven tradeoff. Historical recovery combined salvage of non-corrupted files to a rescue drive with reacquisition as the strategy for missing or damaged replaceable media.

The portable SSD is planned for additional backup use after its current media contents are relocated. That is a future intention, not a current backup capability.

## Dependencies

| Dependency | Why it matters |
|---|---|
| Attached media filesystems | Hold the Jellyfin media archive and NFS-served data |
| Private home LAN | Carries local client, NFS, Docker-published, and backup traffic |
| Tailscale and IONOS VPS | Provide the established public Jellyfin access path |
| Proxmox host | Runs the broader media-automation, monitoring, reverse-proxy, and optional game workloads |
| Dedicated approximately 125 GB SSD | Holds Timeshift RSYNC-mode system snapshots |
| HDD enclosure / attached media storage | Holds much of the unprotected media archive |

The Proxmox host depends on this node for NFS and backup storage, while this node participates in a broader service environment containing automation and monitoring workloads on Proxmox. Exact application API dependencies are not documented.

## Operational considerations

- **Concentrated role:** one physical host provides Jellyfin, shared media, NFS, Docker workloads, the Timeshift snapshot target, and the Proxmox backup target.
- **Archive capacity:** filesystems at 96–100% utilization are expected archival conditions, but they leave limited growth headroom.
- **Single-copy media:** the archive is not replicated or backed up; recovery depends on reacquiring replaceable content.
- **System capacity:** the root filesystem was 74% used at baseline.
- **Portable SSD transition:** current high use is expected, and the device's future backup role is not yet active.
- **Public-path dependency:** normal public Jellyfin access depends on the Tailscale overlay and IONOS VPS in addition to the local host and Internet connection. A restricted trusted-user Tailscale path has provided temporary access during a VPS outage, but it is not equivalent to public ingress.
