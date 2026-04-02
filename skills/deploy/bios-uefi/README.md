# Deploy: BIOS and UEFI Configuration

BIOS and UEFI configuration via Redfish and iLOrest on HPE ProLiant Gen10/Gen11 servers. Covers reading and writing BIOS attributes, boot mode, boot order, Secure Boot, workload-optimized profiles, and the pending-settings apply pattern.

---

## Prerequisites

- iLO connectivity established — see `skills/foundation/connectivity/README.md`
- Server is accessible at a known iLO IP (referred to as `192.168.1.101` in examples)
- Credentials with **Administrator** or **Operator** role (BIOS changes require operator minimum; Secure Boot key management requires administrator)
- **Back up current BIOS config before any changes** (see Backup section below)
- Understand the pending-settings pattern: PATCH to `/Bios/Settings` stages changes; they take effect only after the server reboots. The server does not reboot automatically.

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| Current BIOS settings | `/redfish/v1/Systems/1/Bios` | GET |
| Pending BIOS changes | `/redfish/v1/Systems/1/Bios/Settings` | GET, PATCH |
| HPE boot order (OEM) | `/redfish/v1/Systems/1/Bios/Oem/Hpe/Boot` | GET |
| HPE boot order pending | `/redfish/v1/Systems/1/Bios/Oem/Hpe/Boot/Settings` | GET, PATCH |
| BIOS base configurations | `/redfish/v1/Systems/1/Bios/Oem/Hpe/Baseconfigs` | GET |
| Secure Boot | `/redfish/v1/Systems/1/SecureBoot` | GET, PATCH |
| System (boot override) | `/redfish/v1/Systems/1` | GET, PATCH |
| Server reset (apply BIOS) | `/redfish/v1/Systems/1/Actions/ComputerSystem.Reset` | POST |

---

## Backup Current BIOS Config

Always capture current settings before making changes. This creates a local snapshot you can diff against after reboot or use for rollback reference.

```bash
# Save full BIOS attributes to file
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios \
  | jq '.Attributes' > bios-backup-$(date +%Y%m%d-%H%M%S).json

# Verify backup was written
ls -lh bios-backup-*.json
```

Via iLOrest — saves schema-validated output:

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest select Bios.
ilorest get > bios-backup-$(date +%Y%m%d-%H%M%S).txt
ilorest logout
```

---

## Reading Current BIOS Settings

### Via Redfish (curl)

```bash
# All attributes (large — typically 200+ key/value pairs)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios \
  | jq '.Attributes'

# Specific attribute
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios \
  | jq '.Attributes.BootMode'

# Key operational attributes at a glance
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios | jq '.Attributes | {
    BootMode,
    HyperThreading,
    IntelTurboBoostTech,
    PowerRegulator,
    WorkloadProfile,
    IntelProcVtx,
    IntelVtd,
    SriovEnable,
    NumaGroupSizeOpt,
    SubNumaClustering,
    SecureBootStatus
  }'
```

### Via iLOrest

```bash
# Interactive session — select and get all BIOS attributes
ilorest login https://192.168.1.101 -u admin -p password
ilorest select Bios.
ilorest get

# Single attribute
ilorest get BootMode --select Bios.

# Logout when done
ilorest logout
```

### Key Attributes to Inspect

| Attribute | Values | Notes |
|---|---|---|
| `BootMode` | `Uefi`, `LegacyBios` | Current boot mode; changing may require OS reinstall |
| `HyperThreading` | `Enabled`, `Disabled` | SMT; affects core count seen by OS |
| `IntelTurboBoostTech` | `Enabled`, `Disabled` | CPU turbo frequency |
| `PowerRegulator` | `DynamicPowerSavings`, `StaticHighPerf`, `OsControl`, `StaticLowPower` | Power/perf balance |
| `WorkloadProfile` | `GeneralPowerEfficientCompute`, `Virtualization-MaxPerformance`, `GeneralPeakFrequencyCompute`, `TransactionalApplicationProcessing` | HPE preset profiles |
| `IntelProcVtx` | `Enabled`, `Disabled` | Intel VT-x (hardware virtualization) |
| `IntelVtd` | `Enabled`, `Disabled` | Intel VT-d / IOMMU (device passthrough) |
| `SriovEnable` | `Enabled`, `Disabled` | SR-IOV for NIC virtualization |
| `NumaGroupSizeOpt` | `Flat`, `Clustered` | NUMA memory topology |
| `SubNumaClustering` | `Enabled`, `Disabled` | SNC (Sub-NUMA Clustering) |
| `EnergyEfficientTurbo` | `Enabled`, `Disabled` | Allows turbo to trade off for power savings |
| `SecureBootStatus` | `Enabled`, `Disabled` | Read-only reflection of Secure Boot state |

---

## Boot Mode Configuration

Boot mode controls whether the server uses UEFI or Legacy BIOS (CSM) to boot.

### Check current boot mode

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios \
  | jq '.Attributes.BootMode'
```

