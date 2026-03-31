# Operations: Health Check

Hardware health validation for HPE ProLiant servers using iLO Redfish. Run this before and after every operation — firmware updates, configuration changes, hardware additions, and maintenance windows.

> **Critical Rule #8:** Always verify server health before making any changes. A degraded server can become unrecoverable when further changes are applied to it. Verify health after changes to confirm the operation did not introduce new problems.

---

## Prerequisites

- iLO connectivity established — see `skills/foundation/connectivity/README.md`
- `curl` and `jq` available on the workstation running checks
- Credentials with at least **ReadOnly** role (health checks are read-only operations)

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| System overview | `/redfish/v1/Systems/1` | GET |
| Chassis | `/redfish/v1/Chassis/1` | GET |
| Processors | `/redfish/v1/Systems/1/Processors` | GET |
| Memory collection | `/redfish/v1/Systems/1/Memory` | GET |
| Memory DIMM | `/redfish/v1/Systems/1/Memory/{id}` | GET |
| Thermal | `/redfish/v1/Chassis/1/Thermal` | GET |
| Power | `/redfish/v1/Chassis/1/Power` | GET |
| Storage (Gen10) | `/redfish/v1/Systems/1/SmartStorage/ArrayControllers` | GET |
| Storage (Gen11) | `/redfish/v1/Systems/1/Storage` | GET |
| IML log entries | `/redfish/v1/Systems/1/LogServices/IML/Entries` | GET |
| iLO Manager | `/redfish/v1/Managers/1` | GET |
| AHS download (OEM) | `/redfish/v1/Systems/1/Oem/Hpe/AHSData` | GET |

---

## Quick Health Check (60 Seconds)

Five commands in sequence. If all pass, the server is healthy. If any return non-OK status, proceed to the comprehensive check for that layer.

### 1. Overall System Health

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{
      Model: .Model,
      SerialNumber: .SerialNumber,
      Health: .Status.Health,
      HealthRollup: .Status.HealthRollup,
      PowerState: .PowerState,
      PostState: .Oem.Hpe.PostState,
      BiosVersion: .BiosVersion
    }'
```

Expected: `Health` and `HealthRollup` both `"OK"`, `PostState` `"FinishedPost"`.

### 2. Memory Health

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Memory \
  | jq '[.Members[].Status | select(.Health != null)] | {
      Total: length,
      Unhealthy: [.[] | select(.Health != "OK")] | length
    }'
```

Expected: `Unhealthy: 0`.

> This gives a count only. For DIMM-level details (slot, errors, size) proceed to Layer 3 of the comprehensive check.

### 3. Thermal — Inlet Temperature

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Thermal \
  | jq '[.Temperatures[] | select(.Name | test("Inlet|Ambient"; "i"))] |
      .[] | {
        Name,
        Reading: .ReadingCelsius,
        UpperCaution: .UpperThresholdNonCritical,
        UpperCritical: .UpperThresholdCritical,
        Status: .Status.Health
      }'
```

Expected: `Status: "OK"`, reading well below caution threshold.

### 4. Power Supply Redundancy

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '{
      Redundancy: .PowerSupplies | map(.Status.Health),
      RedundancyState: .Redundancy[0].Mode // "N/A",
      PSU_Count: (.PowerSupplies | length),
      Unhealthy: [.PowerSupplies[] | select(.Status.Health != "OK")] | length
    }'
```

Expected: all PSU statuses `"OK"`, `Unhealthy: 0`.

### 5. Storage Health (Gen10 Smart Array)

```bash
# Gen10 — Smart Array OEM path
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers \
  | jq '[.Members[]] | map(.Status.Health)'

# Gen11 — Standard Redfish Storage path
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage \
  | jq '[.Members[] | .Status.Health] | {
      Controllers: length,
      Unhealthy: [.[] | select(. != "OK")] | length
    }'
```

Expected: all controllers `"OK"`.

### Quick Check Interpretation

