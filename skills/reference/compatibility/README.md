# Reference: Compatibility Matrices

Firmware and component compatibility matrices for HPE ProLiant Gen10 and Gen11 servers.

---

## How to Use This Skill

**Check this reference before:**

- Any firmware update (Critical Rule #3 — verify target firmware is supported on your platform before applying)
- Mixing components from different generations (storage controllers, NICs, drives)
- Planning a new deployment and selecting controller or NIC options
- Diagnosing issues where a version mismatch may be the root cause

**Important limitation:** These matrices are point-in-time snapshots compiled from HPE documentation as of early 2026. HPE releases updated SPPs and firmware every six months and revises compatibility data continuously. Always verify against the current HPE Support Center before making production decisions. See [Where to Find Current Matrices Online](#where-to-find-current-matrices-online).

**Cross-references:**

- Firmware update workflow: `skills/deploy/firmware/README.md`
- Troubleshooting version mismatch symptoms: `skills/diagnose/troubleshooting/README.md`
- Known firmware defects by version: `skills/reference/known-issues/README.md`

---

## SPP Version to Server Generation Support

HPE ships separate SPP ISO images for Gen10/Gen10 Plus and Gen11 starting with the 2023.04.0 release. Earlier SPPs covered both in a single bundle. When searching the HPE Support Center, select the correct SPP stream for your generation.

| SPP Version | Release | Gen9 | Gen10 | Gen10 Plus | Gen11 | Notes |
|-------------|---------|------|-------|------------|-------|-------|
| 2024.09.0 | Sep 2024 | No | Yes | Yes | Yes | Current release; separate ISO per gen |
| 2024.04.0 | Apr 2024 | No | Yes | Yes | Yes | Separate ISO per gen |
| 2023.09.0 | Sep 2023 | No | Yes | Yes | Yes | Separate ISO per gen |
| 2023.04.0 | Apr 2023 | No | Yes | Yes | Yes (limited) | First SPP with Gen11 content; limited Gen11 model coverage |
| 2022.09.0 | Sep 2022 | No | Yes | Yes | No | Gen10/Gen10 Plus only |
| 2022.03.0 | Mar 2022 | No | Yes | Yes | No | Gen10/Gen10 Plus only |
| 2021.10.0 | Oct 2021 | Yes | Yes | Yes | No | Last SPP with Gen9 content |
| 2021.04.0 | Apr 2021 | Yes | Yes | Yes | No | Gen9 through Gen10 Plus |

**Gen10 server families covered (DL/ML/BL):**

| Server | Gen | Form Factor | Notes |
|--------|-----|-------------|-------|
| DL20 Gen10 | Gen10 | 1U rack | Entry |
| DL160 Gen10 | Gen10 | 1U rack | |
| DL180 Gen10 | Gen10 | 2U rack | |
| DL360 Gen10 | Gen10 | 1U rack | High-density compute |
| DL360 Gen10 Plus | Gen10 Plus | 1U rack | Ice Lake Xeon |
| DL380 Gen10 | Gen10 | 2U rack | Mainstream 2S |
| DL380 Gen10 Plus | Gen10 Plus | 2U rack | Ice Lake Xeon |
| DL385 Gen10 Plus | Gen10 Plus | 2U rack | AMD EPYC 7002/7003 |
| ML110 Gen10 | Gen10 | Tower | |
| ML350 Gen10 | Gen10 | Tower/rack | |
| BL460c Gen10 | Gen10 | Blade | |

**Gen11 server families covered:**

| Server | Gen | Form Factor | Notes |
|--------|-----|-------------|-------|
| DL20 Gen11 | Gen11 | 1U rack | Entry |
| DL360 Gen11 | Gen11 | 1U rack | Sapphire Rapids Xeon |
| DL380 Gen11 | Gen11 | 2U rack | Mainstream 2S |
| DL385 Gen11 | Gen11 | 2U rack | AMD EPYC 9004 |
| DL560 Gen11 | Gen11 | 4U rack | 4-socket |
| ML110 Gen11 | Gen11 | Tower | |
| ML350 Gen11 | Gen11 | Tower/rack | |

---

## iLO Firmware Feature Availability

iLO 5 ships with Gen10/Gen10 Plus. iLO 6 ships with Gen11. They are not interchangeable — iLO 5 firmware cannot be installed on Gen11, and iLO 6 firmware cannot be installed on Gen10.

The minimum version listed is the first release where the feature became available. Earlier versions do not support it.

| Feature | iLO 5 Min Version | iLO 6 Min Version | Notes |
|---------|-------------------|-------------------|-------|
| Redfish 1.6 conformance / URI standardization | 1.40 | 1.05 (all) | iLO 6 ships Redfish-compliant from v1.05 |
| OData `$filter` query parameter | Not supported | 1.10 | iLO 6 only |
| `?only` query parameter (single-member collections) | 1.40 | 1.05 (all) | |
| PCIe slot and device enumeration via Redfish | 2.30 | 1.05 (all) | |
| NVMe drive PCIe interface info | 2.10 | 1.05 (all) | |
| TelemetryService (CPU utilization metrics) | 1.40 | 1.05 (all) | |
| Workload Performance Advisor | 1.40 | Not supported | iLO 5 only |
| Persistent memory domain/chunk management | 1.40 | Not supported | iLO 5 only |
| One-button secure erase | 1.40 | 1.05 (all) | |
| iLO configuration backup/restore service | 1.40 | 1.05 (all) | |
| Thermal configuration (optimal/increased/max/enhanced) | 2.30 | 1.05 (all) | |
| Storage controller resources via Redfish | 2.70 | 1.10 | |
| Drive secure erase reporting | 2.33 | 1.05 (all) | |
| Automatic Certificate Enrollment (SCEP) | 2.60 | 1.05 (all) | |
| LLDP / Link Layer Discovery Protocol | 2.65 | 1.05 (all) | |
| User-defined temperature thresholds | 2.60 | 1.05 (all) | |
| DPU / SmartNIC dedicated log service | 2.72 | 1.05 (all) | |
| TLS version management (enable/disable per version) | 2.72 | 1.05 (all) | |
| TLS 1.0/1.1 modification | 2.78 | 1.05 (all) | |
| SNMPv1/v3 enable/disable granular control | 2.90 | 1.40 | |
| Session timeout configuration | 2.90 | 1.40 | |
| Two-factor authentication (TFA) | 2.95 | 1.50 | Requires iLO Advanced license |
| SMTP configuration for TFA | 2.95 | 1.50 | |
| Location indicator LEDs via Redfish | 2.95 | 1.10 | |
| Fabric / switch / port management | 2.99 | 1.55 | |
| Storage enclosure chassis type | 3.01 | 1.56 | |
| Telemetry: power, CPU, memory, I/O utilization | 2.99 | 1.51 | |
| HTTP Basic Authentication control (Unadvertised mode) | 3.07 | 1.62 | |
| Cloud Connect / GreenLake workspace ID | 3.09 | 1.59 | |
| ComponentIntegrity (SPDM) framework | Not supported | 1.10 | iLO 6 / Gen11 only |
| SPDM measurements and signed attestation | Not supported | 1.45 | iLO 6 only |
| Security policy schema | Not supported | 1.71 | iLO 6 only |
| Secure Boot database management via Redfish | Not supported | 1.20 | iLO 6 only |
| KMIP key management integration | Not supported | 1.73 | iLO 6 only |
| HBM memory support | Not supported | 1.40 | Gen11 HBM-equipped SKUs only |
| Enhanced ThermalSubsystem / ThermalMetrics | Not supported | 1.64 | iLO 6 only |
| SSH key management URIs | Not supported | 1.64 | iLO 6 only |
| Storage controller certificate collections | Not supported | 1.67 | iLO 6 only |
| Syslog TLS / UDP event destinations | Not supported | 1.68 | iLO 6 only |
| Drive.SecureErase with sanitization type parameter | Not supported | 1.74 | iLO 6 only |
| Common Criteria Certification (firmware version) | 2.73 | 1.11 | |

**License notes:**

- Most Redfish read operations require no license.
- Two-factor authentication requires iLO Advanced.
- Federation features require iLO Advanced Premium Security Edition on iLO 5.
- GreenLake Compute Ops Management integration requires iLO Advanced on Gen11.

---

## Storage Controller Compatibility

### Gen10 / Gen10 Plus — Smart Array SR Controllers

These controllers use the HPE Smart Array driver stack and are managed via HPE SSA (Smart Storage Administrator) and the `SmartStorage` Redfish OEM extension (iLO 5 v2.70+).

| Controller | Form Factor | Interface | Cache | RAID Levels | Max Direct Drives | Firmware Source |
|------------|------------|-----------|-------|-------------|-------------------|-----------------|
| Smart Array E208i-a SR | Modular (Type-a) | 12G SAS/SATA | No cache (HBA mode capable) | 0, 1, 5, 10 | 8 | SPP |
| Smart Array E208i-p SR | PCIe plug-in | 12G SAS/SATA | No cache (HBA mode capable) | 0, 1, 5, 10 | 8 | SPP |
| Smart Array P204i-a SR | Modular (Type-a) | 12G SAS/SATA | 1 GB FBWC | 0, 1, 5, 10 | 8 | SPP |
| Smart Array P408i-a SR | Modular (Type-a) | 12G SAS/SATA | 2 GB FBWC | 0, 1, 1ADM, 5, 6, 10, 10ADM, 50, 60 | 32 | SPP |
| Smart Array P408i-p SR | PCIe plug-in | 12G SAS/SATA | 2 GB FBWC | 0, 1, 1ADM, 5, 6, 10, 10ADM, 50, 60 | 32 | SPP |
| Smart Array P816i-a SR | Modular (Type-a) | 12G SAS/SATA | 4 GB FBWC | 0, 1, 1ADM, 5, 6, 10, 10ADM, 50, 60 | 32 internal + 64 expander | SPP |

Notes:
- FBWC = Flash-Backed Write Cache. Requires capacitor module; verify capacitor is seated before firmware update.
- "Type-a" modular slot is the server-embedded controller slot; not a PCIe slot.
- HBA mode (passthrough, no RAID) is available on all Gen10 Smart Array controllers.
- Maximum drive counts increase with SAS expanders. The number above is direct-attach.

### Gen10 Plus — MR and SR Controllers (Transition Generation)

Gen10 Plus introduced MR-series (Broadcom MegaRAID-based) controllers alongside legacy Smart Array.

| Controller | Form Factor | Interface | Cache | RAID Levels | Max Drives | Firmware Source |
|------------|------------|-----------|-------|-------------|------------|-----------------|
| Smart Array P408i-a SR (Gen10 Plus) | Modular | 12G SAS/SATA | 2 GB FBWC | 0, 1, 5, 6, 10, 50, 60 | 32 | SPP |
| MR416i-a Gen10 Plus | Modular (OCP) | 12G SAS/SATA + NVMe | 4 GB DDR4 | 0, 1, 5, 6, 10, 50, 60 | 240 | SPP |
| MR416i-p Gen10 Plus | PCIe plug-in | 12G SAS/SATA + NVMe | 4 GB DDR4 | 0, 1, 5, 6, 10, 50, 60 | 240 | SPP |
| SR932i-p Gen10 Plus | PCIe x16 plug-in | 24G SAS/SATA + NVMe PCIe 4.0 | 8 GB DDR4 | 0, 1, 1TP, 5, 6, 10, 10TP, 50, 60 | 32 SAS/SATA + 16 NVMe | SPP |

Notes:
- MR-series controllers use a different management stack than Smart Array: HPE MR Storage Administrator (MRSA), not SSA.
- MR-series Redfish paths differ from Smart Array: use `StorageControllers` schema, not `SmartStorage` OEM extension.
- SR932i-p supports Tri-Mode (SAS + SATA + NVMe simultaneously) per physical port.

### Gen11 — MR and SR Controllers

Gen11 does not ship Smart Array SR controllers. All Gen11 storage controllers are MR-series (Broadcom) or SR-series Tri-Mode.

| Controller | Form Factor | Interface | Cache | RAID Levels | Max Drives | Firmware Source |
|------------|------------|-----------|-------|-------------|------------|-----------------|
| MR216i-a Gen11 | Modular (OCP) | 12G SAS/SATA | 2 GB DDR4 | 0, 1, 5, 10 | 16 | SPP |
| MR216i-p Gen11 | PCIe plug-in | 12G SAS/SATA | 2 GB DDR4 | 0, 1, 5, 10 | 16 | SPP |
| MR416i-a Gen11 | Modular (OCP) | 12G SAS/SATA | 4 GB DDR4 | 0, 1, 5, 6, 10, 50, 60 | 240 | SPP |
| MR416i-p Gen11 | PCIe plug-in | 12G SAS/SATA | 4 GB DDR4 | 0, 1, 5, 6, 10, 50, 60 | 240 | SPP |
| SR932i-p Gen11 | PCIe x16 plug-in | 24G SAS + PCIe 4.0 NVMe | 8 GB DDR4 | 0, 1, 1TP, 5, 6, 10, 10TP, 50, 60 | 32 SAS/SATA + 16 NVMe | SPP |

Notes:
- Gen11 MR controllers include SPDM (Security Protocol and Data Model) attestation support for supply chain security verification via iLO 6 v1.67+.
- SR932i-p Gen11 adds 24Gb/s SAS (vs 12Gb/s on Gen10 Plus variant) and PCIe 4.0 host interface.
- Firmware for all storage controllers is bundled in SPP. Do not apply MR firmware from non-HPE SPP sources — OEM customizations may be overwritten.

### Controller Cross-Generation Compatibility

| Scenario | Supported? | Notes |
|----------|------------|-------|
| Gen10 Smart Array P408i-a in Gen11 server | No | Wrong slot type; Gen11 uses OCP 3.0 |
| Gen10 Plus MR416i-a in Gen11 server | Check QuickSpecs | Some MR416i-a units are qualified for Gen11 — verify part number |
| Gen11 MR416i-p in Gen10 Plus server | Check QuickSpecs | PCIe form factor may fit but firmware support must be confirmed |
| Smart Array drives (SAS/SATA) migrated to MR controller | Conditional | Data accessible; RAID metadata format differs — rebuild required |
| NVMe drives from Gen10 in Gen11 | Generally yes | Check drive vendor qualification list |

---

## NIC Compatibility

### Gen10 / Gen10 Plus NIC Options

Gen10 uses FlexibleLOM (modular) and PCIe slots. FlexibleLOM adapters occupy the dedicated modular slot and do not consume PCIe slots.

| Model | Speed | Form Factor | Connector | Interface | Notes |
|-------|-------|------------|-----------|-----------|-------|
| Ethernet 1GbE 4-port 331T | 1GbE | PCIe 2.0 x4 | RJ-45 | Intel I350 | Entry quad-port |
| Ethernet 1GbE 4-port 366T | 1GbE | PCIe 2.0 x4 | RJ-45 | — | |
| Ethernet 1GbE 2-port 332T | 1GbE | PCIe 2.0 x1 | RJ-45 | — | Low-profile |
| Ethernet 10GbE 2-port 530SFP+ | 10GbE | PCIe 2.0 x8 | SFP+ | Intel X540 | |
| Ethernet 10GbE 2-port 530T | 10GbE | PCIe 2.0 x8 | RJ-45 | Intel X540 | Copper 10GbE |
| Ethernet 10GbE 2-port 562SFP+ FLR | 10GbE | FlexibleLOM | SFP+ | — | |
| Ethernet 10/25GbE 2-port 640SFP28 | 10/25GbE | PCIe 3.0 x8 | SFP28 | Mellanox ConnectX-4 Lx | |
| Ethernet 10/25GbE 2-port 640FLR-SFP28 | 10/25GbE | FlexibleLOM | SFP28 | Mellanox ConnectX-4 Lx | |
| Ethernet 100GbE 2-port 840QSFP28 | 100GbE | PCIe 3.0 x16 | QSFP28 | Mellanox ConnectX-5 | |
| Ethernet 10/25GbE 2-port 631FLR-SFP28 | 10/25GbE | FlexibleLOM | SFP28 | — | |

### Gen11 NIC Options

Gen11 replaces FlexibleLOM with OCP 3.0 Small Form Factor (SFF) modular NICs. OCP 3.0 adapters from Gen10 Plus may be physically compatible but must be verified against the server's QuickSpecs.

| Model | Speed | Form Factor | Connector | Silicon | Notes |
|-------|-------|------------|-----------|---------|-------|
| Broadcom BCM57412 2-port 10GbE SFP+ | 10GbE | OCP 3.0 / PCIe | SFP+ | Broadcom 57412 | |
| Broadcom BCM57414 2-port 10/25GbE SFP28 | 10/25GbE | OCP 3.0 | SFP28 | Broadcom 57414 | Standard Gen11 modular NIC |
| Broadcom BCM57504 4-port 10/25GbE SFP28 | 10/25GbE | PCIe 4.0 x16 | SFP28 | Broadcom 57504 | |
| Intel E810-XXVDA2 2-port 10/25GbE SFP28 | 10/25GbE | PCIe 4.0 x8 | SFP28 | Intel E810 | RDMA (RoCEv2) capable |
| Intel E810-CQDA2 2-port 100GbE QSFP28 | 100GbE | PCIe 4.0 x16 | QSFP28 | Intel E810 | RDMA (RoCEv2) capable |
| Intel E810-CQDA2 2-port 100GbE OCP | 100GbE | OCP 3.0 | QSFP28 | Intel E810 | OCP 3.0 form factor |

**PCIe generation note:** Gen11 servers have PCIe 5.0 host slots. PCIe 4.0 NICs operate in those slots at PCIe 4.0 speeds — full bandwidth for all listed adapters above. A dual-port 100GbE NIC requires PCIe 4.0 x16 or PCIe 5.0 x8 to run both ports at line rate.

### NIC Cross-Generation Notes

| Scenario | Supported? |
|----------|------------|
| Gen10 FlexibleLOM in Gen11 OCP 3.0 slot | No — different mechanical and electrical standard |
| Gen10 Plus OCP 3.0 adapter in Gen11 OCP 3.0 slot | Check QuickSpecs per adapter part number |
| Gen10 PCIe NIC in Gen11 PCIe slot | Generally yes if PCIe 3.0 card in PCIe 5.0 slot (backward compatible) |
| Gen11 PCIe NIC in Gen10 PCIe slot | Generally yes if PCIe 4.0 card in PCIe 3.0 slot (backward compatible, half bandwidth) |

---

## Checking Current Versions via Redfish

Use the Redfish firmware inventory to audit all installed firmware versions and compare against these matrices before planning updates.

### Pull Firmware Inventory

```bash
# Full component firmware inventory
curl -sk -u admin:password \
  https://ilo-ip/redfish/v1/UpdateService/FirmwareInventory \
  | jq '.'

# List firmware names and versions (iLO 6 — uses $expand)
curl -sk -u admin:password \
  "https://ilo-ip/redfish/v1/UpdateService/FirmwareInventory?\$expand=.(\$levels=1)" \
  | jq -r '.Members[] | "\(.Name // "?")\t\(.Version // "?")"'
```

### Pull iLO Version Specifically

```bash
curl -sk -u admin:password \
  https://ilo-ip/redfish/v1/Managers/1 \
  | jq '{Model, FirmwareVersion, RedfishVersion}'
```

### Pull Storage Controller Info

```bash
# List storage controller URIs (iLO 5 v2.70+ / iLO 6 all)
curl -sk -u admin:password \
  https://ilo-ip/redfish/v1/Systems/1/Storage \
  | jq -r '.Members[]."@odata.id"'

# Then fetch each controller:
curl -sk -u admin:password \
  https://ilo-ip/redfish/v1/Systems/1/Storage/DE00A000 \
  | jq '.StorageControllers[] | {Model, FirmwareVersion, SerialNumber}'
```

### Automation Tip: Baseline Comparison

Fetch `FirmwareInventory` from each server, serialize to JSON, and diff against a golden baseline file. A minimal pattern:

```python
# Collect: curl -sk -u admin:pass https://ilo-ip/redfish/v1/UpdateService/FirmwareInventory
# Expand members, build dict: { item["Name"]: item["Version"] for item in members }
# Compare against baseline dict loaded from JSON — flag any key where values differ
# Baseline JSON format: { "iLO Firmware": "3.09", "System ROM": "U30 v2.82 (04/01/2024)" }
```

**Cross-reference:** For the full firmware update workflow, see `skills/deploy/firmware/README.md`.

---

## Where to Find Current Matrices Online

These tables are reference snapshots. Before any production firmware change, verify against the authoritative HPE sources below.

| Resource | URL | What it Contains |
|----------|-----|-----------------|
| HPE Support Center — SPP | https://support.hpe.com/connect/s/product?kmpmoid=5104018 | SPP downloads, release notes, server model compatibility |
| SPP Gen10 Software Details | https://support.hpe.com/connect/s/softwaredetails?collectionId=MTX-7a27f4cba9ca469d | Gen9/Gen10/Gen10 Plus SPP collection |
| SPP Gen11 Software Details | https://support.hpe.com/connect/s/softwaredetails?collectionId=MTX-a2a2747055284d4e | Gen11 SPP collection |
| iLO 5 Redfish Changelog | https://servermanagementportal.ext.hpe.com/docs/redfishservices/ilos/ilo5/ilo5_changelog | Feature additions by iLO 5 version |
| iLO 6 Redfish Changelog | https://servermanagementportal.ext.hpe.com/docs/redfishservices/ilos/ilo6/ilo6_changelog | Feature additions by iLO 6 version |
| HPE Ethernet Adapters QuickSpecs | https://www.hpe.com/psnow/doc/a00073559enw | NIC model list, speeds, form factors, gen support |
| HPE Compute MR Controllers QuickSpecs | https://www.hpe.com/psnow/doc/a50004311enw | MR-series controller specs and compatibility |
| HPE SR Gen10 Plus Controllers QuickSpecs | https://www.hpe.com/us/en/collaterals/collateral.a50002562enw.html | SR-series controller specs |
| HPE Transceiver and Cable Matrix | https://www.hpe.com/psnow/doc/a00002507enw | Transceiver compatibility by NIC and switch |
| HPE SPOCK (Single Point of Connectivity Knowledge) | https://h20564.www2.hpe.com/hpsc/swd/public/readIndex?sp4ts.oid=1008862656 | Storage interoperability: controllers, drives, OS, HBA |

**Disclaimer:** HPE releases SPP updates approximately every six months (March/April and September). iLO firmware is released independently and more frequently. Compatibility data in this document reflects HPE documentation available as of early 2026. Component qualifications, maximum drive counts, and feature availability may change with new releases. Always treat production firmware decisions as requiring verification against the live HPE Support Center and the specific QuickSpecs for your server model and option part number.
