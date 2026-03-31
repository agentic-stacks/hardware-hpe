# Deploy: Network and Boot Configuration

Hardware-level NIC discovery, iLO management network configuration, and network boot setup on HPE ProLiant Gen10/Gen11 via Redfish and iLOrest. Covers FlexibleLOM and PCIe NICs, PXE and UEFI HTTP boot, and the handoff checklist to OS provisioning.

**Scope:** Hardware NIC enumeration and boot configuration only. OS-level networking (bonding, IP assignment, routing, VLAN tagging in the OS) is explicitly out of scope — that is handled by the OS stack that composes with this hardware stack.

---

## Prerequisites

- iLO connectivity established — see `skills/foundation/connectivity/README.md`
- iLO IP accessible at `192.168.1.101` (substitute your iLO IP throughout)
- Credentials with **Operator** or **Administrator** role
- For boot order changes: understand the pending-settings pattern from `skills/deploy/bios-uefi/README.md`

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| Network adapter collection | `/redfish/v1/Chassis/1/NetworkAdapters` | GET |
| Single network adapter | `/redfish/v1/Chassis/1/NetworkAdapters/{id}` | GET |
| Network ports (Gen10) | `/redfish/v1/Chassis/1/NetworkAdapters/{id}/NetworkPorts` | GET |
| Network ports (Gen11) | `/redfish/v1/Chassis/1/NetworkAdapters/{id}/Ports` | GET |
| Port settings (Gen11 pending) | `/redfish/v1/Chassis/1/NetworkAdapters/{id}/Ports/{portId}/Settings` | GET, PATCH |
| Network device functions | `/redfish/v1/Chassis/1/NetworkAdapters/{id}/NetworkDeviceFunctions/{id}` | GET, PATCH |
| System ethernet interfaces | `/redfish/v1/Systems/1/EthernetInterfaces` | GET |
| System ethernet interface | `/redfish/v1/Systems/1/EthernetInterfaces/{id}` | GET, PATCH |
| iLO management NIC | `/redfish/v1/Managers/1/EthernetInterfaces/1` | GET, PATCH |
| iLO network protocol | `/redfish/v1/Managers/1/NetworkProtocol` | GET, PATCH |
| System (boot override) | `/redfish/v1/Systems/1` | GET, PATCH |
| HPE OEM boot order | `/redfish/v1/Systems/1/Bios/Oem/Hpe/Boot/Settings` | GET, PATCH |

---

## NIC Discovery and Enumeration

### List all network adapters

```bash
# List adapter collection (returns @odata.id links)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/NetworkAdapters \
  | jq '.Members[].@odata.id'
```

Typical output on a dual-adapter server:

```
"/redfish/v1/Chassis/1/NetworkAdapters/DA000000"
"/redfish/v1/Chassis/1/NetworkAdapters/DA000001"
```

### Get adapter details

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/NetworkAdapters/DA000000 \
  | jq '{
      Name,
      Manufacturer,
      Model,
      PartNumber,
      SerialNumber,
      Status
    }'
```

Key fields to check:

| Field | What It Tells You |
|---|---|
| `Name` | Human-readable name, often includes "FlexibleLOM" or "PCIe" |
| `Model` | NIC model string (e.g., `HPE Ethernet 10Gb 2-port 535FLR-T Adapter`) |
| `Manufacturer` | Typically `HPE` or `Broadcom` |
| `Status.Health` | `OK`, `Warning`, `Critical` |
| `Status.State` | `Enabled`, `Absent`, `Disabled` |

### Gen10 vs Gen11 differences

| Feature | Gen10 (iLO 5) | Gen11 (iLO 6) |
|---|---|---|
| Port collection | `NetworkPorts/` under adapter | `Ports/` under adapter (DMTF standard) |
| HPE OEM properties | Heavy use of `Oem.Hpe` for port details | Reduced OEM; most data in standard fields |
| BaseNetworkAdapters | Present as HPE OEM (`/redfish/v1/Systems/1/BaseNetworkAdapters`) | Removed — use `Chassis/1/NetworkAdapters` |
| VLAN on device functions | Via `Oem.Hpe` PATCH | Via standard `Ethernet.VLAN` field |

On Gen10, also check the HPE OEM path for a simpler adapter summary:

```bash
# Gen10 only — HPE OEM adapter list (simpler structure)
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/BaseNetworkAdapters \
  | jq '.Members[].@odata.id'
