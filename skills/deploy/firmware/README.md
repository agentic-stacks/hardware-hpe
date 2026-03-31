# Deploy: Firmware Management

Firmware update procedures for HPE ProLiant servers using Redfish, iLOrest, and HPE OneView. Covers SPP, component-by-component update ordering, Redfish SimpleUpdate, rollback, and offline update via virtual media.

> **Critical safety rule:** Never update firmware on multiple servers simultaneously. One server at a time, verify health after each component, have a rollback plan before starting.

---

## Prerequisites

- iLO connectivity established — see `skills/foundation/connectivity/README.md`
- Server health is known-good before starting — see `skills/operations/health-check/README.md`
- Compatibility verified — see `skills/reference/compatibility/README.md`
- Credentials with **Administrator** role (firmware update requires administrator)
- Firmware image source accessible (HTTPS server, local path, or SPP ISO)
- Maintenance window confirmed — most component updates require a server reboot

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| UpdateService | `/redfish/v1/UpdateService` | GET |
| Firmware inventory | `/redfish/v1/UpdateService/FirmwareInventory` | GET |
| Software inventory | `/redfish/v1/UpdateService/SoftwareInventory` | GET |
| SimpleUpdate action | `/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate` | POST |
| HPE UpdateService extensions | `/redfish/v1/UpdateService/Oem/Hpe` | GET |
| Task service | `/redfish/v1/TaskService/Tasks` | GET |
| iLO Manager (for iLO version) | `/redfish/v1/Managers/1` | GET |
| System (for BIOS version) | `/redfish/v1/Systems/1` | GET |

---

## SPP (Service Pack for ProLiant) Overview

### What SPP Is

The Service Pack for ProLiant is HPE's primary firmware distribution mechanism for ProLiant servers. It is a tested, bundled firmware update package that includes firmware for every component in a server — all validated together for compatibility by HPE before release.

**Key properties:**

- Distributed as a bootable ISO image and as individual component packages (`.fwpkg` files)
- HPE tests the bundle holistically — all firmware versions in an SPP are validated to work together on the target generation
- Updated quarterly (approximately); named by year and quarter (e.g., SPP 2024.09.0)
- Available from HPE's Support Center (requires HPE account) or via HPE OneView as a firmware baseline

### Components Included in an SPP

| Component | Package Type | Reboot Required |
|---|---|---|
| iLO firmware | iLO firmware image (`.bin` / `.fwpkg`) | No (iLO resets independently) |
| System ROM (BIOS) | System ROM package | Yes |
| NIC firmware | Component firmware package | Usually yes |
| Storage controller firmware | Component firmware package | Usually yes |
| Drive firmware (SSD/HDD) | Drive firmware package | Usually yes |
| Smart Array RAID controller | Component firmware package | Yes |
| MegaRAID / SmartRAID (Gen11) | Component firmware package | Yes |
| Power management controller | Component firmware package | Yes |
| Chassis firmware | Component firmware package | Yes |
| Intel/AMD CPU microcode | Embedded in System ROM | Yes (applied during POST) |

### SPP Naming and Versioning

```
SPP YYYY.QQ.patch
     ^^^^  Year
          ^^  Quarter (03=Q1, 06=Q2, 09=Q3, 11=Q4)
             ^ Minor patch number (0 = first release of that quarter)
```

Example: `SPP 2024.09.0` = September 2024 (Q3), initial release.

Component packages extracted from SPP are versioned independently per-component. The SPP version is the "bundle" identifier; individual component version strings (e.g., `iLO 5 v3.06`, `U46 v2.80`) reflect the component's own versioning scheme.

### Online vs Offline SPP

| Mode | How | When to Use |
|---|---|---|
| Online (in-band) | Smart Update Tools (SUT) runs inside the OS and stages updates to iLO | Server is running and OS is accessible; avoids full outage |
| Online (out-of-band) | Redfish SimpleUpdate or iLOrest pushes individual component packages directly to iLO | OS is not needed; preferred for automation |
| Offline | Boot from SPP ISO (virtual media or physical DVD/USB) | OS is not accessible; or need to update components that cannot be updated online |

For automation, use the out-of-band Redfish approach (individual `.fwpkg` files via SimpleUpdate) rather than booting the full SPP ISO. The ISO-based offline method is a fallback for problematic servers or when network paths to the firmware repository are unavailable.

