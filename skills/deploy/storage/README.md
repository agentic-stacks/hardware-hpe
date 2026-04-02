# Deploy: Storage and RAID Configuration

RAID and disk management for HPE ProLiant Gen10 and Gen11 servers. Covers controller discovery, physical drive inventory, RAID level selection, logical drive creation and deletion, hot spares, cache configuration, secure erase, and OneView storage profiles.

**Critical:** The storage API is completely different between Gen10 and Gen11. Always identify the server generation before issuing storage commands. Using Gen10 paths on a Gen11 server (or vice versa) will return 404 or operate on the wrong subsystem.

---

## Gen10 vs Gen11 Storage Architecture

| Attribute | Gen10 (iLO 5) | Gen11 (iLO 6) |
|---|---|---|
| Controller family | HPE Smart Array (P408i-a, P816i-a, E208i-a, etc.) | Broadcom MR (MegaRAID) and SR (SmartRAID) |
| API namespace | HPE OEM — `HpeSmartStorage*` | DMTF standard Redfish Storage model |
| Storage root | `/redfish/v1/Systems/1/SmartStorage/` | `/redfish/v1/Systems/1/Storage/` |
| Controller list | `/redfish/v1/Systems/1/SmartStorage/ArrayControllers/` | `/redfish/v1/Systems/1/Storage/` (members are controllers) |
| Logical drives | `/redfish/v1/Systems/1/SmartStorage/ArrayControllers/{id}/LogicalDrives/` | `/redfish/v1/Systems/1/Storage/{id}/Volumes/` |
| Physical drives | `/redfish/v1/Systems/1/SmartStorage/ArrayControllers/{id}/DiskDrives/` | `/redfish/v1/Systems/1/Storage/{id}/Drives/{driveId}` |
| Create RAID | POST to OEM LogicalDrives URI | POST to standard Volumes URI |
| iLOrest type | `SmartStorageArrayController.` | `StorageController.` |
| OData type prefix | `HpeSmartStorage*` | `Storage.v*`, `Volume.v*`, `Drive.v*` |

Gen11 removed the `SmartStorage` OEM namespace entirely. If you query `/redfish/v1/Systems/1/SmartStorage/` on a Gen11 server, you will receive a 404.

---

## Identifying Server Generation

Before any storage operation, confirm which generation you are on.

```bash
# Method 1: Check iLO generation from manager resource
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/ \
  | jq '{Model, FirmwareVersion}'
# iLO 5.x.x = Gen10; iLO 6.x.x = Gen11

# Method 2: Check system model string
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/ \
  | jq '.Model'
# "ProLiant DL380 Gen10" or "ProLiant DL380 Gen11"

# Method 3: Probe both paths — Gen10 has SmartStorage, Gen11 has only Storage
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ \
  | jq '.@odata.type'
# If returns HpeSmartStorage type => Gen10
# If returns 404 => Gen11, use /Systems/1/Storage/ instead
```

Via iLOrest:

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest get Model --select ComputerSystem.
ilorest logout
```

---

## Controller Discovery

### Gen11 — Standard DMTF Redfish

```bash
# List all storage controllers (each member is a controller)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/ \
  | jq '.Members[].@odata.id'

# Example output:
# "/redfish/v1/Systems/1/Storage/DA000000"
# "/redfish/v1/Systems/1/Storage/DE040000"

# Get details for a specific controller
CTRL_ID="DA000000"
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/ \
  | jq '{
      Id,
      Name,
      Status,
      StorageControllers: (.StorageControllers[]? | {
        Model,
        FirmwareVersion,
        SupportedRAIDTypes,
        CacheSummary
      }),
      Drives: (.Drives | length),
      Volumes: (.Volumes."@odata.id")
    }'

# Controller health summary across all controllers
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/ \
  | jq '.Members[].@odata.id' | while read -r uri; do
    uri=$(echo "$uri" | tr -d '"')
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq --arg uri "$uri" '{controller: $uri, status: .Status, name: .Name}'
  done
```

Via iLOrest (Gen11):

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest select Storage.
ilorest get Name Status StorageControllers
ilorest logout
```

