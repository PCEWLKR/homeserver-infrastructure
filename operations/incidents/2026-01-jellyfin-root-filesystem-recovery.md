# Jellyfin Root-Filesystem Failure and Timeshift Recovery

**Incident date:** Mid-January 2026 (approximate)

**Documentation status:** Retrospectively reconstructed. Durations and data quantities are approximate unless stated otherwise.

## Summary

The Jellyfin physical host suffered a root-filesystem failure after an improper destructive deletion operation was executed with elevated privileges. The first visible symptom was the sudden loss of ordinary shell commands while the host was still running. The host was shut down immediately to limit further damage, and recovery began from an Ubuntu live USB without attempting another normal boot.

The installed operating system was determined to be impractical to repair in place. A Timeshift snapshot approximately one month old, stored on a physically separate SSD inside the same laptop, was restored. The host booted successfully on the first post-recovery attempt. Timeshift directly recovered the operating system and important Jellyfin configuration and user files; Docker configuration and a GPU-transcoding patch were copied manually from recoverable data.

Total service downtime was approximately two hours. Retrospective assessment indicates that important data was preserved. Approximately 200 GB of replaceable in-transit data was lost, causing an additional noncritical reacquisition delay of roughly three to five hours.

## Impact

The following services on the Jellyfin physical host were unavailable during recovery:

- Jellyfin;
- Seerr;
- Bazarr;
- Prowlarr.

The outage also temporarily removed the host's NFS role and other services provided by the laptop. Validation after recovery included confirming NFS availability from the Proxmox side.

Core Jellyfin configuration and user data were preserved. Lost data was limited to approximately 200 GB of replaceable in-transit content, based on the incident reconstruction.

## Detection and initial symptoms

The incident began while the host was running. Common shell commands, including basic navigation and file-listing commands, suddenly became unavailable. The known improper deletion operation indicated active root-filesystem damage rather than an isolated shell or path problem.

The host was powered down immediately. No normal reboot was attempted. This decision prioritized preservation of remaining data over further diagnosis on the damaged live system.

## Timeline

Times are relative and approximate; exact timestamps were not retained.

| Relative time | Event |
|---|---|
| T+0 | Ordinary shell commands disappeared while the host was running. |
| Immediately afterward | The destructive deletion event was recognized as the likely initiating event, and the host was shut down. |
| First approximately 15 minutes | Initial diagnosis began from an Ubuntu live USB. This diagnostic period is included in the total outage. |
| Recovery phase | The installed root filesystem was mounted from the live environment and entered through a chroot for inspection. |
| Recovery decision | In-place repair was judged impractical; an approximately month-old Timeshift snapshot was selected. |
| Restore phase | Timeshift restored the operating system and important Jellyfin configuration and user files. |
| Data-copy phase | Recoverable Docker configuration files and the existing NVIDIA transcoding patch were copied manually to accelerate return to service. |
| First boot attempt | The restored host booted successfully. No second boot attempt was required. |
| Validation phase | Host connectivity, Jellyfin and supporting applications, and Proxmox-side NFS availability were checked. |
| Approximately T+2 hours | Core service recovery and validation were complete. |
| Following approximately 3–5 hours | Replaceable in-transit data was reacquired; this did not extend the core service outage. |

## Diagnosis and operational reasoning

The missing shell commands, known destructive deletion event, and live-environment inspection pointed to broad root-filesystem damage. A live USB was chosen to avoid relying on the damaged installed operating system and to preserve access to surviving files.

The root filesystem was mounted from the live environment, and a chroot was used to inspect the installed system. After inspection, in-place repair was rejected in favor of snapshot restoration. The reasoning was to restore a coherent operating-system state quickly while retaining the option to copy recoverable post-snapshot configuration from the damaged filesystem.

The selected Timeshift snapshot was approximately one month old. It resided on a separate physical SSD housed in the same laptop, not on another partition of the failed root device. This physical separation made the snapshot available despite root-filesystem damage, although it does not provide host- or site-level disaster isolation.

## Recovery sequence

