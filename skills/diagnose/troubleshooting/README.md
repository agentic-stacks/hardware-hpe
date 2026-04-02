# Diagnose: Troubleshooting

Symptom-based diagnostic decision trees for HPE ProLiant servers. This is the first stop when something goes wrong. Each tree starts with what the operator observes, branches on what the API and hardware report, and terminates at a resolution or an explicit escalation point.

> **How to use this skill:** Identify the symptom the operator has reported. Jump to that section. Follow the decision tree top to bottom, executing each query as you reach it. Every branch either resolves the issue, hands off to a more specific skill, or escalates to HPE support.

---

## Key URIs Referenced in This Skill

| Resource | URI | Notes |
|---|---|---|
| System overview | `/redfish/v1/Systems/1` | Health, PowerState, PostState |
| Chassis | `/redfish/v1/Chassis/1` | Physical enclosure |
| IML log entries | `/redfish/v1/Systems/1/LogServices/IML/Entries` | Integrated Management Log |
| iLO event log | `/redfish/v1/Managers/1/LogServices/IEL/Entries` | iLO-generated events |
| Security log | `/redfish/v1/Systems/1/LogServices/SL/Entries` | Security events |
| Memory collection | `/redfish/v1/Systems/1/Memory` | All DIMMs |
| Single DIMM | `/redfish/v1/Systems/1/Memory/{id}` | Individual DIMM detail |
| Processors | `/redfish/v1/Systems/1/Processors` | CPU status |
| Thermal | `/redfish/v1/Chassis/1/Thermal` | Fans and temperatures |
| Power | `/redfish/v1/Chassis/1/Power` | PSUs and power readings |
| Storage (Gen11) | `/redfish/v1/Systems/1/Storage` | Controllers, volumes, drives |
| SmartStorage (Gen10) | `/redfish/v1/Systems/1/SmartStorage/ArrayControllers` | Gen10 OEM path |
| Network adapters | `/redfish/v1/Chassis/1/NetworkAdapters` | NIC ports and link status |
| iLO interfaces | `/redfish/v1/Managers/1/EthernetInterfaces` | iLO network config |
| iLO Manager | `/redfish/v1/Managers/1` | iLO status and actions |
| UpdateService | `/redfish/v1/UpdateService` | Firmware update staging |
| Task list | `/redfish/v1/TaskService/Tasks` | Long-running operation status |
| AHS data (OEM) | `/redfish/v1/Systems/1/Oem/Hpe/AHSData` | Active Health System download |

---

## Symptom Index

