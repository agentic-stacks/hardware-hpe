# HPE ProLiant architecture, iLO management model, Redfish API structure, and core terminology — the mental model for all other skills in this stack.

---

## What Is iLO (Integrated Lights-Out)

iLO is HPE's out-of-band management engine embedded in every ProLiant server. It is not software installed on the server — it is a dedicated ARM-based management processor soldered to the system board with its own firmware, memory, and network interface.

**Key architectural properties:**

- Runs independently of the server's CPU, OS, and power state — it operates as long as the server has power (even when off)
- Has its own dedicated management NIC (or can share the server's NIC via NCSI — Network Controller Sideband Interface)
- Provides a full REST API (Redfish-compliant) accessible over HTTPS on port 443
- Hosts the iLO web UI, virtual serial port, virtual media, remote console (iLO HTML5 KVM), and event logs
- Persists hardware health data (IML, AHS) independently of the OS

**What iLO lets you do without touching the OS:**

- Power on, off, or reset the server
- View real-time hardware health (temperatures, fan speeds, power draw, component status)
- Mount virtual media (ISO) for OS installation
- Open a remote console (keyboard/video/mouse over the network)
- Configure BIOS/UEFI settings
- Update firmware (iLO, System ROM, NIC, RAID controller, drives)
- Review hardware event logs (IML) and continuous telemetry (AHS)
- Configure server RAID arrays (via SmartArray/MR adapter integration)

**Network access:** iLO listens on a dedicated management network interface. Best practice is to connect it to a separate out-of-band management VLAN so it remains accessible when the OS network is down.

---

## iLO 5 (Gen10) vs iLO 6 (Gen11)

| Attribute | iLO 5 | iLO 6 |
|---|---|---|
| Server Generation | ProLiant Gen10, Gen10 Plus | ProLiant Gen11 |
| Management Processor | ARM Cortex-A9 based | ARM Cortex-A72 based (faster, more RAM) |
| Redfish API Version | Redfish 1.6.0 (schema bundle 8010_2021.4) | Redfish 1.6.0, full DMTF conformance |
| iLO REST API | iLO RESTful API v2.x (Redfish + HPE OEM) | iLO RESTful API v1.x (fully Redfish-conformant, OEM reduced) |
| Legacy REST support | Yes (pre-Redfish iLO REST API coexists) | No — all pre-Redfish support removed |
| RAID model | HPE SmartStorage OEM (`HpeSmartStorage*`) | DMTF Redfish Storage Model only (`/Systems/1/Storage/`) |
| RAID controllers | Smart Array (HPE proprietary) | MegaRAID / SmartRAID (Broadcom-based, MR/SR) |
| Network adapter API | `BaseNetworkAdapters` (HPE OEM) + standard | Standard `NetworkAdapters` + `Port` only (OEM deprecated) |
| Security | TLS, iLO Standard/Advanced license tiers | TLS + SPDM + TPM ComponentIntegrity, 2FA (v1.50+), Trusted OS Security scanning |
| Secure Boot DB mgmt | Basic | Full UEFI DB management (KEK, PK, db, dbx, dbr, dbt) |
| OData query support | Limited | `$expand`, `$filter`, `$top`/`$skip`, `$count`, `only` |
| DPU / SmartNIC support | Partial (v2.72+) | Full DPU system type, separate DPU log service |
| GPU firmware tracking | No | Yes (via OEM extensions) |
| Key new security | — | ComponentIntegrity (SPDM/TPM), HpeSecurityService integrity policies |

**When you see "Gen10" assume iLO 5. When you see "Gen11" assume iLO 6.** The API structure is largely the same — the core Redfish URIs are identical — but the storage and network adapter APIs differ significantly between generations. Always check which generation you are targeting before writing automation.

---

## Redfish API Architecture

### Overview

Redfish is a DMTF standard REST API for hardware management. HPE's iLO implements Redfish as its primary API. All resources are JSON documents accessed over HTTPS. The API is self-describing: start at `/redfish/v1/` and traverse links — do not hardcode URIs beyond the service root.

### Resource Tree

```
/redfish/v1/                              ← Service root (no auth required)
├── Systems/
│   └── 1/                               ← The server (ComputerSystem)
│       ├── Processors/
│       ├── Memory/
│       ├── Storage/
│       │   └── {id}/
│       │       ├── Controllers/
│       │       ├── Volumes/
│       │       └── Drives/{driveId}/
│       ├── EthernetInterfaces/
│       ├── NetworkInterfaces/
│       ├── Bios/
│       │   └── Oem/Hpe/                 ← HPE BIOS extensions
│       ├── LogServices/
│       │   ├── Event/Entries/           ← IML (Integrated Management Log)
│       │   └── DPU/                     ← DPU logs (Gen11 only)
│       ├── SecureEraseReportService/
│       └── Actions/
│           └── ComputerSystem.Reset
├── Chassis/
│   └── 1/                               ← Physical enclosure
│       ├── Thermal/                     ← Temperatures + fan speeds
│       ├── Power/                       ← Power supplies + consumption
│       ├── NetworkAdapters/             ← NIC inventory
│       │   └── {id}/Ports/
│       └── PCIeDevices/
├── Managers/
│   └── 1/                               ← The iLO processor itself
│       ├── NetworkProtocol/             ← iLO network settings (IP, DNS, ports)
│       ├── EthernetInterfaces/          ← iLO NIC config
│       ├── LogServices/IEL/Entries/     ← iLO Event Log
│       ├── SecurityService/             ← TLS, certs, security policy
│       │   ├── AutomaticCertificateEnrollment/
│       │   └── PlatformCert/Certificates/
│       ├── RemoteSupportService/        ← HPE Insight Remote Support
│       ├── LicenseService/              ← iLO license management
│       ├── SnmpService/                 ← SNMP config
│       ├── BackupRestoreService/        ← iLO config backup/restore (iLO5 v2.72+)
│       └── ActiveHealthSystem/          ← AHS log download
├── AccountService/                      ← iLO user accounts and roles
├── SessionService/
│   └── Sessions/                        ← Session management (login/logout)
├── EventService/                        ← Event subscriptions
├── UpdateService/
│   ├── FirmwareInventory/               ← All component firmware versions
│   └── SoftwareInventory/
├── CertificateService/                  ← Certificate management
├── ComponentIntegrity/                  ← SPDM/TPM integrity (Gen11 only)
└── Registries/                          ← Message registries for event codes
```

### Standard Base URIs Reference

| Resource | URI | What It Exposes |
|---|---|---|
| Service root | `/redfish/v1/` | API version, links to all top-level collections |
| Systems collection | `/redfish/v1/Systems/` | List of compute nodes (always `1` for standalone iLO) |
| System | `/redfish/v1/Systems/1/` | CPU, memory, BIOS version, boot order, power state, health |
| Chassis | `/redfish/v1/Chassis/1/` | Physical enclosure: thermal, power, LEDs, PCIe |
| Thermal | `/redfish/v1/Chassis/1/Thermal/` | All temperatures and fan speeds with status |
| Power | `/redfish/v1/Chassis/1/Power/` | Power supplies, power consumption, power capping |
| Manager (iLO) | `/redfish/v1/Managers/1/` | iLO firmware version, network config, services |
| iLO network | `/redfish/v1/Managers/1/NetworkProtocol/` | Hostname, IP, DNS, SNMP, SSH, IPMI settings |
| iLO NIC | `/redfish/v1/Managers/1/EthernetInterfaces/1/` | iLO IP address, MAC, VLAN configuration |
| Accounts | `/redfish/v1/AccountService/` | User accounts, LDAP, LDAPS, directory settings |
| Sessions | `/redfish/v1/SessionService/Sessions/` | Active sessions; POST here to authenticate |
| Firmware inventory | `/redfish/v1/UpdateService/FirmwareInventory/` | All component firmware versions |
| Update service | `/redfish/v1/UpdateService/` | Firmware upload and staging |
| IML log | `/redfish/v1/Systems/1/LogServices/IML/Entries/` | Hardware event log entries |
| iLO event log | `/redfish/v1/Managers/1/LogServices/IEL/Entries/` | iLO management event log |
| BIOS settings | `/redfish/v1/Systems/1/Bios/` | Current BIOS settings |
| BIOS pending | `/redfish/v1/Systems/1/Bios/Settings/` | Pending BIOS changes (apply on next reboot) |
| Storage | `/redfish/v1/Systems/1/Storage/` | Storage controllers, logical volumes, physical drives |
| Network adapters | `/redfish/v1/Chassis/1/NetworkAdapters/` | NIC cards, ports, firmware |
| AHS log | `/redfish/v1/Managers/1/ActiveHealthSystem/` | Active Health System download link |
| License | `/redfish/v1/Managers/1/LicenseService/` | iLO license info and activation |
| ComponentIntegrity | `/redfish/v1/ComponentIntegrity/` | SPDM/TPM measurements (Gen11 only) |

### OData Conventions

Every Redfish resource uses OData annotations:

| Property | Meaning |
|---|---|
| `@odata.id` | The canonical URI of this resource (use this for navigation) |
| `@odata.type` | Schema type, e.g., `#ComputerSystem.v1_13_0.ComputerSystem` |
| `@odata.context` | Metadata context URL |
| `Members` | Array of `@odata.id` links in a collection |
| `Members@odata.count` | Total count of members in a collection |
| `Actions` | Available actions with their target URIs |
| `Links` | Related resources (not inline, requires separate GET) |

**iLO 6 OData query operators** (append to collection URIs):

| Operator | Example | Effect |
|---|---|---|
| `$expand=.` | `/redfish/v1/Systems/?$expand=.` | Inline all linked resources (1 level deep) |
| `$filter` | `?$filter=Status/Health eq 'Critical'` | Filter collection members |
| `$top` / `$skip` | `?$top=10&$skip=20` | Paginate large collections |
| `$count` | `?$count=true` | Include total count in response |
| `only` | `?only` | Return single-member collection directly |

### Authentication

**Method 1 — Session token (preferred for scripts):**

```bash
# Login: get X-Auth-Token
curl -sk -X POST https://<ilo>/redfish/v1/SessionService/Sessions/ \
  -H "Content-Type: application/json" \
  -d '{"UserName": "admin", "Password": "password"}' \
  -D - | grep -i x-auth-token

# Use the token in subsequent requests
curl -sk https://<ilo>/redfish/v1/Systems/1/ \
  -H "X-Auth-Token: <token>" | jq .

# Logout (DELETE the session URI from Location header)
curl -sk -X DELETE https://<ilo>/redfish/v1/SessionService/Sessions/<session-id> \
  -H "X-Auth-Token: <token>"
```

**Method 2 — Basic auth (simple queries):**

```bash
curl -sk -u admin:password https://<ilo>/redfish/v1/Systems/1/ | jq .
```

**iLO 6 additional methods:** Certificate-based login, Kerberos (with licensing), LDAP (with licensing), Two-Factor Auth (v1.50+).

Unauthenticated requests return `HTTP 401` with `NoValidSession`. The service root `/redfish/v1/` is the only resource accessible without authentication.

### HPE OEM Extensions

HPE extends standard Redfish via the `Oem.Hpe` namespace. When a resource has HPE-specific properties, they appear under:

```json
{
  "Oem": {
    "Hpe": {
      "@odata.type": "#HpeComputerSystemExt.v2_11_0.HpeComputerSystemExt",
      "PostState": "InPostDiscoveryComplete",
      "AggregateHealthStatus": { ... },
      ...
    }
  }
}
```

Key HPE OEM resources:

| OEM Resource | Path | Purpose |
|---|---|---|
| `HpeComputerSystemExt` | `Systems/1/Oem/Hpe` | PostState, AggregateHealthStatus, PowerAllocationLimit |
| `HpeThermalExt` | `Chassis/1/Thermal/Oem/Hpe` | Thermal config: OptimalCooling, EnhancedCooling, MaximumCooling |
| `HpeiLOActiveHealthSystem` | `Managers/1/ActiveHealthSystem/` | AHS log download and status |
| `HpeSecurityService` | `Managers/1/SecurityService/` | TLS config, cert enrollment, integrity policies |
| `HpeRemoteSupportService` | `Managers/1/RemoteSupportService/` | HPE Insight Remote Support registration |
| `HpeiLOSnmpService` | `Managers/1/SnmpService/` | SNMPv1/v3 config and traps |
| `HpeiLOBackupRestoreService` | `Managers/1/BackupRestoreService/` | iLO config export/import |
| `HpeiLOUpdateServiceExt` | `UpdateService/Oem/Hpe` | FWPKG 2.0, PLDM firmware staging |
| `HpeWorkloadPerformanceAdvisor` | `Systems/1/Oem/Hpe/WorkloadPerformanceAdvisor/` | CPU/memory tuning recommendations |
| `HpeSecureEraseReportService` | `Systems/1/SecureEraseReportService/` | NIST SP800-88 secure erase audit trail |

**Gen10 only (iLO 5):** `HpeSmartStorage*` resources for RAID management.
**Gen11 only (iLO 6):** `ComponentIntegrity` for SPDM/TPM, `HpeSecurityService` integrity scanning.

---

## iLOrest CLI

iLOrest is HPE's official command-line tool that wraps the Redfish API. It supports both remote connections (over network) and local in-band connections (from within the OS via the iLO channel interface driver).

```bash
# Remote connection
ilorest --url https://<ilo> -u admin -p password <command>

# Local in-band (from within OS on the managed server — no credentials needed)
ilorest <command>

# Common patterns
ilorest login https://<ilo> -u admin -p password
ilorest get --select ComputerSystem.
ilorest get PowerState --select ComputerSystem.
ilorest select Bios.
ilorest set QuietBoot=Disabled
ilorest commit
ilorest logout
```

iLOrest is preferred over raw curl for multi-step operations (BIOS changes, firmware updates) because it handles session management, schema validation, and pending settings automatically.

---

## OneView Concepts

### What OneView Is

HPE OneView is a fleet management platform that manages multiple servers through a single pane of glass. Where iLO manages a single server, OneView manages hundreds or thousands.

| Capability | iLO (single server) | OneView (fleet) |
|---|---|---|
| Scope | One server | Hundreds of servers |
| Interface | HTTPS direct to iLO IP | HTTPS to OneView appliance |
| Server profiles | No | Yes — consistent config templates |
| Firmware baseline | Manual per-server | Baseline compliance across fleet |
| Discovery | Manual | Auto-discovers servers via iLO |
| Alerts | Per-server | Aggregated, correlated alerts |
| API | Redfish | HPE OneView REST API (separate from Redfish) |

### Server Profiles and Templates

A **Server Profile Template** defines the desired state for a class of server: BIOS settings, network connections, storage configuration, boot order, and firmware baseline.

A **Server Profile** is an instance of a template applied to a physical server. OneView enforces compliance — if a server drifts from its profile, OneView alerts and can remediate.

```
Server Profile Template (e.g., "ESXi-Host-Template")
    └── Server Profile (applied to Physical Server 1)
    └── Server Profile (applied to Physical Server 2)
    └── Server Profile (applied to Physical Server 3)
```

### When to Use iLO vs OneView

Use **iLO** when:
- Managing a single server or a small number
- Doing one-off operations (firmware update, reboot, console access)
- The operator has direct access to iLO IPs
- No OneView appliance is deployed

Use **OneView** when:
- Managing a fleet of 10+ servers
- Enforcing consistent BIOS/network/storage configuration across servers
- Running firmware compliance campaigns
- Onboarding new servers at scale with server profile templates

---

## HPE Terminology Reference

| Term | Full Name | What It Is |
|---|---|---|
| iLO | Integrated Lights-Out | Out-of-band management processor embedded in every ProLiant |
| IML | Integrated Management Log | Hardware event log stored in iLO; records component failures, temperature events, POST errors |
| AHS | Active Health System | Continuous telemetry log capturing hardware state every few minutes; stored in iLO non-volatile memory; downloadable as `.ahs` file for HPE support analysis |
| SPP | Service Pack for ProLiant | Bundled firmware update ISO containing all component firmware (ROM, iLO, NIC, HBA, RAID) tested together by HPE for a given generation |
| SUT | Smart Update Tools | HPE agent that applies SPP updates from within the OS or via iLO |
| Smart Array | (none) | Gen10 HPE-proprietary RAID controller family (e.g., P408i-a, P816i-a); managed via `HpeSmartStorage` OEM API in iLO 5 |
| MR | MegaRAID | Gen11 RAID controller (Broadcom MegaRAID-based); managed via standard Redfish Storage model |
| SR | SmartRAID | Gen11 HPE-branded Broadcom RAID controller; same API as MR |
| NS | No RAID (JBOD) | Non-RAID storage configuration; drives exposed directly to OS |
| FlexibleLOM | Flexible LAN on Motherboard | Replaceable network adapter slot in the server's motherboard for plug-in NIC modules (avoids full PCIe slot use) |
| mezzanine | (none) | Blade server equivalent of FlexibleLOM |
| UID | Unit Identification | Blue LED on front/rear of server used to physically locate a server in a rack; can be triggered via iLO API |
| RBSU | ROM-Based Setup Utility | Legacy F9 BIOS setup screen; replaced by UEFI System Utilities in Gen10+ |
| UEFI SU | UEFI System Utilities | Modern replacement for RBSU; accessible at POST via F9 |
| POST | Power-On Self Test | Server firmware self-check on boot; POST codes visible in IML and via remote console |
| iLO Federation | (none) | iLO 4/5 feature allowing one iLO to manage a group of iLOs on the same network segment without OneView |
| Virtual Media | (none) | iLO feature to mount an ISO or USB image over the network as a virtual CD/DVD or USB drive |
| Remote Console | (none) | iLO HTML5 KVM (keyboard/video/mouse) session for full graphical access to the server console |
| IPMI | Intelligent Platform Mgmt Interface | Legacy out-of-band protocol supported by iLO (deprecated in favor of Redfish; disabled by default in iLO 6) |
| CHIF | Channel Interface | In-band driver allowing OS-side tools (iLOrest, SUT) to communicate with iLO without network |
| SID | Server ID | HPE internal identifier for a server hardware configuration |
| SEL | System Event Log | IPMI-compatible event log; iLO presents this alongside IML |
| iSUT | Integrated Smart Update Tools | Version of SUT integrated with iLO for agentless firmware staging |
| FWPKG | Firmware Package | HPE firmware bundle format (`.fwpkg`) consumed by iLO UpdateService |
| PLDM | Platform Level Data Model | DMTF standard used by iLO 5/6 for firmware transfer and update |
| Gen10 | Generation 10 | ProLiant server hardware generation (2017–2020); uses iLO 5 |
| Gen10 Plus | Generation 10 Plus | Refresh of Gen10 with AMD/Intel Xeon 3rd gen; still iLO 5 |
| Gen11 | Generation 11 | ProLiant server hardware generation (2023–present); uses iLO 6 |
| OCA | Online Certificate Authority | Used with iLO automatic certificate enrollment (SCEP) |
| SCEP | Simple Certificate Enroll Protocol | Protocol for automatic SSL cert provisioning to iLO |
| SPDM | Security Protocol and Data Model | DMTF standard for authenticating firmware/hardware component integrity (Gen11) |
| RBAC | Role-Based Access Control | iLO privilege model: Administrator, Operator, ReadOnly, plus custom roles |

---

## Quick Reference: Common Redfish Queries for Agents

### Read Operations (safe, no side effects)

| Task | curl Command |
|---|---|
| Check server power state | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/ \| jq '.PowerState'` |
| Check overall server health | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/ \| jq '.Status'` |
| Get BIOS version | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/ \| jq '.BiosVersion'` |
| Get iLO firmware version | `curl -sk -u user:pass https://<ilo>/redfish/v1/Managers/1/ \| jq '.FirmwareVersion'` |
| List all firmware versions | `curl -sk -u user:pass https://<ilo>/redfish/v1/UpdateService/FirmwareInventory/ \| jq '.Members[].@odata.id'` |
| Get CPU info | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/Processors/ \| jq '.Members'` |
| Get memory info | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/Memory/ \| jq '.Members[].@odata.id'` |
| Get temperatures | `curl -sk -u user:pass https://<ilo>/redfish/v1/Chassis/1/Thermal/ \| jq '.Temperatures[] \| {Name,ReadingCelsius,Status}'` |
| Get fan status | `curl -sk -u user:pass https://<ilo>/redfish/v1/Chassis/1/Thermal/ \| jq '.Fans[] \| {Name,Reading,Status}'` |
| Get power consumption | `curl -sk -u user:pass https://<ilo>/redfish/v1/Chassis/1/Power/ \| jq '.PowerControl[].PowerConsumedWatts'` |
| Get iLO IP/network config | `curl -sk -u user:pass https://<ilo>/redfish/v1/Managers/1/EthernetInterfaces/1/ \| jq '{IPv4Addresses,IPv6Addresses,MACAddress}'` |
| Get storage controllers | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/Storage/ \| jq '.Members[].@odata.id'` |
| Get logical volumes | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/Storage/1/Volumes/ \| jq '.Members[].@odata.id'` |
| Get physical drives | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/Storage/1/Drives/ \| jq '.'` |
| Get NIC inventory | `curl -sk -u user:pass https://<ilo>/redfish/v1/Chassis/1/NetworkAdapters/ \| jq '.Members[].@odata.id'` |
| Get IML events | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/LogServices/IML/Entries/ \| jq '.Members[] \| {Created,Message,Severity}'` |
| Get iLO event log | `curl -sk -u user:pass https://<ilo>/redfish/v1/Managers/1/LogServices/IEL/Entries/ \| jq '.Members[] \| {Created,Message,Severity}'` |
| Get current BIOS settings | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/Bios/ \| jq '.Attributes'` |
| Check boot order | `curl -sk -u user:pass https://<ilo>/redfish/v1/Systems/1/ \| jq '.Boot'` |
| Get iLO license info | `curl -sk -u user:pass https://<ilo>/redfish/v1/Managers/1/LicenseService/ \| jq '.'` |
| Get user accounts | `curl -sk -u user:pass https://<ilo>/redfish/v1/AccountService/Accounts/ \| jq '.Members[].@odata.id'` |

### Write Operations (change state — use with care)

| Task | curl Command |
|---|---|
| Power on server | `curl -sk -u user:pass -X POST https://<ilo>/redfish/v1/Systems/1/Actions/ComputerSystem.Reset -H "Content-Type: application/json" -d '{"ResetType":"On"}'` |
| Graceful shutdown | `curl -sk -u user:pass -X POST https://<ilo>/redfish/v1/Systems/1/Actions/ComputerSystem.Reset -H "Content-Type: application/json" -d '{"ResetType":"GracefulShutdown"}'` |
| Force power off | `curl -sk -u user:pass -X POST https://<ilo>/redfish/v1/Systems/1/Actions/ComputerSystem.Reset -H "Content-Type: application/json" -d '{"ResetType":"ForceOff"}'` |
| Force restart | `curl -sk -u user:pass -X POST https://<ilo>/redfish/v1/Systems/1/Actions/ComputerSystem.Reset -H "Content-Type: application/json" -d '{"ResetType":"ForceRestart"}'` |
| Enable UID LED | `curl -sk -u user:pass -X PATCH https://<ilo>/redfish/v1/Systems/1/ -H "Content-Type: application/json" -d '{"LocationIndicatorActive":true}'` |
| Set one-time boot to PXE | `curl -sk -u user:pass -X PATCH https://<ilo>/redfish/v1/Systems/1/ -H "Content-Type: application/json" -d '{"Boot":{"BootSourceOverrideTarget":"Pxe","BootSourceOverrideEnabled":"Once"}}'` |
| Set one-time boot to CD/ISO | `curl -sk -u user:pass -X PATCH https://<ilo>/redfish/v1/Systems/1/ -H "Content-Type: application/json" -d '{"Boot":{"BootSourceOverrideTarget":"Cd","BootSourceOverrideEnabled":"Once"}}'` |
| Reset iLO (not the server) | `curl -sk -u user:pass -X POST https://<ilo>/redfish/v1/Managers/1/Actions/Manager.Reset -H "Content-Type: application/json" -d '{"ResetType":"GracefulRestart"}'` |

### iLOrest Equivalents

| Task | iLOrest Command |
|---|---|
| Connect to iLO | `ilorest login https://<ilo> -u admin -p password` |
| Check server health | `ilorest get Status --select ComputerSystem.` |
| Check power state | `ilorest get PowerState --select ComputerSystem.` |
| Get firmware versions | `ilorest list --select SoftwareInventory.` |
| View BIOS settings | `ilorest select Bios. && ilorest get` |
| Change a BIOS setting | `ilorest select Bios. && ilorest set <Attribute>=<Value> && ilorest commit` |
| View IML log | `ilorest list --select LogEntry. --filter '@odata.type=/#LogEntry'` |
| Power off server | `ilorest reboot ForceOff` |
| Restart server | `ilorest reboot ForceRestart` |
| Upload firmware | `ilorest flashfwpkg <firmware.fwpkg>` |
| Disconnect | `ilorest logout` |

---

## How Redfish Relates to Other HPE Management Tools

```
                        ┌─────────────────────────────────┐
                        │         Operator / Agent        │
                        └────────────┬────────────────────┘
                                     │
              ┌──────────────────────┼─────────────────────┐
              │                      │                      │
    ┌─────────▼────────┐  ┌──────────▼──────────┐  ┌───────▼──────────┐
    │   curl / Python  │  │     iLOrest CLI      │  │   HPE OneView    │
    │  (direct Redfish)│  │  (Redfish wrapper)   │  │ (fleet REST API) │
    └─────────┬────────┘  └──────────┬──────────┘  └───────┬──────────┘
              │                      │                      │
              └──────────────────────┼─────────────────────┘
                                     │ HTTPS (port 443)
                              ┌──────▼──────┐
                              │   iLO 5/6   │
                              │ (Redfish API│
                              │ on server)  │
                              └──────┬──────┘
                                     │ internal bus
                              ┌──────▼──────┐
                              │  ProLiant   │
                              │   Server    │
                              │  Hardware   │
                              └─────────────┘
```

**OneView** communicates with each server's iLO under the hood — it uses the same Redfish API but adds fleet orchestration on top. When you use OneView, you are still ultimately talking to iLO.

---

## Key Concepts for Agents

1. **Always start at `/redfish/v1/`** — discover URIs dynamically rather than hardcoding paths beyond the service root. The tree above shows where things live, but follow `@odata.id` links.

2. **The server is always `Systems/1`** for standalone iLO. Chassis is always `Chassis/1`. iLO manager is always `Managers/1`. These IDs can differ in blade/synergy contexts.

3. **Gen10 vs Gen11 matters for storage and networking.** If you need to query drives on Gen10, use `HpeSmartStorage` OEM paths. On Gen11, use standard `Storage/` only.

4. **BIOS changes are pending until reboot.** PATCH to `/Bios/Settings/` stages the change. It applies only after the server reboots. Check `/Bios/` (current) vs `/Bios/Settings/` (pending) to see staged changes.

5. **IML is the first place to look when hardware acts up.** It is the definitive record of hardware events from the server's perspective. AHS provides deeper telemetry for HPE support cases.

6. **iLO license tiers matter.** Standard license is free; Advanced license unlocks features like LDAP, iLO Federation advanced features, power regulation, and some telemetry features. Check with `GET /redfish/v1/Managers/1/LicenseService/`.

7. **Never assume the OS is running** — iLO is the ground truth. `PowerState` from iLO is authoritative. An OS may have crashed while iLO still reports `On`.
