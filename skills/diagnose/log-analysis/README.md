# Diagnose: Log Analysis

Deep log interpretation for HPE ProLiant servers: IML hardware events, iLO Event Log management activity, and Active Health System telemetry. This skill is the detailed investigation layer — the troubleshooting skill routes here when logs need systematic analysis.

> **When to use this skill:** You have a hardware event to investigate, you need to establish a timeline of what happened, you are preparing for an HPE support call, or you are doing proactive maintenance review. For symptom-based triage, start at `skills/diagnose/troubleshooting/README.md` first.

---

## The Three Log Types

| Log | Full Name | What It Records | Retention | Access |
|-----|-----------|-----------------|-----------|--------|
| IML | Integrated Management Log | Hardware events: failures, warnings, repairs, POST errors, component status changes | Persistent — survives reboots, power loss, and OS reinstall | Redfish + iLOrest |
| IEL | iLO Event Log | Management events: logins, config changes, firmware updates, remote console sessions, alerts | Persistent — survives reboots, power loss | Redfish + iLOrest |
| AHS | Active Health System | Continuous telemetry: temperatures, voltages, fan speeds, error counters, config snapshots — sampled on a rolling basis | Rolling ~7 days of dense data; older data is compressed | Download binary via Redfish OEM endpoint |

**Why all three matter:** IML tells you *what* hardware event occurred. IEL tells you *who or what* triggered management activity around the same time. AHS gives you the environmental *context* — what the sensors showed in the minutes and hours surrounding the event. Together they form a complete picture.

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| IML entries | `/redfish/v1/Systems/1/LogServices/IML/Entries` | GET |
| IML single entry | `/redfish/v1/Systems/1/LogServices/IML/Entries/{id}` | GET |
| IML clear | `/redfish/v1/Systems/1/LogServices/IML/Actions/LogService.ClearLog` | POST |
| IEL entries | `/redfish/v1/Managers/1/LogServices/IEL/Entries` | GET |
| IEL single entry | `/redfish/v1/Managers/1/LogServices/IEL/Entries/{id}` | GET |
| IEL clear | `/redfish/v1/Managers/1/LogServices/IEL/Actions/LogService.ClearLog` | POST |
| AHS download (OEM) | `/redfish/v1/Systems/1/Oem/Hpe/AHSData` | GET |
| AHS capture action | `/redfish/v1/Managers/1/ActiveHealthSystem/Actions/HpeiLOActiveHealthSystem.CaptureSystemLog` | POST |

---

## Reading the IML

### Fetch All Entries

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries \
  | jq '.Members[] | {
      Id,
      Severity: .Severity,
      Created: .Created,
      Message: .Message,
      Class: .Oem.Hpe.ClassDescription,
      Code: .Oem.Hpe.Code,
      Count: .Oem.Hpe.Count,
      Repaired: .Oem.Hpe.Repaired,
      Action: .Oem.Hpe.RecommendedAction
    }'
```

### Fetch Only Critical Entries (iLO 6 OData filter)

iLO 6 supports OData `$filter` on log entries:

```bash
# Standard Severity field
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Severity eq 'Critical'" \
  | jq '.Members[] | {Id, Created, Message, Severity}'

# HPE OEM severity (iLO 6 — includes "Repaired" state)
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Oem.Hpe.Severity eq 'Critical'" \
  | jq '.Members[] | {Id, Created, Message}'
```

### Fetch Recent Entries (pagination)

```bash
# Last 20 entries
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$top=20" \
  | jq '[.Members[] | {Id, Severity, Created, Message}]'

# Entries since a specific date
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Created gt '2026-01-15T00:00:00Z'" \
  | jq '.Members[] | {Id, Severity, Created, Message}'
