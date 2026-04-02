# Known Issues: iLO 5 Firmware

Applies to HPE ProLiant Gen10 and Gen10 Plus servers running iLO 5.
Latest iLO 5 release as of 2026-03: **v3.18** (released 2026-02-13).

---

## Version 3.x Issues

### iLO 5 3.06 Update Causes Management Plane Outage Under ESXi
**Symptom:** After updating iLO 5 from 3.05 to 3.06, iLO becomes unresponsive on servers running VMware ESXi. In severe cases, entire vSphere clusters lost management connectivity. iLO health LED may show Degraded. HPE Agentless Management Service (AMS) stops reporting.
**Cause:** A defect introduced in 3.06 caused a compatibility regression with the HPE Agentless Management Service driver stack on ESXi hosts. The iLO subsystem enters a partial-reset loop that does not self-recover.
**Workaround:**
1. Do not update iLO 5 to 3.06 on ESXi hosts without first testing on a non-production node.
2. If already updated, perform a full iLO reset via IPMI or physical server iLO reset button.
3. Downgrade to iLO 5 3.05 using offline SPP if AMS connectivity is required and 3.06 exhibits the issue.
4. Monitor HPE advisory channel before re-applying 3.06.
**Affected versions:** iLO 5 3.06
**Status:** Fixed in iLO 5 3.07

---

### Redfish Session Exhaustion — Management Lockout
**Symptom:** All Redfish and iLO web interface logins fail with "Maximum sessions reached" or HTTP 503. Existing automation scripts begin returning authentication errors. Physical access to iLO is the only remaining path to recovery.
**Cause:** iLO 5 enforces a hard limit on simultaneous sessions. Scripts using token-based authentication that do not call `DELETE /redfish/v1/SessionService/Sessions/{id}` on exit leave orphaned sessions. These persist until the `SessionTimeout` expires (default: 30 minutes). Under heavy polling, the limit is reached before any session expires.
**Workaround:**
1. Identify orphaned sessions: `GET /redfish/v1/SessionService/Sessions` — list all active sessions.
2. Delete stale sessions: `DELETE /redfish/v1/SessionService/Sessions/{id}` for each orphaned entry.
3. If fully locked out, perform an iLO reset (cold or warm) via the physical iLO reset button or IPMI.
4. In all scripts, add an explicit session DELETE in the finally/cleanup block.
5. For monitoring scripts, use HTTP Basic Authentication (self-cleaning, no persistent session) or set `SessionTimeout` to a lower value (e.g., 5 minutes).
6. Avoid using `ilorest --nocache` in loops — each invocation creates a new session.
**Affected versions:** All iLO 5 versions
**Status:** Open — architectural limit; mitigated by proper session hygiene

---

### Incorrect Memory Sensor Status Reported to VMware vSphere
**Symptom:** VMware vSphere Web Client displays false memory health warnings (yellow or red) for DIMMs that are fully functional. `esxcli hardware memory get` and POST memory tests show no errors. iLO 5 Event Log is clean.
**Cause:** A defect in iLO 5 firmware 2.30 caused the DIMM health status reported via CIM/WBEM to VMware to be misencoded. The sensor value was valid internally but translated incorrectly when passed to the ESXi health framework.
**Workaround:**
1. Verify actual DIMM health directly in iLO 5 web UI under System Information > Memory.
2. Check POST logs for any real DIMM errors.
3. Update iLO 5 to version 2.31 or later to resolve the CIM reporting bug.
**Affected versions:** iLO 5 2.30
**Status:** Fixed in iLO 5 2.31

---

### Virtual Media Disconnects During Large ISO Transfers
**Symptom:** Virtual CD/DVD mounted via iLO 5 HTML5 or scripted virtual media disconnects mid-transfer when streaming ISOs larger than approximately 4 GB. OS installation stalls or fails with "media not found" errors. The iLO event log records a virtual media disconnection event.
**Cause:** iLO 5 virtual media uses an internal HTTP/HTTPS session for the network-backed ISO stream. Network path instability, high-latency links (e.g., VPN), or TCP checksum offload mismatches on the client NIC can cause the stream to stall, triggering iLO's internal timeout and dropping the virtual media connection.
**Workaround:**
1. Host the ISO on a server within the same Layer-2 network segment as the iLO management port to minimize latency.
2. Disable TCP offload on the management workstation NIC if TCP checksum errors are suspected.
3. Use iLO scripted virtual media (`hpilo` or Redfish `VirtualMediaInsert`) with a stable HTTP server rather than direct browser mounting.
4. Set the iLO Idle Connection Timeout to a larger value (Security > Access Settings > Idle Connection Timeout) before starting a long transfer.
5. For ISOs > 4 GB, prefer SPP online update or iLO Repository firmware delivery over virtual media.
**Affected versions:** iLO 5 all versions prior to 2.78 (partial fix); fully mitigated by network path improvements
**Status:** Open — network-dependent; partial fix in 2.78 for browser-based virtual media stability