---

## Firmware Inventory Collection

### Via Redfish

List all firmware components and their versions:

```bash
# List all firmware inventory member URIs
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/UpdateService/FirmwareInventory \
  | jq '.Members[].@odata.id'

# Get full inventory with name and version for each component
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/UpdateService/FirmwareInventory \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{Name, Version, Updateable, Status}'
  done

# Quick summary — name and version for every component
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/UpdateService/FirmwareInventory \
  | jq '[.Members[] | {name: .Name, version: .Version}]'
```

> On iLO 6 (Gen11), use `$expand=.` to retrieve all members in one request:
>
> ```bash
> curl -sk -u admin:password \
>   "https://192.168.1.101/redfish/v1/UpdateService/FirmwareInventory?\$expand=." \
>   | jq '.Members[] | {Name, Version, Updateable}'
> ```

### Key Components to Track

| Component | Redfish `Name` (typical) | Example Version | Reboot Required |
|---|---|---|---|
| iLO 5 firmware | `iLO 5` | `3.06 Oct 30 2024` | No |
| iLO 6 firmware | `iLO 6` | `1.57 Oct 30 2024` | No |
| System ROM (BIOS) | `System ROM` | `U46 v2.80 (03/15/2024)` | Yes |
| NIC firmware | `HPE Ethernet 10/25Gb ...` | `20.28.60` | Yes |
| Smart Array controller (Gen10) | `HPE Smart Array ...` | `5.20` | Yes |
| MegaRAID / SmartRAID (Gen11) | `HPE MR Gen11 ...` | `52.24.0-4726` | Yes |
| Drive firmware | `MB1200JVXLR` (model-based) | `HPD8` | Yes |
| Power management | `System Programmable Logic Device` | `0x27` | Yes |
| Chassis firmware | `HPE ProLiant ...` | varies | Yes |

### Via iLOrest

```bash
# Connect and list all firmware
ilorest login https://192.168.1.101 -u admin -p password
ilorest list --select SoftwareInventory.
ilorest logout

# Firmware integrity check (validates installed firmware against HPE signing keys)
ilorest firmwareintegritycheck --url https://192.168.1.101 -u admin -p password
```

### Compliance Checking Against a Baseline

Capture inventory as a snapshot before updating, and after updating to verify changes:

```bash
# Snapshot current firmware state
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/UpdateService/FirmwareInventory?\$expand=." \
  | jq '[.Members[] | {Name, Version}] | sort_by(.Name)' \
  > firmware-inventory-$(date +%Y%m%d-%H%M%S).json

# Compare two snapshots to see what changed
diff \
  <(jq -S . firmware-inventory-before.json) \
  <(jq -S . firmware-inventory-after.json)
```

---

## Recommended Update Ordering

Always follow this sequence. The order is not arbitrary:

| Step | Component | Reason |
|---|---|---|
| 1 | iLO firmware | iLO is the management plane and runs the update service itself. Updating iLO first ensures the update infrastructure is current before updating anything else. |
| 2 | System ROM (BIOS) | BIOS microcode and initialization sequences affect how all other components initialize. Update before storage and NIC firmware. |
| 3 | Storage controller firmware | Storage controllers depend on BIOS initialization. Update storage before drives. |
| 4 | Drive firmware (SSD/HDD) | Update after storage controller — controller firmware may affect how drive updates are applied. |
| 5 | NIC / HBA firmware | Network adapters are last; updating them last minimizes risk of losing management connectivity during earlier steps. |
| — | Health check after each step | Run `skills/operations/health-check` after each component update to catch problems before proceeding. |

> **Never skip the health check between components.** A failed storage controller update, if missed, can cause data corruption when the server reboots with the next update. Verify clean health at each step.

---

## Update Procedures by Component

### 1. iLO Firmware Update (No Server Reboot Required)

iLO firmware updates without rebooting the server OS. After upload, iLO resets itself — the connection to iLO drops for approximately 3–5 minutes while it comes back up. The server continues running normally.

**Via Redfish SimpleUpdate:**

```bash
curl -k -X POST \
  https://192.168.1.101/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "ImageURI": "https://firmware-server.example.com/ilo5_306.bin",
    "Targets": ["/redfish/v1/Managers/1"]
  }'
```

