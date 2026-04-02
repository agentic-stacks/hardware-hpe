# Operations: Server Lifecycle Management

Warranty, asset tracking, and decommissioning for HPE ProLiant servers. Covers collecting asset information, preparing HPE support packages, recovering resources before decommission, secure erase, iLO factory reset, and re-provisioning for repurposing.

> **Lifecycle context:** This skill bookends the stack — from first touch (see `skills/foundation/concepts/README.md`) to last touch. Every other skill represents a phase in the server's life. This skill covers what to do before a server arrives in inventory, and what to do when it leaves.

---

## Prerequisites

- iLO connectivity established — see `skills/foundation/connectivity/README.md`
- `curl` and `jq` available on the workstation
- Credentials with **Administrator** role for any write operations (asset tag updates, secure erase, factory reset)
- **ReadOnly** role is sufficient for asset information collection

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| System overview | `/redfish/v1/Systems/1` | GET, PATCH |
| iLO Manager overview | `/redfish/v1/Managers/1` | GET |
| iLO network interfaces | `/redfish/v1/Managers/1/EthernetInterfaces` | GET |
| Chassis | `/redfish/v1/Chassis/1` | GET, PATCH |
| Firmware inventory | `/redfish/v1/UpdateService/FirmwareInventory` | GET |
| System erase (OEM) | `/redfish/v1/Systems/1/Actions/Oem/Hpe/HpeComputerSystemExt.SystemReset` | POST |
| Storage secure erase | `/redfish/v1/Systems/1/Storage/{id}/Drives/{driveId}/Actions/Drive.SecureErase` | POST |
| iLO factory defaults (OEM) | `/redfish/v1/Managers/1/Actions/Oem/Hpe/HpeiLO.ResetToFactoryDefaults` | POST |

> iLO 4 (Gen8) uses slightly different OEM namespace paths — `Hp` instead of `Hpe`. Where paths diverge, this skill notes it explicitly.

---

## Asset Information Collection

### System Identity Fields

The `GET /redfish/v1/Systems/1` response contains the primary identity fields used for asset tracking, support cases, and inventory systems.

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{
      Model,
      SerialNumber,
      SKU,
      AssetTag,
      PartNumber: .PartNumber,
      UUID,
      BiosVersion,
      PowerState,
      PostState: .Oem.Hpe.PostState
    }'
```

Key fields:

| Field | Description | Example |
|---|---|---|
| `Model` | Product name string | `"ProLiant DL380 Gen10"` |
| `SerialNumber` | Chassis serial number — primary identifier for HPE support | `"CZ123456AB"` |
| `SKU` | HPE product number (P/N) | `"P02462-B21"` |
| `AssetTag` | User-configurable tag for inventory tracking | `"DC01-RACK3-U12"` |
| `PartNumber` | HPE part number (sometimes same as SKU) | `"P02462-B21"` |
| `UUID` | System UUID — unique, used by some CMDB tools | `"31393150-3135-5A43-4339-..."` |
| `BiosVersion` | Installed BIOS/UEFI version string | `"U30 v2.46 (11/15/2022)"` |

### iLO Manager Information

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1 \
  | jq '{
      FirmwareVersion,
      ManagerType,
      Model,
      UUID,
      iLOHostname: .Oem.Hpe.Hostname,
      License: .Oem.Hpe.License
    }'
```

| Field | Description |
|---|---|
| `FirmwareVersion` | iLO firmware version — required for support cases |
| `iLOHostname` | DNS name assigned to the iLO interface |
| `License.LicenseType` | Current iLO license tier |

### iLO Network Configuration

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/EthernetInterfaces \
  | jq '.Members[]["@odata.id"]'

# Then fetch each interface (typically there is one dedicated iLO NIC)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/EthernetInterfaces/1 \
  | jq '{
      FQDN,
      MACAddress,
      IPv4: [.IPv4Addresses[] | {Address, SubnetMask, Gateway, AddressOrigin}],
      DHCPEnabled: .DHCPv4.DHCPEnabled
    }'