### Change boot mode

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d '{"Attributes": {"BootMode": "Uefi"}}'
```

Reboot to apply:

```bash
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "ForceRestart"}'
```

Via iLOrest:

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest select Bios.
ilorest set BootMode=Uefi
ilorest commit
ilorest reboot ForceRestart
ilorest logout
```

> **WARNING:** Switching from `LegacyBios` to `Uefi` (or vice versa) on a running OS installation will almost certainly prevent the OS from booting. Only change boot mode on a freshly provisioned server, or when performing a full OS reinstall. Confirm the target OS image supports the desired boot mode before changing.

---

## Boot Order Configuration

### Read current boot order

Boot order from the standard Redfish System resource (boot override and current boot source):

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '.Boot'
```

HPE OEM extended boot order (full ordered list):

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Oem/Hpe/Boot \
  | jq '.'
```

Via iLOrest:

```bash
ilorest bootorder --select ComputerSystem.
```

### Modify persistent boot order

Use the HPE OEM boot endpoint to reorder persistent boot targets:

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Oem/Hpe/Boot/Settings \
  -H "Content-Type: application/json" \
  -d '{
    "DefaultBootOrder": [
      "Cd",
      "Usb",
      "HardDisk",
      "EmbeddedStorage",
      "PcieSlotStorage",
      "EmbeddedFlexLOM",
      "PcieSlotNic",
      "UefiShell"
    ]
  }'
```

Changes take effect on next reboot (same pending pattern as BIOS attributes).

### One-time boot override

One-time boot does not require a reboot-to-apply cycle — it takes effect on the next boot only and then reverts to the normal order. Patch the System resource directly:

```bash
# One-time boot to PXE (network boot)
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "Pxe", "BootSourceOverrideEnabled": "Once"}}'

# One-time boot to CD/virtual media
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "Cd", "BootSourceOverrideEnabled": "Once"}}'

# One-time boot to UEFI shell
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "UefiShell", "BootSourceOverrideEnabled": "Once"}}'

# Clear boot override (return to normal)
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideEnabled": "Disabled"}}'
```

Valid `BootSourceOverrideTarget` values: `None`, `Pxe`, `Floppy`, `Cd`, `Usb`, `Hdd`, `BiosSetup`, `Utilities`, `Diags`, `UefiShell`, `UefiTarget`, `SDCard`, `UefiHttp`.

Via iLOrest:

```bash
# One-time PXE boot
ilorest bootorder --onetimeboot=Pxe --select ComputerSystem.
ilorest commit
```

---

## Secure Boot

Secure Boot enforces UEFI signature validation — only binaries signed with keys in the Secure Boot database (db) are allowed to execute during boot.

### Check Secure Boot status

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SecureBoot \
  | jq '{SecureBootEnable, SecureBootCurrentBoot, SecureBootMode}'
```

Response fields:

| Field | Meaning |
|---|---|
| `SecureBootEnable` | Whether Secure Boot is configured to be enabled |
| `SecureBootCurrentBoot` | Whether Secure Boot was enforced on the current boot |
| `SecureBootMode` | `UserMode` (custom keys), `SetupMode` (no PK enrolled), `AuditMode`, `DeployedMode` |

### Enable Secure Boot

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/SecureBoot \
  -H "Content-Type: application/json" \
  -d '{"SecureBootEnable": true}'