| Result | Action |
|---|---|
| All five pass | Server is healthy — proceed with planned operation |
| System Health is `Warning` | Run comprehensive check; identify which component is degraded |
| System Health is `Critical` | Do not proceed with planned operation — investigate and resolve first |
| Memory has unhealthy DIMMs | Run Layer 3 comprehensive check; identify failed DIMM by slot |
| Inlet temp above caution | Check fan status and airflow before proceeding |
| PSU redundancy degraded | Identify failed PSU; do not proceed with operations that could affect power |
| Storage controller warning | Run Layer 4 comprehensive check; do not update storage firmware until resolved |

---

## Comprehensive Health Check (Layered)

Work through each layer in order. Each layer gives full detail for that subsystem. Stop at the first layer with problems and resolve before proceeding with planned operations.

---

### Layer 1: System Overview

Full system identity and overall health status.

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{
      Model: .Model,
      SerialNumber: .SerialNumber,
      UUID: .UUID,
      PartNumber: .PartNumber,
      BiosVersion: .BiosVersion,
      PowerState: .PowerState,
      Health: .Status.Health,
      HealthRollup: .Status.HealthRollup,
      State: .Status.State,
      PostState: .Oem.Hpe.PostState,
      PostCode: .Oem.Hpe.PostCodeDump // "none",
      IndicatorLED: .IndicatorLED
    }'
```

iLO firmware version and model:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1 \
  | jq '{
      iLO_Model: .Model,
      FirmwareVersion: .FirmwareVersion,
      Health: .Status.Health,
      DateTime: .DateTime
    }'
```

**Key fields:**

| Field | Healthy Value | Action if Different |
|---|---|---|
| `Status.Health` | `OK` | Run deeper layer checks to identify degraded component |
| `Status.HealthRollup` | `OK` | A rolled-up warning means something lower in the hierarchy is degraded |
| `PostState` | `FinishedPost` | Server still in POST or stuck — check IML and remote console |
| `IndicatorLED` | `Off` | `Lit` means attention is required; `Blinking` means active locating |
| `PowerState` | `On` | `Off` or `PoweringOn` requires attention if unexpected |

---

### Layer 2: Processors

```bash
# List all processors
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Processors \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{
          Socket: .Socket,
          Model: .Model,
          TotalCores: .TotalCores,
          TotalThreads: .TotalThreads,
          MaxSpeedMHz: .MaxSpeedMHz,
          Health: .Status.Health,
          State: .Status.State
        }'
  done
```

Expected: each processor shows `Health: "OK"`, `State: "Enabled"`.

**What to look for:**

- `State: "Absent"` on a populated socket indicates a seating issue or failed processor
- `Health: "Warning"` or `"Critical"` indicates processor hardware fault — check IML for thermal events
- Mismatched processor models in a dual-socket server may cause BIOS warnings

---

### Layer 3: Memory

#### DIMM Inventory and Status

```bash
# Full DIMM inventory with health and identification
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Memory \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{
          Slot: .DeviceLocator,
          CapacityMiB: .CapacityMiB,
          SpeedMHz: .OperatingSpeedMhz,
          Manufacturer: .Manufacturer,
          PartNumber: .PartNumber,
          Health: .Status.Health,
          State: .Status.State,
          ErrorCorrection: .ErrorCorrection,
          MemoryType: .MemoryType
        }'
  done
```

#### Error Counts (HPE OEM)

iLO exposes per-DIMM error counters via the OEM extension. Correctable errors (CE) are expected in small numbers; uncorrectable errors (UE) indicate DIMM failure.

```bash
# Check OEM error counters (iLO 5 / Gen10)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Memory \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{
          Slot: .DeviceLocator,
          CorrectableErrors: .Oem.Hpe.DIMMStatus // "N/A",
          Health: .Status.Health
        }'
  done
```

#### Identifying a Failed DIMM for Replacement

When a DIMM shows `Health: "Warning"` or `"Critical"`:

1. Record the `DeviceLocator` field — this is the physical slot label (e.g., `PROC 1 DIMM 1A`)
2. Record `PartNumber` and `Manufacturer` for ordering a replacement
3. Record `CapacityMiB` and `OperatingSpeedMhz` — replacement must match or exceed the speed

```bash
# Find unhealthy DIMMs and output identification info
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Memory \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq 'select(.Status.Health != "OK" and .Status.State != "Absent") | {
          Slot: .DeviceLocator,
          Health: .Status.Health,
          CapacityMiB: .CapacityMiB,
          SpeedMHz: .OperatingSpeedMhz,
          Manufacturer: .Manufacturer,
          PartNumber: .PartNumber
        }'
  done
```