```

### Via iLOrest

```bash
ilorest login 192.168.1.101 -u admin -p password
ilorest serverlogs --selectlog=IML
ilorest serverlogs --selectlog=IML --filter Severity=Critical
ilorest logout
```

### IML Entry Fields

| Field | Path | Description |
|-------|------|-------------|
| Standard Severity | `.Severity` | `"OK"`, `"Warning"`, `"Critical"` |
| Created timestamp | `.Created` | ISO 8601 UTC timestamp |
| Message text | `.Message` | Human-readable event description |
| Entry type | `.EntryType` | Always `"Oem"` for IML entries |
| OEM format | `.OemRecordFormat` | `"Hpe-IML"` |
| HPE Severity | `.Oem.Hpe.Severity` | Finer-grained: `"Informational"`, `"Caution"`, `"Critical"`, `"Repaired"` |
| Event number | `.Oem.Hpe.EventNumber` | Unique sequence number |
| Class description | `.Oem.Hpe.ClassDescription` | Category label (e.g., `"Memory"`, `"Power"`, `"Cooling"`) |
| Class code | `.Oem.Hpe.Class` | Numeric class identifier |
| Event code | `.Oem.Hpe.Code` | Specific event code within class |
| Occurrence count | `.Oem.Hpe.Count` | How many times this event fired |
| Repaired flag | `.Oem.Hpe.Repaired` | `true` if condition has self-cleared |
| Recommended action | `.Oem.Hpe.RecommendedAction` | HPE-suggested remediation text |
| Learn more link | `.Oem.Hpe.LearnMoreLink` | URL to HPE support article |
| Last updated | `.Oem.Hpe.Updated` | Timestamp of most recent occurrence |

### Severity Levels Explained

| Standard `.Severity` | HPE `.Oem.Hpe.Severity` | Meaning |
|---|---|---|
| `"Critical"` | `"Critical"` | Active hardware failure — requires immediate action |
| `"Warning"` | `"Caution"` | Degraded condition — monitor or schedule repair |
| `"OK"` | `"Repaired"` | Event that previously fired but condition has cleared |
| `"OK"` | `"Informational"` | Normal operational event (config change, server boot, etc.) |

> **Note:** The `.Oem.Hpe.Repaired` boolean field is distinct from severity. An event with `Repaired: true` means the hardware condition self-corrected (e.g., memory error that stopped occurring). Repaired events still warrant review — they indicate a condition that existed.

---

## Common IML Patterns and What They Mean

| IML Message Pattern | Classification | Meaning | Action |
|---------------------|---------------|---------|--------|
| `"Correctable Memory Error"` — Count increasing over days | Memory / Warning | DIMM is developing errors but ECC is correcting them — early failure sign | Monitor count rate. If count doubles week-over-week, schedule DIMM replacement. |
| `"Correctable Memory Error"` — Sudden spike (hundreds in minutes) | Memory / Critical | DIMM failing rapidly, ECC correction rate approaching limit | Replace DIMM in next maintenance window. Do not wait. |
| `"Uncorrectable Memory Error"` | Memory / Critical | ECC cannot correct — actual data corruption occurring or DIMM completely failed | Replace DIMM immediately. System may be unstable. |
| `"Memory Training Failure"` | Memory / Critical | DIMM did not initialise at POST — slot may be empty, DIMM unseated, or failed | Check seating. If properly seated, replace DIMM. |
| `"Drive Failure"` / `"Physical Drive Status Change to Failed"` | Storage / Critical | Hard drive or SSD has failed | Identify slot from message. Replace drive. Verify RAID rebuild if in array. |
| `"Predictive Spare Drive Enabled"` | Storage / Warning | Controller has flagged a drive as failing and activated a spare | Review drive health. Confirm rebuild is progressing. Replace flagged drive. |
| `"Redundancy Lost"` (Power) | Power / Critical | One PSU in a redundant pair has failed | Replace failed PSU. Check `.Chassis/1/Power.PowerSupplies` to identify which one. |
| `"Power Supply Failed"` | Power / Critical | PSU hardware failure | Replace immediately. Single-PSU servers are now unprotected from power events. |
| `"Caution: Ambient temperature"` / `"Inlet Temperature exceeded"` | Thermal / Warning | Air temperature at server inlet is above safe operating threshold | Check room cooling, rack airflow, blanking panels. Reduce ambient temperature. |
| `"Fan Degraded"` / `"Fan at reduced speed"` | Thermal / Warning | Fan spinning below expected speed (possible bearing wear or obstruction) | Inspect fan for dust. If clean and degraded, schedule fan replacement. |
| `"Fan Failed"` | Thermal / Critical | Fan completely stopped | Replace fan immediately. Server may begin throttling or shutting down. |
| `"POST Error"` — Memory | Boot / Critical | BIOS POST could not initialise memory | Check DIMM seating and compatibility. See DIMM event entries for detail. |
| `"POST Error"` — Storage controller | Boot / Critical | RAID controller did not initialise | Check controller seating, battery, and firmware. |
| `"Firmware Flash Started"` / `"Firmware Flash Completed"` | Administration / Informational | Firmware update event | Normal — correlate with IEL to confirm who initiated update. |
| `"IML Cleared"` | Administration / Informational | Someone cleared the IML | Correlate with IEL to identify who did it and when. |
| `"Automatic Server Recovery"` (ASR) | Reliability / Warning | iLO watchdog triggered server reset because OS stopped responding | Investigate OS hang cause. Increasing frequency indicates OS or hardware instability. |
| `"Server power restored"` | Power / Informational | Server recovered from unexpected power loss | Check PDU and facility power. Review AHS for voltage dips before event. |
| `"Network adapter link down"` | Network / Warning | NIC link lost | Check cable and switch port. May be intentional (maintenance) or hardware failure. |
| `"Processor error"` / `"CPU fault"` | Processor / Critical | CPU hardware failure or thermal event | Check CPU seating and thermal compound. If confirmed CPU fault, escalate to HPE support. |

### Reading the `Count` Field

The `Oem.Hpe.Count` field is critical for memory errors. A single correctable memory error that fired once and has `Repaired: true` is far less concerning than an error with `Count: 847` that keeps incrementing. Always check the count trend:

```bash
# Watch count of memory-class IML entries
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Oem.Hpe.ClassDescription eq 'Memory'" \
  | jq '.Members[] | {
      Id,
      Created: .Created,
      Updated: .Oem.Hpe.Updated,
      Count: .Oem.Hpe.Count,
      Repaired: .Oem.Hpe.Repaired,
      Message: .Message
    } | select(.Count > 0)'
