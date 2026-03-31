# Known Issues: Gen10 Platform

Platform-specific hardware, BIOS, and storage issues for HPE ProLiant Gen10 and Gen10 Plus servers.
Covers DL360/380/560/580 Gen10, ML350 Gen10, and Gen10 Plus variants.

---

### Smart Array P408i-a SR Gen10 — Write-Back Cache Loss After Reboot
**Symptom:** Server POST fails with error `1831-Slot X Drive Array – Data in Write-Back Smart Cache has been lost`. This occurs after an OS reboot on systems using the Smart Array P408i-a SR Gen10 controller. Smart Cache must rebuild, causing elevated I/O latency for extended periods. On Hyper-V hosts, VMs may report storage errors during the rebuild window.
**Cause:** A driver incompatibility between certain Smart Array P408i-a SR Gen10 driver versions and the controller firmware causes the cache metadata to become inconsistent across power cycles. The cache write-back journal is not properly flushed before the OS signals shutdown completion, leaving dirty cache entries that the controller cannot reconcile on the next boot.
**Workaround:**
1. Identify the current driver version: Device Manager > Storage Controllers > HPE Smart Array P408i-a SR Gen10.
2. Install the oldest available driver from the HPE Support Center for this controller — specifically the version shipped with SPP 2020.09.0 or earlier.
3. Apply the latest controller firmware (not the driver — firmware is separate) from HPE Support.
4. Avoid pairing the newest driver with older firmware or vice versa — use a matched SPP bundle version.
5. After driver rollback, monitor for Smart Cache rebuild (`smartpqisrc` kernel messages on Linux; Event Viewer on Windows) and verify cache returns to Write-Back mode.
6. As a risk mitigation, switch the cache policy to Write-Through until a stable driver/firmware combination is confirmed.
**Affected versions:** Smart Array P408i-a SR Gen10 with SPP 2021.x and 2022.x driver bundles
**Status:** Fixed — HPE engineering released a corrected driver; install SPP 2023.03.0 or later for the matched fix

---

### NVMe M.2 Drives Not Detected in BIOS or OS When S100i Enabled
**Symptom:** M.2 NVMe drives installed in Gen10 servers do not appear in the BIOS storage menu or in the OS device list. `lspci` / Device Manager show no NVMe controller or drive. The slot appears empty despite a confirmed physical installation. Issue appears or worsens after enabling the HPE Smart Array S100i software RAID controller in BIOS.
**Cause:** Enabling the Smart Array S100i controller reassigns PCIe M.2 slot resources and remaps the M.2 controller mode from NVMe passthrough to AHCI-emulation or RAID mode, depending on BIOS version. This conflicts with NVMe drives that require PCIe passthrough. Additionally, some early Gen10 BIOS versions had a PCIe enumeration bug that failed to detect M.2 NVMe on riser cards under certain slot combinations.
**Workaround:**
1. Enter UEFI System Utilities (F9 at POST) > System Configuration > BIOS/Platform Configuration (RBSU) > Storage Options.
2. Set "Smart Array S100i SW RAID Support" to **Disabled** if M.2 NVMe is the intended storage.
3. Set "M.2 SATA/NVMe" mode to **NVMe** (not RAID or AHCI) for each affected M.2 slot.
4. Save settings and reboot.
5. If the drive still does not appear, update the system BIOS to the latest version from HPE Support and retry.
6. Verify backplane compatibility: U2-type backplanes support NVMe only; U3-type support NVMe/SAS/SATA. Do not install SATA M.2 drives in an NVMe-only slot.
**Affected versions:** Gen10 BIOS versions prior to 2.30 (P89); issue resurfaces when S100i is re-enabled on any BIOS version
**Status:** Partially fixed — BIOS 2.30 resolves the enumeration bug; S100i conflict is by design (configuration issue)

---

### BIOS Settings Reset to Defaults After CMOS Battery Replacement
**Symptom:** After replacing the CMOS/RTC battery on a Gen10 server, the server boots with all BIOS settings returned to factory defaults. Custom boot order, Workload Profile, memory settings, NUMA configuration, and PCI slot assignments are lost. iLO settings are unaffected but the system may fail to boot the OS if the boot order is not restored.
**Cause:** Gen10 servers store all BIOS configuration in NVRAM backed by the CMOS battery. When the battery is removed (or fully depleted), NVRAM contents are lost. Unlike some enterprise platforms that mirror BIOS settings to iLO or a separate NVRAM chip, Gen10 BIOS settings are stored in battery-backed CMOS only.
**Workaround:**
1. Before replacing the battery, export the current BIOS configuration via iLO Redfish: `GET /redfish/v1/Systems/1/Bios` and save the JSON response.
2. Alternatively, use ilorest: `ilorest save --selector Bios --json --output bios_backup.json`.
3. Replace the battery with the server powered off but **still connected to AC power** to maintain NVRAM charge during the swap (varies by chassis — check the Gen10 Maintenance Guide).
4. After replacement, restore settings: `ilorest load --filename bios_backup.json` or re-enter manually via UEFI System Utilities.
5. Verify the Workload Profile and boot order before returning the server to service.
**Affected versions:** All Gen10 and Gen10 Plus server models
**Status:** Open — hardware behavior; preventive export is the only mitigation

---

### FlexibleLOM NIC Recognized but Link Fails After Hot-Firmware-Update to Certain Versions
**Symptom:** After an online firmware update of the FlexibleLOM adapter (Mellanox or Intel-based OCP 2.0 mezzanine) using SPP, the NIC is visible in the OS and reports "Link Up" in the driver but no traffic flows. `ethtool` shows link detected but zero RX/TX packets. Rebooting the OS does not fix it; a full server cold boot is required.
**Cause:** Certain FlexibleLOM firmware versions require a PCIe slot power cycle to reinitialize the controller's internal state machine after a firmware flash. The online update flashes the firmware but leaves the adapter in a transitional state that the PCIe driver cannot fully reset via software. A cold boot forces the PCIe bus to re-enumerate the device, completing initialization.
**Workaround:**
1. After any online FlexibleLOM firmware update, schedule a full server cold boot (power off, wait 10 seconds, power on) rather than a warm reboot.
2. Do not attempt traffic testing until after the cold boot.
3. If the issue is already present: power off the server (not just reboot), wait 10 seconds for PCIe capacitors to discharge, then power on.
4. In SPP-based update workflows, add a cold-boot step to the post-update validation checklist for FlexibleLOM updates.
**Affected versions:** FlexibleLOM adapters updated via SPP 2021.04.0 through 2022.09.0; specific adapter models: HPE FlexibleLOM 10/25Gb 2-port SFP28 (Intel), HPE Synergy 10Gb/25Gb FlexLOM (Mellanox ConnectX-4/5 based)
**Status:** Fixed in SPP 2023.03.0 which includes FlexibleLOM firmware with corrected post-flash reset sequence
