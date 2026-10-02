# Proxmox/NFS Startup Dependency Failure and Remediation

**Incident date:** Approximately July 2026

**Documentation status:** Retrospectively reconstructed. Exact timestamps were not retained, and durations are approximate.

## Summary

A brief storm-related power outage caused the Proxmox and Jellyfin physical hosts to return in an unfavorable order. Proxmox attempted to mount three NFS media shares before the Jellyfin host was available. The mount process entered a persistent failure loop that continued to affect Proxmox after the Jellyfin host and its shares had recovered.

The Proxmox web interface was unavailable and media-management workloads could not provide their required function. Administrative SSH access to the Proxmox host remained available and was used for diagnosis and recovery. The Jellyfin host returned in under approximately 10 minutes, and user-facing Jellyfin access remained otherwise available. Media-management services were disrupted for approximately 45–60 minutes.

Recovery required confirming that NFS was available to other LAN clients, restarting Proxmox, observing the failure recur, then changing the Proxmox host's mount configuration to use `nofail` behavior with a reduced timeout. After another restart, host connectivity returned. The mount design was also consolidated at the Proxmox host level and selected paths were exposed to the LXCs that use media storage.

The remediation was validated by powering off the Jellyfin host and reproducing the original dependency condition. Proxmox and its LXCs started normally while NFS was unavailable. The workloads could run, but their media-dependent functions remained degraded until shared storage returned.

## Impact

### User impact

Jellyfin itself regained connectivity within approximately 10 minutes and remained available to users. The incident primarily affected the Proxmox-hosted media-management pipeline rather than direct media playback.

### Infrastructure impact

- The Proxmox web interface was unavailable.
- Proxmox retained enough functionality for administrative SSH access.
- qBittorrent, Sonarr, Radarr, and SABnzbd were confirmed unable to perform their required media workflows.
- All three host-level NFS mounts were involved.
- Monitoring through the later Prometheus/Grafana implementation was not available because that implementation was not complete at incident time.

Nginx Proxy Manager impact was not established. No claim is made about workloads not specifically confirmed during the incident.

## Trigger and contributing conditions

The initiating event was a brief local power outage during storms. The resulting host startup order exposed a timing dependency:

1. Proxmox started while the Jellyfin/NFS host was still down.
2. Proxmox attempted to bind the NFS-backed media storage.
3. The mount process fell into a loop or persistent failed state.
4. Jellyfin and its NFS shares returned, but Proxmox did not recover the mounts automatically.
5. Media-management services remained unavailable until host-side remediation.

At incident time, the mount configuration treated unavailable NFS storage as startup-blocking. The earlier design also required more manual handling of individual HDD-backed mounts rather than the later consolidated enclosure-level structure.

## Symptoms

The observable symptoms were:

- unavailable Proxmox web interface;
- reduced Proxmox host functionality;
- SSH still available for administration;
- media-management applications unable to perform normal sorting and management workflows;
- persistent NFS-related errors despite the Jellyfin host having returned;
- all three expected NFS mounts unavailable from the Proxmox workload path.

Although the affected LXCs can start and run without NFS after remediation, their intended media workflows require access to the shared storage.

## Timeline

| Relative time | Event |
|---|---|
| T+0 | Brief storm-related power outage affected both physical hosts. |
| Early recovery | Proxmox started before the Jellyfin host and attempted to mount unavailable NFS shares. |
| Within approximately 10 minutes | Jellyfin regained connectivity and its shares were confirmed available to other LAN devices. |
| Initial diagnosis | The Proxmox web interface was attempted and found unavailable; administrative work moved to SSH. |
| Log review | Host error and boot logs pointed to the NFS mount process. |
| First recovery attempt | Proxmox was restarted after NFS availability was confirmed, based on the hypothesis that only startup order was responsible. |
| Recurrence | The same failure returned after reboot, disproving the startup-order-only hypothesis. |
| Remediation | `nofail` behavior was enabled for the host mounts and mount timeout behavior was reduced. |
| Second restart | Proxmox connectivity and workload startup returned. |
| Approximately T+45–60 minutes | Media-management service disruption ended. |
| Post-incident validation | The Jellyfin host was deliberately powered off and Proxmox startup was tested under the original unavailable-NFS condition. |

Exact timestamps were not retained.

## Diagnosis and operational reasoning

The unavailable web interface required moving to host SSH for diagnosis. Error and boot logs identified NFS mounting as the blocking condition.

The Jellyfin host was then checked directly, and the shares were verified as available from other devices on the LAN. This separated NFS server availability from the stale or blocking state on the Proxmox client.

The first Proxmox restart tested the hypothesis that correct host order alone would restore normal operation now that NFS was online. When the same behavior recurred, the issue was treated as a mount error-handling problem rather than only a one-time ordering problem.

The remediation choice was to allow Proxmox and its guests to start in a degraded state instead of treating media storage as mandatory for boot. This preserves management access and service lifecycle control while allowing media-dependent applications to report errors until storage becomes available.