```

---

## Reading the iLO Event Log (IEL)

The IEL records management-plane activity — what iLO itself did or what was done to iLO. It does not record hardware component events (those go to IML). Use it to establish who was managing the server and when.

### Fetch All IEL Entries

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/LogServices/IEL/Entries \
  | jq '.Members[] | {
      Id,
      Severity: .Severity,
      Created: .Created,
      Message: .Message
    }'
```

### Fetch Recent Critical IEL Entries

```bash
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Managers/1/LogServices/IEL/Entries?\$filter=Severity eq 'Critical'&\$top=20" \
  | jq '.Members[] | {Id, Severity, Created, Message}'
```

### Via iLOrest

```bash
ilorest login 192.168.1.101 -u admin -p password
ilorest serverlogs --selectlog=IEL
ilorest logout
```

### What to Look For in the IEL

| IEL Event Pattern | What It Means | Action |
|---|---|---|
| `"Login failed"` — multiple in short succession | Failed authentication attempts (brute force probe) | Review IP source. Consider restricting iLO network access or enabling login security lockout. |
| `"Login failed"` from unexpected IP | Unauthorized access attempt | Audit who has iLO credentials. Rotate passwords if suspicious. |
| `"User account created/deleted/modified"` | Account management change | Verify this was authorised. |
| `"Configuration change"` | iLO or BIOS setting modified | Correlate with change management record. |
| `"Firmware update"` | Firmware was flashed | Confirm version and that update was planned. Cross-reference IML for any POST errors post-update. |
| `"Remote console session started"` | Someone opened KVM console | Verify this was authorised. |
| `"Virtual Media activity"` | ISO mounted/unmounted | Should correlate with OS install or maintenance activity. |
| `"IML cleared"` | Someone cleared the hardware event log | Note who and when. Clearing IML loses hardware history. |
| `"Power action"` — unexpected time | Server was powered on/off/reset | Verify this was intentional. Unplanned resets during business hours warrant investigation. |
| `"iLO reset"` | iLO processor was rebooted | Usually firmware update. If unexpected, investigate. |
| `"SSL certificate"` event | Certificate installed or expired | Check certificate validity. |
| `"License activated"` | iLO license key applied | Confirm matches procurement records. |

### IEL vs IML: The Key Distinction

- **IEL** events are *management plane* — they answer "who did what to the server management interface"
- **IML** events are *hardware plane* — they answer "what did the hardware do"
- If IML shows an unexpected server reset and IEL shows a `"Power action"` at the same time from a known admin account, it was likely deliberate maintenance
- If IML shows a server reset and IEL shows nothing at that time, the reset was hardware-triggered (OS watchdog, PSU failure, etc.)

---

## Active Health System (AHS)

### What AHS Captures

AHS runs continuously inside iLO, recording a dense stream of telemetry samples regardless of whether any alerts are firing:

- **Thermal:** All temperature sensor readings (inlet, CPU, DIMM slots, PCIe slots, system board zones) sampled every few minutes
- **Power:** Input power, CPU power, DIMM power, efficiency readings
- **Fans:** Speed readings (RPM and percentage) for all fans
- **Error counters:** Cumulative correctable memory error counts, PCIe error counts
- **Hardware config snapshots:** DIMM population, CPU config, storage topology — recorded at boot and when changes are detected
- **Firmware inventory:** All component firmware versions, recorded at each boot
- **System events:** Cross-referenced against IML entries