**Memory health interpretation:**

| Condition | Meaning | Action |
|---|---|---|
| `State: "Absent"` | No DIMM installed in this slot | Normal if slot intentionally empty |
| `Health: "OK"` | DIMM is functioning correctly | No action |
| `Health: "Warning"` + correctable errors | DIMM is degrading | Plan replacement; monitor closely |
| `Health: "Critical"` | DIMM failed or too many errors | Replace immediately; system may have disabled the DIMM |
| Uncorrectable error in IML | Data corruption risk | Replace DIMM before next boot if possible |

---

### Layer 4: Storage

#### Gen10 — Smart Array (HPE OEM Path)

```bash
# Controller status
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{
          Model: .Model,
          SerialNumber: .SerialNumber,
          FirmwareVersion: .FirmwareVersion,
          Health: .Status.Health,
          State: .Status.State,
          CacheStatus: .CacheModuleStatus,
          CacheSizeMiB: .CacheMemorySizeMiB
        }'
  done

# Logical drives (RAID arrays)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/LogicalDrives \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{
          LogicalDrive: .LogicalDriveNumber,
          RaidLevel: .Raid,
          CapacityMiB: .CapacityMiB,
          Status: .Status.Health,
          DriveCount: .DriveGeometry.SpindleCount
        }'
  done

# Physical drives
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/DiskDrives \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{
          Location: .Location,
          Model: .Model,
          CapacityGB: .CapacityGB,
          MediaType: .MediaType,
          Health: .Status.Health,
          PredictiveFailure: .PredictiveFailureCount
        }'
  done
```

#### Gen11 — Standard Redfish Storage Path

```bash
# Controller and volume summary
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{
          Controller: .StorageControllers[0].Model,
          FirmwareVersion: .StorageControllers[0].FirmwareVersion,
          Health: .Status.Health,
          Volumes: [.Volumes.Members[]] | length,
          Drives: [.Drives[]] | length
        }'
  done

# Physical drives
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/DA000000/Drives \
  | jq '.Members[].@odata.id' -r | while read uri; do
    curl -sk -u admin:password "https://192.168.1.101${uri}" \
      | jq '{
          Name: .Name,
          CapacityBytes: .CapacityBytes,
          MediaType: .MediaType,
          Protocol: .Protocol,
          Health: .Status.Health,
          PredictiveFailure: .FailurePredicted
        }'
  done
```

**Storage health interpretation:**

| Condition | Meaning | Action |
|---|---|---|
| Controller `Health: "OK"` | Controller and cache are healthy | No action |
| Controller `Health: "Warning"` | Controller degraded — possibly cache or battery issue | Check IML; do not update firmware until resolved |
| Logical drive `Health: "OK"` | RAID array is intact | No action |
| Logical drive `Health: "Warning"` | Array rebuilding or a drive failed; RAID still protecting data | Identify failed drive; replace before another fails |
| Logical drive `Health: "Critical"` | RAID has failed — data may be at risk | Emergency — do not write to array; check backup status |
| Drive `PredictiveFailure: true` | SMART data indicates impending drive failure | Replace drive during next maintenance window |
| Drive `Health: "Critical"` | Drive has failed | Replace immediately; check if RAID is still protecting data |

> For full storage investigation and RAID rebuild procedures, see `skills/deploy/storage/README.md`.

---

### Layer 5: Thermal

#### Temperature Sensors

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Thermal \
  | jq '.Temperatures[] | select(.Status.State != "Absent") | {
      Name,
      ReadingCelsius,
      UpperCaution: .UpperThresholdNonCritical,
      UpperCritical: .UpperThresholdCritical,
      Status: .Status.Health
    }'
```

#### Fan Status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Thermal \
  | jq '.Fans[] | {
      Name,
      Reading: .Reading,
      ReadingUnits: .ReadingUnits,
      Health: .Status.Health,
      State: .Status.State
    }'
```