### Gen10 — HPE SmartStorage OEM

```bash
# List array controllers
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/ \
  | jq '.Members[].@odata.id'

# Example output:
# "/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/"

# Get controller details
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/ \
  | jq '{
      Model,
      SerialNumber,
      FirmwareVersion: .FirmwareVersion.Current.VersionString,
      Status,
      CacheMemorySizeMiB,
      EncryptionEnabled,
      LocationFormat,
      BackupPowerSourceStatus
    }'
```

Key controller fields to check:

| Field | Where | What It Means |
|---|---|---|
| `Status.Health` | Both | `OK`, `Warning`, `Critical` — overall controller health |
| `Status.State` | Both | `Enabled` = active and functional |
| `FirmwareVersion` | Both | Controller firmware — update via SPP if outdated |
| `CacheSummary.TotalCacheSizeMiB` | Gen11 | Cache RAM size in MiB |
| `CacheMemorySizeMiB` | Gen10 OEM | Cache RAM size in MiB |
| `BackupPowerSourceStatus` | Gen10 OEM | `Present` = capacitor/battery healthy; `NotPresent` = cache write-through forced |
| `CacheSummary.PersistentCacheSizeMiB` | Gen11 | Non-volatile cache (capacitor-backed) |
| `SupportedRAIDTypes` | Gen11 | Array of RAID levels this controller supports |

---

## Physical Drive Inventory

### Gen11

```bash
CTRL_ID="DA000000"

# List all drives on a controller
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/ \
  | jq '.Drives[].@odata.id'

# Get details for a specific drive
DRIVE_URI="/redfish/v1/Systems/1/Storage/DA000000/Drives/0/"
curl -sk -u admin:password \
  https://192.168.1.101${DRIVE_URI} \
  | jq '{
      Name,
      MediaType,
      Protocol,
      CapacityBytes,
      RotationSpeedRPM,
      SerialNumber,
      Model,
      Manufacturer,
      Status,
      PhysicalLocation: .PhysicalLocation.PartLocation,
      PredictiveFailureAnalysis: .Oem.Hpe.DriveStatus
    }'

# Full drive inventory — all drives on all controllers
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/ \
  | jq -r '.Members[].@odata.id' | while read -r ctrl_uri; do
    curl -sk -u admin:password "https://192.168.1.101${ctrl_uri}" \
      | jq -r '.Drives[]?.@odata.id'
  done | while read -r drive_uri; do
    curl -sk -u admin:password "https://192.168.1.101${drive_uri}" \
      | jq --arg uri "$drive_uri" '{
          uri: $uri,
          name: .Name,
          media: .MediaType,
          protocol: .Protocol,
          gb: (.CapacityBytes / 1073741824 | round),
          status: .Status.Health,
          serial: .SerialNumber
        }'
  done
```

### Gen10

```bash
# List physical disk drives on controller 0
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/DiskDrives/ \
  | jq '.Members[].@odata.id'

# Get drive details
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/DiskDrives/0/ \
  | jq '{
      Name,
      Model,
      SerialNumber,
      CapacityGB,
      InterfaceType,
      MediaType,
      RotationalSpeedRpm,
      Status,
      Location,
      CurrentTemperatureCelsius,
      MinimumGoodFirmwareVersion
    }'

# NVMe drives appear under a separate path on Gen10
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/StorageEnclosures/ \
  | jq '.'
```

Physical drive fields reference:

| Field | Values | Meaning |
|---|---|---|
| `MediaType` | `HDD`, `SSD`, `SMR` | Drive technology |
| `Protocol` | `SAS`, `SATA`, `NVMe` | Interface protocol |
| `Status.Health` | `OK`, `Warning`, `Critical` | Current health |
| `CapacityBytes` / `CapacityGB` | integer | Raw capacity |
| `RotationSpeedRPM` | integer or `0` | `0` for SSD/NVMe |
| `PredictiveFailureAnalysis` (Gen11 OEM) | `OK`, `Failure Predicted` | SMART status |
| `PhysicalLocation` | bay/slot string | Physical location in enclosure |

