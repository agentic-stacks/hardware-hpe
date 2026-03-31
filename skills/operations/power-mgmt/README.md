# Operations: Power Management

Power state control, consumption monitoring, power capping, PSU health, and UID LED operations for HPE ProLiant servers via iLO Redfish.

---

## Prerequisites

- iLO connectivity established — see `skills/foundation/connectivity/README.md`
- `curl` and `jq` available on the workstation
- Credentials with at least **Operator** role for power state changes; **ReadOnly** for monitoring
- Power capping and power regulator features require **iLO Advanced** license

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| System (power state, reset action) | `/redfish/v1/Systems/1` | GET, PATCH |
| Reset action target | `/redfish/v1/Systems/1/Actions/ComputerSystem.Reset` | POST |
| Chassis (UID LED) | `/redfish/v1/Chassis/1` | GET, PATCH |
| Power (consumption, PSUs, capping) | `/redfish/v1/Chassis/1/Power` | GET, PATCH |

---

## Power State Operations

### Check Current Power State

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '.PowerState'
```

Valid return values:

| Value | Meaning |
|---|---|
| `"On"` | Server is powered on and running |
| `"Off"` | Server is powered off (iLO still active) |
| `"PoweringOn"` | Transitioning to powered-on state (iLO 6 v1.10+) |
| `"PoweringOff"` | Transitioning to powered-off state (iLO 6 v1.10+) |

> iLO remains accessible regardless of `PowerState` — it runs from standby power. `PowerState` reflects the main server, not iLO.

### ResetType Values

| ResetType | Effect | OS Involvement |
|---|---|---|
| `On` | Powers on from off state | None — hardware only |
| `ForceOff` | Immediate power cut, like pulling the plug | None — OS gets no warning |
| `GracefulShutdown` | Sends ACPI shutdown signal; OS performs clean shutdown | OS shuts down cleanly |
| `ForceRestart` | Immediate hardware reboot without OS notification | None — OS gets no warning |
| `GracefulRestart` | Sends ACPI restart signal; OS reboots cleanly | OS restarts cleanly |
| `PushPowerButton` | Simulates a physical front-panel button press | Depends on OS power button policy |
| `Nmi` | Generates a Non-Maskable Interrupt | OS captures for kernel dump/debugging |
| `PowerCycle` | Full power cycle (off then on) | None — hardware only |

**Safety rule:** Always attempt `GracefulShutdown` or `GracefulRestart` first. Use `ForceOff` or `ForceRestart` only if the OS is unresponsive and graceful methods have timed out.

### Power On a Server

Use when the server is in `Off` state:

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "On"}' \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset
```

Expected response: `HTTP 200 OK` with empty body or success message.

### Graceful Shutdown

OS writes are flushed; services stop cleanly. Preferred for planned maintenance:

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "GracefulShutdown"}' \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset
```

> Wait for `PowerState` to reach `"Off"` before proceeding. Poll with `GET /redfish/v1/Systems/1 | jq '.PowerState'`.

### Force Power Off

Use only when the OS is completely unresponsive and graceful shutdown has failed:

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "ForceOff"}' \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset
```

**Risk:** Data loss or filesystem corruption is possible. Only use as a last resort.

### Graceful Restart

Cleanest way to reboot a healthy OS:

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "GracefulRestart"}' \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset
```

### Force Restart

Immediate hardware reboot — use when OS is hung and graceful restart does not respond:

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "ForceRestart"}' \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset
```

### Simulate Power Button Press

Behavior depends on the OS power button configuration (most Linux/Windows defaults: initiate shutdown):

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "PushPowerButton"}' \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset
```

### iLOrest Equivalents

```bash
# Power on
ilorest reboot On --url https://192.168.1.101 -u admin -p password

# Graceful shutdown
ilorest reboot GracefulShutdown --url https://192.168.1.101 -u admin -p password

# Force off
ilorest reboot ForceOff --url https://192.168.1.101 -u admin -p password

# Graceful restart
ilorest reboot GracefulRestart --url https://192.168.1.101 -u admin -p password

# Force restart
ilorest reboot ForceRestart --url https://192.168.1.101 -u admin -p password

# Check power state only
ilorest get PowerState --select ComputerSystem. \
  --url https://192.168.1.101 -u admin -p password
```

---

## Power Consumption Monitoring

### Current Wattage

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.PowerControl[] | {
      Name,
      PowerConsumedWatts,
      PowerCapacityWatts,
      PowerRequestedWatts: .PowerMetrics.AverageConsumedWatts
    }'
```

Key fields in `PowerControl[]`:

| Field | Description |
|---|---|
| `PowerConsumedWatts` | Current power draw in watts (instantaneous) |
| `PowerCapacityWatts` | Maximum rated power capacity of the server |
| `PowerRequestedWatts` | Power requested by the server (may differ from consumed) |