**Via iLOrest (preferred — handles staging and confirmation automatically):**

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest flashfwpkg /path/to/ilo5_306.fwpkg
ilorest logout
```

**Wait for iLO to come back up:**

```bash
# Poll until iLO responds again (allow up to 10 minutes)
for i in $(seq 1 40); do
  if curl -sk -u admin:password \
    https://192.168.1.101/redfish/v1/Managers/1 \
    | jq -e '.FirmwareVersion' > /dev/null 2>&1; then
    echo "iLO is back online"
    break
  fi
  echo "Waiting for iLO... attempt $i/40"
  sleep 15
done
```

**Verify new iLO version:**

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1 \
  | jq '.FirmwareVersion'
```

---

### 2. System ROM (BIOS) Update (Requires Server Reboot)

System ROM updates are staged to iLO and applied when the server reboots. The server OS continues running after staging — the update takes effect on the next reboot.

**Stage via Redfish SimpleUpdate:**

```bash
curl -k -X POST \
  https://192.168.1.101/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "ImageURI": "https://firmware-server.example.com/U46_2.80_10_15_2024.fwpkg",
    "Targets": ["/redfish/v1/Systems/1/Bios"]
  }'
```

**Via iLOrest:**

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest flashfwpkg /path/to/U46_2.80.fwpkg --forceupload
ilorest logout
```

**Reboot to apply:**

```bash
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "ForceRestart"}'
```

**Verify BIOS version after reboot** (wait for POST to complete first):

```bash
# Poll until POST complete
until curl -sk -u admin:password https://192.168.1.101/redfish/v1/Systems/1 \
  | jq -e '.Oem.Hpe.PostState == "FinishedPost"' > /dev/null 2>&1; do
  echo "Waiting for POST to complete..."
  sleep 15
done

# Check new BIOS version
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '.BiosVersion'
```

---

### 3. NIC / HBA Firmware Update

NIC firmware updates typically require a reboot. Stage the update then reboot.

**Via Redfish SimpleUpdate:**

```bash
curl -k -X POST \
  https://192.168.1.101/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "ImageURI": "https://firmware-server.example.com/HPE_NIC_20.28.60.fwpkg"
  }'
```

> The `Targets` field is optional for NIC updates — iLO will auto-detect the applicable component from the package metadata. If multiple NIC adapters are present and you need to target a specific one, use the NIC member URI from the FirmwareInventory response.

**Via iLOrest:**

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest flashfwpkg /path/to/HPE_NIC_20.28.60.fwpkg
ilorest logout
```

Reboot to apply (same `ComputerSystem.Reset` call as System ROM above). Verify NIC firmware version after reboot by checking `FirmwareInventory` for the NIC entry.

---

### 4. Storage Controller Firmware Update

**Check controller cache status before updating.** If the controller has a write cache with dirty data, a firmware update may reset the cache — potentially causing data loss. Ensure the cache is clean or flush it before proceeding.

**Check cache status (iLO 5 / Gen10 — SmartArray OEM path):**

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0 \
  | jq '.CacheModuleStatus, .CacheMemorySizeMiB'
```

**Check cache status (iLO 6 / Gen11 — standard Storage path):**

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/DA000000 \
  | jq '.StorageControllers[0] | {CacheSummary}'
```

**Stage via Redfish SimpleUpdate:**

```bash
curl -k -X POST \
  https://192.168.1.101/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "ImageURI": "https://firmware-server.example.com/HPE_SmartArray_5.20.fwpkg"
  }'
```

**Via iLOrest:**

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest flashfwpkg /path/to/HPE_SmartArray_5.20.fwpkg
ilorest logout
```

Reboot to apply. After reboot, verify the controller firmware version in `FirmwareInventory` and re-check controller health status before declaring success.

---

## Redfish SimpleUpdate — Full Reference

### Action Endpoint

```
POST /redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate
```

### Payload Fields

| Field | Required | Description |
|---|---|---|
| `ImageURI` | Yes | HTTPS (or HTTP) URI to the firmware image file. iLO fetches directly. |
| `Targets` | No | Array of Redfish URIs targeting which component to update. If omitted, iLO detects from package metadata. |
| `TransferProtocol` | No | `HTTPS` (default), `HTTP`, `NFS`, `CIFS`. |
| `Username` | No | Credential for authenticated image URI. |
| `Password` | No | Credential for authenticated image URI. |

### Full Example with Explicit Target

```bash
curl -k -X POST \
  https://192.168.1.101/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "ImageURI": "https://firmware-server.example.com/ilo5_306.bin",
    "Targets": ["/redfish/v1/Managers/1"]
  }'