Drive health indicators — escalate if you see:

- `Status.Health: "Warning"` — drive degraded, replacement recommended
- `Status.Health: "Critical"` — drive failed or imminent failure
- `PredictiveFailureAnalysis: "Failure Predicted"` — SMART threshold exceeded; replace before array fails

---

## RAID Level Decision Guide

| RAID | Min Drives | Fault Tolerance | Usable Capacity | Read Perf | Write Perf | Best For |
|---|---|---|---|---|---|---|
| 0 | 1 | None (data loss on any failure) | 100% | High | High | Temp scratch space, caches |
| 1 | 2 | 1 drive | 50% | High | Medium | Boot/OS volume |
| 5 | 3 | 1 drive | (N-1)/N | High | Medium | General workloads, read-heavy |
| 6 | 4 | 2 drives | (N-2)/N | High | Low | Critical data, archive |
| 10 | 4 | 1 drive per mirror pair | 50% | Very High | High | Databases, IOPS-intensive |
| 50 | 6 | 1 drive per RAID-5 group | ~83% | Very High | Medium | Large arrays, balanced |
| 60 | 8 | 2 drives per RAID-6 group | ~75% | Very High | Low | Large arrays, maximum protection |

**Selection heuristics:**

- Boot/OS: RAID 1 (2 drives) — simplicity and predictability
- Database (OLTP): RAID 10 — write performance and fault tolerance
- File/object storage: RAID 6 — tolerate 2 simultaneous failures
- High-capacity, write-light: RAID 60 — maximum capacity with strong protection
- Avoid RAID 5 for arrays larger than 6 drives — rebuild risk increases with array size
- Never use RAID 0 for persistent application data

**Capacity calculation examples** (with 8 × 1.8 TB drives):

| RAID | Usable Capacity |
|---|---|
| RAID 0 | 14.4 TB |
| RAID 1 | 1.8 TB |
| RAID 5 | 12.6 TB |
| RAID 6 | 10.8 TB |
| RAID 10 | 7.2 TB |
| RAID 60 | 10.8 TB |

---

## Creating Logical Drives

**WARNING: Creating a logical drive on drives that already contain data will destroy that data. Verify all target drives are unassigned (show as `Unconfigured Good` or equivalent) before proceeding. This operation cannot be undone without rebuilding from backup.**

### Gen11 — Standard Redfish POST to Volumes

```bash
CTRL_ID="DA000000"

# First, identify available (unassigned) drives
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/ \
  | jq '.Drives[].@odata.id'
# Note the @odata.id values for drives you intend to use

# Create a RAID 5 logical drive across 4 drives
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/ \
  -d '{
    "RAIDType": "RAID5",
    "Drives": [
      {"@odata.id": "/redfish/v1/Systems/1/Storage/DA000000/Drives/0"},
      {"@odata.id": "/redfish/v1/Systems/1/Storage/DA000000/Drives/1"},
      {"@odata.id": "/redfish/v1/Systems/1/Storage/DA000000/Drives/2"},
      {"@odata.id": "/redfish/v1/Systems/1/Storage/DA000000/Drives/3"}
    ],
    "DisplayName": "DataVol-RAID5",
    "ReadCachePolicy": "ReadAhead",
    "WriteCachePolicy": "WriteBackWithBBU"
  }' | jq '{status: .error // "Created", location: .["@odata.id"]}'

# Create a RAID 10 logical drive across 4 drives
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/ \
  -d '{
    "RAIDType": "RAID10",
    "Drives": [
      {"@odata.id": "/redfish/v1/Systems/1/Storage/DA000000/Drives/4"},
      {"@odata.id": "/redfish/v1/Systems/1/Storage/DA000000/Drives/5"},
      {"@odata.id": "/redfish/v1/Systems/1/Storage/DA000000/Drives/6"},
      {"@odata.id": "/redfish/v1/Systems/1/Storage/DA000000/Drives/7"}
    ],
    "DisplayName": "DBVol-RAID10",
    "ReadCachePolicy": "ReadAhead",
    "WriteCachePolicy": "WriteBackWithBBU",
    "StripSizeBytes": 65536
  }' | jq '.'
```