```

### Chassis Information

The Chassis resource surfaces the physical enclosure serial (may differ from the system serial on blade servers) and the asset tag at the chassis level:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1 \
  | jq '{
      Model,
      SerialNumber,
      AssetTag,
      ChassisType,
      SKU
    }'
```

On rack servers the Chassis and System serial numbers are typically the same. On blade servers they will differ — the Chassis serial identifies the enclosure, the System serial identifies the blade.

---

## Asset Tag Management

The asset tag is a free-text field you control. Use it to record the server's role, location, rack position, owner, or any identifier your inventory system requires.

### Reading the Asset Tag

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq -r '.AssetTag'
```

An empty string means no tag has been set.

### Setting the Asset Tag

PATCH the `AssetTag` field. This change takes effect immediately and survives reboots.

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"AssetTag": "DC01-RACK3-U12-PROD-WEB"}' \
  https://192.168.1.101/redfish/v1/Systems/1
```

On success: `HTTP 200 OK`. Verify:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq -r '.AssetTag'
# Expected: "DC01-RACK3-U12-PROD-WEB"
```

### Asset Tag Conventions

Recommended tag format for rack servers: `<datacenter>-<rack>-<unit>-<environment>-<role>`

Examples:
- `DC01-RACK03-U12-PROD-WEB` — production web server, rack 3, unit 12
- `DC02-RACK07-U04-DEV-DB` — dev database server
- `COLO-A4-U22-STAGING` — staging server in colocation cage A4

The asset tag is also writable on the Chassis resource via the same PATCH method. On rack servers, keep both in sync:

```bash
# Set on both System and Chassis
for RESOURCE in Systems/1 Chassis/1; do
  curl -sk -u admin:password \
    -X PATCH \
    -H "Content-Type: application/json" \
    -d '{"AssetTag": "DC01-RACK3-U12-PROD-WEB"}' \
    https://192.168.1.101/redfish/v1/$RESOURCE
done
```

---

## Full Asset Report

Collect all identity and configuration information in a single pass. Run this at server intake (record baseline) and again before decommissioning (record final state).

```bash
#!/usr/bin/env bash
# collect-asset-report.sh
# Usage: ./collect-asset-report.sh <ilo-ip> <username> <password>

ILO="${1:?Usage: $0 <ilo-ip> <user> <pass>}"
USER="${2:?}"
PASS="${3:?}"
AUTH="-u $USER:$PASS"
BASE="https://$ILO"
TIMESTAMP=$(date -u +"%Y-%m-%dT%H:%M:%SZ")

echo "=== HPE Server Asset Report ==="
echo "Collected: $TIMESTAMP"
echo "iLO IP: $ILO"
echo ""

echo "--- System Identity ---"
curl -sk $AUTH "$BASE/redfish/v1/Systems/1" \
  | jq '{
      Model,
      SerialNumber,
      SKU,
      AssetTag,
      UUID,
      BiosVersion,
      PowerState,
      PostState: .Oem.Hpe.PostState
    }'

echo ""
echo "--- iLO Manager ---"
curl -sk $AUTH "$BASE/redfish/v1/Managers/1" \
  | jq '{
      FirmwareVersion,
      Model,
      UUID,
      Hostname: .Oem.Hpe.Hostname,
      LicenseType: .Oem.Hpe.License.LicenseType
    }'

echo ""
echo "--- iLO Network ---"
curl -sk $AUTH "$BASE/redfish/v1/Managers/1/EthernetInterfaces/1" \
  | jq '{
      FQDN,
      MACAddress,
      IPv4: [.IPv4Addresses[] | {Address, SubnetMask, Gateway, AddressOrigin}]
    }'

echo ""
echo "--- Chassis ---"
curl -sk $AUTH "$BASE/redfish/v1/Chassis/1" \
  | jq '{Model, SerialNumber, AssetTag, ChassisType}'

echo ""
echo "--- Firmware Inventory (Summary) ---"
curl -sk $AUTH "$BASE/redfish/v1/UpdateService/FirmwareInventory" \
  | jq -r '.Members[]["@odata.id"]' \
  | while read -r URI; do
      curl -sk $AUTH "$BASE$URI" | jq -r '"\(.Name): \(.Version)"'
    done

