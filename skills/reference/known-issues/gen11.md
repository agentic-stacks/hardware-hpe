# Known Issues: Gen11 Platform

Platform-specific hardware, BIOS, storage controller, and automation issues for HPE ProLiant Gen11 servers.
Covers DL360/380/560/580 Gen11, ML350 Gen11, Alletra, and Compute Gen11 portfolio.

---

### Automation Breaks — Scripts Using SmartStorage OEM Paths Fail on Gen11
**Symptom:** Any script or tool that queries or manages storage via the HPE OEM SmartStorage Redfish namespace fails on Gen11 with HTTP 404. Affected endpoints include: `/redfish/v1/Systems/1/SmartStorage/`, `SmartStorageConfig`, `HpeSmartStorageArrayController`, `HpeSmartStorageLogicalDrive`, and related OEM URIs. Monitoring dashboards show "storage controller unknown" for Gen11 nodes.
**Cause:** Gen11 servers replaced the proprietary HPE SmartStorage model with the DMTF Redfish standard `Storage` model. Gen11 uses new MR (Tri-Mode, 16e/8i) and SR (SAS/SATA) Gen11 controllers that expose Redfish storage exclusively via the standard namespace. The OEM paths do not exist on these systems.
**Workaround:**
1. Map all storage queries to the standard path: `GET /redfish/v1/Systems/1/Storage/` to list controllers.
2. List volumes: `GET /redfish/v1/Systems/1/Storage/{controllerId}/Volumes/`.
3. List physical drives: `GET /redfish/v1/Systems/1/Storage/{controllerId}/Drives/`.
4. Create/modify RAID: `POST /redfish/v1/Systems/1/Storage/{controllerId}/Volumes` with a standard Redfish `Volume` body.
5. Update iLO REST tool (`ilorest`) to version 4.0+ which uses standard Storage paths.
6. For mixed Gen10/Gen11 environments, add a generation check at script startup: inspect `GET /redfish/v1/Systems/1/` → `Model` field, then branch to OEM or standard storage paths accordingly.
**Affected versions:** All Gen11 servers
**Status:** Won't fix — permanent architectural change; migrate to standard Redfish Storage

---

### Gen11 BIOS Attribute Names and OEM Paths Differ From Gen10 — Automation Breaks
**Symptom:** Redfish BIOS automation scripts written for Gen10 fail on Gen11 with "attribute not found" errors, incorrect values, or silent no-ops. Specific failures: scripts setting BIOS attributes by name that have been renamed between generations; scripts reading BIOS OEM settings from `BaseConfigs` at the Gen10 path; HPE OneView Server Profile BIOS templates from Gen10 cannot be applied to Gen11 server profiles without modification.
**Cause:** Two distinct changes break Gen10 BIOS automation on Gen11: (1) The `BaseConfigs` URI changed — Gen10 uses `/redfish/v1/Systems/1/Bios/BaseConfigs`, while Gen10 Plus and Gen11 use `/redfish/v1/Systems/1/Bios/Oem/Hpe/BaseConfigs`. (2) Individual BIOS attribute names were updated between BIOS/ROM registry versions across the platform generations. HPE does not publish a consolidated diff — registry comparison is required.
**Workaround:**
1. Update `BaseConfigs` path references in scripts from `/redfish/v1/Systems/1/Bios/BaseConfigs` to `/redfish/v1/Systems/1/Bios/Oem/Hpe/BaseConfigs` for Gen11 targets.
2. Download the BIOS Attribute Registry for your specific Gen11 BIOS version: HPE Support Center > Server model > BIOS component > download Schema/Registry zip.
3. Compare attribute names between the Gen10 and Gen11 registries using a JSON diff tool to identify renamed or removed attributes.
4. Add generation-aware branching to automation scripts that configure BIOS attributes.
5. For HPE OneView: Gen10 Server Profile templates with BIOS customization must be re-validated against a Gen11 test server before production rollout — do not assume template portability.
**Affected versions:** All Gen11 servers vs. Gen10 BIOS automation
**Status:** Open — permanent; documentation gap in HPE published cross-generation attribute diff

---