1. The host was shut down immediately after root-filesystem damage was recognized.
2. An Ubuntu live USB recovery environment was booted.
3. The installed root filesystem was mounted and entered through a chroot for diagnosis.
4. In-place repair was ruled out as a practical path to rapid, reliable recovery.
5. The approximately month-old Timeshift snapshot on the separate internal SSD was selected.
6. The operating system and snapshot-protected Jellyfin configuration and user files were restored.
7. Recoverable Docker configuration and the NVIDIA transcoding patch for the concurrent-transcoding setup were copied manually.
8. The installed system was booted.
9. Host connectivity and every service running on the laptop were validated.
10. NFS availability was validated from the Proxmox host to confirm cross-node operation.
11. Missing replaceable in-transit data was reacquired.

Exact commands, paths, identifiers, and patch details are intentionally omitted.

## Validation and outcome

The restored system booted successfully on the first attempt. Validation covered:

- basic host connectivity;
- Jellyfin availability;
- Seerr, Bazarr, and Prowlarr availability;
- restored Jellyfin configuration and user files;
- Docker configuration copied during recovery;
- NFS visibility from the Proxmox host;
- broader inter-host connectivity.

The recovery was considered successful because the operating system and required services returned to operation, important Jellyfin state was present, and shared storage connectivity was restored.

The approximately two-hour duration covers detection, initial diagnosis, live-environment work, snapshot restoration, manual copying, reboot, and service validation. It is a historical outcome, not a recovery-time objective or guarantee.

Retrospective assessment indicates that important data was preserved. This outcome must not be confused with the current approximately 52 GB Timeshift snapshot size or interpreted as the amount copied by Timeshift.

## Data loss

Approximately 200 GB of in-transit, replaceable data was lost. Reacquisition took approximately three to five hours and caused noncritical procurement delay. No irreplaceable data was lost.

## Contributing conditions

- An elevated destructive deletion operation could affect the root filesystem broadly.
- The available Timeshift snapshot was approximately one month old at incident time.
- Some post-snapshot Docker configuration and the GPU-transcoding patch required manual copying.
- Snapshot storage was local to the same physical laptop, although on a separate physical drive.

This report does not assign a formal root-cause classification beyond the initiating deletion event identified during incident reconstruction.

## Changes made afterward

### Backup changes

- Timeshift scheduling was increased to daily snapshots.
- Retention was set to seven daily snapshots.
- The current target remains a dedicated, separately mounted approximately 125 GB SSD in the laptop.
- The current completed-downloads directory is explicitly excluded as replaceable data.
- The current manual baseline snapshot is approximately 52 GB.
- Seerr, Bazarr, and Prowlarr configuration is currently documented as included in Timeshift scope, although a restore of that current Docker configuration has not been separately tested.

### Access and command-safety changes

Permissions and ownership were partitioned more deliberately between the superuser and the regular SSH account. The regular account is now oriented toward work outside the root filesystem and toward cleanup, configuration, and monitoring tasks that do not require unrestricted root access.

The intent is to reduce the routine need for elevated filesystem access. The effectiveness of these controls remains unverified.

## Lessons learned

- Rapid shutdown can be appropriate when ongoing destructive filesystem activity is suspected and preserving remaining data is the priority.
- A snapshot on a separate physical drive can remain usable when the root filesystem is damaged.
- Snapshot restoration and selective manual copying can be combined when a coherent OS baseline is needed but some newer configuration remains recoverable.
- Service validation must include cross-host dependencies such as NFS, not only the recovered host's local applications.
- Replaceable transit data should remain outside the protected set when its backup cost exceeds its recovery value.
- Privilege and ownership boundaries should reduce how often routine work requires unrestricted root access.

## Remaining limitations

- The incident timeline is approximate and no command transcript or detailed log archive is available.
- The exact restored-file manifest is unknown.
- Docker configuration is now included in Timeshift scope, but that current configuration has not been restore-tested independently.
- The NVIDIA patch was manually recovered; its current backup status was not separately verified.
- The snapshot target remains inside the same physical laptop.
- No formal RPO, RTO, or guaranteed future recovery time is established.
- Similar destructive deletion has not recurred, but this does not constitute a complete control validation.
