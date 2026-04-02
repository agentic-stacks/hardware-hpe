# Decision Guide: Firmware Update Strategies

Use this guide to choose the right update method, cadence, and sequencing for HPE server firmware. Apply this before planning a maintenance window or responding to a security advisory.

---

## Update Method Selection

| Approach | How It Works | Server Downtime | When to Use |
|----------|-------------|-----------------|-------------|
| Online via iLO | Update components via iLO web UI or iLOrest while OS is running | Minimal — iLO reset only; reboot required for BIOS/CPLD | Routine iLO, NIC, storage controller updates |
| Online via OS tools | Run SUT (Smart Update Tools) or iLOrest from within the OS | Reboot required for most components | Batch updates to many components at once |
| Offline via SPP ISO | Boot server from Service Pack for ProLiant ISO | Full downtime required | Initial server setup, major version upgrades, recovery |

### Online via iLO

The lowest-disruption method. iLO firmware itself can be updated with only a brief iLO reset (no server reboot). Other components (NICs, storage controllers) require a server reboot to activate the new firmware but the update can be staged online and activated at the next scheduled reboot. Use this for individual targeted updates between scheduled maintenance windows.

Tool: iLO web UI > Firmware & OS Software > Update Firmware, or `ilorest firmware update`.

### Online via OS Tools

HPE Smart Update Tools (SUT) runs inside the OS and can apply an entire SPP bundle or individual components. Most components require a reboot to activate. Use this when updating multiple components at once — SUT handles dependency ordering automatically. The OS must be running and healthy before starting.

Tool: `sut` command or HPE SUM (Smart Update Manager) orchestration.

### Offline via SPP ISO

Boot the server from the Service Pack for ProLiant ISO (downloaded from HPE Support). The ISO inventories the server, identifies applicable updates, and applies them without an OS present. Required for:

- Initial system setup before OS installation
- Situations where the OS cannot be booted (recovery scenarios)
- Major version jumps where online tools may not support the source version
- Ensuring a clean, known baseline across a fleet

---

## Update Frequency

| Update Type | Target Timeline |
|-------------|----------------|
| Critical security patches (CVEs, iLO vulnerabilities) | Apply within the next scheduled maintenance window after release — do not defer beyond 30 days |
| Quarterly SPP release | Evaluate within 30 days of release; apply within 90 days |
| Minor component updates (NIC firmware, drive firmware) | Batch with the quarterly SPP cycle unless a specific bug fix is needed |
| BIOS / CPLD updates (no known issues) | Apply with the quarterly cycle; not every release requires immediate action |

Security advisories from HPE (HPSBHF / HPSBGN series) take precedence over the quarterly cycle. Check HPE Security Bulletin page when a new advisory is published.

---

## Rolling vs Batch Updates

| Strategy | Description | Risk | When to Use |
|----------|-------------|------|-------------|
| Rolling (one at a time) | Update one server, verify health, proceed to next | Lower — failure scope is one server | Production environments, always |
| Batch (multiple simultaneously) | Update several servers in parallel | Higher — a bad update affects multiple servers at once | Lab / dev environments, or when time pressure is high and risk is accepted |

**Recommendation: always use rolling updates in production.**

Rolling procedure:
1. Update the first server (or a designated canary server)
2. Wait for reboot and full OS initialization
3. Verify: health status in iLO, OS boots cleanly, application responds
4. If healthy, proceed to the next server
5. If unhealthy, stop and investigate before continuing

Batch is acceptable in non-production environments where simultaneous downtime is tolerable and a bad outcome affects no customers or SLAs.

---

## Pre-Update Checklist

Complete all items before beginning any firmware update session.

1. **Check compatibility matrix** — Verify the target SPP or component version is listed as supported for the server generation, BIOS version, and OS version. HPE Compatibility Matrix: https://www.hpe.com/us/en/servers/server-components/firmware.html

2. **Check known issues for the target version** — Review the SPP release notes and individual component release notes for regressions or caveats specific to your hardware configuration. See `skills/reference/known-issues/` for curated issues.

3. **Back up current configuration** — Export iLO configuration (iLO web UI > Administration > Backup and Restore), server BIOS settings, and RAID configuration before updating. Some updates reset settings to defaults.

4. **Verify server health before starting** — Check iLO event log and system health summary. Do not update a server with active hardware faults — resolve hardware issues first. A degraded RAID array or a failing drive will be more vulnerable during the reboot that follows firmware updates.

5. **Plan rollback procedure** — Know the previous firmware version. For iLO firmware, a previous version backup exists in iLO's flash (iLO 5/6 support revert to previous). For BIOS, confirm the SPP ISO or individual SoftPaq for the prior version is available before starting.

6. **Schedule maintenance window** — Communicate the window to stakeholders. Even online updates require at least one reboot. For rolling updates across many servers, estimate time as: (per-server update + reboot + verification) x number of servers.

---

## Component Update Order (When Updating Multiple Components)

HPE recommends the following order to avoid dependency conflicts:

1. iLO firmware (update first — it provides the update infrastructure)
2. System ROM (BIOS)
3. CPLD / programmable logic
4. Storage controller firmware
5. Drive firmware (HDD/SSD)
6. NIC / adapter firmware
7. Power management controller (if applicable)

SUT and SPP handle this ordering automatically. If updating manually, follow this sequence.

---

## HPE-Specific Notes

- **SPP versions are tied to server generations** — Gen10 SPPs do not apply to Gen9, and vice versa. Confirm the SPP matches the ProLiant or Synergy generation.
- **iLO license requirements** — iLO Advanced license is not required for firmware updates, but remote console and some iLO REST API features require it.
- **Intelligent Provisioning** — On Gen9/Gen10, Intelligent Provisioning (F10 at POST) includes a firmware update workflow that can target the HPE SDR (Software Delivery Repository) online or a local SPP ISO.
- **Service Pack for ProLiant ISO download** — Available at: https://www.hpe.com/servers/spp — requires HPE Passport account and valid support contract for access to current releases.