Valid `RAIDType` values for Gen11 MR/SR controllers: `RAID0`, `RAID1`, `RAID5`, `RAID6`, `RAID10`, `RAID50`, `RAID60`

Valid `WriteCachePolicy` values:

| Value | Meaning |
|---|---|
| `WriteBackWithBBU` | Write-back when backup power (capacitor/battery) is present; falls back to write-through if BBU fails |
| `AlwaysWriteBack` | Write-back always (risk of data loss on power failure without BBU) |
| `WriteThrough` | Write-through always (safe but slower) |

### Gen10 — HPE OEM SmartStorage

```bash
CTRL_URI="/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0"

# List unconfigured drives first
curl -sk -u admin:password \
  https://192.168.1.101${CTRL_URI}/DiskDrives/ \
  | jq '.Members[] | select(.Status.State == "Enabled") | {id: .@odata.id, loc: .Location}'

# Create RAID 5 logical drive (Gen10 OEM schema)
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  https://192.168.1.101${CTRL_URI}/LogicalDrives/ \
  -d '{
    "Raid": "Raid5",
    "DataDrives": [
      "/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/DiskDrives/0/",
      "/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/DiskDrives/1/",
      "/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/DiskDrives/2/",
      "/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/DiskDrives/3/"
    ],
    "LogicalDriveName": "DataVol-RAID5",
    "StripSizeBytes": 262144,
    "ReadCachePolicy": "ReadAhead",
    "WriteCachePolicy": "WriteBackWithMirror"
  }' | jq '.'
```

Via iLOrest (Gen11):

```bash
ilorest login https://192.168.1.101 -u admin -p password

# List drives available for configuration
ilorest select Drive.
ilorest get Name Status PhysicalLocation

# Create logical drive using smartarray command (works Gen10 and Gen11)
ilorest createlogicaldrive raid5 --drives=1I:1:1,1I:1:2,1I:1:3,1I:1:4 \
  --label="DataVol-RAID5" --readpolicy=ReadAhead --writepolicy=WriteBackWithBBU

ilorest logout
```

**After creating a logical drive, the array will initialize.** For large arrays this can take hours. The server is operational during initialization, but full performance is not available until the build completes. Monitor progress via `VolumeType` and `Operations` fields on the Volume resource.

---

## Listing Existing Logical Drives

### Gen11

```bash
CTRL_ID="DA000000"

# List logical volumes
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/ \
  | jq '.Members[].@odata.id'

# Volume details
VOL_ID="1"
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/${VOL_ID}/ \
  | jq '{
      Name,
      RAIDType,
      CapacityBytes,
      Status,
      ReadCachePolicy,
      WriteCachePolicy,
      StripSizeBytes,
      VolumeUsage,
      Operations
    }'
```

### Gen10

```bash
# List logical drives
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/LogicalDrives/ \
  | jq '.Members[].@odata.id'

# Logical drive details
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/LogicalDrives/1/ \
  | jq '{
      LogicalDriveName,
      Raid,
      CapacityMiB,
      Status,
      RebuildStatus,
      TransformationCompletionPercentage,
      DriveLocationFormat
    }'
```

---

## Deleting Logical Drives

> **CRITICAL — DESTRUCTIVE OPERATION**
>
> Deleting a logical drive permanently destroys all data on that volume. This cannot be undone. No data recovery is possible after deletion.
>
> Per Critical Rule #7: this operation requires explicit operator approval before execution. Always confirm:
> 1. The correct volume URI is targeted (verify name and capacity before deleting)
> 2. Any data on the volume has been backed up or is not needed
> 3. No OS, application, or virtual machine is mounted on this volume
>
> Do not proceed without an explicit confirmation from the operator.

### Gen11

```bash
CTRL_ID="DA000000"
VOL_ID="1"

# VERIFY before deleting — confirm you have the right volume
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/${VOL_ID}/ \
  | jq '{Name, RAIDType, CapacityBytes, Status}'

# DELETE — only proceed after operator confirms the above details
curl -sk -u admin:password \
  -X DELETE \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/${VOL_ID}/
# HTTP 200 or 204 = success; HTTP 405 = operation not supported in current state
```