```

---

## FlexibleLOM vs PCIe NICs

**FlexibleLOM** (Flexible LAN on Motherboard) is a replaceable NIC module that plugs into a dedicated slot on the ProLiant motherboard — it uses a non-standard form factor specific to HPE servers. FlexibleLOM avoids consuming a PCIe expansion slot, making it the default embedded NIC for most ProLiant rack and tower servers. Typical examples: HPE Ethernet 10Gb 2-port 535FLR-T, HPE Ethernet 10/25Gb 2-port 631FLR-SFP28.

**PCIe NICs** are standard PCIe add-in cards installed in expansion slots.

### Identifying adapter type from Redfish

Check the `Name` or `Model` field of the adapter:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/NetworkAdapters \
  | jq -r '.Members[].@odata.id' \
  | while read uri; do
      curl -sk -u admin:password "https://192.168.1.101${uri}" \
        | jq -r '"URI: \(.["@odata.id"]) | Model: \(.Model // "N/A") | Name: \(.Name)"'
    done
```

Identification patterns:

| Model string contains | Type |
|---|---|
| `FLR`, `FlexibleLOM` | FlexibleLOM (embedded slot) |
| PCIe slot reference, or no `FLR` | PCIe add-in card |
| `iLO`, `Management` | iLO dedicated management NIC (not a data NIC) |

Common HPE NIC models:

| Model | Type | Ports | Speed |
|---|---|---|---|
| HPE Ethernet 10Gb 2-port 535FLR-T | FlexibleLOM | 2x RJ45 | 10GbE |
| HPE Ethernet 10/25Gb 2-port 631FLR-SFP28 | FlexibleLOM | 2x SFP28 | 10/25GbE |
| HPE Ethernet 1Gb 4-port 331FLR | FlexibleLOM | 4x RJ45 | 1GbE |
| HPE Ethernet 10Gb 2-port 562SFP+ | PCIe | 2x SFP+ | 10GbE |
| HPE Ethernet 10/25Gb 2-port 640SFP28 | PCIe | 2x SFP28 | 10/25GbE |
| HPE Ethernet 100Gb 2-port 960QSFP28 | PCIe | 2x QSFP28 | 100GbE |

---

## NIC Port Status and Link Information

### List ports on an adapter

Gen10 (iLO 5):

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/NetworkAdapters/DA000000/NetworkPorts \
  | jq '.Members[].@odata.id'
```

Gen11 (iLO 6):

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/NetworkAdapters/DA000000/Ports \
  | jq '.Members[].@odata.id'
```

### Get port details (link, MAC, speed)

Gen10:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/NetworkAdapters/DA000000/NetworkPorts/1 \
  | jq '{
      Id,
      Name,
      LinkStatus,
      CurrentLinkSpeedMbps,
      AssociatedNetworkAddresses,
      ActiveLinkTechnology,
      SupportedLinkCapabilities
    }'
```

Gen11:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/NetworkAdapters/DA000000/Ports/1 \
  | jq '{
      Id,
      LinkStatus,
      CurrentSpeedGbps,
      Ethernet: {MACAddress: .Ethernet.AssociatedMACAddresses},
      Status
    }'
```

Key port fields:

| Field | Values | Meaning |
|---|---|---|
| `LinkStatus` | `Up`, `Down`, `NoMedia` | Physical link state |
| `CurrentLinkSpeedMbps` (Gen10) | `1000`, `10000`, `25000` | Negotiated speed in Mbps |
| `CurrentSpeedGbps` (Gen11) | `1`, `10`, `25` | Negotiated speed in Gbps |
| `AssociatedNetworkAddresses` | MAC address array | Port MAC addresses |
| `ActiveLinkTechnology` | `Ethernet` | Protocol on this port |