---

### Slow Redfish API Response Under Heavy Polling
**Symptom:** Redfish `GET` requests to `/redfish/v1/Systems/1/` or `/redfish/v1/Chassis/1/Thermal` take 5–30 seconds to respond when polled from multiple concurrent clients. iLO CPU utilization climbs to 90–100% (visible in iLO Diagnostics). Other iLO functions (web UI, IPMI) also slow.
**Cause:** iLO 5 runs on constrained embedded hardware. Its HTTP server is single-threaded for certain endpoints. Monitoring systems polling thermal, power, and system status at sub-60-second intervals across multiple sessions overwhelm the iLO CPU, starving all other management functions.
**Workaround:**
1. Increase polling interval to a minimum of 60 seconds per endpoint per iLO.
2. Use the iLO Telemetry service (`/redfish/v1/TelemetryService/`) with server-push subscriptions instead of polling, where supported (iLO 5 2.60+).
3. Limit concurrent Redfish connections to a single session per iLO where possible.
4. Use SNMP polling for basic health metrics — much lower CPU cost than Redfish for read-only status.
5. If response times remain unacceptable, perform an iLO warm reset to clear any runaway internal processes.
**Affected versions:** iLO 5 all versions (hardware constraint)
**Status:** Open — architectural; mitigated by reduced polling frequency and Telemetry service

---

### SNMP Trap Storm Under Fan/Thermal Fault Conditions
**Symptom:** iLO 5 floods the SNMP trap receiver with thousands of repeated trap messages per minute when a fan or temperature sensor crosses a threshold and oscillates around it. Trap receivers log storage fills rapidly; SNMP management systems become unresponsive.
**Cause:** iLO 5 generates an SNMP trap each time a sensor crosses a configured threshold. When a sensor value oscillates (e.g., a failing fan that spins up and down), it repeatedly crosses the threshold in both directions, generating a trap for every crossing with no dampening or de-duplication logic.
**Workaround:**
1. Identify the oscillating sensor: `GET /redfish/v1/Chassis/1/Thermal` — look for values near the warning threshold.
2. Address the root hardware issue (replace the failing fan or clean blocked airflow) to stop the oscillation.
3. As a short-term measure, increase the SNMP trap threshold hysteresis in iLO 5 if supported by your firmware version.
4. Configure your SNMP trap receiver to de-duplicate traps with a minimum repeat interval (receiver-side filtering).
5. Temporarily disable SNMP traps for the affected sensor via iLO Management > SNMP Settings if the root fix is underway and the storm is impacting operations.
**Affected versions:** iLO 5 all versions
**Status:** Open — no firmware dampening added; workaround is hardware remediation or receiver-side filtering

---

### AHS Log Corruption in Specific Firmware Versions
**Symptom:** Downloading the Active Health System (AHS) log fails with an HTTP 500 error, or the downloaded `.ahs` file cannot be opened by HPE's AHS Viewer. iLO reports "AHS Log Error" in the Diagnostics panel. The AHS log size shown in iLO is either 0 bytes or an implausibly large value.
**Cause:** A defect in iLO 5 firmware versions 2.55 through 2.60 caused AHS log write operations to produce corrupt index entries under high-frequency event conditions (e.g., rapid firmware updates or mass health alerts). The corruption is self-perpetuating — subsequent writes fail because the index cannot be parsed.
**Workaround:**
1. Attempt a partial AHS download using the date-range filter: `GET /redfish/v1/Managers/1/ActiveHealthSystem?downloadEntries=<days>`.
2. If the full log is corrupt, clear the AHS log: iLO Web UI > Information > Active Health System > Clear.
3. Note: clearing AHS destroys all historical diagnostic data for that server.
4. Update iLO 5 to version 2.63 or later before re-accumulating AHS data.
5. As a preventive measure, schedule regular AHS log downloads and offsite archival via the Redfish endpoint.
**Affected versions:** iLO 5 2.55 through 2.60
**Status:** Fixed in iLO 5 2.63