```

### Monitoring Update Progress via Task Service

SimpleUpdate returns an HTTP `202 Accepted` with a `Location` header pointing to a task URI. Poll this task to monitor completion:

```bash
# Capture the task URI from the response Location header
TASK_URI=$(curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/UpdateService/Actions/UpdateService.SimpleUpdate \
  -H "Content-Type: application/json" \
  -d '{"ImageURI": "https://firmware-server.example.com/U46_2.80.fwpkg"}' \
  -D - -o /dev/null \
  | grep -i '^location:' | awk '{print $2}' | tr -d '\r')

echo "Task URI: $TASK_URI"

# Poll task until complete
while true; do
  TASK=$(curl -sk -u admin:password "https://192.168.1.101${TASK_URI}")
  STATE=$(echo "$TASK" | jq -r '.TaskState')
  PCT=$(echo "$TASK" | jq -r '.PercentComplete // "unknown"')
  echo "State: $STATE  Progress: ${PCT}%"
  if [[ "$STATE" == "Completed" || "$STATE" == "Exception" || "$STATE" == "Killed" ]]; then
    echo "$TASK" | jq '{TaskState, TaskStatus, Messages}'
    break
  fi
  sleep 10
done
```

### Task States

| TaskState | Meaning |
|---|---|
| `New` | Task created, not yet started |
| `Starting` | iLO is fetching or staging the image |
| `Running` | Firmware is being applied |
| `Completed` | Update finished successfully |
| `Exception` | Update failed — check `Messages` for error details |
| `Killed` | Task was cancelled |

---

## iLOrest Firmware Commands

```bash
# Connect to iLO
ilorest login https://192.168.1.101 -u admin -p password

# List all installed firmware versions
ilorest list --select SoftwareInventory.

# Upload and apply a single firmware package
ilorest flashfwpkg /path/to/component.fwpkg

# Force upload even if version appears current (useful for re-applying known-good firmware)
ilorest flashfwpkg /path/to/component.fwpkg --forceupload

# List pending firmware updates (staged, not yet applied)
ilorest list --select HpeiLOUpdateServiceExt.

# Run firmware integrity check (verifies HPE signing on installed firmware)
ilorest firmwareintegritycheck

# Disconnect
ilorest logout
```

> `ilorest flashfwpkg` is the preferred command for individual package updates. It reads the package metadata, selects the correct component, stages the update to iLO, and reports status — handling the staging logic automatically.

---

## OneView Firmware Baselines

When managing a fleet, use HPE OneView firmware baselines instead of per-server scripting.

### Creating a Firmware Baseline from SPP

1. Download the SPP ISO from HPE Support Center
2. In OneView UI: **OneView > Firmware Bundles > Add firmware bundle**
3. Upload the SPP ISO — OneView extracts component packages automatically
4. The baseline appears in the firmware bundles list with the SPP version as its name

Via OneView REST API:

```bash
# Upload SPP ISO as firmware bundle (multipart upload)
curl -sk -X POST \
  -H "X-Auth-Token: $OV_TOKEN" \
  -H "uploadfilename: SPP_2024.09.0.iso" \
  -F "file=@/path/to/SPP_2024.09.0.iso" \
  https://192.168.1.10/rest/firmware-bundles
```

### Compliance Checking Across Fleet

```bash
# List servers and their firmware compliance state
curl -sk -H "X-Auth-Token: $OV_TOKEN" \
  https://192.168.1.10/rest/server-profiles \
  | jq '.members[] | {name: .name, firmwareComplianceState: .firmware.complianceState}'