#### Summary — Critical Sensors Only

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Thermal \
  | jq '{
      InletTemp: (.Temperatures[] | select(.Name | test("Inlet|01-Inlet"; "i")) | .ReadingCelsius),
      UnhealthyTemps: [.Temperatures[] | select(.Status.Health != "OK" and .Status.State != "Absent")] | length,
      UnhealthyFans: [.Fans[] | select(.Status.Health != "OK" and .Status.State != "Absent")] | length,
      TotalFans: [.Fans[] | select(.Status.State != "Absent")] | length
    }'
```

**Thermal thresholds (typical DL380 Gen10):**

| Sensor | Normal | Caution | Critical |
|---|---|---|---|
| Inlet ambient | 18–25°C | 42°C | 46°C |
| CPU (under load) | 50–70°C | 80°C | 90°C |
| Exhaust | 30–45°C | 60°C | 70°C |

> Thresholds vary by server model and configuration. Always read `UpperThresholdNonCritical` and `UpperThresholdCritical` from the actual sensor data rather than using hardcoded values.

**Thermal health interpretation:**

| Condition | Meaning | Action |
|---|---|---|
| All fans `OK`, inlet < caution | Thermal health normal | No action |
| Inlet above caution threshold | Ambient temperature too high | Investigate datacenter cooling before running high-load operations |
| One fan `Warning` | Fan speed degraded or single fan failure | Check fan seating; replace fan; verify redundancy |
| One fan `Critical` or absent | Fan failed | Replace immediately; monitor temperatures closely |
| CPU temp above caution | Thermal throttling likely — performance degraded | Check airflow, fan status, heatsink seating |

---

### Layer 6: Power

#### PSU Inventory and Status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.PowerSupplies[] | {
      Name,
      Model: .Model,
      SerialNumber: .SerialNumber,
      Manufacturer: .Manufacturer,
      PowerCapacityWatts: .PowerCapacityWatts,
      LastPowerOutputWatts: .LastPowerOutputWatts,
      Health: .Status.Health,
      State: .Status.State,
      HotPluggable: .HotPluggable
    }'
```

#### Power Redundancy Status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '{
      RedundancyMode: .Redundancy[0].Mode,
      RedundancyHealth: .Redundancy[0].Status.Health,
      RedundancyState: .Redundancy[0].Status.State,
      MinRequired: .Redundancy[0].MinNumNeeded,
      MaxSupported: .Redundancy[0].MaxNumSupported
    }'
```

#### Power Consumption

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.PowerControl[] | {
      Name,
      ConsumedWatts: .PowerConsumedWatts,
      AverageConsumedWatts: .PowerMetrics.AverageConsumedWatts,
      MinConsumedWatts: .PowerMetrics.MinConsumedWatts,
      MaxConsumedWatts: .PowerMetrics.MaxConsumedWatts,
      IntervalMinutes: .PowerMetrics.IntervalInMin
    }'
```

#### HPE OEM — Chassis Input Power

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.Oem.Hpe | {
      ChassisInputPowerWatts,
      SNMPPowerThresholdAlert: .SNMPPowerThresholdAlert
    }'
```

**Power redundancy states:**

| `Mode` | Meaning | Action |
|---|---|---|
| `Failover` | N+1 redundancy, all PSUs healthy | Normal |
| `NotRedundant` | Single PSU or redundancy disabled | Review if this is expected; add PSU if not |
| `Failover` + `Health: Warning` | Redundancy mode set but one PSU degraded | Replace degraded PSU before it affects operation |
| `Health: Critical` | Active PSU failure affecting power delivery | Emergency — may impact server stability |

---

### Layer 7: Logs (IML)

The Integrated Management Log (IML) records hardware events from iLO. Always check the last 24 hours before and after any operation.

#### Recent IML Entries (Last 24 Hours)

```bash
# All IML entries, newest first
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$orderby=Created%20desc&\$top=50" \
  | jq '.Members[] | {
      Timestamp: .Created,
      Severity: .Severity,
      Message: .Message,
      Category: .Oem.Hpe.Class // "General"
    }'
```

#### Filter for Critical and Warning Only

```bash
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$top=200" \
  | jq '[.Members[] | select(.Severity == "Critical" or .Severity == "Warning")] |
      sort_by(.Created) | reverse |
      .[] | {Timestamp: .Created, Severity, Message}'