### Historical Power Metrics

iLO tracks rolling min/average/max across a 24-hour window:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.PowerControl[] | {
      Name,
      CurrentWatts: .PowerConsumedWatts,
      AverageWatts: .PowerMetrics.AverageConsumedWatts,
      MinWatts: .PowerMetrics.MinConsumedWatts,
      MaxWatts: .PowerMetrics.MaxConsumedWatts,
      IntervalMinutes: .PowerMetrics.IntervalInMin
    }'
```

Sample output:
```json
{
  "Name": "Server Power Control",
  "CurrentWatts": 312,
  "AverageWatts": 298,
  "MinWatts": 245,
  "MaxWatts": 385,
  "IntervalMinutes": 20
}
```

`IntervalInMin` is the rolling window over which min/average/max are calculated. This window resets automatically.

### Power Meter Readings (OEM)

HPE provides a more detailed power history via OEM extensions. Available on servers with iLO Advanced license:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.Oem.Hpe.PowerAllocatedWatts,
        .Oem.Hpe.PowerAvailableWatts,
        .Oem.Hpe.PowerCapacityWatts,
        .Oem.Hpe.PowerLimit.ActualSetPoints'
```

### iLOrest Power Monitoring

```bash
# View full power resource
ilorest get --select Power. --url https://192.168.1.101 -u admin -p password

# Get just consumption values
ilorest get PowerConsumedWatts --select Power. \
  --url https://192.168.1.101 -u admin -p password
```

---

## Power Capping

Power capping sets a hard ceiling on how many watts the server can draw. iLO enforces this by throttling CPU and memory performance to keep consumption under the limit.

**When to use power capping:**
- Data center PDU circuits have a fixed amperage budget shared across many servers
- You need to guarantee power draw stays within a rack's capacity
- Compliance requirements mandate maximum power consumption per server

> **Warning:** Power capping throttles CPU and memory. Workload performance will degrade in proportion to how aggressively the cap is set. Never cap below the server's idle power draw — this causes instability. Set caps at least 20% above average observed consumption.

### Read the Current Power Cap

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.PowerControl[] | {
      Name,
      LimitInWatts: .PowerLimit.LimitInWatts,
      LimitException: .PowerLimit.LimitException,
      CorrectionInMs: .PowerLimit.CorrectionInMs
    }'
```

| Field | Description |
|---|---|
| `LimitInWatts` | Active power cap in watts (`null` = no cap) |
| `LimitException` | Action if limit is exceeded: `"NoAction"`, `"HardPowerOff"`, `"LogEventOnly"`, `"Oem"` |
| `CorrectionInMs` | How quickly (in milliseconds) iLO corrects to stay under the cap |

### Set a Power Cap

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{
    "PowerControl": [{
      "PowerLimit": {
        "LimitInWatts": 400,
        "LimitException": "LogEventOnly"
      }
    }]
  }' \
  https://192.168.1.101/redfish/v1/Chassis/1/Power
```

Replace `400` with your target watt ceiling. `LogEventOnly` records an IML event if the cap is hit without powering off the server.

### Remove a Power Cap

Set `LimitInWatts` to `null` to disable capping:

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{
    "PowerControl": [{
      "PowerLimit": {
        "LimitInWatts": null
      }
    }]
  }' \
  https://192.168.1.101/redfish/v1/Chassis/1/Power
```

### Monitor Capping Status

After setting a cap, verify it was accepted and check if the server is currently being throttled:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.PowerControl[] | {
      CurrentWatts: .PowerConsumedWatts,
      CapWatts: .PowerLimit.LimitInWatts,
      Headroom: (.PowerLimit.LimitInWatts - .PowerConsumedWatts)
    }'
```

Negative `Headroom` means the server is at or over the cap and iLO is actively throttling.

---

## Power Supply Status

### PSU Inventory and Health

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.PowerSupplies[] | {
      Name,
      Model,
      SerialNumber,
      PowerSupplyType,
      LineInputVoltage,
      LastPowerOutputWatts,
      Health: .Status.Health,
      State: .Status.State,
      FirmwareVersion
    }'
```

Key `Status.Health` values:

| Health | Meaning |
|---|---|
| `"OK"` | PSU is healthy and operating normally |
| `"Warning"` | PSU has a non-critical issue — monitor closely |
| `"Critical"` | PSU has failed or is about to fail — act immediately |

`Status.State` values:

| State | Meaning |
|---|---|
| `"Enabled"` | PSU is present and active |
| `"Absent"` | PSU slot is empty |
| `"UnavailableOffline"` | PSU is present but not currently supplying power (standby in redundant mode) |

### PSU Redundancy Status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.Redundancy[] | {
      Name,
      Mode: .Mode,
      Health: .Status.Health,
      MinNumNeeded,
      MaxNumSupported,
      RedundancySet: [.RedundancySet[].@odata.id]
    }'
```