### MAC address retrieval

Quick MAC listing across all ports of an adapter:

```bash
# Gen10 — NetworkPorts
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Chassis/1/NetworkAdapters/DA000000/NetworkPorts \
  | jq -r '.Members[].@odata.id' \
  | while read uri; do
      curl -sk -u admin:password "https://192.168.1.101${uri}" \
        | jq -r '"Port \(.Id): \(.AssociatedNetworkAddresses[])"'
    done
```

Via iLOrest:

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest list --select NetworkPort.
ilorest logout
```

---

## iLO Management NIC Configuration

iLO has its own NIC, separate from the server's data NICs. It can run in two modes:

| Mode | Description |
|---|---|
| **Dedicated** | Uses a physically separate RJ45 port on the server rear (labeled iLO or MGMT). Preferred — completely isolated from data traffic. |
| **Shared NIC** | iLO shares a data NIC via NCSI (Network Controller Sideband Interface). Uses the same physical cable and port as a data NIC. Use only when a dedicated port is unavailable. |

### Read current iLO NIC configuration

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/EthernetInterfaces/1 \
  | jq '{
      Name,
      MACAddress,
      IPv4Addresses,
      IPv6Addresses,
      DHCPv4,
      VLANs: .VLAN,
      FullDuplex,
      SpeedMbps,
      Status
    }'
```

The `IPv4Addresses` array shows the current IP, subnet, and gateway. `DHCPv4.DHCPEnabled` indicates whether DHCP is active.

### Configure static IP for iLO management NIC

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Managers/1/EthernetInterfaces/1 \
  -H "Content-Type: application/json" \
  -d '{
    "DHCPv4": {
      "DHCPEnabled": false
    },
    "IPv4StaticAddresses": [
      {
        "Address": "192.168.10.50",
        "SubnetMask": "255.255.255.0",
        "Gateway": "192.168.10.1"
      }
    ]
  }'
```

> **Note:** After changing the iLO IP, you will lose the current connection. Reconnect to the new IP. If the new IP is incorrect or unreachable, physical console access may be required to recover.

### Configure DNS for iLO management NIC

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Managers/1/EthernetInterfaces/1 \
  -H "Content-Type: application/json" \
  -d '{
    "NameServers": ["192.168.10.2", "192.168.10.3"],
    "StaticNameServers": ["192.168.10.2", "192.168.10.3"]
  }'
```

### Configure VLAN on iLO management NIC

Use when the iLO management port connects to a trunk port that must be tagged:

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Managers/1/EthernetInterfaces/1 \
  -H "Content-Type: application/json" \
  -d '{
    "VLAN": {
      "VLANEnable": true,
      "VLANId": 100
    }
  }'
```

### iLO NIC mode (dedicated vs shared)

Check current mode via iLO network protocol resource:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/NetworkProtocol \
  | jq '.Oem.Hpe.NICMode'
```

Values: `Dedicated` or `Shared`. To change NIC mode use iLOrest (this setting is under the HPE OEM extension):

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest select iLOEthernetInterface.
ilorest get NICEnabled
ilorest logout
```

---

## Server Data NIC Configuration

The system ethernet interfaces reflect data NIC port state from the OS perspective (as reported to iLO via NCSI or host-driver channels). These are readable but most configuration is handled at the OS level.

### Read system ethernet interfaces

```bash
# List all
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/EthernetInterfaces \
  | jq '.Members[].@odata.id'

# Get details for a specific interface
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/EthernetInterfaces/1 \
  | jq '{
      Name,
      MACAddress,
      SpeedMbps,
      LinkStatus,
      IPv4Addresses,
      VLAN,
      Status
    }'