### MR Gen11 Controller Firmware Must Be Updated Before RAID Configuration
**Symptom:** Attempting to create or modify RAID volumes on an MR Gen11 controller (e.g., MR408i-o, MR416i-a) immediately after server deployment results in errors: `Operation failed — controller firmware incompatible`, Redfish returns HTTP 400 or 503 on Volume POST, or the controller shows "Degraded" status in iLO 6 despite no drive failures. Issue is most common on new servers received with SPP firmware from factory.
**Cause:** MR Gen11 controllers shipped in early Gen11 production runs contained factory firmware (1.x) that had known stability issues with the PLDM-for-Redfish device enablement layer used by iLO 6 to expose the DMTF Storage model. The controller and iLO 6 must be at compatible minimum firmware versions before RAID operations are reliable via Redfish.
**Workaround:**
1. Before any storage configuration on a new Gen11 server, update iLO 6 to the latest firmware.
2. Update the MR Gen11 controller firmware via SPP: boot SPP ISO and allow firmware baseline to apply.
3. The recommended minimum combined baseline is SPP 2023.10.0 or later — do not perform RAID configuration below this baseline.
4. After SPP update, reboot the server and wait for iLO 6 to fully re-discover the storage controller (allow 3–5 minutes post-boot before querying storage).
5. Verify controller status: `GET /redfish/v1/Systems/1/Storage/` — controller `Status.Health` must be `OK` before creating volumes.
6. If the controller remains Degraded after firmware update, perform an iLO 6 cold reset and re-check.
**Affected versions:** MR Gen11 controllers with firmware below SPP 2023.10.0 baseline
**Status:** Fixed — SPP 2023.10.0 and later include compatible MR controller and iLO 6 firmware

---

### Gen11 Server Watchdog Timer Triggers Unexpected iLO 6 Reset During POST
**Symptom:** In rare cases, a Gen11 server triggers its watchdog timer during POST, causing iLO 6 to reset. This appears as an unexpected server restart with no operator-initiated action. The iLO 6 event log records a watchdog event. The server typically completes POST and boots normally after the reset, but the event causes false alerts in monitoring systems and may interrupt automated provisioning workflows.
**Cause:** A timing race condition between the server's POST sequence and iLO 6's internal watchdog initialization, present in early iLO 6 firmware, causes the watchdog to fire before POST completes the expected acknowledgment handshake. This is most likely to occur when POST is unusually slow (e.g., large memory population, many PCIe devices, or first boot after new hardware installation). Vendor advisory a00138595en_us documents this for both iLO 5 and iLO 6.
**Workaround:**
1. Update iLO 6 firmware to the latest available version — the timing window was narrowed in subsequent firmware releases.
2. If the reset loop repeats (more than two consecutive watchdog events), check POST for underlying hardware errors that are slowing POST completion (memory training failures, PCIe device enumeration delays).
3. Clear the iLO 6 event log after firmware update to suppress stale watchdog alerts in monitoring.
4. For automated provisioning, add a watchdog-event check in the post-deployment validation step and automatically retry if a single watchdog event was recorded during first boot.
**Affected versions:** iLO 6 all versions (reduced frequency in newer releases); Vendor advisory: a00138595en_us
**Status:** Open — partially mitigated in newer firmware; root watchdog race condition not fully resolved

---

### Storage Controller Firmware Update Ordering Requirement — iLO 6 Must Update First
**Symptom:** Applying SPP in an unordered manner (e.g., updating MR/SR Gen11 controller firmware before iLO 6) results in the storage controller becoming unmanageable via Redfish after reboot. iLO 6 reports the controller as "Not Responding" or the Storage endpoint returns no controllers. The physical drives and RAID volumes are intact but cannot be managed until iLO 6 is brought to a compatible version.
**Cause:** The Gen11 storage controller exposes itself to iLO 6 via PLDM-for-Redfish. The controller firmware version dictates the PLDM protocol version and schema version it uses to register with iLO 6. If the controller firmware is ahead of iLO 6's supported PLDM version, iLO 6 cannot parse the controller's registration and drops it from the Redfish tree.
**Workaround:**
1. Always update iLO 6 firmware **first**, before updating any storage controller firmware on Gen11 servers.
2. Recommended update order: (1) iLO 6 firmware, (2) System BIOS/UEFI, (3) Storage controller firmware, (4) Drive firmware.
3. If the controller is already in the unmanageable state: update iLO 6 to the latest version and cold-reboot the server (full power cycle, not warm reset). The controller will re-register with iLO 6 using the compatible PLDM version.
4. Use SPP in guided mode (rather than manual component selection) to automatically resolve update ordering requirements.
5. If using Redfish automation for firmware updates, sequence component updates using the Redfish `UpdateService` task queue and verify each stage before proceeding.
**Affected versions:** Gen11 servers with mixed SPP component update ordering; all iLO 6 versions
**Status:** Open — by design; mitigated by enforcing iLO-first update order