```

#### Count by Severity

```bash
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$top=200" \
  | jq '{
      Critical: [.Members[] | select(.Severity == "Critical")] | length,
      Warning: [.Members[] | select(.Severity == "Warning")] | length,
      Informational: [.Members[] | select(.Severity == "OK" or .Severity == "Informational")] | length
    }'
```

**IML severity guide:**

| Severity | Meaning | Action |
|---|---|---|
| `OK` / `Informational` | Normal operational events | No action; review if unexpected |
| `Warning` | Degraded state; operation continues | Investigate cause; plan remediation |
| `Critical` | Significant hardware fault | Investigate immediately; do not proceed with changes until resolved |

> For IML event interpretation and common error patterns, see `skills/diagnose/log-analysis/README.md`.

---

## Health Report Format

Use this structured format when documenting health status in change records, tickets, or handoff notes.

```
SERVER HEALTH REPORT
====================
Server:      ProLiant DL380 Gen10
Serial:      MXQ12345AB
iLO:         192.168.1.101 (iLO 5 v3.06)
Timestamp:   2026-01-15T10:30:00Z

Overall Status: HEALTHY | DEGRADED | UNHEALTHY

Layer 1 - System:     OK
  - Model:            ProLiant DL380 Gen10
  - POST State:       FinishedPost
  - System ROM:       U30 v2.80
  - Indicator LED:    Off

Layer 2 - Processors: OK
  - CPU 1:            Intel Xeon Gold 6248R (24C/48T) — OK
  - CPU 2:            Intel Xeon Gold 6248R (24C/48T) — OK

Layer 3 - Memory:     OK
  - Total:            256 GB (8x 32GB DDR4-2933)
  - Populated:        16 of 24 slots
  - Errors:           0 correctable, 0 uncorrectable

Layer 4 - Storage:    OK
  - Controller:       Smart Array P408i-a (firmware 5.20) — OK
  - Logical Drive 1:  RAID 5 (3x 600GB SAS) — OK
  - Drives:           3/3 OK, 0 predictive failure

Layer 5 - Thermal:    OK
  - Inlet:            22°C (caution: 42°C, critical: 46°C)
  - CPU 1:            58°C
  - CPU 2:            56°C
  - Fans:             7/7 OK

Layer 6 - Power:      OK
  - PSU 1:            800W — OK
  - PSU 2:            800W — OK
  - Redundancy:       Full (Failover mode)
  - Current Draw:     342W

Layer 7 - Logs:       OK
  - Recent IML:       0 critical, 0 warning (last 24h)

Issues:
  (none)
```

**Filling in the Overall Status:**

| Overall Status | Condition |
|---|---|
| `HEALTHY` | All seven layers OK, no issues in IML |
| `DEGRADED` | One or more layers showing Warning; server operational but not at full redundancy |
| `UNHEALTHY` | One or more layers showing Critical; do not proceed with changes |

---

## AHS Log Collection

The Active Health System (AHS) log is a continuous ring buffer of all hardware telemetry, events, and configuration history maintained by iLO. It is the primary artifact HPE Support requires to diagnose hardware failures.

### When to Collect

- Before calling HPE Support for any hardware issue
- After any significant hardware event (PSU failure, DIMM failure, storage fault)
- Before and after a major maintenance window as a reference record
- When a server has exhibited intermittent behaviour that is not currently active

### Download via Redfish OEM

```bash
# Download AHS log (may take several minutes — the file can be several hundred MB)
SERVER=192.168.1.101
SERIAL=$(curl -sk -u admin:password https://${SERVER}/redfish/v1/Systems/1 | jq -r '.SerialNumber')
DATE=$(date +%Y%m%d)
OUTFILE="logs/ahs/${SERIAL}-${DATE}.ahs"

mkdir -p logs/ahs

curl -sk -u admin:password \
  "https://${SERVER}/redfish/v1/Systems/1/Oem/Hpe/AHSData/Actions/HpeWfmAHSData.DownloadAHSData" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"DirectDownload": true}' \
  -o "${OUTFILE}"