```

### Disable Secure Boot

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/SecureBoot \
  -H "Content-Type: application/json" \
  -d '{"SecureBootEnable": false}'
```

### Reset Secure Boot keys to factory defaults

Removes all custom keys and re-enrolls HPE/Microsoft factory keys. Use when Secure Boot blocks a signed OS loader after key modifications.

```bash
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/SecureBoot/Actions/SecureBoot.ResetKeys \
  -H "Content-Type: application/json" \
  -d '{"ResetKeysType": "ResetAllKeysToDefault"}'
```

### iLO 6 (Gen11) — Advanced key management

iLO 6 exposes full UEFI Secure Boot database management (PK, KEK, db, dbx, dbr, dbt). Access individual key databases under:

```
/redfish/v1/Systems/1/SecureBoot/SecureBootDatabases/
```

List available databases:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/SecureBoot/SecureBootDatabases \
  | jq '.Members[].@odata.id'
```

> **Note:** Secure Boot changes require BootMode=Uefi. Secure Boot is unavailable (and the endpoint returns an error) when BootMode=LegacyBios.

---

## Performance Profiles (Workload-Optimized BIOS)

Apply these attribute sets before OS installation. Changes are staged to `/Bios/Settings` and apply on reboot. The `WorkloadProfile` attribute is a convenience shortcut on iLO 5 that sets many sub-attributes automatically — where available, prefer it and supplement with explicit overrides.

### Virtualization Baseline (VMware ESXi, KVM, Hyper-V)

Enables hardware virtualization features and SR-IOV for NIC passthrough.

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d '{
    "Attributes": {
      "WorkloadProfile": "Virtualization-MaxPerformance",
      "IntelProcVtx": "Enabled",
      "IntelVtd": "Enabled",
      "SriovEnable": "Enabled",
      "HyperThreading": "Enabled",
      "PowerRegulator": "StaticHighPerf",
      "EnergyEfficientTurbo": "Disabled"
    }
  }'
```

| Attribute | Value | Reason |
|---|---|---|
| `WorkloadProfile` | `Virtualization-MaxPerformance` | HPE preset tuning for virt workloads |
| `IntelProcVtx` | `Enabled` | VT-x required for hypervisor guest execution |
| `IntelVtd` | `Enabled` | IOMMU — required for PCIe device passthrough to VMs |
| `SriovEnable` | `Enabled` | SR-IOV — allows NIC to present virtual functions to VMs |
| `HyperThreading` | `Enabled` | Increases vCPU density per socket |
| `PowerRegulator` | `StaticHighPerf` | Prevents CPU frequency scaling from reducing VM performance |
| `EnergyEfficientTurbo` | `Disabled` | Ensures turbo runs at maximum, not throttled for efficiency |

### Database Baseline (Oracle, SQL Server, PostgreSQL)

NUMA-aware configuration with high memory bandwidth and consistent CPU frequency.

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d '{
    "Attributes": {
      "WorkloadProfile": "TransactionalApplicationProcessing",
      "NumaGroupSizeOpt": "Clustered",
      "SubNumaClustering": "Disabled",
      "HyperThreading": "Enabled",
      "PowerRegulator": "StaticHighPerf",
      "IntelTurboBoostTech": "Enabled",
      "EnergyEfficientTurbo": "Disabled"
    }
  }'
```

| Attribute | Value | Reason |
|---|---|---|
| `WorkloadProfile` | `TransactionalApplicationProcessing` | HPE preset for OLTP-style workloads |
| `NumaGroupSizeOpt` | `Clustered` | Exposes NUMA topology to OS for memory-local allocation |
| `SubNumaClustering` | `Disabled` | Keeps memory local within NUMA node; SNC can fragment memory for DB workloads |
| `HyperThreading` | `Enabled` | SQL Server licensing is per-socket; maximize logical core use |
| `PowerRegulator` | `StaticHighPerf` | Eliminates frequency scaling latency under query bursts |

### HPC Baseline (CFD, molecular dynamics, MPI workloads)

Deterministic single-thread performance; disables SMT to eliminate cross-thread interference.

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d '{
    "Attributes": {
      "WorkloadProfile": "GeneralPeakFrequencyCompute",
      "HyperThreading": "Disabled",
      "IntelTurboBoostTech": "Enabled",
      "PowerRegulator": "StaticHighPerf",
      "EnergyEfficientTurbo": "Disabled",
      "NumaGroupSizeOpt": "Flat",
      "SubNumaClustering": "Disabled"
    }
  }'
```