```

### NIC teaming and bonding

NIC teaming, bonding, and LAG (Link Aggregation Group) configuration is **OS-level** and is not managed via iLO Redfish. iLO can see individual physical port state but cannot create or configure bonds:

- **Linux:** `nmcli`, `ip link`, or `/etc/network/interfaces` bonding
- **VMware ESXi:** vSphere standard/distributed switch teaming policies
- **Windows:** NIC teaming via Server Manager or `New-NetLbfoTeam`

This hardware stack does not configure bonding. Hand off to the OS stack once hardware NICs are confirmed healthy.

---

## PXE Boot Setup

### Legacy PXE (BIOS mode)

Legacy PXE requires `BootMode=LegacyBios` and the NIC must be PXE-enabled in BIOS. Set via BIOS attributes:

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d '{
    "Attributes": {
      "BootMode": "LegacyBios",
      "PxeBoot": "Enabled"
    }
  }'
```

Changes are pending until reboot. See `skills/deploy/bios-uefi/README.md` for the full pending-settings apply pattern.

### UEFI PXE boot (recommended)

In UEFI mode, PXE is enabled per NIC port via BIOS attributes. The attribute names follow the pattern `PciSlot<N>BootEnable` (PCIe NICs) or `EmbNic<N>Enable` (FlexibleLOM / embedded NICs):

```bash
# Check NIC-specific PXE BIOS attributes
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1/Bios \
  | jq '.Attributes | with_entries(select(.key | test("NicBoot|PxeEnable|FlexLom|EmbNic"; "i")))'

# Enable PXE on FlexibleLOM port 1 (attribute name varies by model)
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d '{"Attributes": {"FlexLom1NicBoot1": "NetworkBoot"}}'
```

> The exact BIOS attribute name for enabling PXE on a specific port varies by server model and NIC. Inspect the `Attributes` blob from `/Bios` and grep for keys matching the port you want. Common patterns: `FlexLom1NicBoot1`, `EmbNicPort1Boot`, `NicBoot1`.

### UEFI HTTP Boot (modern approach)

UEFI HTTP Boot is the preferred replacement for PXE in modern deployments:

- **Works across VLANs** — uses standard IP routing (no broadcast dependency like DHCP/TFTP PXE)
- **Faster** — serves large images over HTTP/HTTPS at wire speed without TFTP limitations
- **Supports HTTPS** — encrypted boot image delivery; can validate server certificate
- **Simpler infrastructure** — no TFTP server required, just an HTTP/HTTPS server

Configure HTTP Boot URL in BIOS:

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Settings \
  -H "Content-Type: application/json" \
  -d '{
    "Attributes": {
      "BootMode": "Uefi",
      "UefiHttpBootUriVlan": 0,
      "HttpBootUri": "https://boot.example.com/efi/boot/bootx64.efi"
    }
  }'
```

One-time boot to UEFI HTTP:

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "UefiHttp", "BootSourceOverrideEnabled": "Once"}}'
```

---

## Network Boot Order

### Set NIC as first persistent boot device

Place network boot ahead of local storage in the persistent boot order. Uses the HPE OEM boot order endpoint (pending — applies on next reboot):

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1/Bios/Oem/Hpe/Boot/Settings \
  -H "Content-Type: application/json" \
  -d '{
    "DefaultBootOrder": [
      "EmbeddedFlexLOM",
      "PcieSlotNic",
      "EmbeddedStorage",
      "PcieSlotStorage",
      "HardDisk",
      "Cd",
      "Usb",
      "UefiShell"
    ]
  }'