echo "AHS saved to: ${OUTFILE}"
ls -lh "${OUTFILE}"
```

> The AHS download endpoint may vary between iLO versions. On some iLO 5 systems the endpoint is at `/redfish/v1/Managers/1/ActiveHealthSystem`. Verify with `GET /redfish/v1/Systems/1/Oem/Hpe` to find the `AHSData` link.

#### Alternative — iLOrest

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest downloadahs --filename logs/ahs/${SERIAL}-${DATE}.ahs
ilorest logout
```

---

## Pre-Operation Checklist

Run before any change: firmware update, configuration change, hardware addition, OS operation.

```
PRE-OPERATION HEALTH CHECK
==========================
Date:     [timestamp]
Server:   [model + serial]
iLO:      [IP]
Operator: [who is performing the operation]
Change:   [brief description of planned change]

[ ] Quick check passed (all 5 commands OK)
[ ] System Health: OK
[ ] System HealthRollup: OK
[ ] PostState: FinishedPost
[ ] No active IML Critical events in last 24h
[ ] PSU redundancy: Full
[ ] Thermal: within normal range
[ ] Storage: all controllers and arrays OK

Baseline firmware snapshot captured: [yes/no]
  File: [path to snapshot]

Decision: [ ] PROCEED  [ ] HOLD — reason: ___________
```

If the pre-operation check fails on any item marked above, do not proceed until the issue is resolved or explicitly accepted as a known risk with sign-off.

---

## Post-Operation Checklist

Run after any change to confirm it did not introduce new problems.

```
POST-OPERATION HEALTH CHECK
===========================
Date:       [timestamp]
Server:     [model + serial]
Operation:  [brief description of what was changed]
Duration:   [start time] to [end time]

[ ] Quick check passed (all 5 commands OK)
[ ] System Health: OK (same or better than pre-operation)
[ ] No new IML Critical or Warning events since operation started
[ ] Thermal: within normal range
[ ] Power: redundancy maintained
[ ] Storage: all arrays still OK
[ ] If firmware updated: firmware version confirmed correct

Comparison to pre-operation baseline:
  Health: [IMPROVED | SAME | DEGRADED]
  New IML events: [count and severity]
  Notes: ___________

Decision: [ ] COMPLETE — server returned to service
          [ ] INVESTIGATE — new issues found: ___________
          [ ] ROLLBACK — rolling back due to: ___________
```

---

## Health Check Script Template

Save as `scripts/health-check.sh`. Runs the full quick check and prints a structured summary.