AHS data is binary — it is not human-readable. It is designed for analysis by HPE's diagnostic tools and support engineers. Do not attempt to parse it manually.

### When to Collect AHS

- **Before calling HPE support** — HPE will ask for it. Have it ready.
- **After any hardware event** (IML Critical entry) — AHS shows what the environment looked like before, during, and after the event
- **When you suspect intermittent thermal issues** — AHS sensor history will reveal temperature spikes that the current reading does not show
- **At regular intervals** for critical servers (weekly or monthly) as a diagnostic baseline

### Download AHS Data via Redfish

```bash
# Download AHS data — output is a binary .ahs file
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/Oem/Hpe/AHSData" \
  -o "logs/ahs/MXQ12345AB-$(date +%Y-%m-%d).ahs"
```

> **Note on the AHS endpoint:** The OEM AHS download path is `/redfish/v1/Systems/1/Oem/Hpe/AHSData`. On iLO 5, this is a direct GET that streams the binary. Verify the exact endpoint is present by first querying `/redfish/v1/Systems/1/Oem/Hpe` and checking for the `AHSData` link in the response. If the link is absent, query `/redfish/v1/Managers/1/ActiveHealthSystem` as an alternate path.

### Download with Date Range (if supported)

Some iLO versions accept query parameters to limit the AHS window:

```bash
# Request only last 3 days of AHS data
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/Oem/Hpe/AHSData?from=$(date -v-3d +%Y-%m-%d)&to=$(date +%Y-%m-%d)" \
  -o "logs/ahs/MXQ12345AB-$(date +%Y-%m-%d)-3day.ahs"
```

### Capture AHS Snapshot (iLO 5 OEM action)

On iLO 5, you can trigger an explicit AHS capture:

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{}' \
  "https://192.168.1.101/redfish/v1/Managers/1/ActiveHealthSystem/Actions/HpeiLOActiveHealthSystem.CaptureSystemLog"
```

### Via iLOrest

```bash
ilorest login 192.168.1.101 -u admin -p password
ilorest serverlogs --selectlog=AHS --directorypath ./logs/ahs/
ilorest logout
```

### AHS File Naming Convention

Use a consistent naming scheme so files are easy to match to a specific server and event:

```
logs/ahs/<serial>-<date>.ahs
logs/ahs/<serial>-<date>-<reason>.ahs

# Examples:
logs/ahs/MXQ12345AB-2026-01-15.ahs
logs/ahs/MXQ12345AB-2026-01-15-memory-errors.ahs
logs/ahs/MXQ12345AB-2026-01-20-before-hpe-support-call.ahs
```

Store AHS files with the server's maintenance records. HPE support engineers will typically ask you to upload the file to HPE's case portal.

---

## Correlating Events Across All Three Logs

The most powerful log analysis combines all three sources. Here is the workflow:

### Step 1 — Establish the timeline from IML

```bash
# Get all non-informational IML entries, sorted by time
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Severity ne 'OK'" \
  | jq '[.Members[] | {
      Time: .Created,
      Severity: .Severity,
      Source: "IML",
      Message: .Message,
      Count: .Oem.Hpe.Count,
      Repaired: .Oem.Hpe.Repaired
    }] | sort_by(.Time)'
```

Note the timestamps of Critical and Warning events.

### Step 2 — Check IEL for management activity around those times

```bash
# All IEL entries — look for activity within +/- 30 minutes of IML events
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Managers/1/LogServices/IEL/Entries?\$filter=Created gt '2026-01-15T02:00:00Z'" \
  | jq '.Members[] | {
      Time: .Created,
      Source: "IEL",
      Severity: .Severity,
      Message: .Message
    }'
```

### Step 3 — Download AHS for the time window

Collect AHS covering the period of the IML events. Upload to HPE support case or analyse in HPE's iLO Amplifier Pack / OneView.

### Correlation Patterns

| IML Shows | IEL Shows | AHS Context | Root Cause |
|---|---|---|---|
| Memory correctable errors increasing | Nothing notable | Temperature spike at DIMM location | Thermal stress causing memory errors — check cooling and DIMM slot airflow |
| Memory correctable errors increasing | Nothing notable | Normal temperatures, errors at specific slot only | DIMM hardware failure — replace |
| Server reset | Nothing at reset time | Voltage drop on input power | Power event (PDU or facility issue) caused reset |
| Server reset | `"Power action"` at same time | Normal | Deliberate admin action — planned maintenance or unplanned admin reset |
| `"Redundancy Lost"` for PSU | Nothing | Power draw near PSU limit | PSU failed — check if running near rated capacity; check PSU LED |
| IML cleared | `"IML Cleared"` by user X | N/A | Admin deliberately cleared logs — verify it was planned |
| No IML events | Failed login attempts in IEL | N/A | Security probe — no hardware impact but access control review needed |

---

## Clearing Logs

### When Clearing Is Appropriate

- After all logged issues have been fully resolved and documented
- During server decommissioning (clearing before repurposing)
- When log is full and new events are not being recorded (check `.LogService.OverWritePolicy`)

### When NOT to Clear

- Never clear IML before fully understanding all logged events
- Never clear logs before an HPE support call — support needs the history
- Never clear logs as a first response to problems — that destroys the diagnostic record

### Clear IML

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{}' \
  https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Actions/LogService.ClearLog
```