echo ""
echo "--- System Health ---"
curl -sk $AUTH "$BASE/redfish/v1/Systems/1" \
  | jq '{Health: .Status.Health, HealthRollup: .Status.HealthRollup}'
```

Save the output to a file dated with the server serial number:

```bash
./collect-asset-report.sh 192.168.1.101 admin password \
  > asset-report-CZ123456AB-$(date +%Y%m%d).txt
```

> For detailed firmware version collection, see `skills/deploy/firmware/README.md`. The firmware skill covers iLOrest-based full inventory and component-level version comparison.

---

## Warranty and Support Package

### What HPE Support Requires

When opening an HPE support case, gather these items before calling or opening a ticket. Having them ready reduces resolution time significantly.

| Item | Where to Get It | Command |
|---|---|---|
| Server serial number | `GET /redfish/v1/Systems/1` → `SerialNumber` | See asset report script above |
| Product number (SKU) | `GET /redfish/v1/Systems/1` → `SKU` | Same |
| iLO firmware version | `GET /redfish/v1/Managers/1` → `FirmwareVersion` | See asset report script above |
| BIOS version | `GET /redfish/v1/Systems/1` → `BiosVersion` | Same |
| IML log (last 24h events) | `GET /redfish/v1/Systems/1/LogServices/IML/Entries` | See log-analysis skill |
| AHS log archive | iLO OEM AHSData endpoint | See health-check skill |
| Current health status | `GET /redfish/v1/Systems/1` → `Status` | See health-check skill |

The server serial number is the single most important piece of information. HPE support uses it to look up the hardware configuration, warranty entitlement, and service history. Without it, HPE cannot validate the service contract or pre-position parts.

### Collecting the Support Package

```bash
ILO=192.168.1.101
AUTH="-u admin:password"

echo "=== Support Case Preparation ==="
echo ""
echo "1. System Identity"
curl -sk $AUTH https://$ILO/redfish/v1/Systems/1 \
  | jq '{SerialNumber, SKU, Model, BiosVersion}'

echo ""
echo "2. iLO Firmware"
curl -sk $AUTH https://$ILO/redfish/v1/Managers/1 \
  | jq '{FirmwareVersion}'

echo ""
echo "3. Current Health"
curl -sk $AUTH https://$ILO/redfish/v1/Systems/1 \
  | jq '{
      Health: .Status.Health,
      HealthRollup: .Status.HealthRollup,
      PowerState,
      PostState: .Oem.Hpe.PostState
    }'

echo ""
echo "4. IML — Last 10 events (most recent first)"
curl -sk $AUTH \
  "https://$ILO/redfish/v1/Systems/1/LogServices/IML/Entries?\$orderby=Created%20desc&\$top=10" \
  | jq -r '.Members[] | "\(.Created)  [\(.Severity)]  \(.Message)"'
```

> For the full IML log analysis workflow and severity filtering, see `skills/diagnose/log-analysis/README.md`.

> For downloading the AHS (Active Health System) log archive for HPE case upload, see `skills/operations/health-check/README.md`. AHS logs contain up to 1.5 years of health telemetry and are the primary diagnostic input for HPE support engineers.

### HPE Support Contact

- **Warranty check and case opening:** https://support.hpe.com
- **Entitlement lookup by serial number:** https://www.hpe.com/us/en/contact-hpe.html
- **iLO support community:** https://community.hpe.com/t5/ilo-management-engine/bd-p/server-management

---

## Decommissioning Procedure

Decommissioning removes a server from production permanently. Follow all steps in order. Do not skip steps — they exist to protect data and preserve reusable assets.

> **STOP before starting:** Confirm the server is taken out of all load balancers, orchestration systems, and monitoring. Verify no workloads are running. Get explicit approval from the asset owner.

### Step 1 — Collect Final Asset Report

Before any destructive operation, record the final state of the server.

```bash
./collect-asset-report.sh 192.168.1.101 admin password \
  > decommission-report-CZ123456AB-$(date +%Y%m%d).txt