Redundancy modes:

| Mode | Description |
|---|---|
| `"Failover"` | N+1 — one PSU can fail and the other carries full load |
| `"Sparing"` | Hot spare — standby PSU takes over on failure |
| `"Sharing"` | Load sharing across all PSUs (non-redundant but efficient) |
| `"NotRedundant"` | Single PSU, no redundancy — loss of this PSU = server power loss |

### Gen11 High-Efficiency Mode (OEM)

Gen11 servers support domain-based PSU redundancy and efficiency modes:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.Oem.Hpe.Domains[] | {
      DomainName,
      HighEfficiencyMode,
      PowerSupplyRedundancy
    }'
```

`HighEfficiencyMode` values: `"Auto"`, `"Balanced"`, `"Even"`, `"Odd"`. In `"Auto"` mode, iLO keeps one PSU in standby and routes all load through the other for maximum efficiency, activating the standby only on demand.

### Responding to a PSU Failure

1. Identify which PSU failed:
   ```bash
   curl -sk -u admin:password \
     https://192.168.1.101/redfish/v1/Chassis/1/Power \
     | jq '.PowerSupplies[] | select(.Status.Health != "OK") | {Name, Health: .Status.Health, State: .Status.State}'
   ```

2. Check if redundancy is maintained:
   ```bash
   curl -sk -u admin:password \
     https://192.168.1.101/redfish/v1/Chassis/1/Power \
     | jq '.Redundancy[] | {Name, Health: .Status.Health}'
   ```
   - `"OK"` redundancy + one failed PSU = safe to hot-swap the failed unit
   - `"Critical"` redundancy = server is running on a single PSU with no failover — hot-swap urgently or schedule a maintenance window

3. HPE ProLiant PSUs are hot-swappable when redundancy is intact. Do not power off the server to replace a PSU if the remaining PSU can carry the full load.

4. After replacement, verify the new PSU comes online:
   ```bash
   curl -sk -u admin:password \
     https://192.168.1.101/redfish/v1/Chassis/1/Power \
     | jq '.PowerSupplies[] | {Name, Health: .Status.Health, State: .Status.State, LastPowerOutputWatts}'
   ```

---

## UID LED Control

The Unit Identification (UID) LED is a blue LED on the front and rear of every HPE ProLiant server. Activating it lets you physically locate a specific server in a dense rack without having to read serial numbers.

**When to use:** Before physically accessing a server for hardware replacement, cabling work, or decommissioning — especially in colocation facilities or large data halls.

### Check Current UID State

**iLO 6 (Gen11) — preferred method:**

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '.LocationIndicatorActive'
```

Returns `true` (UID on) or `false` (UID off).

**iLO 5 (Gen10) — legacy method:**

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1 \
  | jq '.IndicatorLED'
```

Returns `"Lit"`, `"Off"`, or `"Blinking"`.

### Turn UID LED On

**iLO 6 (Gen11):**

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"LocationIndicatorActive": true}' \
  https://192.168.1.101/redfish/v1/Systems/1
```

**iLO 5 (Gen10) / backward-compatible OEM path:**

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"IndicatorLED": "Lit"}' \
  https://192.168.1.101/redfish/v1/Chassis/1
```

### Turn UID LED Off

**iLO 6 (Gen11):**

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"LocationIndicatorActive": false}' \
  https://192.168.1.101/redfish/v1/Systems/1
```

**iLO 5 (Gen10):**

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"IndicatorLED": "Off"}' \
  https://192.168.1.101/redfish/v1/Chassis/1
```

### Set UID LED to Blinking (iLO 5 / OEM)

Blinking is useful to distinguish a specific server when multiple UIDs are on nearby:

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"IndicatorLED": "Blinking"}' \
  https://192.168.1.101/redfish/v1/Chassis/1
```

> `IndicatorLED` is deprecated in iLO 6 in favor of the boolean `LocationIndicatorActive`. `Blinking` is only available via the OEM `Oem.Hpe.IndicatorLED` extension on Gen11. Use `LocationIndicatorActive: true` for standard Gen11 automation.

### iLOrest UID Control

```bash
# Turn UID on (Gen11)
ilorest set LocationIndicatorActive=True --select ComputerSystem. \
  --url https://192.168.1.101 -u admin -p password && ilorest commit

# Turn UID off (Gen11)
ilorest set LocationIndicatorActive=False --select ComputerSystem. \
  --url https://192.168.1.101 -u admin -p password && ilorest commit
```

---

## Power Efficiency Modes

### BIOS Power Regulator

HPE ProLiant servers have a BIOS-level power regulator that controls how the CPU scales frequency and voltage in response to load. This is configured via BIOS settings, not via the Power resource directly.