### Clear IEL

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{}' \
  https://192.168.1.101/redfish/v1/Managers/1/LogServices/IEL/Actions/LogService.ClearLog
```

### Via iLOrest

```bash
ilorest login 192.168.1.101 -u admin -p password
ilorest serverlogs --selectlog=IML --clearlog
ilorest serverlogs --selectlog=IEL --clearlog
ilorest logout
```

> Clearing either log creates an informational entry in the IEL recording who cleared it and when. This entry itself cannot be cleared by the same action that clears the IML — it is written to the IEL automatically.

---

## Proactive Log Monitoring

### Regular IML Review

Check IML at minimum weekly for production servers. Look for:

1. Any new Critical entries since last review
2. Warning entries with increasing `Count` values (especially memory)
3. Repaired entries — even if self-corrected, track whether they recur

```bash
# IML entries from last 7 days, non-informational only
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Created gt '$(date -v-7d -u +%Y-%m-%dT%H:%M:%SZ)' and Severity ne 'OK'" \
  | jq '[.Members[] | {Severity, Created, Message, Count: .Oem.Hpe.Count}]'
```

### Tracking Correctable Memory Error Trends

Correctable memory errors are the most important count to track over time. A single isolated event may be a transient. A growing count is a DIMM failing:

```bash
# Extract memory error entries with counts for trend analysis
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Oem.Hpe.ClassDescription eq 'Memory'" \
  | jq '[.Members[] | {
      Created,
      Updated: .Oem.Hpe.Updated,
      Count: .Oem.Hpe.Count,
      Repaired: .Oem.Hpe.Repaired,
      Message,
      Action: .Oem.Hpe.RecommendedAction
    }]'
```

Thresholds to act on (conservative guidance — adjust to your risk tolerance):

| Error Rate | Recommendation |
|---|---|
| Isolated event, `Count` stable, `Repaired: true` | Monitor — check again in 7 days |
| `Count` increasing slowly (< 50/week) | Schedule DIMM replacement at next maintenance window |
| `Count` > 500 or doubling weekly | Expedite replacement — do not wait for maintenance window |
| Any Uncorrectable Memory Error | Replace immediately |

### Alerting Without Advanced License

HPE iLO Advanced license provides SNMP traps and configurable alertmail. Without it, proactive monitoring requires polling. A simple approach:

```bash
#!/bin/bash
# check-iml-critical.sh — run via cron, alerts if new Critical IML entries found
ILO="192.168.1.101"
USER="admin"
PASS="password"
THRESHOLD_DATE=$(date -v-1d -u +%Y-%m-%dT%H:%M:%SZ)  # macOS date syntax

CRITICAL=$(curl -sk -u "$USER:$PASS" \
  "https://$ILO/redfish/v1/Systems/1/LogServices/IML/Entries?\$filter=Severity eq 'Critical' and Created gt '$THRESHOLD_DATE'" \
  | jq '.Members | length')

if [ "$CRITICAL" -gt 0 ]; then
  echo "ALERT: $CRITICAL new Critical IML entries on $ILO in last 24 hours" >&2
  exit 1
fi
```

---

## Integration With Other Skills

- **Health Check** (`skills/operations/health-check/`) — performs a quick IML scan as part of the standard health check. Routes to this skill when Critical entries are found.
- **Troubleshooting** (`skills/diagnose/troubleshooting/`) — symptom-based decision trees. Each tree that involves log analysis calls out specific IML patterns and routes here for deeper interpretation.
- **Power Management** (`skills/operations/power-management/`) — unexpected power events (server resets, PSU failures) generate both IML and IEL entries. Use this skill to correlate.
- **HPE Support escalation** — always collect AHS before opening a support case. The support engineer will ask for it, and having it ready avoids a round-trip delay.