```

Archive this file in your CMDB or ticketing system. It provides an audit trail for what was installed on the server at the time of decommissioning.

Also capture a BIOS configuration snapshot for reference:

```bash
# Capture BIOS config snapshot via iLOrest
ilorest save --selector BiosRegistry. \
  --url https://192.168.1.101 -u admin -p password \
  -f bios-snapshot-CZ123456AB.json
```

> For full BIOS configuration capture, see `skills/deploy/bios-uefi/README.md`.

### Step 2 — Recover the iLO License Key

If the server has an iLO Advanced or iLO Advanced Premium Security license installed, retrieve the key before decommissioning. The license can be reused on another server.

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService/1 \
  | jq '{
      LicenseType: .Oem.Hpe.LicenseType,
      LicenseKey: .Oem.Hpe.LicenseKey
    }'
```

Record the license key. Then remove it from this server so its status in the HPE license portal is clear:

```bash
curl -sk -u admin:password \
  -X DELETE \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService/1
```

> For full license recovery and portability details, see `skills/operations/licensing/README.md`.

> **Note on key masking:** Some iLO firmware versions mask the key after installation (shown as `XXXXX-XXXXX-...`). If masked, retrieve the original key from the HPE iLO license portal at https://www.hpe.com/us/en/servers/integrated-lights-out-ilo.html using your HPE Passport account.

### Step 3 — Secure Erase All Drives

**WARNING: This permanently destroys all data on the drives. There is no recovery.** Confirm backups or data retention requirements have been satisfied before proceeding.

#### Option A: Per-Drive Secure Erase (Recommended — Gen11)

On Gen11 (iLO 6) servers, issue the standard Redfish `Drive.SecureErase` action per physical drive.

```bash
# First, discover all drives
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Storage \
  | jq -r '.Members[]["@odata.id"]' \
  | while read -r CTRL_URI; do
      echo "Controller: $CTRL_URI"
      curl -sk -u admin:password \
        "https://192.168.1.101$CTRL_URI" \
        | jq -r '.Drives[]["@odata.id"]'
    done
```

Trigger secure erase on each drive:

```bash
# Replace /redfish/v1/Systems/1/Storage/DA000000/Drives/0 with actual path
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{}' \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/DA000000/Drives/0/Actions/Drive.SecureErase
```

The operation is asynchronous. Monitor progress via the task returned in the `Location` header or by polling the drive's `Operations` field.

#### Option B: Per-Drive Secure Erase (Gen10 — SmartStorage OEM)

On Gen10 (iLO 5) servers with HPE Smart Array controllers, use the OEM `HpeSmartStorageDiskDrive.Erase` action:

```bash
# Discover drives under SmartStorage
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers \
  | jq -r '.Members[]["@odata.id"]' \
  | while read -r CTRL; do
      curl -sk -u admin:password \
        "https://192.168.1.101$CTRL/DiskDrives" \
        | jq -r '.Members[]["@odata.id"]'
    done

# Trigger erase on a specific drive (ErasePattern: "SanitizeRestrictedOverwrite" is most thorough)
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ErasePattern": "SanitizeRestrictedOverwrite"}' \
  "https://192.168.1.101/redfish/v1/Systems/1/SmartStorage/ArrayControllers/0/DiskDrives/0/Actions/HpeSmartStorageDiskDrive.Erase"
```

> For storage controller discovery, RAID teardown, and drive-level operations in full detail, see `skills/deploy/storage/README.md`.

#### Option C: Full System Erase via HPE OEM Action

HPE provides a single OEM action to erase all drives and reset NVRAM in one operation. Use this when you need a full wipe of all storage simultaneously. **This also resets the system ROM and iLO configuration — it is equivalent to running all decommission steps in one command.**

```bash
# WARNING: This erases ALL drives, clears NVRAM, resets iLO to factory defaults,
# and resets BIOS. The server will need complete re-provisioning after this.
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"SystemROMAndiLOEraseComponents": "SystemROMAndiLO"}' \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/Oem/Hpe/HpeComputerSystemExt.SystemReset
```

