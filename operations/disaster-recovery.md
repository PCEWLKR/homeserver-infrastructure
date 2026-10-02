# Disaster Recovery

**Baseline date:** 2026-10-01

## Purpose

This document records the environment's current recovery posture, protected and replaceable data, dependency-driven recovery priorities, known single points of failure, and actual recovery experience. It is not a step-by-step runbook.

Exact commands, device identifiers, mount paths, addresses, domains, credentials, firewall rules, and VPN configuration are intentionally omitted. No formal recovery point objective (RPO), recovery time objective (RTO), guaranteed restore time, or generalized recovery capability is defined; historical outcomes describe specific incidents rather than future service-level commitments.

## Recovery posture

The environment separates smaller system/configuration recovery from bulk-media recovery:

- Jellyfin host system snapshots use Timeshift in RSYNC mode.
- Proxmox has a scheduled backup job with defined retention on storage hosted by the Jellyfin node.
- The media archive has no backup or replica; recovery relies on salvaging readable files and reacquiring replaceable content.
- Public Jellyfin access normally depends on the IONOS VPS and Tailscale, with a restricted Tailscale fallback available to a subset of trusted users.
- NFS mount behavior has been changed so storage unavailability no longer blocks Proxmox startup.

This approach protects smaller operational state while accepting the cost and delay of media reacquisition. Both backup targets remain within the same local environment, so they do not provide off-site or site-level disaster protection.

## Recovery priorities

Recovery order is event-dependent rather than a single fixed sequence. The following priorities are based on established dependencies and historical practice:

| Priority area | Recovery objective | What is known |
|---|---|---|
| Preserve recoverable data | Avoid overwriting salvageable system or media data during a storage event | Historical media recovery copied non-corrupted files to a rescue drive from a live environment |
| Restore physical-host operation | Recover the affected operating system sufficiently to provide its host role | Timeshift successfully restored the Jellyfin OS after a root-filesystem failure |
| Restore configuration and user state | Recover important Jellyfin configuration and user files | The historical Timeshift restore recovered these items |
| Restore Proxmox workloads | Recover required guests from the Proxmox backup set when necessary | VM 100 was successfully restored as a test; complete guest coverage remains unknown |
| Restore shared storage availability | Re-establish NFS-dependent media access without blocking the Proxmox host itself | Current mount behavior permits Proxmox startup while an unavailable share reports a warning |
| Restore service access | Re-establish normal VPS/Tailscale ingress or provide restricted fallback access | Restricted Tailscale access restored Jellyfin connectivity for approximately 10 users during a VPS outage |
| Recover replaceable media | Salvage readable files first, then reacquire missing or corrupted media | Historical disk salvage supports this approach; the archive remains dependent on reacquisition |

This table describes goals and dependencies, not a fixed procedural order for every failure scenario.

## Protected, replaceable, and unresolved data

### Protected by established configuration

#### Jellyfin host system snapshots

Timeshift is configured in RSYNC mode with:

- daily system snapshots;
- retention of seven daily snapshots;
- a dedicated, separately mounted approximately 125 GB SSD target;
- large media HDD filesystems outside snapshot scope;
- an explicit exclusion for the completed-downloads directory.

An initial manual snapshot created on 2026-10-01 was approximately 52 GB. The root filesystem contained approximately 639 GB before exclusions, of which approximately 578 GB was attributable to the excluded replaceable-downloads directory.

A historical recovery restored the operating system and important Jellyfin configuration and user files. Every current application file is not necessarily present in every snapshot.

#### Proxmox backups

The Proxmox backup job is configured to:

- run daily at 04:00 local time;
- target a dedicated partition on the Jellyfin host;
- retain the latest seven backups and four weekly backups.

VM 100 was successfully restored from backup as a test, and its expected state was recovered. Complete guest selection, routine job history, artifact integrity, and restore behavior for other guests remain unknown.

### Explicitly replaceable or unprotected

The large media archive is not backed up or replicated. This is an accepted budget-driven risk. The recovery strategy prioritizes rapid reacquisition of missing replaceable media rather than bulk backup of the archive.

The completed-downloads directory is explicitly excluded from Timeshift and contained approximately 578 GB at baseline. Its contents are treated as replaceable.

The reacquisition pipeline has been used operationally and is supported by prior salvage experience. No measured full-archive recovery time or validated throughput is documented.