### Gen10

```bash
# VERIFY first
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/LogicalDrives/1/ \
  | jq '{LogicalDriveName, Raid, CapacityMiB}'

# DELETE after explicit operator confirmation
curl -sk -u admin:password \
  -X DELETE \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/LogicalDrives/1/
```

Via iLOrest:

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest deletelogicaldrive --all  # Interactive prompt will list and confirm each
# Or target specific volume:
ilorest deletelogicaldrive 1
ilorest logout
```

---

## Hot Spare Configuration

Hot spares allow the controller to automatically begin a RAID rebuild when a drive fails, without manual intervention.

**Types:**

| Type | Scope | Behavior |
|---|---|---|
| Dedicated | Assigned to a specific array | Rebuilds only that array; most predictable |
| Global/Roaming | Available to any array on the controller | More efficient use of spare drives in mixed environments |

### Gen11 — Assign Hot Spare via Volume PATCH

```bash
CTRL_ID="DA000000"
VOL_ID="1"          # The volume this spare belongs to
SPARE_DRIVE="/redfish/v1/Systems/1/Storage/DA000000/Drives/8"

# Assign dedicated hot spare to a volume
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/${VOL_ID}/ \
  -d '{
    "Oem": {
      "Hpe": {
        "SpareResourceSet": [
          {"@odata.id": "'${SPARE_DRIVE}'"}
        ]
      }
    }
  }' | jq '.'
```

### Gen10 — Assign Hot Spare via OEM

```bash
CTRL_URI="/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0"
SPARE_DRIVE="${CTRL_URI}/DiskDrives/4/"

# Assign as dedicated spare to a specific array (via logical drive PATCH)
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  https://192.168.1.101${CTRL_URI}/LogicalDrives/1/ \
  -d '{
    "SpareDrives": ["'${SPARE_DRIVE}'"],
    "SpareDriveType": "Dedicated"
  }' | jq '.'

# Global/roaming spare — assign at controller level
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  https://192.168.1.101${CTRL_URI}/ \
  -d '{
    "GlobalSpareDrives": ["'${SPARE_DRIVE}'"]
  }' | jq '.'
```

Via iLOrest:

```bash
ilorest login https://192.168.1.101 -u admin -p password
# Assign drive at bay 1I:1:5 as a dedicated spare for logical drive 1
ilorest createlogicaldrive --spares=1I:1:5 --sparetype=Dedicated 1
ilorest logout
```

**Hot spare sizing rule:** The spare drive must be equal to or larger in capacity than the smallest drive in the array it protects.

---

## Controller Cache Settings

RAID controllers use battery-backed or capacitor-backed write cache to absorb write bursts and improve write performance. Cache health directly impacts data safety and performance.

### Cache Status Check

```bash
# Gen11 — cache summary in controller StorageControllers array
CTRL_ID="DA000000"
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/ \
  | jq '.StorageControllers[] | {
      Model,
      CacheSummary: .CacheSummary,
      Status: .Status
    }'

# CacheSummary fields:
# TotalCacheSizeMiB     — total installed cache RAM
# PersistentCacheSizeMiB — non-volatile (capacitor-backed) portion
# Status.Health         — "OK" = healthy; "Warning" = degraded; "Critical" = cache disabled
```

```bash
# Gen10 — cache and battery status from controller resource
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/ \
  | jq '{
      CacheMemorySizeMiB,
      BackupPowerSourceStatus,
      WriteCacheBypassThresholdKiB,
      CurrentOperatingMode
    }'