> If using Option C, skip Steps 4 and 5 below — they are already performed by this action.

### Step 4 — Clear NVRAM and Persistent Memory

If the server has NVRAM-backed storage or Intel Optane persistent memory, ensure it is cleared before the server leaves your custody.

Clear NVRAM via BIOS settings (requires reboot into BIOS):

```bash
# Set BIOS to clear NVRAM on next boot via Redfish BIOS pending settings
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"Oem": {"Hpe": {"IntelligentProvisioningIndex": 0}}}' \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings
```

For persistent memory (NVDIMMs), use the BIOS security menu or the platform management tools. HPE Persistent Memory configuration and sanitize options are accessible via the BIOS setup — see `skills/deploy/bios-uefi/README.md` for BIOS access and pending-settings workflow.

### Step 5 — Factory Reset iLO

> **Critical Rule — Requires Approval:** Factory resetting iLO erases ALL iLO configuration permanently, including:
> - All user accounts (including the Administrator account)
> - Network configuration (IP address, hostname, DNS)
> - iLO license (if not already removed in Step 2)
> - Security settings (SSL certificates, 2FA, directory auth)
> - Federation group memberships
> - Alert and syslog configuration
> - Session history and event logs
>
> After a factory reset, iLO returns to out-of-box defaults with only the `Administrator` account using the factory-default password printed on the server's iLO tag. **You will lose remote access via the current IP address.** Physical access or reconfiguring iLO from a local connection is required to re-establish management.
>
> Get explicit approval before executing this step. Log the approval in your ticket.

#### Via Redfish OEM Action

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "Default"}' \
  https://192.168.1.101/redfish/v1/Managers/1/Actions/Oem/Hpe/HpeiLO.ResetToFactoryDefaults
```

Expected response: `HTTP 200 OK`. iLO reboots immediately. The management interface becomes unreachable within seconds and takes 60–90 seconds to come back online with default settings.

#### Via iLOrest

```bash
ilorest factorydefaults \
  --url https://192.168.1.101 -u admin -p password
```

iLOrest prints a confirmation prompt before executing. The `factorydefaults` command maps to the same OEM action above.

#### Verifying Factory Reset Completed

After iLO reboots, connect using the default iLO credentials printed on the server tag. If the iLO login page appears with a blank hostname and the default credentials work, the factory reset completed successfully. The absence of any previously configured accounts confirms a clean reset.

### Step 6 — Update Inventory

Remove the server from your inventory systems:

1. Delete or archive the server entry in `inventory.yaml`
2. Close the server record in your CMDB
3. Remove from DNS and IPAM (both iLO IP and server IPs)
4. Remove from monitoring (Nagios, Prometheus, Zabbix, etc.)
5. Remove from configuration management (Ansible inventory, Puppet node, Salt minion)
6. Attach the asset report and decomm ticket to the hardware disposal record

---

## Server Repurposing

Repurposing resets the server to a clean state for a new workload without physically decommissioning it.

### Repurposing Checklist

1. **Collect baseline** — run the asset report script to record current state
2. **Recover the iLO license** if it will be replaced with a different tier
3. **Secure erase drives** — see Step 3 above (Option A or B; use per-drive erase to preserve the controller config)
4. **Reset BIOS to defaults** — clears all tuning and custom boot order:

```bash
# Reset BIOS to manufacturing defaults via Redfish
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{}' \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Actions/Bios.ResetBios
```

> For full BIOS reset and default restore procedures, see `skills/deploy/bios-uefi/README.md`.

5. **Clear RAID configuration** — remove all logical drives and arrays so storage presents as raw disks to the new OS:

```bash
# Gen11: delete volumes via standard Redfish
curl -sk -u admin:password \
  -X DELETE \
  https://192.168.1.101/redfish/v1/Systems/1/Storage/DA000000/Volumes/1
```

> For full RAID teardown and controller reset, see `skills/deploy/storage/README.md`.

6. **Set new asset tag** — update to reflect the new role:

```bash
curl -sk -u admin:password \
  -X PATCH \
  -H "Content-Type: application/json" \
  -d '{"AssetTag": "DC01-RACK3-U12-PROD-CACHE"}' \
  https://192.168.1.101/redfish/v1/Systems/1