### Configuration requiring validation

Seerr, Bazarr, and Prowlarr configuration files were copied manually during the historical recovery and are now included in Timeshift scope. Their current snapshot presence has not been independently inspected, and the current Docker configuration has not been separately restore-tested.

The exact Proxmox backup scope is also unknown. VM 100 recovery succeeded in one test, but that result does not show protection for all LXCs, VM 103, host configuration, or every current guest disk.

## Dependency and failure analysis

### Jellyfin physical host

The Jellyfin node is a major concentration point. It provides:

- Jellyfin;
- attached media filesystems;
- three NFS media exports to Proxmox;
- Docker-hosted supporting applications;
- the Timeshift snapshot target;
- the Proxmox backup target.

Loss of the host can therefore affect media access, NFS consumers, normal Jellyfin service, supporting Docker applications, and access to both documented backup targets. The Timeshift SSD and Proxmox target are separately described, but whether they share a lower-level storage failure domain has not been established.

### Proxmox physical host

The standalone Proxmox host runs all six service LXCs and the two known QEMU VMs. There is no second Proxmox node for host-level failover. A host outage affects monitoring export, download/media automation, reverse-proxy components, and VM 100 when it is in use.

### NFS

NFS is a shared-service dependency between the physical hosts. During a storm-related power outage, Proxmox started before the Jellyfin/NFS host. All three mounts entered a persistent blocking state that affected the Proxmox web interface and media-management workflows for approximately 45–60 minutes.

The Proxmox host mount configuration now uses `nofail` with a reduced timeout. Storage is mounted at the host layer and selected paths are exposed to qBittorrent, Sonarr, Radarr, and SABnzbd. A deliberate test with the Jellyfin host powered off confirmed that Proxmox and its LXCs start in a degraded state. Applications can run, but media functions remain unavailable until NFS returns.

Automatic restart and Wake-on-LAN procedures were enabled for the Jellyfin host. NFS availability is now monitored through Prometheus and Grafana; exact scrape and alert behavior remain unknown.

### Public ingress

Normal public Jellyfin and game-service paths depend on an IONOS VPS and Tailscale. The exact reverse-proxy chain remains unresolved.

During a historical IONOS outage, the normal public path was unavailable for approximately 12 hours. About 10 of the 20+ users had access to a restricted, sanitized Tailscale path and restored Jellyfin connectivity promptly after enabling the VPN. Users without that access remained affected until the provider path recovered.

The fallback reduces impact for authorized users but is not equivalent to full public-service continuity. Exact peer membership, ACLs, and connection details are intentionally omitted.

### Locality and site risk

Both documented backup targets are local to the Jellyfin node. No off-site, cloud, or geographically separate backup has been established. A physical-site event, local power event, or Jellyfin-host/storage event could affect source data and backup availability together.

## Historical recovery and outage experience

### Jellyfin root-filesystem failure

| Item | Outcome |
|---|---|
| Initiating event | Improper elevated deletion operation |
| Impact | Root-filesystem/OS failure and Jellyfin service outage |
| Recovery mechanism | Timeshift system restore |
| Recovered | Operating system plus important Jellyfin configuration and user files |
| Total service downtime | Approximately 2 hours |
| Data outcome | Important data preserved |
| Docker-state recovery | Docker configuration copied manually; current Timeshift inclusion is configured but not separately restore-tested |
| Data loss | Approximately 200 GB of replaceable in-transit data |
| Additional reacquisition delay | Approximately 3–5 hours; noncritical |
| Isolated restore-operation duration | Not recorded separately from total downtime |
| Overall result | Restore went as planned and service recovery succeeded |

The two-hour outcome applies to this incident and is not a recovery-time guarantee. The data preserved during recovery must not be interpreted as part of the current approximately 52 GB Timeshift snapshot; media filesystems are outside Timeshift scope.

The host was not rebooted normally after the destructive deletion was recognized. Recovery began immediately from an Ubuntu live USB. Timeshift restored the OS and important Jellyfin state directly; Docker configuration and an NVIDIA concurrent-transcoding patch were copied manually from recoverable data. The restored host booted successfully on the first attempt, followed by host, application, and Proxmox-side NFS validation.

See [Jellyfin Root-Filesystem Failure and Timeshift Recovery](incidents/2026-01-jellyfin-root-filesystem-recovery.md) for the full incident record.