```

Compliance states: `Compliant` (matches baseline), `NonCompliant` (one or more components differ), `Unknown` (baseline not assigned or not scanned yet).

### Applying a Baseline via Server Profile

```bash
# PATCH server profile to assign a firmware baseline
curl -sk -X PATCH \
  -H "X-Auth-Token: $OV_TOKEN" \
  -H "Content-Type: application/json" \
  https://192.168.1.10/rest/server-profiles/<profile-id> \
  -d '{
    "firmware": {
      "manageFirmware": true,
      "firmwareBaselineUri": "/rest/firmware-drivers/SPP_2024_09_0",
      "firmwareInstallType": "FirmwareAndOSDrivers",
      "forceInstallFirmware": false
    }
  }'
```

### Staged vs Immediate Updates

| Update Type | `firmwareInstallType` | When Applied |
|---|---|---|
| Stage only | `FirmwareOnly` | Staged to iLO; applied on next server reboot |
| Immediate | `FirmwareAndOSDrivers` | Applied immediately; may trigger reboot if required |
| OS drivers too | `FirmwareAndOSDrivers` | Includes OS driver packages from SPP |

Use `forceInstallFirmware: true` only when deliberately downgrading or re-applying a version that appears current.

---

## Firmware Rollback

### When Rollback Is Available

| Component | Rollback Possible | Notes |
|---|---|---|
| iLO firmware | Yes | iLO keeps the previous firmware image in its flash and can revert |
| System ROM (BIOS) | Yes | System ROM partition retains previous version for one-step rollback |
| NIC firmware | Generally No | Most NIC firmware updates are one-way |
| Storage controller firmware | Generally No | Smart Array and MR controllers do not support firmware rollback |
| Drive firmware | No | Drive firmware is not reversible |

### iLO Firmware Rollback

iLO maintains a backup image of the previous firmware version. To revert:

```bash
# Trigger iLO firmware rollback via HPE OEM action
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Managers/1/Actions/Oem/Hpe/HpeiLO.iLOFirmwareRecovery \
  -H "Content-Type: application/json" \
  -d '{}'
```

Alternatively via iLO web UI: **Firmware & OS Software > iLO Firmware > Rollback**.

After rollback, iLO resets and comes back on the previous version. Verify:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1 \
  | jq '.FirmwareVersion'
```

### System ROM (BIOS) Rollback

System ROM maintains a redundant partition. To revert via iLOrest:

```bash
ilorest login https://192.168.1.101 -u admin -p password
# Select the backup ROM partition for next boot
ilorest rawpatch /redfish/v1/Systems/1/Bios/Settings \
  --body '{"Attributes": {"BootOrderPolicy": "AttemptOnce"}}'
ilorest logout
```

For explicit ROM partition selection, use iLO web UI: **Firmware & OS Software > System ROM > Redundant System ROM**.

### When Rollback Is Not Possible

- Drive firmware — never reversible; drives do not retain previous firmware
- NIC and HBA firmware — almost always one-way; check component release notes before updating
- Storage controller firmware — not supported; verify thoroughly before applying
- Any component where the previous version is no longer stored on-device

For components without rollback support, your recovery path is re-flashing the component with an older known-good package via SimpleUpdate — which requires having that older package available.

---

## Offline Update via Virtual Media

### When to Use

- iLO is reachable but no network path exists from iLO to the firmware image repository
- Component requires offline environment (OS must not be running during update)
- Recovering a server where network firmware update has failed and left firmware in a bad state
- Full SPP sweep preferred over individual component updates

### Mount SPP ISO via Virtual Media

```bash
# Connect virtual media (iLO mounts the ISO as a virtual DVD)
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.InsertMedia \
  -H "Content-Type: application/json" \
  -d '{
    "Image": "https://fileserver.example.com/SPP_2024.09.0.iso",
    "Oem": {
      "Hpe": {
        "BootOnNextServerReset": true
      }
    }
  }'
```

> `VirtualMedia/2` is typically the DVD drive slot. Verify with `GET /redfish/v1/Managers/1/VirtualMedia` to find the correct ID.

### Set One-Time Boot to Virtual CD

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "Cd", "BootSourceOverrideEnabled": "Once"}}'
```

### Reboot to Boot from SPP ISO

```bash
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "ForceRestart"}'
```

The server boots into the SPP maintenance environment. Use the iLO remote console (`skills/operations/remote-console`) to interact with the SPP GUI or CLI to select components to update. The SPP environment runs the update, then prompts for reboot.

After offline update completes and server reboots to OS:

1. Verify firmware inventory via Redfish (`FirmwareInventory`)
2. Run health check (`skills/operations/health-check`)
3. Eject virtual media

```bash
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.EjectMedia \
  -H "Content-Type: application/json" \
  -d '{}'