```

7. **Re-provision** — follow the new-server workflow from `skills/foundation/concepts/README.md`, applying networking, firmware, BIOS, and storage configuration for the new role.

---

## Batch Asset Collection Across a Fleet

For inventorying multiple servers at once, iterate over a list of iLO addresses:

```bash
#!/usr/bin/env bash
# fleet-asset-summary.sh
# Prints a one-line summary per server: IP | Serial | Model | AssetTag | iLO FW

USER="${1:?Usage: $0 <username> <password> <ilo1> [ilo2 ...]}"
PASS="${2:?}"
shift 2

printf "%-18s %-14s %-30s %-25s %s\n" \
  "iLO IP" "SerialNumber" "Model" "AssetTag" "iLO FW"
printf '%0.s-' {1..110}; echo ""

for ILO in "$@"; do
  SYS=$(curl -sk -u "$USER:$PASS" \
    "https://$ILO/redfish/v1/Systems/1" 2>/dev/null)
  MGR=$(curl -sk -u "$USER:$PASS" \
    "https://$ILO/redfish/v1/Managers/1" 2>/dev/null)

  SERIAL=$(echo "$SYS" | jq -r '.SerialNumber // "unknown"')
  MODEL=$(echo "$SYS"  | jq -r '.Model // "unknown"')
  TAG=$(echo "$SYS"    | jq -r '.AssetTag // ""')
  FW=$(echo "$MGR"     | jq -r '.FirmwareVersion // "unknown"')

  printf "%-18s %-14s %-30s %-25s %s\n" \
    "$ILO" "$SERIAL" "$MODEL" "$TAG" "$FW"
done
```

Usage:

```bash
./fleet-asset-summary.sh admin password \
  192.168.1.101 192.168.1.102 192.168.1.103
```

Output example:

```
iLO IP             SerialNumber   Model                          AssetTag                  iLO FW
--------------------------------------------------------------------------------------------------------------
192.168.1.101      CZ123456AB     ProLiant DL380 Gen10           DC01-RACK3-U12-PROD-WEB   iLO 5 v2.72 (...)
192.168.1.102      CZ789012CD     ProLiant DL380 Gen10           DC01-RACK3-U14-PROD-DB    iLO 5 v2.72 (...)
192.168.1.103      MXQ1234567     ProLiant DL380 Gen11           DC01-RACK3-U16-DEV        iLO 6 v1.55 (...)
```

---

## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| PATCH AssetTag returns `HTTP 400` | Value exceeds length limit (max 32 chars on some firmware) | Shorten the tag value |
| PATCH AssetTag returns `HTTP 403` | Account does not have write permission | Use an account with Operator or Administrator role |
| `SerialNumber` field is empty | Server has not been POST-completed (still in pre-boot) | Power on and wait for POST to complete; check `PostState` |
| `SKU` field returns `null` | Uncommon on older Gen9 firmware | Use `PartNumber` as a fallback; check system label on chassis |
| Factory reset OEM action returns `HTTP 404` | Path differs on iLO 4 | On iLO 4 use `Actions/Oem/Hp/HpiLO.ResetToFactoryDefaults` (note `Hp` not `Hpe`) |
| Drive.SecureErase returns `HTTP 405` | Drive is part of an active RAID array | Remove the drive from the array first, or delete the logical drive |
| Drive.SecureErase returns `HTTP 400` | Drive type does not support the secure erase method | SAS/SATA HDDs use overwrite; NVMe drives use cryptographic erase — check `SupportedErasePatterns` |
| iLO unreachable after factory reset | Expected behaviour — iLO resets network config to DHCP | Connect via physical iLO port or use the factory-default IP printed on the server tag |
| `ilorest factorydefaults` fails with auth error mid-command | iLO reset its accounts before the command fully completed | This is normal; the reset succeeded |
| Secure erase task stuck in progress | Controller firmware bug or drive unresponsive | Check IML for drive errors; a hard reboot may be required; contact HPE support with drive model and controller firmware version |