## Recovery sequence

1. Access through the Proxmox web interface was attempted and found unavailable.
2. Administrative SSH was used to connect to the Proxmox host.
3. Host error and boot logs were reviewed.
4. NFS mount failures were identified as the common blocking condition.
5. The Jellyfin host was checked to verify that its NFS service and shares were online.
6. NFS availability was confirmed through other LAN devices.
7. Proxmox was restarted to test whether restored NFS availability and corrected boot order were sufficient.
8. The same failure recurred after restart.
9. The Proxmox host's NFS entries were configured with `nofail` behavior and shorter timeout handling.
10. Proxmox was restarted again.
11. Host connectivity and workload startup were confirmed.
12. The host-level storage design was consolidated, with required paths exposed to qBittorrent, Sonarr, Radarr, and SABnzbd.

Exact mount paths, server addresses, and timeout values are intentionally omitted.

## Remediation design

### Non-blocking host mounts

The Proxmox host now treats unavailable NFS media storage as a degraded condition rather than a boot-blocking requirement. The established mechanism uses `nofail` in the host mount configuration and a reduced mount timeout.

This allows:

- Proxmox to start without the Jellyfin host;
- the web and SSH management paths to become available;
- LXCs to start without the NFS shares;
- applications to report missing storage rather than blocking host startup.

The applications can run without NFS, but qBittorrent, Sonarr, Radarr, and SABnzbd cannot perform their intended media workflows until storage returns.

### Consolidated host-level mount model

The final design mounts NFS storage on the Proxmox host and exposes selected paths to the LXCs that require it. The relevant workloads are:

- qBittorrent;
- Sonarr;
- Radarr;
- SABnzbd.

The storage design was consolidated around the broader HDD-enclosure hierarchy and its subdirectories rather than manually treating each HDD-backed path as an independent LXC-level binding. Exact exports and bind paths are omitted.

This centralizes the Proxmox-side failure point and makes degraded mount behavior manageable at the host layer. It does not remove the Jellyfin host or NFS service as runtime dependencies for media access.

### Power-recovery measures

Additional measures were enabled for future power events:

- automatic restart behavior for the Jellyfin host;
- Wake-on-LAN procedures for bringing the Jellyfin host back when required.

Exact firmware, network, and Wake-on-LAN configuration is intentionally not documented.

## Validation

The remediation was tested by powering off the Jellyfin host rather than merely disabling its NFS export. This reproduced the broader original condition in which the storage server itself was unavailable.

Validation outcome:

- Proxmox started normally;
- non-media host functionality remained available;
- LXCs started normally;
- missing NFS storage produced degraded behavior/errors rather than blocking startup;
- media-dependent functions remained unavailable until the Jellyfin host and storage returned.

This test directly validates the intended startup behavior. It does not establish behavior for every possible NFS failure mode, filesystem corruption, or simultaneous host fault.

## Monitoring and recurrence

The incident occurred before the current Prometheus/Grafana implementation was complete, so that stack did not contribute to initial detection or diagnosis.

NFS availability is now monitored through Prometheus and Grafana. Exact scrape configuration, dashboards, thresholds, and alert notification behavior remain undocumented.

The failure has not recurred since remediation.

## Operational outcome

Media-management services were unavailable for approximately 45–60 minutes. Jellyfin user connectivity returned in under approximately 10 minutes and was otherwise unaffected by the Proxmox client-side mount loop.

The remediation changed the preferred failure mode from startup blocking to visible degradation. No workload is intended to be prevented from starting solely because NFS is offline, although media workflows cannot operate normally without their data.

The duration is a historical observation, not an RTO or guaranteed future recovery time.

## Lessons learned

- Shared storage should not block management-plane or guest startup when workloads can safely enter a degraded state.
- Service availability on the NFS server does not guarantee that a client has recovered from an earlier blocking mount failure.
- Testing the initial hypothesis with a restart exposed that startup order was not the only problem.
- Host-level mount error handling provides a clearer recovery boundary than repeated manual guest-level binding.
- Power restoration order matters when one physical host supplies storage to another.
- A realistic validation should remove the storage host, not only disable an export.
- Automatic restart and Wake-on-LAN provide additional recovery options after local power events.
- Monitoring added after the incident can improve visibility, but its exact alerting behavior still requires documentation.

## Remaining limitations

- Exact timestamps, log excerpts, timeout values, and prior mount configuration were not retained.
- Exact NFS export and bind-mount paths are intentionally omitted.
- NFS availability monitoring is documented, but scrape and alert-delivery details remain unknown.
- There is no second NFS server or replicated media archive.
- Workloads start without NFS but cannot complete their intended media operations until shared storage returns.
- Wake-on-LAN and automatic restart behavior are documented only at an architectural level, not as a tested procedural runbook.
- No formal severity, RPO, RTO, or availability objective is assigned.