```

---

## Safety Rules

These rules apply to every firmware update operation without exception.

**1. One server at a time.** Never run firmware updates on multiple servers simultaneously. If one server requires recovery, you need the capacity to absorb the downtime — parallel failures multiply impact and exhaust resources.

**2. Verify health before starting.** Run the health check skill before any firmware operation. Do not update a server with existing hardware warnings — resolve them first. Firmware updates on an already-degraded server can convert recoverable problems into unrecoverable ones.

**3. Verify health after each component.** After updating each component, run a health check before moving to the next. A failed update may not be immediately obvious — catching it between components allows targeted recovery before the next update makes the situation worse.

**4. Always check compatibility first.** Confirm the firmware version is supported on the server generation, OS, and existing software stack. See `skills/reference/compatibility/README.md`.

**5. Have a rollback plan before starting.** Identify which components support rollback. For those that do not, confirm you have the previous firmware package available and can re-flash if needed.

**6. Never interrupt a firmware update in progress.** Interrupting a flash operation (power loss, forced reset, network disconnect) can corrupt component firmware and may brick the component. iLO has a watchdog to survive interrupted iLO updates, but System ROM and storage controller corruption requires physical intervention.

**7. Capture inventory before and after.** Snapshot `FirmwareInventory` before starting. Diff before-and-after to confirm exactly what changed. Keep both snapshots for audit trail and change documentation.

---

## Firmware Update Workflow (End-to-End)

```
1. Pre-flight
   ├── health-check skill → confirm server is healthy
   ├── compatibility skill → confirm firmware versions are compatible
   ├── snapshot FirmwareInventory → baseline record
   └── confirm maintenance window

2. Update iLO firmware
   ├── SimpleUpdate or ilorest flashfwpkg
   ├── Wait for iLO to reconnect (3–5 min)
   └── health-check → confirm iLO healthy

3. Update System ROM
   ├── SimpleUpdate or ilorest flashfwpkg
   ├── Reboot server
   ├── Wait for POST complete
   └── health-check → confirm no POST errors

4. Update storage controller firmware
   ├── Verify cache is clean
   ├── SimpleUpdate or ilorest flashfwpkg
   ├── Reboot server
   └── health-check → confirm controller healthy

5. Update NIC / HBA firmware
   ├── SimpleUpdate or ilorest flashfwpkg
   ├── Reboot server
   └── health-check → confirm NIC healthy

6. Post-update verification
   ├── snapshot FirmwareInventory → post-update record
   ├── diff before vs after → confirm expected changes
   ├── health-check → full server health pass
   └── return server to service
```

---

## Decision Guide for Agents

| Situation | Action |
|---|---|
| Need current firmware versions | `GET /redfish/v1/UpdateService/FirmwareInventory` — list all components |
| Firmware image is on an HTTPS server | Use Redfish SimpleUpdate with `ImageURI` pointing to the HTTPS server |
| Firmware image is a local `.fwpkg` file | Use `ilorest flashfwpkg /path/to/file.fwpkg` |
| iLO firmware update needed | SimpleUpdate with `Targets: ["/redfish/v1/Managers/1"]`; no reboot needed |
| System ROM update needed | SimpleUpdate then `ComputerSystem.Reset` with `ForceRestart` |
| Update failed mid-way | Check task state via TaskService; check IML for hardware errors; attempt rollback if component supports it |
| Storage controller cache has dirty data | Do not update storage controller firmware until cache is clean; let writes flush or force-flush via controller utility |
| No network path from iLO to firmware repo | Use offline SPP ISO via virtual media |
| Fleet-wide firmware compliance needed | Use OneView firmware baselines — assign baseline to server profiles, check `complianceState` |
| iLO update seems stuck | Check IML for errors; allow 10 minutes before concluding failure; iLO resets take 3–5 minutes normally |
| Need to verify update actually applied | Check `FirmwareInventory` for the component after update and reboot; diff against pre-update snapshot |