1. [Server Won't POST / No Video Output](#symptom-server-wont-post--no-video-output)
2. [Degraded Health Status (Amber/Red Indicators)](#symptom-degraded-health-status-amberred-indicators)
3. [Memory Errors](#symptom-memory-errors)
4. [Storage Failures](#symptom-storage-failures)
5. [Thermal / Fan Alerts](#symptom-thermal--fan-alerts)
6. [Power Supply Failure or Redundancy Lost](#symptom-power-supply-failure-or-redundancy-lost)
7. [Network Connectivity Issues](#symptom-network-connectivity-issues)
8. [Firmware Update Failed](#symptom-firmware-update-failed)
9. [Unexpected Reboot or Shutdown](#symptom-unexpected-reboot-or-shutdown)
10. [iLO Unresponsive](#symptom-ilo-unresponsive)
11. [Processor (CPU) Fault](#symptom-processor-cpu-fault)
12. [BIOS/UEFI Configuration Problem](#symptom-biosuefi-configuration-problem)
13. [Escalation: When to Call HPE Support](#escalation-when-to-call-hpe-support)

---

## Symptom: Server Won't POST / No Video Output

```
Is the power LED on?
│
├─ NO (no power LED, system completely dark)
│   ├─ Check physical power:
│   │   → Verify IEC power cable is seated at server and PDU/wall
│   │   → Verify PDU breaker has not tripped
│   │   → Verify outlet/PDU is energised (test with known-good device)
│   ├─ Is iLO reachable on the network?
│   │   ├─ YES → Query power state:
│   │   │       GET /redfish/v1/Systems/1
│   │   │       Check: .PowerState
│   │   │       → "Off": server is off; attempt power on:
│   │   │             POST /redfish/v1/Systems/1/Actions/ComputerSystem.Reset
│   │   │             Body: {"ResetType":"On"}
│   │   │             If no response from server after 3 minutes → PSU hardware fault
│   │   │             → Escalate: physical PSU inspection required
│   │   │       → "On": power state inconsistency — iLO thinks server is on
│   │   │             → Check IML for PSU events
│   │   │             → Check .Oem.Hpe.AggregateHealthStatus.PowerSupplies
│   │   └─ NO → Physical access required
│   │             → Inspect PSU fault LED (amber = PSU fault)
│   │             → Reseat PSUs
│   │             → If dual PSU: remove one at a time to isolate faulty unit
│   │             → If still no power: escalate to HPE support
│
└─ YES (power LED on, fans spinning, but no POST progress)
    ├─ Is iLO reachable on the network?
    │   ├─ YES → Check POST state:
    │   │         GET /redfish/v1/Systems/1
    │   │         Check: .Oem.Hpe.PostState
    │   │
    │   │         PostState values and actions:
    │   │         → "InPost": server is actively POSTing — wait up to 5 minutes
    │   │         → "InPostDiscoveryStart": device discovery begun — wait
    │   │         → "InPostDiscoveryComplete": POST finished but OS not yet started
    │   │               → Video output issue: check display cable/adapter
    │   │               → Video output on iLO remote console may still work:
    │   │                     GET /redfish/v1/Managers/1 → check RemoteConsole URI
    │   │         → "FinishedPost": OS loaded — not a POST problem
    │   │               → Re-diagnose: check OS layer or network
    │   │         → "PowerOff": server powered off unexpectedly during POST
    │   │               → Check IML for error at time of shutdown
    │   │         → "Halted": POST halted on error
    │   │               → Check IML immediately for POST error detail
    │   │
    │   │         Check IML for POST errors:
    │   │         GET /redfish/v1/Systems/1/LogServices/IML/Entries
    │   │         Filter to recent critical entries:
    │   │           ?$filter=Severity eq 'Critical'&$orderby=Created desc&$top=10
    │   │         → Memory training failure → see Symptom: Memory Errors
    │   │         → CPU fault → see Symptom: Processor CPU Fault
    │   │         → Storage controller init failure → see Symptom: Storage Failures
    │   │
    │   └─ NO → iLO is unreachable
    │             → Check iLO dedicated NIC cable (rear panel, labelled iLO)
    │             → Try iLO dedicated port if currently using shared NIC
    │             → Check DHCP table for iLO MAC address assignment
    │             → Try iLO static IP if DHCP not resolving
    │             → Physical access: check LCD panel for iLO IP (Gen10/Gen11 LCD)
    │             → Physical access: enter UEFI (F9 at POST) → iLO configuration
    │             → If iLO still unreachable with confirmed network: iLO hardware fault
    │                   → Escalate to HPE support
    │
    └─ No video but system otherwise operational:
          → Use iLO virtual console instead of physical display
          → Check video cable and display separately
          → POST results accessible via iLO web console regardless of physical video
```

---

## Symptom: Degraded Health Status (Amber/Red Indicators)

Amber front panel LED indicates `Warning`. Red indicates `Critical`. Start here to identify which subsystem is responsible.

### Step 1: Identify the unhealthy subsystem

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{
      Health: .Status.Health,
      HealthRollup: .Status.HealthRollup,
      AggregateHealth: .Oem.Hpe.AggregateHealthStatus
    }'
```

The `AggregateHealthStatus` object breaks health down by subsystem:

```
AggregateHealthStatus subsystems and where to go:
│
├─ AgentlessManagementService: "Warning"/"Critical"
│   → iLO agentless management issue
│   → Check iLO firmware version: GET /redfish/v1/Managers/1 → .FirmwareVersion
│   → Update iLO firmware if outdated — see skills/deploy/firmware/README.md
│
├─ BiosOrHardwareHealth: "Warning"/"Critical"
│   → BIOS-detected hardware fault
│   → Check IML: GET /redfish/v1/Systems/1/LogServices/IML/Entries
│   → Look for Critical entries with Oem.Hpe.Class = "POST Messages"
│   → See Symptom: BIOS/UEFI Configuration Problem
│
├─ FanRedundancy: "Warning"
│   → Fan redundancy degraded (not failed, but below redundancy threshold)
│   → See Symptom: Thermal / Fan Alerts
│
├─ Fans: "Warning"/"Critical"
│   → One or more fans degraded or failed
│   → See Symptom: Thermal / Fan Alerts
│
├─ Memory: "Warning"/"Critical"
│   → DIMM error or failure
│   → See Symptom: Memory Errors
│
├─ Network: "Warning"/"Critical"
│   → NIC fault
│   → See Symptom: Network Connectivity Issues
│
├─ PowerSupplies: "Warning"/"Critical"
│   → PSU fault
│   → See Symptom: Power Supply Failure or Redundancy Lost
│
├─ PowerSupplyRedundancy: "Warning"
│   → PSU redundancy degraded (one PSU still functional)
│   → See Symptom: Power Supply Failure or Redundancy Lost
│
├─ Processors: "Warning"/"Critical"
│   → CPU fault or degraded state
│   → See Symptom: Processor CPU Fault
│
├─ SmartStorageBattery: "Warning"/"Critical"
│   → Storage controller cache battery/capacitor degraded
│   → Battery protects write cache; if failed, controller may disable write cache
│   → GET /redfish/v1/Systems/1/SmartStorage (Gen10) for battery status
│   → Schedule battery/capacitor replacement
│   → Escalate if write cache is disabled and performance is critical
│
├─ Storage: "Warning"/"Critical"
│   → Storage controller, logical drive, or physical drive issue
│   → See Symptom: Storage Failures
│
└─ Temperatures: "Warning"/"Critical"
    → Thermal threshold exceeded
    → See Symptom: Thermal / Fan Alerts
```

### Step 2: Confirm with chassis-level check

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1 \
  | jq '{
      Health: .Status.Health,
      HealthRollup: .Status.HealthRollup,
      IndicatorLED: .IndicatorLED
    }'
```

`IndicatorLED` values: `"Off"` (healthy), `"Blinking"` (attention), `"Lit"` (fault identified).

---

## Symptom: Memory Errors

### Step 1: Enumerate all DIMMs

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Memory \
  | jq '[.Members[]] | map({
      id: .["@odata.id"],
      Name: .Name,
      Health: .Status.Health,
      State: .Status.State
    })'
```

```
For each DIMM, check Status.Health:
│
├─ "OK" (all DIMMs healthy)
│   → If system still shows memory warning:
│       → Check OEM correctable error counters (see Step 2)
│       → Low correctable error count is normal; high/increasing count is not
│
├─ "Warning" (correctable errors accumulating)
│   → Inspect the specific DIMM:
│         GET /redfish/v1/Systems/1/Memory/{id}
│         Check .Oem.Hpe.ErrorCorrectionCount or equivalent error field
│   → If error count increasing:
│         → Schedule DIMM replacement at next maintenance window
│         → Identify replacement part:
│               .PartNumber   — HPE part number for ordering
│               .MemoryLocation.Socket / .Channel / .Slot — physical location
│               .CapacityMiB, .MemoryDeviceType, .OperatingSpeedMhz — spec match
│   → If error count stable:
│         → Monitor: re-check in 24 hours
│         → If >100 correctable errors in 24 hours: treat as "Critical" — replace
│
├─ "Critical" (uncorrectable error — DIMM has failed)
│   → DIMM has failed; server may have auto-disabled it
│   → Check IML for memory event:
│         GET /redfish/v1/Systems/1/LogServices/IML/Entries
│         ?$filter=Severity eq 'Critical'
│         Look for entries with "memory" in message, slot identification
│   → Note the physical slot from IML or DIMM MemoryLocation
│   → Replace DIMM immediately:
│         1. Schedule maintenance window (memory replacement = server down)
│         2. Power off server cleanly
│         3. Replace DIMM in identified slot
│         4. Power on, verify POST completes
│         5. Re-check: GET /redfish/v1/Systems/1/Memory/{id} → Health should be "OK"
│         6. Verify error counts cleared or zeroed
│   → If server fails to POST after replacement:
│         → DIMM may be incompatible or improperly seated
│         → Reseat DIMM; verify compatibility matrix
│         → See skills/reference/compatibility/README.md
│
└─ "Absent" / State: "Absent"
    → Slot is empty or DIMM is not detected
    → If DIMM should be present: reseat required
    → If unexpected absence: check IML for "memory removed" event
```

### Step 2: Check correctable error counts (OEM)

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Memory/proc1dimm1 \
  | jq '{
      Name: .Name,
      Location: .MemoryLocation,
      PartNumber: .PartNumber,
      Health: .Status.Health,
      OemData: .Oem.Hpe
    }'
```

Replace `proc1dimm1` with the actual DIMM ID from the collection. The `Oem.Hpe` block contains HPE-specific error counters when available.

---

## Symptom: Storage Failures

### Step 1: Determine generation and get controller status

```bash
# Gen11 — standard Redfish Storage
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage \
  | jq '[.Members[]] | map({
      id: .["@odata.id"],
      Health: .Status.Health,
      State: .Status.State
    })'

# Gen10 — HPE SmartStorage OEM path
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers \
  | jq '[.Members[]] | map({
      id: .["@odata.id"],
      Model: .Model,
      Health: .Status.Health,
      State: .Status.State
    })'
```

```
Controller Status:
│
├─ "OK" → Controller healthy; check logical and physical drives below
│
├─ "Warning" → Controller degraded
│   → GET /redfish/v1/Systems/1/Storage/{id} for detail
│   → Check IML for storage controller events
│   → Common causes: cache battery failing, firmware mismatch
│   → If cache battery: check SmartStorageBattery in AggregateHealthStatus
│   → Update controller firmware — see skills/deploy/firmware/README.md
│
└─ "Critical" → Controller failed
    → Do NOT attempt to modify RAID configuration
    → Collect IML and AHS logs
    → Escalate to HPE support immediately
    → See Escalation section at end of this skill
```

### Step 2: Check logical drive status

```bash
# Gen11
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/{ControllerId}/Volumes \
  | jq '[.Members[]] | map({
      Name: .Name,
      Health: .Status.Health,
      VolumeType: .VolumeType,
      RAIDType: .RAIDType,
      Capacity: .CapacityBytes
    })'
```

```
Logical Drive Status:
│
├─ "OK" → Logical drive healthy
│
├─ "Warning" (RAID degraded — member drive failed)
│   → A physical drive has failed; RAID is running without full redundancy
│   → Identify failed member:
│         GET /redfish/v1/Systems/1/Storage/{id}/Drives
│         Check each drive .Status.Health
│   → Is a hot spare present and rebuilding?
│         Check .Oem.Hpe.RebuildStatus or equivalent rebuild progress field
│         → If rebuild in progress: monitor progress, do not power off
│         → If no spare and no rebuild: identify failed drive, replace manually
│   → Replace failed drive:
│         1. Identify drive by slot (physical bay LED should be blinking amber)
│         2. Hot-swap replacement (if controller supports hot-swap)
│         3. Assign as replacement or assign hot spare if not auto-assigned
│         4. Monitor rebuild: re-check volume Health every 10 minutes
│         5. Rebuild time: ~2-4 hours per TB depending on workload
│   → After rebuild completes: verify Health returns to "OK"
│
├─ "Critical" (RAID failed — multiple drives lost, data loss likely)
│   → STOP — do not attempt any RAID modifications
│   → Do not power cycle if avoidable
│   → Immediate actions:
│         1. Collect IML: GET /redfish/v1/Systems/1/LogServices/IML/Entries
│         2. Download AHS: GET /redfish/v1/Systems/1/Oem/Hpe/AHSData
│         3. Note exact time of failure from IML
│   → Escalate to HPE support with IML and AHS logs
│   → Data recovery may require HPE or third-party recovery services
│
└─ Logical drive missing from collection:
    → Controller may have lost the volume configuration
    → Do NOT run RAID reconfiguration
    → Check IML for "logical drive deleted" or "array configuration changed"
    → Escalate to HPE support
```

### Step 3: Check physical drive status

```bash
# Gen11
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/{ControllerId}/Drives \
  | jq '[.Members[]] | map({
      Name: .Name,
      Health: .Status.Health,
      State: .Status.State,
      Location: .PhysicalLocation.PartLocation.ServiceLabel,
      CapacityGB: .CapacityBytes,
      PredictiveFailure: .PredictiveFailureAlertCapable
    })'
```

```
Physical Drive Status:
│
├─ "OK" → Drive healthy
│
├─ "Warning" (predictive failure alert)
│   → Drive SMART data indicates imminent failure
│   → Do NOT ignore — replace before actual failure
│   → Actions:
│         1. Identify bay: check .PhysicalLocation.PartLocation.ServiceLabel
│         2. Order replacement drive (same capacity, interface, speed)
│         3. Schedule replacement at next opportunity (do not wait for failure)
│         4. After replacement: rebuild if in RAID array
│
├─ "Critical" (drive failed)
│   → Drive has failed
│   → If in RAID: RAID is now degraded — see Logical Drive Warning branch above
│   → Replace immediately:
│         1. Identify bay by amber fault LED on drive bay
│         2. Hot-swap if controller supports it (most Gen10/Gen11 Smart Array do)
│         3. New drive auto-assigned if hot spare pool configured
│         4. Monitor rebuild progress
│
└─ Drive absent / not detected:
    → Drive may be unseated
    → Check bay for proper seating
    → Check IML for "drive removed" event to determine if this is expected
```

---

## Symptom: Thermal / Fan Alerts

### Step 1: Check temperature readings

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Thermal \
  | jq '{
      OverTemp: [.Temperatures[] | select(.Status.Health != "OK" and .Status.Health != null)],
      AllTemps: [.Temperatures[] | {
        Name: .Name,
        Reading: .ReadingCelsius,
        UpperCaution: .UpperThresholdNonCritical,
        UpperCritical: .UpperThresholdCritical,
        Health: .Status.Health
      }]
    }'
```

```
Temperature Status:
│
├─ All readings "OK" → No thermal fault at sensor level
│   → If AggregateHealthStatus.Temperatures still shows Warning:
│       → Trend check: is any sensor close to UpperThresholdNonCritical?
│       → Check fan status (Step 2) — insufficient airflow can cause temps to rise
│
├─ Reading above UpperThresholdNonCritical (Warning)
│   → Server approaching dangerous temperature
│   → Immediate actions:
│       → Check ambient room temperature (should be <27°C / 81°F for most configs)
│       → Check front-to-back airflow path: blanks installed in empty bays?
│       → Check fan status (Step 2)
│       → Verify no cables blocking airflow inside chassis
│       → If ambient OK and fans OK: check for recent workload spike
│
└─ Reading above UpperThresholdCritical (Critical)
    → CRITICAL: Server will perform emergency thermal shutdown if sustained
    → Check if server already shut down: GET /redfish/v1/Systems/1 → .PowerState
    → Immediate physical check required:
        → Room temperature
        → Cooling system (CRAC/CRAH unit) status
        → Blanking panels present in all empty rack units
        → Hot aisle/cold aisle containment integrity
    → Do NOT restart until thermal cause is resolved
    → See Symptom: Unexpected Reboot or Shutdown if shutdown has already occurred
```

### Step 2: Check fan status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Thermal \
  | jq '{
      FanIssues: [.Fans[] | select(.Status.Health != "OK" and .Status.Health != null)],
      AllFans: [.Fans[] | {
        Name: .Name,
        RPM: .Reading,
        Health: .Status.Health,
        State: .Status.State
      }],
      Redundancy: .Redundancy
    }'
```

```
Fan Status:
│
├─ All fans "OK" → Fan hardware healthy
│   → If temperatures still elevated: airflow path obstruction likely
│
├─ One fan "Warning" or "Critical" (single fan fault)
│   → Fan has degraded or failed
│   → Remaining fans will increase speed to compensate (fan redundancy feature)
│   → Actions:
│       → Identify fan by name/position from API response
│       → Fan fault LED on the fan module (amber)
│       → Hot-swap fan replacement on Gen10/Gen11 (no server shutdown required)
│       → After replacement: re-check Health within 2 minutes
│   → If server is in a high-ambient environment:
│       → Monitor temperatures closely during replacement — thermal risk is elevated
│
├─ Multiple fans "Warning"/"Critical" (fan redundancy lost)
│   → Thermal risk is high — temperatures will rise faster
│   → Immediate priority: replace failed fans
│   → If temperatures are already elevated: consider emergency workload reduction
│
└─ Fan at 0 RPM with State "Enabled":
    → Fan module has failed completely or is not detected
    → Reseat fan module first
    → If still 0 RPM after reseat: replace fan module
```

---

## Symptom: Power Supply Failure or Redundancy Lost

### Step 1: Check PSU status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '{
      PSUs: [.PowerSupplies[] | {
        Name: .Name,
        Health: .Status.Health,
        State: .Status.State,
        InputWatts: .PowerInputWatts,
        OutputWatts: .PowerOutputWatts,
        LineInputVoltage: .LineInputVoltage,
        Model: .Model,
        SerialNumber: .SerialNumber,
        PartNumber: .SparePartNumber,
        LastPowerOutputWatts: .LastPowerOutputWatts
      }],
      Redundancy: .Redundancy,
      PowerControl: [.PowerControl[] | {
        ConsumedWatts: .PowerConsumedWatts,
        CapacityWatts: .PowerCapacityWatts
      }]
    }'
```

```
PSU Status:
│
├─ Both PSUs "OK" → PSUs healthy
│   → If redundancy warning still shows: check Redundancy.Status.Health
│   → Redundancy may be "Warning" if PSUs are mismatched (different wattage)
│   → Verify both PSUs are same model/wattage for N+1 redundancy
│
├─ One PSU "Critical" or State "Absent" (single PSU fault)
│   → Server running on single PSU — no redundancy
│   → DO NOT perform risky operations (firmware updates, reboots) until resolved
│   → Is the server currently within the remaining PSU's capacity?
│         Check PowerConsumedWatts vs PSU OutputWatts
│         If consumption > 80% of remaining PSU capacity: reduce load before replacement
│   → Actions:
│       → Hot-swap replacement supported on all Gen10/Gen11 with dual PSU
│       → Identify PSU bay from API Name field ("Power Supply 1" = Bay 1)
│       → Replace with identical model (matching wattage and part number)
│       → After insertion: Health should return to "OK" within 30 seconds
│       → Re-verify redundancy status
│
└─ Both PSUs "Critical" (total power loss imminent or occurred)
    → Server may have already shut down
    → Check PowerState: GET /redfish/v1/Systems/1 → .PowerState
    → This is a critical event — check IML for timing and cause
    → Physical inspection: input power cables, PDU breakers, upstream power
    → Escalate to HPE support if no obvious physical cause
```

---

## Symptom: Network Connectivity Issues

```
What is the specific network symptom?
│
├─ NIC link is down (server OS cannot reach network)
│   → Step 1: Physical check
│         → Verify cable seated at server NIC port
│         → Verify cable seated at switch port
│         → Try known-good cable
│   → Step 2: Check NIC status via API
│         GET /redfish/v1/Chassis/1/NetworkAdapters
│         List adapters and ports:
│           jq '[.Members[]] | map(.["@odata.id"])'
│         For each adapter:
│           GET /redfish/v1/Chassis/1/NetworkAdapters/{id}
│           Check .Controllers[].Links.NetworkPorts
│         For each port:
│           GET /redfish/v1/Chassis/1/NetworkAdapters/{id}/Ports/{portId}
│           Check: .LinkStatus ("Up"/"Down"), .CurrentSpeedMbps, .ActiveLinkTechnology
│   → Step 3: If port shows "Down" with cable connected:
│         → Check switch port: is it enabled? Correct VLAN?
│         → Check NIC firmware version — outdated firmware can cause link issues
│         → Update NIC firmware — see skills/deploy/firmware/README.md
│   → Step 4: If port shows "Up" but OS reports no connectivity:
│         → OS-layer issue: check OS network configuration, drivers
│         → Not a hardware issue — outside scope of this skill
│
├─ iLO is unreachable (cannot reach management interface)
│   → Is the ping response present?
│   │   ├─ YES (ping responds but web/API does not):
│   │   │   → iLO process may be hung
│   │   │   → If server OS is accessible (SSH or console):
│   │   │         Use iLOrest in-band to reset iLO:
│   │   │           ilorest login --no-proxy
│   │   │           ilorest rawpost /redfish/v1/Managers/1/Actions/Manager.Reset \
│   │   │             '{"ResetType":"GracefulRestart"}'
│   │   │   → If OS is not accessible:
│   │   │         Physical: locate iLO reset button on rear panel
│   │   │         (small recessed button, labelled "iLO Reset", requires pin/paperclip)
│   │   │         Press and hold 10 seconds
│   │   │         Wait 90 seconds for iLO to reinitialise
│   │   │   → After reset: verify iLO API responds
│   │   │         curl -sk https://{ilo-ip}/redfish/v1 | jq .RedfishVersion
│   │   │
│   │   └─ NO (no ping response):
│   │       → Check network cable on iLO dedicated port (rear panel, "iLO" label)
│   │       → Verify switch port is active and in correct VLAN (management VLAN)
│   │       → Is shared NIC mode configured?
│   │             Shared NIC requires the server OS to be running for iLO to share
│   │             the host NIC — if OS is down, iLO is unreachable on shared NIC
│   │             Switch to dedicated iLO NIC if shared NIC is the issue
│   │       → Physical access: check LCD panel for iLO IP
│   │       → Physical access: UEFI F9 → iLO Configuration → View/Set IP
│   │       → If iLO still unreachable with confirmed network and cable:
│   │             iLO NIC hardware fault — escalate to HPE support
│
├─ Slow network throughput
│   → Check negotiated speed and duplex:
│         GET /redfish/v1/Chassis/1/NetworkAdapters/{id}/Ports/{portId}
│         Check: .CurrentSpeedMbps — does it match expected?
│         Expected: 1000 (GbE), 10000 (10GbE), 25000 (25GbE), etc.
│   → If speed is lower than expected:
│         → Auto-negotiation may have fallen back to lower speed
│         → Check switch port statistics for errors (CRC errors, FCS errors)
│         → Faulty cable or SFP/DAC transceiver — replace cable first
│         → If persists: check NIC and switch port duplex mismatch
│
└─ Intermittent connectivity drops
    → Check IML for NIC events:
          GET /redfish/v1/Systems/1/LogServices/IML/Entries
          ?$filter=contains(Message,'NIC') or contains(Message,'network')
    → Check for pattern: drops at specific time? Load-related?
    → NIC firmware bug: check HPE advisory for known issues
    → See skills/reference/known-issues/README.md
    → Consider NIC firmware update — see skills/deploy/firmware/README.md
```

---

## Symptom: Firmware Update Failed

### Step 1: Check update task status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/TaskService/Tasks \
  | jq '[.Members[]] | map({
      id: .["@odata.id"],
      Name: .Name,
      TaskState: .TaskState,
      TaskStatus: .TaskStatus,
      StartTime: .StartTime,
      EndTime: .EndTime,
      Messages: .Messages
    })'
```

Retrieve the specific failed task for full message detail:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/TaskService/Tasks/{taskId} \
  | jq '{
      TaskState: .TaskState,
      TaskStatus: .TaskStatus,
      Messages: [.Messages[] | {Message: .Message, Severity: .Severity}]
    }'
```

```
Task state interpretation:
│
├─ "Running" → Update is still in progress
│   → Wait; firmware updates can take 5-30 minutes
│   → Do NOT interrupt: power cycling during flash can brick the component
│   → Re-check every 2 minutes
│
├─ "Completed" with TaskStatus "OK" → Update succeeded
│   → The "failed" report may have been premature
│   → Verify component firmware version matches target:
│         GET /redfish/v1/Systems/1 → .BiosVersion (for BIOS)
│         GET /redfish/v1/Managers/1 → .FirmwareVersion (for iLO)
│
├─ "Exception" (update failed)
│   → Read the Messages array carefully — it contains the failure reason
│   │
│   ├─ "Incompatible firmware version" / "Version not supported"
│   │   → Firmware package does not match this hardware model or generation
│   │   → Verify target component part number matches firmware package
│   │   → Check compatibility matrix — see skills/reference/compatibility/README.md
│   │   → Download correct firmware from HPE Support (support.hpe.com)
│   │
│   ├─ "Insufficient flash storage" / "Not enough space"
│   │   → iLO flash partition is full
│   │   → Update iLO firmware first (iLO self-update frees space for other components)
│   │   → Then retry the target component update
│   │
│   ├─ "Update interrupted" / "Connection lost during update"
│   │   → Transfer was interrupted before flash began
│   │   → Component is likely still functional (flash not started)
│   │   → Retry the upload and update
│   │
│   ├─ "Signature verification failed"
│   │   → Firmware package is corrupted or tampered
│   │   → Re-download firmware package from HPE (check SHA256 hash)
│   │   → Ensure iLO Secure Boot / firmware verification settings allow this package
│   │
│   ├─ "Server must be in maintenance mode" / "Requires server reboot"
│   │   → Some components require the server to be off or in maintenance mode
│   │   → Schedule the update during a maintenance window with planned reboot
│   │   → See skills/deploy/firmware/README.md for staged update procedure
│   │
│   └─ Unrecognised error message:
│       → Capture full Messages array from task
│       → Check IML: GET /redfish/v1/Systems/1/LogServices/IML/Entries
│         Filter to the time of the update attempt
│       → Escalate to HPE support with task messages and IML
│
└─ "Killed" / "Interrupted" → Task was cancelled or timed out
    → Re-attempt update
    → If repeatedly killed: check iLO resource usage, consider iLO reset first
```

### Step 2: Verify component state after failure

If the update failed, confirm the component is still functional before retrying:

```bash
# Check BIOS version (if BIOS update failed)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{BiosVersion: .BiosVersion, Health: .Status.Health}'

# Check iLO firmware version (if iLO update failed)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1 \
  | jq '{FirmwareVersion: .FirmwareVersion, Health: .Status.Health}'
```

If the component is non-functional after a failed flash, check for rollback. See `skills/deploy/firmware/README.md` for the rollback procedure. If no rollback is possible, escalate to HPE support.

---

## Symptom: Unexpected Reboot or Shutdown

### Step 1: Confirm what happened

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{
      PowerState: .PowerState,
      Health: .Status.Health,
      HealthRollup: .Status.HealthRollup,
      LastResetReason: .LastResetType,
      PostState: .Oem.Hpe.PostState
    }'
```

### Step 2: Check IML around the time of the event

```bash
# Get the 20 most recent IML entries
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$orderby=Created desc&\$top=20" \
  | jq '[.Members[] | {
      Created: .Created,
      Severity: .Severity,
      Message: .Message,
      Class: .Oem.Hpe.Class
    }]'
```

```
Look for these IML events and follow the corresponding branch:
│
├─ "Thermal shutdown" / "Critical temperature exceeded"
│   → Server shut down to prevent hardware damage
│   → Do NOT restart until thermal cause is resolved
│   → Check current temperatures before restart:
│         GET /redfish/v1/Chassis/1/Thermal → check all .ReadingCelsius values
│   → Check ambient and airflow — see Symptom: Thermal / Fan Alerts
│   → Check fan status — a failed fan preceding the thermal event is common
│   → Only restart when temperatures are below UpperThresholdNonCritical
│
├─ "Power supply failure" / "Power fault"
│   → PSU failed and redundancy could not sustain load
│   → Check current PSU status: GET /redfish/v1/Chassis/1/Power
│   → Identify failed PSU, replace before restart
│   → See Symptom: Power Supply Failure or Redundancy Lost
│
├─ "Watchdog timeout" / "System watchdog reset"
│   → BMC or OS watchdog detected a hung system and issued a reset
│   → Two subtypes:
│         OS Watchdog: OS stopped responding; watchdog rebooted it
│           → Check OS logs (not iLO scope) for cause of hang before watchdog fired
│           → This is often a software/OS issue
│         iLO Watchdog: iLO's own watchdog fired
│           → Check iLO event log for iLO health events around same time
│           → GET /redfish/v1/Managers/1/LogServices/IEL/Entries
│
├─ "Server reset by user" / "Reset initiated"
│   → A user or automated process issued a reset command
│   → Check iLO event log to identify who initiated:
│         GET /redfish/v1/Managers/1/LogServices/IEL/Entries
│         Look for reset/power action entries with username
│   → Check iLO Event Log for the initiating account
│   → If automated system (monitoring tool, ILO Amplifier): check that system
│
├─ "Server powered off" / "Graceful shutdown"
│   → Ordered shutdown — OS shut down cleanly before server powered off
│   → Most likely a scheduled event or administrator action
│   → Check iLO event log for who initiated
│
├─ "NMI" / "Non-Maskable Interrupt"
│   → Hardware-level interrupt indicating hardware fault
│   → Check for accompanying memory, CPU, or PCIe errors in same time window
│   → Collect AHS logs: GET /redfish/v1/Systems/1/Oem/Hpe/AHSData
│   → Escalate to HPE support with AHS if hardware fault suspected
│
└─ No relevant IML entry found:
    → Power event may not have been logged if iLO lost power simultaneously
    → Check upstream power infrastructure (PDU logs, UPS logs)
    → Check OS-level crash logs if OS is accessible after restart
    → Download AHS for HPE support analysis
```

### Step 3: Check iLO event log for management actions

```bash
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Managers/1/LogServices/IEL/Entries?\$orderby=Created desc&\$top=20" \
  | jq '[.Members[] | {
      Created: .Created,
      Severity: .Severity,
      Message: .Message,
      Username: .Oem.Hpe.Username
    }]'
```

The `Oem.Hpe.Username` field identifies which iLO account initiated the action.

---

## Symptom: iLO Unresponsive

```
What is the exact failure mode?
│
├─ iLO IP responds to ping but web interface / Redfish API does not respond
│   → iLO management daemon is hung while network stack is alive
│   → Step 1: Try iLO reset via in-band (if server OS is reachable):
│         SSH to the server OS
│         If ilorest is installed:
│           ilorest login --no-proxy
│           ilorest rawpost /redfish/v1/Managers/1/Actions/Manager.Reset \
│             '{"ResetType":"GracefulRestart"}'
│         Via curl from within the OS (if OS has iLO access):
│           curl -sk -u admin:password \
│             -X POST https://localhost/redfish/v1/Managers/1/Actions/Manager.Reset \
│             -H "Content-Type: application/json" \
│             -d '{"ResetType":"GracefulRestart"}'
│   → Step 2: If OS is not accessible or in-band reset fails:
│         Physical: locate iLO reset button on rear panel
│         Labelled "iLO", recessed, requires pin or straightened paperclip
│         Press and hold 10 seconds
│         Wait 90 seconds for iLO to restart
│   → Step 3: After reset, verify iLO API responds:
│         curl -sk https://{ilo-ip}/redfish/v1 | jq .RedfishVersion
│   → If iLO remains unresponsive after physical reset:
│         iLO hardware fault — escalate to HPE support
│
├─ iLO IP does not respond to ping
│   → Network-layer issue (no connectivity) or iLO hardware fault
│   → Step 1: Verify iLO network configuration
│         Physical: check cable on iLO dedicated NIC port (rear panel, "iLO" label)
│         Physical: verify switch port is active (link LED on switch)
│   → Step 2: Confirm iLO IP address
│         Physical LCD panel: iLO IP is displayed (Gen10/Gen11 have LCD)
│         UEFI: press F9 during POST → iLO Configuration → View Network Settings
│   → Step 3: Shared NIC check
│         If configured for shared NIC mode: iLO shares the host's network port
│         Shared NIC requires OS to be running for iLO to use the port
│         If OS is down, iLO on shared NIC is unreachable by design
│         Workaround: configure iLO for dedicated NIC port, or access via OS console
│   → Step 4: If network is confirmed and IP is correct and still no ping:
│         iLO firmware may be corrupted or iLO hardware has failed
│         Physical iLO reset (see above)
│         If no recovery: escalate to HPE support — iLO replacement may be required
│
└─ iLO responds intermittently
    → Check iLO health: when accessible, GET /redfish/v1/Managers/1 → .Status.Health
    → Check iLO firmware version — known iLO bugs can cause intermittent hangs
    → Check iLO event log for internal errors when accessible
    → Update iLO firmware to latest — see skills/deploy/firmware/README.md
    → If intermittency persists after firmware update: escalate to HPE support
```

---

## Symptom: Processor (CPU) Fault

### Step 1: Check processor status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Processors \
  | jq '[.Members[]] | map({
      id: .["@odata.id"],
      Socket: .Socket,
      Model: .Model,
      Health: .Status.Health,
      State: .Status.State,
      TotalCores: .TotalCores,
      TotalThreads: .TotalThreads,
      MaxSpeedMHz: .MaxSpeedMHz
    })'
```

```
CPU Status:
│
├─ All CPUs "OK" → CPU hardware healthy
│   → If AggregateHealthStatus.Processors still shows warning:
│       → Check for thermal throttling in IML
│       → Check if all CPU cores are enabled (BIOS settings)
│
├─ CPU "Warning"
│   → CPU degraded — may be thermal throttling or configuration issue
│   → Check IML for CPU-related events:
│         GET /redfish/v1/Systems/1/LogServices/IML/Entries
│         ?$filter=contains(Message,'processor') or contains(Message,'CPU')
│   → Check thermal status — throttling occurs when temps approach limits
│   → If persistent Warning without thermal cause: CPU hardware degradation
│         → Schedule replacement; escalate to HPE support for CPU RMA
│
├─ CPU "Critical"
│   → CPU has failed or has uncorrectable errors
│   → Check IML for CPU fault events
│   → Server may not POST if primary CPU (CPU 1) has failed
│   → CPU replacement is a complex operation:
│         → Escalate to HPE support
│         → CPU replacement requires specialised tooling and anti-static procedures
│         → Do not attempt without HPE guidance
│
└─ CPU State "Absent" (socket should be populated)
    → CPU is not detected in socket
    → Physical: verify CPU is properly seated
    → If recently installed: re-seat with correct torque (HPE specifies in server guide)
    → If not recently installed: CPU may have shifted — server guide for re-seating
    → After re-seating: retry POST
```

---

## Symptom: BIOS/UEFI Configuration Problem

### Step 1: Check BIOS health and current settings

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{
      BiosVersion: .BiosVersion,
      Health: .Status.Health,
      PostState: .Oem.Hpe.PostState,
      BootSourceOverride: .Boot,
      SecureBoot: .SecureBoot
    }'
```

```
Common BIOS/UEFI problems:
│
├─ Server POST halts with BIOS error message
│   → Check IML for POST error detail:
│         GET /redfish/v1/Systems/1/LogServices/IML/Entries
│         ?$filter=Oem/Hpe/Class eq 'POST Messages'
│   → Common POST halt causes:
│         "CPU not supported" → CPU incompatible with current BIOS version
│           → BIOS update required before installing new CPU generation
│         "Memory configuration error" → DIMM in wrong slot or unsupported config
│           → Check server QuickSpecs for supported memory configurations
│         "No bootable device" → Boot device missing or misconfigured
│           → See Boot configuration below
│
├─ Server boots to wrong device / wrong OS
│   → Check boot order:
│         GET /redfish/v1/Systems/1/Bios
│         Look for BootOrderPolicy or persistent boot order settings
│   → One-time boot override (does not change persistent order):
│         PATCH /redfish/v1/Systems/1
│         Body: {"Boot": {"BootSourceOverrideTarget": "Pxe",
│                         "BootSourceOverrideEnabled": "Once"}}
│         Valid targets: "None", "Pxe", "Hdd", "Cd", "Usb", "SDCard", "UefiTarget"
│   → Persistent boot order changes: use BIOS settings via iLO or UEFI
│         POST /redfish/v1/Systems/1/Bios/Settings
│         (Requires server reboot to apply)
│
├─ Secure Boot blocking OS or driver load
│   → Check Secure Boot state:
│         GET /redfish/v1/Systems/1/SecureBoot
│         Check: .SecureBootEnable, .SecureBootCurrentBoot ("Enabled"/"Disabled")
│   → To temporarily disable for testing:
│         PATCH /redfish/v1/Systems/1/SecureBoot
│         Body: {"SecureBootEnable": false}
│         Reboot required
│   → To add custom key (for unsigned drivers/bootloader):
│         GET /redfish/v1/Systems/1/SecureBoot/SecureBootDatabases
│         Enrol key into DB (Signature Database) via API or UEFI UI
│
└─ BIOS settings reset to defaults unexpectedly
    → Check IML for "NVRAM cleared" or "BIOS defaults loaded" event
    → Common cause: CMOS battery failure (server loses settings on power loss)
    → Physical: replace CMOS battery (coin cell, location in server guide)
    → After battery replacement: reconfigure BIOS settings
    → For large deployments: use BIOS baseline profile to restore settings
          See skills/deploy/bios-uefi/README.md
```

---

## Escalation: When to Call HPE Support

### Escalate immediately when:

- Hardware failure confirmed and replacement is not straightforward (CPU, system board, iLO hardware)
- IML shows critical hardware event that is not user-fixable (NMI without clear cause, persistent CPU errors)
- RAID logical drive is in "Failed" state (data recovery risk)
- Multiple component failures suggesting a systemic or cascading issue
- Server will not POST after reseating all components and following decision trees
- Firmware update has left a component non-functional and no rollback is available
- iLO remains unreachable after physical reset

### What to prepare before calling HPE support:

```bash
# 1. Capture system identity
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{Model, SerialNumber, UUID, BiosVersion}'

# 2. Export recent IML (last 50 critical and warning entries)
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Severity ne 'OK'&\$orderby=Created desc&\$top=50" \
  | jq '[.Members[] | {Created, Severity, Message}]' > iml_export.json

# 3. Download AHS (Active Health System) log — contains full hardware history
# AHS download returns a binary .ahs file; save it
curl -sk -u admin:password \
  -o ahs_$(date +%Y%m%d).ahs \
  "https://192.168.1.101/redfish/v1/Systems/1/Oem/Hpe/AHSData/AHSDataFile"
```

Have ready:
- Server serial number (from API or rear panel sticker)
- IML export covering the time of the fault
- AHS log file (HPE support will analyse this automatically)
- Description of the fault: what was observed, when it started, what changed before it started
- Current status: is the server up, degraded, or completely down?

### HPE support channels:

- HPE Support Portal: support.hpe.com
- HPE Warranty/contract check: includes SLA response time for your hardware
- For critical outages: escalate to 24/7 support if under active support contract

---

## Cross-References

| Topic | Skill |
|---|---|
| Reviewing IML and AHS logs in depth | `skills/diagnose/log-analysis/README.md` |
| Running a full health check before operations | `skills/operations/health-check/README.md` |
| Updating firmware | `skills/deploy/firmware/README.md` |
| BIOS/UEFI configuration and baselines | `skills/deploy/bios-uefi/README.md` |
| Storage RAID configuration | `skills/deploy/storage/README.md` |
| Networking and NIC configuration | `skills/deploy/networking/README.md` |
| iLO connectivity and setup | `skills/foundation/connectivity/README.md` |
| HPE hardware concepts | `skills/foundation/concepts/README.md` |
| Compatibility matrices | `skills/reference/compatibility/README.md` |
| Known issues and HPE advisories | `skills/reference/known-issues/README.md` |
| Server lifecycle (decommission, AHS collection) | `skills/operations/lifecycle/README.md` |