```bash
#!/usr/bin/env bash
# health-check.sh — HPE ProLiant quick health check via Redfish
# Usage: ./health-check.sh <ilo-ip> <username> <password>
# Example: ./health-check.sh 192.168.1.101 admin password

set -euo pipefail

ILO_HOST="${1:?Usage: $0 <ilo-ip> <username> <password>}"
ILO_USER="${2:?Missing username}"
ILO_PASS="${3:?Missing password}"
BASE="https://${ILO_HOST}"
AUTH="-u ${ILO_USER}:${ILO_PASS}"
CURL="curl -sk ${AUTH}"

ISSUES=()

echo "SERVER HEALTH REPORT"
echo "===================="

# --- System identity ---
SYSTEM=$($CURL "${BASE}/redfish/v1/Systems/1")
MODEL=$(echo "$SYSTEM" | jq -r '.Model')
SERIAL=$(echo "$SYSTEM" | jq -r '.SerialNumber')
HEALTH=$(echo "$SYSTEM" | jq -r '.Status.Health')
ROLLUP=$(echo "$SYSTEM" | jq -r '.Status.HealthRollup')
POSTSTATE=$(echo "$SYSTEM" | jq -r '.Oem.Hpe.PostState')
BIOS=$(echo "$SYSTEM" | jq -r '.BiosVersion')

MGR=$($CURL "${BASE}/redfish/v1/Managers/1")
ILO_VER=$(echo "$MGR" | jq -r '.FirmwareVersion')

printf "Server:      %s\n" "$MODEL"
printf "Serial:      %s\n" "$SERIAL"
printf "iLO:         %s (%s)\n" "$ILO_HOST" "$ILO_VER"
printf "Timestamp:   %s\n" "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
echo ""

# --- Layer 1: System ---
L1="OK"
if [[ "$HEALTH" != "OK" ]] || [[ "$ROLLUP" != "OK" ]]; then
  L1="DEGRADED (Health=$HEALTH, Rollup=$ROLLUP)"
  ISSUES+=("Layer 1: System health is $HEALTH / rollup $ROLLUP")
fi
if [[ "$POSTSTATE" != "FinishedPost" ]]; then
  L1="WARNING (PostState=$POSTSTATE)"
  ISSUES+=("Layer 1: PostState is $POSTSTATE (expected FinishedPost)")
fi
printf "Layer 1 - System:     %s\n" "$L1"
printf "  - Model:            %s\n" "$MODEL"
printf "  - POST State:       %s\n" "$POSTSTATE"
printf "  - System ROM:       %s\n" "$BIOS"
echo ""

# --- Layer 3: Memory ---
MEM_UNHEALTHY=$($CURL "${BASE}/redfish/v1/Systems/1/Memory" \
  | jq '[.Members[] | select(.Status.Health != null and .Status.Health != "OK" and .Status.State != "Absent")] | length' 2>/dev/null || echo "0")
MEM_TOTAL=$($CURL "${BASE}/redfish/v1/Systems/1/Memory" \
  | jq '[.Members[] | select(.Status.State == "Enabled")] | length' 2>/dev/null || echo "?")
L3="OK"
if [[ "$MEM_UNHEALTHY" -gt 0 ]]; then
  L3="DEGRADED ($MEM_UNHEALTHY DIMM(s) unhealthy)"
  ISSUES+=("Layer 3: $MEM_UNHEALTHY DIMM(s) reporting unhealthy status")
fi
printf "Layer 3 - Memory:     %s\n" "$L3"
printf "  - Enabled DIMMs:    %s\n" "$MEM_TOTAL"
echo ""

# --- Layer 5: Thermal ---
THERMAL=$($CURL "${BASE}/redfish/v1/Chassis/1/Thermal")
INLET=$(echo "$THERMAL" | jq '[.Temperatures[] | select(.Name | test("Inlet|01-Inlet"; "i"))] | .[0].ReadingCelsius // "N/A"')
FAN_BAD=$(echo "$THERMAL" | jq '[.Fans[] | select(.Status.Health != "OK" and .Status.State != "Absent")] | length')
FAN_TOTAL=$(echo "$THERMAL" | jq '[.Fans[] | select(.Status.State != "Absent")] | length')
TEMP_BAD=$(echo "$THERMAL" | jq '[.Temperatures[] | select(.Status.Health != "OK" and .Status.State != "Absent")] | length')
L5="OK"
if [[ "$FAN_BAD" -gt 0 ]] || [[ "$TEMP_BAD" -gt 0 ]]; then
  L5="DEGRADED (fans_bad=$FAN_BAD, temps_bad=$TEMP_BAD)"
  [[ "$FAN_BAD" -gt 0 ]] && ISSUES+=("Layer 5: $FAN_BAD fan(s) not OK")
  [[ "$TEMP_BAD" -gt 0 ]] && ISSUES+=("Layer 5: $TEMP_BAD temperature sensor(s) not OK")
fi
printf "Layer 5 - Thermal:    %s\n" "$L5"
printf "  - Inlet:            %s°C\n" "$INLET"
printf "  - Fans:             %s/%s OK\n" "$((FAN_TOTAL - FAN_BAD))" "$FAN_TOTAL"
echo ""

# --- Layer 6: Power ---
POWER=$($CURL "${BASE}/redfish/v1/Chassis/1/Power")
PSU_BAD=$(echo "$POWER" | jq '[.PowerSupplies[] | select(.Status.Health != "OK" and .Status.State != "Absent")] | length')
PSU_TOTAL=$(echo "$POWER" | jq '[.PowerSupplies[] | select(.Status.State != "Absent")] | length')
REDUNDANCY=$(echo "$POWER" | jq -r '.Redundancy[0].Status.Health // "N/A"')
CONSUMED=$(echo "$POWER" | jq '.PowerControl[0].PowerConsumedWatts // "N/A"')
L6="OK"
if [[ "$PSU_BAD" -gt 0 ]]; then
  L6="DEGRADED ($PSU_BAD PSU(s) not OK)"
  ISSUES+=("Layer 6: $PSU_BAD PSU(s) not OK")
fi
if [[ "$REDUNDANCY" != "OK" ]] && [[ "$REDUNDANCY" != "N/A" ]]; then
  L6="DEGRADED (redundancy=$REDUNDANCY)"
  ISSUES+=("Layer 6: PSU redundancy health is $REDUNDANCY")
fi
printf "Layer 6 - Power:      %s\n" "$L6"
printf "  - PSUs:             %s/%s OK\n" "$((PSU_TOTAL - PSU_BAD))" "$PSU_TOTAL"
printf "  - Redundancy:       %s\n" "$REDUNDANCY"
printf "  - Current Draw:     %sW\n" "$CONSUMED"
echo ""

# --- Layer 7: Logs ---
IML_CRITICAL=$($CURL "${BASE}/redfish/v1/Systems/1/LogServices/IML/Entries?\$top=200" \
  | jq '[.Members[] | select(.Severity == "Critical")] | length' 2>/dev/null || echo "0")
IML_WARNING=$($CURL "${BASE}/redfish/v1/Systems/1/LogServices/IML/Entries?\$top=200" \
  | jq '[.Members[] | select(.Severity == "Warning")] | length' 2>/dev/null || echo "0")
L7="OK"
if [[ "$IML_CRITICAL" -gt 0 ]]; then
  L7="CRITICAL ($IML_CRITICAL critical IML events)"
  ISSUES+=("Layer 7: $IML_CRITICAL critical IML event(s) found")
elif [[ "$IML_WARNING" -gt 0 ]]; then
  L7="WARNING ($IML_WARNING warning IML events)"
  ISSUES+=("Layer 7: $IML_WARNING warning IML event(s) found")
fi
printf "Layer 7 - Logs:       %s\n" "$L7"
printf "  - IML:              %s critical, %s warning\n" "$IML_CRITICAL" "$IML_WARNING"
echo ""

# --- Overall status ---
if [[ ${#ISSUES[@]} -eq 0 ]]; then
  echo "Overall Status: HEALTHY"
  echo ""
  echo "Issues:"
  echo "  (none)"
else
  # Determine DEGRADED vs UNHEALTHY
  if echo "${ISSUES[@]}" | grep -q "Critical\|CRITICAL"; then
    echo "Overall Status: UNHEALTHY"
  else
    echo "Overall Status: DEGRADED"
  fi
  echo ""
  echo "Issues:"
  for issue in "${ISSUES[@]}"; do
    echo "  - $issue"
  done
fi
echo ""
```