```bash
# Read current power regulator setting
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios \
  | jq '.Attributes.PowerRegulator'
```

| PowerRegulator Value | Description |
|---|---|
| `"DynamicPowerSavings"` | OS controls frequency scaling (OS-native power management) |
| `"StaticLowPower"` | CPU locked at minimum frequency — maximum power savings, reduced performance |
| `"StaticHighPerf"` | CPU locked at maximum frequency — best performance, highest power |
| `"OsControl"` | Passes control to the OS power management governor (recommended for most workloads) |

### HPE Power Profiles

HPE servers ship with pre-configured power profiles accessible via BIOS. Common profiles:

| Profile | Best For |
|---|---|
| Balanced Power and Performance | General-purpose workloads; default for most deployments |
| Maximum Performance | Latency-sensitive or CPU-intensive workloads (HPC, databases) |
| Minimum Power Usage | Dev/test environments where performance is not critical |
| Custom | Fine-grained control over individual BIOS power settings |

To change the power regulator, PATCH to the BIOS settings (changes apply after reboot):

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"Attributes": {"PowerRegulator": "StaticHighPerf"}}' \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings
```

Then reboot for the change to take effect. See `skills/deploy/bios-uefi/README.md` for full BIOS change workflow.

---

## Common Scenarios

### Scenario: Planned Maintenance Window

```bash
ILO=192.168.1.101
AUTH="-u admin:password"

# 1. Turn on UID so the physical server is identifiable
curl -sk $AUTH -X PATCH -H "Content-Type: application/json" \
  -d '{"LocationIndicatorActive": true}' \
  https://$ILO/redfish/v1/Systems/1

# 2. Confirm server is on and healthy
curl -sk $AUTH https://$ILO/redfish/v1/Systems/1 \
  | jq '{PowerState, Health: .Status.Health}'

# 3. Graceful shutdown
curl -sk $AUTH -X POST -H "Content-Type: application/json" \
  -d '{"ResetType": "GracefulShutdown"}' \
  https://$ILO/redfish/v1/Systems/1/Actions/ComputerSystem.Reset

# 4. Poll until off
until [ "$(curl -sk $AUTH https://$ILO/redfish/v1/Systems/1 | jq -r '.PowerState')" = "Off" ]; do
  echo "Waiting for shutdown..."; sleep 10
done
echo "Server is off — safe to proceed with physical work"
```

### Scenario: Emergency Power Cut (OS Unresponsive)

```bash
ILO=192.168.1.101
AUTH="-u admin:password"

# First try graceful — give it 60 seconds
curl -sk $AUTH -X POST -H "Content-Type: application/json" \
  -d '{"ResetType": "GracefulShutdown"}' \
  https://$ILO/redfish/v1/Systems/1/Actions/ComputerSystem.Reset

sleep 60

STATE=$(curl -sk $AUTH https://$ILO/redfish/v1/Systems/1 | jq -r '.PowerState')
if [ "$STATE" != "Off" ]; then
  echo "Graceful shutdown failed — issuing ForceOff"
  curl -sk $AUTH -X POST -H "Content-Type: application/json" \
    -d '{"ResetType": "ForceOff"}' \
    https://$ILO/redfish/v1/Systems/1/Actions/ComputerSystem.Reset
fi
```

### Scenario: Check Power Budget Before Adding Load

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/Power \
  | jq '.PowerControl[] | {
      CurrentWatts: .PowerConsumedWatts,
      CapacityWatts: .PowerCapacityWatts,
      CapWatts: .PowerLimit.LimitInWatts,
      Utilization: ((.PowerConsumedWatts / .PowerCapacityWatts * 100) | round | tostring + "%")
    }'
```

---

## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| `ResetType: GracefulShutdown` returns 200 but server stays On | OS is not responding to ACPI signal | Check OS via remote console; escalate to `ForceOff` after timeout |
| `ResetType: On` returns 400 | Server is already on, or `ResetType: On` is not allowed when powered on | Check current `PowerState` first; use `ForceRestart` to cycle a running server |
| Power cap set but consumption still exceeds it | Cap is too close to idle power; iLO cannot throttle fast enough | Raise cap by at least 10% above peak observed consumption |
| PSU Health `"Warning"` | PSU is degraded — could be high inlet temp, over-voltage, or fan failure in the PSU | Replace PSU during next maintenance window; monitor redundancy status |
| PSU Health `"Critical"` with Redundancy `"Critical"` | Both PSUs failed or one PSU is absent and the remaining PSU is degraded | Immediate intervention required — server may power off without warning |
| `IndicatorLED` PATCH returns 404 on Gen11 | `IndicatorLED` deprecated on Gen11 | Use `LocationIndicatorActive` on `Systems/1` instead |
| `PowerState` shows `"On"` but OS is unreachable | OS has crashed or network is down | Open remote console via iLO to diagnose OS state |