# BackupPowerSourceStatus values:
# "Present"    — battery or capacitor present and healthy
# "NotPresent" — no backup power; controller forces write-through (performance impact)
# "Charging"   — backup power present but not yet at full charge
```

### Cache Write Policy Behavior

| Scenario | Write Policy | Data Safety | Performance |
|---|---|---|---|
| BBU/capacitor healthy | WriteBackWithBBU | High | Best |
| BBU failed or absent | WriteThrough (forced) | High | Reduced |
| BBU charging | WriteThrough (temporary) | High | Reduced until charged |
| AlwaysWriteBack | WriteBack unconditional | Risk without BBU | Best |

**If `BackupPowerSourceStatus` is `NotPresent` or `Failed`:** The controller automatically falls back to write-through mode. Write performance will be significantly reduced. Replace the cache module (battery or capacitor) to restore write-back capability.

### Adjust Cache Read/Write Policy on a Volume (Gen11)

```bash
CTRL_ID="DA000000"
VOL_ID="1"

curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/${VOL_ID}/ \
  -d '{
    "ReadCachePolicy": "ReadAhead",
    "WriteCachePolicy": "WriteBackWithBBU"
  }' | jq '.'
```

Valid `ReadCachePolicy` values: `Off`, `ReadAhead`

---

## Drive Secure Erase

Use secure erase when decommissioning drives — removes all data cryptographically or by overwriting. Required for NIST SP800-88 compliance.

> **WARNING:** Secure erase is irreversible. Confirm the drive is not part of an active array before erasing. Erase the drive only — not the array unless decommissioning the entire system.

### Gen11 — Via Redfish Action

```bash
CTRL_ID="DA000000"
DRIVE_ID="5"

# Verify drive is not in use before erasing
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Drives/${DRIVE_ID}/ \
  | jq '{Name, Status, MediaType, CapacityBytes}'

# Initiate secure erase — SanitizeOverwrite or SanitizeCryptographicErase
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Drives/${DRIVE_ID}/Actions/Drive.SecureErase \
  -d '{}' | jq '.'

# For NVMe drives — cryptographic erase is near-instant
# For HDD — overwrite-based erase takes hours proportional to drive size
```

### Gen10 — Via iLOrest

```bash
ilorest login https://192.168.1.101 -u admin -p password

# List drives eligible for erase
ilorest select SmartStorageDiskDrive.
ilorest get Location SerialNumber Status

# Secure erase a specific drive (by location)
ilorest drivesanitize 1I:1:3 --mediatype HDD

# For SSD — uses cryptographic erase
ilorest drivesanitize 1I:1:4 --mediatype SSD

ilorest logout
```

Erase status tracking:

```bash
# Poll the drive resource for erase progress (Gen11)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Drives/${DRIVE_ID}/ \
  | jq '.Operations'
# Operations[].PercentageComplete shows progress
```

Audit trail for compliant erase is recorded in `/redfish/v1/Systems/1/SecureEraseReportService/`.

---

## Monitoring RAID Rebuild and Array Health

```bash
# Gen11 — check all volumes for rebuild or degraded status
CTRL_ID="DA000000"
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/${CTRL_ID}/Volumes/ \
  | jq -r '.Members[].@odata.id' | while read -r vol_uri; do
    curl -sk -u admin:password "https://192.168.1.101${vol_uri}" \
      | jq --arg u "$vol_uri" '{
          vol: $u,
          name: .Name,
          raid: .RAIDType,
          health: .Status.Health,
          state: .Status.State,
          ops: (.Operations // [])
        }'
  done

# Gen10 — check logical drive rebuild status
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/LogicalDrives/ \
  | jq -r '.Members[].@odata.id' | while read -r ld_uri; do
    curl -sk -u admin:password "https://192.168.1.101${ld_uri}" \
      | jq '{name: .LogicalDriveName, status: .Status, rebuild: .RebuildStatus, pct: .TransformationCompletionPercentage}'
  done
```

IML will also log RAID events — check the hardware event log when storage health changes:

```bash
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries/?$filter=Severity eq 'Warning' or Severity eq 'Critical'" \
  | jq '.Members[] | select(.Message | test("drive|array|RAID|storage"; "i")) | {Created, Message, Severity}'