Make the script executable:

```bash
chmod +x scripts/health-check.sh
./scripts/health-check.sh 192.168.1.101 admin password
```

---

## Decision Guide for Agents

| Situation | Action |
|---|---|
| Starting any operation on a server | Run quick health check (5 commands) first |
| Quick check passes | Proceed with planned operation |
| `Status.HealthRollup` is `Warning` | Run comprehensive check; identify degraded layer before proceeding |
| `Status.Health` is `Critical` | Do not proceed; investigate and resolve before any changes |
| Need to identify a failed DIMM | Layer 3 comprehensive check; record `DeviceLocator`, `PartNumber`, `CapacityMiB` |
| DIMM shows `Health: Critical` | Replace DIMM; check IML for uncorrectable error events |
| Thermal warnings | Layer 5 check; identify specific sensor; check fan status and datacenter cooling |
| PSU health `Warning` | Layer 6 check; identify which PSU; replace before redundancy is fully lost |
| Before calling HPE Support | Download AHS log first; have serial number, model, and IML critical events ready |
| After any firmware update | Run post-operation health check; compare to pre-operation baseline |
| IML has Critical events | Layer 7 check; cross-reference with `skills/diagnose/log-analysis/README.md` |
| Server stuck in POST | Check `Oem.Hpe.PostState`; check IML for POST error events; use remote console to observe |
| Predictive drive failure flagged | Plan drive replacement; verify RAID redundancy is intact before replacing |
| Need full health documentation | Generate health report in structured format above; attach to change record |