| Attribute | Value | Reason |
|---|---|---|
| `HyperThreading` | `Disabled` | 1 thread per physical core — eliminates SMT contention in tightly coupled MPI |
| `IntelTurboBoostTech` | `Enabled` | Maximizes per-core frequency |
| `PowerRegulator` | `StaticHighPerf` | OS-visible P-states locked at maximum |
| `EnergyEfficientTurbo` | `Disabled` | Prevents turbo from stepping down for thermal/power headroom |
| `NumaGroupSizeOpt` | `Flat` | Presents full memory as single flat NUMA domain (some MPI runtimes prefer this) |

### General Purpose Baseline (mixed workloads, dev/test, infrastructure)

Balanced profile; OS controls power states; efficient for servers with variable load.

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d '{
    "Attributes": {
      "WorkloadProfile": "GeneralPowerEfficientCompute",
      "HyperThreading": "Enabled",
      "PowerRegulator": "DynamicPowerSavings",
      "EnergyEfficientTurbo": "Enabled"
    }
  }'
```

| Attribute | Value | Reason |
|---|---|---|
| `WorkloadProfile` | `GeneralPowerEfficientCompute` | Balanced HPE preset |
| `PowerRegulator` | `DynamicPowerSavings` | Hardware-managed P-states; OS power governor retains control |
| `EnergyEfficientTurbo` | `Enabled` | Allows CPU to trade off some turbo for better power envelope |

---

## Applying BIOS Settings — The Pending Pattern

This is the most important operational concept for BIOS automation. Understand it before scripting any BIOS changes.

```
PATCH /Bios/Settings  →  staged (pending)  →  server reboots  →  settings applied  →  visible in /Bios
```

1. **Stage the change** — PATCH to `/redfish/v1/Systems/1/Bios/Settings`
2. **Verify what is staged** — GET `/redfish/v1/Systems/1/Bios/Settings` (shows pending, not current)
3. **Reboot the server** — POST to `Actions/ComputerSystem.Reset` with `ForceRestart`
4. **Verify the change applied** — GET `/redfish/v1/Systems/1/Bios` and compare

### Verify pending settings before reboot

```bash
# What is staged and waiting to apply
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  | jq '.Attributes'

# Compare current vs pending for a specific attribute
echo "Current:" && \
  curl -sk -u admin:password https://192.168.1.101/redfish/v1/Systems/1/Bios \
  | jq '.Attributes.HyperThreading'
echo "Pending:" && \
  curl -sk -u admin:password https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  | jq '.Attributes.HyperThreading'
```

### Batch BIOS changes via iLOrest

iLOrest buffers all `set` commands and sends them in a single PATCH on `commit`, reducing API round-trips:

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest select Bios.
ilorest set HyperThreading=Disabled
ilorest set PowerRegulator=StaticHighPerf
ilorest set IntelTurboBoostTech=Enabled
ilorest set EnergyEfficientTurbo=Disabled
ilorest commit                      # single PATCH to Bios/Settings
ilorest reboot ForceRestart         # apply on next boot
ilorest logout
```

### Verify settings took effect after reboot

Wait for server to reach POST-complete state (check `Oem.Hpe.PostState` = `FinishedPost`) then compare:

```bash
# Poll until server finishes POST
until curl -sk -u admin:password https://192.168.1.101/redfish/v1/Systems/1 \
  | jq -e '.Oem.Hpe.PostState == "FinishedPost"' > /dev/null 2>&1; do
  echo "Waiting for POST to complete..."
  sleep 15
done
echo "POST complete — verifying BIOS settings"

# Verify key attributes applied
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios \
  | jq '.Attributes | {HyperThreading, PowerRegulator, IntelTurboBoostTech, EnergyEfficientTurbo}'
```