```

---

## OneView Server Profile Storage

When managing storage at fleet scale, define storage configuration in a OneView Server Profile Template. Every server assigned from the template receives the same logical drive layout automatically.

### Storage Block in Server Profile Template (OneView REST API)

```json
{
  "localStorage": {
    "controllers": [
      {
        "deviceSlot": "Embedded",
        "mode": "RAID",
        "initialize": true,
        "logicalDrives": [
          {
            "name": "Boot-OS",
            "raidLevel": "RAID1",
            "bootable": true,
            "numPhysicalDrives": 2,
            "driveMinSizeGB": 480,
            "driveMaxSizeGB": 480,
            "driveTechnology": "SataHdd"
          },
          {
            "name": "Data-RAID5",
            "raidLevel": "RAID5",
            "bootable": false,
            "numPhysicalDrives": 4,
            "driveMinSizeGB": 1800,
            "driveTechnology": "SasHdd"
          }
        ]
      }
    ]
  }
}
```

Benefits of OneView template-defined storage:

- Drive assignment is automatic — OneView selects drives matching size/type criteria
- Re-deploying a server from profile recreates the exact storage layout
- Compliance view shows if any server deviates from the template
- Profile migration (moving profile to a new server) re-creates storage on the new hardware

**OneView storage management requires the server to be managed by OneView** (iLO registered with the OneView appliance). For servers managed directly via iLO only, use the Redfish and iLOrest methods above.

---

## Quick Reference: Storage URIs

| Resource | Gen10 URI | Gen11 URI |
|---|---|---|
| Storage root | `/redfish/v1/Systems/1/SmartStorage/` | `/redfish/v1/Systems/1/Storage/` |
| Controller list | `/redfish/v1/Systems/1/SmartStorage/ArrayControllers/` | `/redfish/v1/Systems/1/Storage/` |
| Controller detail | `.../ArrayControllers/{id}/` | `.../Storage/{id}/` |
| Logical drives | `.../ArrayControllers/{id}/LogicalDrives/` | `.../Storage/{id}/Volumes/` |
| Physical drives | `.../ArrayControllers/{id}/DiskDrives/` | `.../Storage/{id}/Drives/{driveId}` |
| Hot spare (controller) | `.../ArrayControllers/{id}/` PATCH | `.../Storage/{id}/Volumes/{id}/` PATCH |
| Secure erase | `ilorest drivesanitize` | `.../Drives/{id}/Actions/Drive.SecureErase` |

## iLOrest Storage Commands Reference

| Task | Command |
|---|---|
| List controllers | `ilorest select Storage.` then `ilorest get` |
| List drives | `ilorest select Drive.` then `ilorest get` |
| List volumes | `ilorest select Volume.` then `ilorest get` |
| Create logical drive | `ilorest createlogicaldrive raid5 --drives=1I:1:1,1I:1:2,1I:1:3` |
| Delete logical drive | `ilorest deletelogicaldrive 1` |
| Assign hot spare | `ilorest createlogicaldrive --spares=1I:1:5 --sparetype=Dedicated 1` |
| Sanitize drive | `ilorest drivesanitize 1I:1:3` |

---

## Key Operational Rules

1. **Always identify the generation first.** Check iLO firmware version or model string. Gen10 = iLO 5 = SmartStorage OEM paths. Gen11 = iLO 6 = standard Storage paths.

2. **Verify drive state before creating arrays.** Drives must be unassigned. On Gen11, check `Status.State` on the Drive resource. On Gen10, check that drives do not appear in any existing LogicalDrive's `DataDrives`.

3. **Use WriteBackWithBBU, not AlwaysWriteBack.** `AlwaysWriteBack` risks data corruption on power failure. Only use it in environments with redundant power and UPS protection.

4. **Array initialization takes time.** A 48 TB RAID 6 array can take 12-24 hours to fully initialize. The volume is accessible during this period but in a degraded performance state.

5. **Monitor IML for storage events.** Drive failures, rebuild starts, and array degradation are logged in the Integrated Management Log at `/redfish/v1/Systems/1/LogServices/IML/Entries/`.

6. **Hot spares do not protect against controller failure.** They protect against individual drive failure only. For controller-level redundancy, use dual-controller configurations or host-based mirroring.

7. **Secure erase before decommissioning drives.** Use `Drive.SecureErase` (Gen11) or `ilorest drivesanitize` (Gen10). The audit trail is stored in `SecureEraseReportService`.