```

After staging, reboot to apply:

```bash
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "ForceRestart"}'
```

### One-time boot to PXE (does not change persistent order)

For OS deployment without permanently changing boot order. Takes effect on next boot only:

```bash
curl -sk -u admin:password -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "Pxe", "BootSourceOverrideEnabled": "Once"}}'
```

Then power on or restart the server:

```bash
curl -sk -u admin:password -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "On"}'
```

Valid `BootSourceOverrideTarget` values for network boot:

| Value | Description |
|---|---|
| `Pxe` | Legacy PXE or UEFI PXE network boot |
| `UefiHttp` | UEFI HTTP/HTTPS boot |
| `None` | No override — use persistent boot order |

Via iLOrest:

```bash
ilorest login https://192.168.1.101 -u admin -p password
ilorest bootorder --onetimeboot=Pxe --select ComputerSystem.
ilorest commit
ilorest reboot On
ilorest logout
```

---

## Preparing for OS Provisioning Handoff

### What this stack does (hardware scope)

- Enumerate physical NICs and confirm link status
- Identify FlexibleLOM vs PCIe adapters and their port MACs
- Configure the iLO management NIC (IP, VLAN, DNS)
- Enable PXE or UEFI HTTP boot on the appropriate data NIC
- Set boot order or one-time boot override for network boot

### What this stack does NOT do (OS scope)

- Configure network bonds, LAGs, or teaming
- Assign data NIC IP addresses
- Configure VLANs in the OS network stack
- Set up routing, DNS resolvers, or NTP in the OS
- Install or configure network drivers

These are handled by the OS provisioning stack (Ansible, cloud-init, Kickstart, etc.) after the server boots into the OS.

### Pre-handoff checklist

Before handing off to OS provisioning, verify:

```bash
ILO="192.168.1.101"

# 1. Confirm intended NIC is physically linked
curl -sk -u admin:password \
  https://${ILO}/redfish/v1/Chassis/1/NetworkAdapters/DA000000/NetworkPorts/1 \
  | jq '{LinkStatus, CurrentLinkSpeedMbps, AssociatedNetworkAddresses}'

# 2. Confirm boot override is set (if using one-time PXE)
curl -sk -u admin:password \
  https://${ILO}/redfish/v1/Systems/1 \
  | jq '.Boot | {BootSourceOverrideTarget, BootSourceOverrideEnabled}'

# 3. Confirm server power state before triggering boot
curl -sk -u admin:password \
  https://${ILO}/redfish/v1/Systems/1 \
  | jq '.PowerState'
```

Expected pre-handoff state:

| Check | Expected Value |
|---|---|
| NIC link status | `Up` |
| Boot override target | `Pxe` or `UefiHttp` |
| Boot override mode | `Once` |
| Power state | `On` (or ready to power on) |

---

## iLOrest Network Commands

```bash
# Connect
ilorest login https://192.168.1.101 -u admin -p password

# List network adapters
ilorest list --select NetworkAdapter.

# List network ports (Gen10)
ilorest list --select NetworkPort.

# List system ethernet interfaces
ilorest list --select EthernetInterface. --filter 'MACAddress!=None'

# Get iLO management NIC settings
ilorest select iLOEthernetInterface.
ilorest get

# Read current boot state
ilorest bootorder --select ComputerSystem.

# Set one-time PXE boot
ilorest bootorder --onetimeboot=Pxe --select ComputerSystem.
ilorest commit

# Set one-time UEFI HTTP boot
ilorest bootorder --onetimeboot=UefiHttp --select ComputerSystem.
ilorest commit

# Reboot server into network boot
ilorest reboot On

# Disconnect
ilorest logout
```

---

## Decision Guide

| Situation | Action |
|---|---|
| Need to find which NIC port to use for PXE | GET `Chassis/1/NetworkAdapters` and iterate ports; match by MAC or link status |
| FlexibleLOM not showing link | Check physical cable; confirm FlexLOM module is seated; check `Status.Health` |
| Gen10 adapter paths differ from Gen11 | Gen10: `NetworkPorts/`; Gen11: `Ports/`; Gen10 also has `BaseNetworkAdapters` OEM path |
| Need PXE across VLANs | Use UEFI HTTP Boot (`UefiHttp`) instead — HTTP boot uses routed IP, not PXE broadcast |
| iLO IP changed and connection dropped | Reconnect to new IP; if unreachable, use physical console or iLO dedicated port |
| Boot override not triggering on next boot | Verify `BootSourceOverrideEnabled` is `Once` (not `Disabled`); confirm `commit` was called in iLOrest |
| OS networking (bonding, IP assignment) | Out of scope for this skill — hand off to OS provisioning stack |
| Need persistent network-first boot order | PATCH `Bios/Oem/Hpe/Boot/Settings` with `DefaultBootOrder` array; reboot to apply |