---

## OneView Server Profile BIOS Settings

When managing a fleet, encode BIOS settings into a Server Profile Template rather than applying per-server. This ensures consistency and makes BIOS config version-controlled alongside infrastructure code.

### Read BIOS settings from an existing OneView server profile

```bash
# Requires HPE OneView appliance — substitute your OneView IP and session token
curl -sk -H "Auth: $OV_TOKEN" \
  https://192.168.1.10/rest/server-profiles/<profile-id> \
  | jq '.bios'
```

### Apply BIOS via server profile template (OneView REST)

```bash
# PATCH the BIOS section of a server profile template
curl -sk -X PATCH \
  -H "Auth: $OV_TOKEN" \
  -H "Content-Type: application/json" \
  https://192.168.1.10/rest/server-profile-templates/<template-id> \
  -d '{
    "bios": {
      "manageBios": true,
      "overriddenSettings": [
        {"id": "HyperThreading",        "value": "Disabled"},
        {"id": "PowerRegulator",        "value": "StaticHighPerf"},
        {"id": "IntelTurboBoostTech",   "value": "Enabled"},
        {"id": "EnergyEfficientTurbo",  "value": "Disabled"}
      ]
    }
  }'
```

**Benefits of OneView BIOS profiles:**

- Single change propagates to all servers using the template (on next compliance remediation)
- Drift detection — OneView alerts when a server's actual BIOS diverges from the profile
- Audit trail — profile versions are timestamped and tracked
- Scales to hundreds of servers without per-server scripting

---

## Verification and Rollback

### Diff current vs baseline

```bash
# Compare live settings to a saved baseline
CURRENT=$(curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios | jq '.Attributes')
BASELINE=$(cat bios-backup-20240101-120000.json)

diff <(echo "$BASELINE" | jq -S .) <(echo "$CURRENT" | jq -S .)
```

### Rollback procedure

If BIOS changes cause instability (boot failures, degraded performance, application errors):

1. **Stage the original values** — PATCH the backed-up attributes to `/Bios/Settings`
2. **Reboot** — changes apply on next boot
3. **Verify** — confirm IML shows no new hardware errors (`GET /Systems/1/LogServices/IML/Entries`)

```bash
# Re-apply saved baseline attributes
BASELINE_ATTRS=$(cat bios-backup-20240101-120000.json)
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d "{\"Attributes\": $BASELINE_ATTRS}"

# Reboot to apply rollback
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "ForceRestart"}'
```

### Check IML after BIOS change

Look for POST errors or hardware warnings that appeared after the reboot:

```bash
curl -sk -u admin:password \
  "https://192.168.1.101/redfish/v1/Systems/1/LogServices/IML/Entries?\$top=20" \
  | jq '.Members[] | select(.Severity != "OK") | {Created, Message, Severity}'
```

---

## Decision Guide for Agents

| Situation | Action |
|---|---|
| Server will run VMware ESXi or KVM | Apply virtualization baseline; verify `IntelProcVtx` and `IntelVtd` are `Enabled` before installing hypervisor |
| Server will run Oracle DB / SQL Server | Apply database baseline; confirm `NumaGroupSizeOpt=Clustered` post-reboot |
| Server will run MPI/HPC workloads | Apply HPC baseline; `HyperThreading=Disabled` is intentional — do not re-enable |
| Server is general infrastructure / dev | Apply general purpose baseline; `PowerRegulator=DynamicPowerSavings` is correct |
| PXE boot needed for OS deployment | Use one-time boot override (`BootSourceOverrideTarget=Pxe`) — no reboot cycle needed |
| OS not booting after BIOS change | Check pending settings were applied; check IML for POST errors; roll back attributes |
| Secure Boot blocking OS loader | Verify OS bootloader is signed; or temporarily disable Secure Boot; or reset keys to default |
| Fleet-wide BIOS standardization | Use OneView Server Profile Templates instead of per-server scripting |
| Need to confirm change applied | GET `/Bios` (not `/Bios/Settings`) after reboot; `/Bios/Settings` shows pending, not current |