### VM 100 recovery test

VM 100 was recovered from backup as a test, and the expected state was recovered successfully. The test establishes recoverability for that VM and backup instance, but no restore duration or repeatability record is available, and other guests remain unvalidated.

### Media-disk failure

A failed media HDD was connected to an Ubuntu Server live environment, and readable media files were extracted to a rescue drive. The incident successfully salvaged readable files, but did not establish complete archive recoverability.

Missing or corrupted replaceable media is handled through reacquisition. The archive remains single-copy.

### NFS outage

A brief storm-related power outage caused Proxmox to start before the Jellyfin host. The Proxmox web interface became unavailable, while SSH remained usable for diagnosis. After logs identified NFS and a first restart failed to clear the condition, `nofail` and shorter timeout behavior were configured on the host mounts.

The remediation was tested by powering off the Jellyfin host. Proxmox and its LXCs started normally, while media workflows remained degraded until NFS returned. The incident has not recurred, and NFS availability is now monitored through Prometheus and Grafana.

See [Proxmox/NFS Startup Dependency Failure and Remediation](incidents/2026-07-proxmox-nfs-startup-dependency.md) for the full incident record.

### IONOS VPS outage

The normal public ingress path was unavailable for approximately 12 hours due to a provider-side outage. Approximately 10 trusted users regained Jellyfin access promptly by activating the restricted Tailscale fallback. Other users remained without the normal public path until provider service returned.

This incident validates a limited access fallback, not full public-path redundancy.

## Recovery coverage and open questions

| Recovery area | Available information | Missing information |
|---|---|---|
| Jellyfin OS | Timeshift mode, schedule, retention, sanitized target, exclusions, snapshot size, one successful historical restore, and a detailed incident sequence | Full manifest, independent Docker-config inclusion verification, isolated restore duration, repeated validation |
| Proxmox VM | Backup schedule, retention, sanitized target, and successful VM 100 restore test | Complete guest selection, LXC restore tests, VM 103 coverage, restore timing, artifact health history |
| Media archive | Historical live-environment salvage and reacquisition strategy | Full inventory, integrity baseline, recovery throughput, spare-drive capacity, full-archive recovery time |
| NFS outage | Failure sequence, `nofail` and timeout remediation, affected LXC paths, deliberate power-off validation, and current monitoring intent | Exact timeout value, complete monitoring implementation, tested behavior for every failure mode |
| Public ingress | Normal VPS/Tailscale relationship and restricted trusted-user fallback | Exact routing, failover procedure, peer policy, DNS behavior, broader-user fallback |
| Monitoring | Component presence | Alerting, outage detection, notification, dashboard, and monitoring-recovery details |

## What recovery has demonstrated

The documented recovery experience demonstrates:

- Timeshift has successfully recovered the Jellyfin OS and important Jellyfin configuration/user files in one historical incident.
- VM 100 has been successfully restored from backup in a test.
- Non-corrupted media has been salvaged from a failed disk using a live environment and rescue drive.
- Proxmox startup coupling to unavailable NFS storage has been reduced.
- Restricted Tailscale access has maintained Jellyfin availability for a subset of users during an IONOS outage.

This experience does not show:

- a formal RPO or RTO;
- a guaranteed two-hour Jellyfin recovery;
- independently inspected or restore-tested protection for the current Docker configuration;
- recoverability of every Proxmox guest;
- full media-archive recovery from backup;
- full public-ingress redundancy for all users;
- tested site-level disaster recovery;
- validated recovery from simultaneous loss of a source host and its local backup targets.

## Remaining validation needs

Material gaps that would improve recovery readiness are:

1. derive a reusable sanitized Timeshift recovery runbook from the documented incident;
2. independently verify Seerr, Bazarr, and Prowlarr configuration in a completed snapshot and test recovery safely;
3. verify the complete Proxmox backup selection and test at least one LXC restore safely;
4. record backup job history, artifact freshness, and retention behavior;
5. document NFS monitoring behavior and remaining dependency details without exposing unnecessary mount information;
6. define and document the restricted Tailscale fallback activation process for authorized users;
7. inventory media at a level sufficient to drive reacquisition without exposing personal data;
8. evaluate a physically or geographically separate copy for the smallest irreplaceable data set;
9. record timestamps during future tests before deriving any recovery objectives.

These are improvement opportunities, not implied current capabilities.
