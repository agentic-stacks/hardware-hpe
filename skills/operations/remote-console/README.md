# Operations: Remote Console and Virtual Media

Remote access and virtual media operations for HPE ProLiant servers via iLO Redfish. Mount remote ISO images as virtual optical drives, configure boot order, and understand the scope of what agents can automate vs. what requires interactive browser access.

---

## Prerequisites

- iLO connectivity established — see `skills/foundation/connectivity/README.md`
- `curl` and `jq` available on the workstation
- Credentials with at least **Operator** role for virtual media and boot override operations
- **iLO Advanced license** required for virtual media and HTML5 remote console — see `skills/operations/licensing/README.md`
- An HTTP/HTTPS-accessible ISO image server, or NFS/CIFS share (for URL-based mounting)

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| Virtual media collection | `/redfish/v1/Managers/1/VirtualMedia` | GET |
| Floppy/USB slot (slot 1) | `/redfish/v1/Managers/1/VirtualMedia/1` | GET |
| CD/DVD slot (slot 2) | `/redfish/v1/Managers/1/VirtualMedia/2` | GET |
| Insert media action | `/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.InsertMedia` | POST |
| Eject media action | `/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.EjectMedia` | POST |
| System (boot override) | `/redfish/v1/Systems/1` | GET, PATCH |

---

## Virtual Media Overview

Virtual media lets iLO present a remote ISO or IMG file to the server as if a physical disc or USB key were plugged in. The server's BIOS, firmware updater, and OS installer see it as real hardware — no physical media, KVM switch, or on-site access required.

**Two virtual media slots:**

| Slot | ID | Typical Use |
|---|---|---|
| Floppy / USB key | `1` | IMG files, bootable USB images, firmware tools |
| CD / DVD | `2` | ISO images — OS installers, live environments, rescue discs |

**Supported image protocols:**

| Protocol | Example URL |
|---|---|
| HTTPS | `https://iso-server.example.com/rhel-9.3-dvd.iso` |
| HTTP | `http://192.168.1.50/images/ubuntu-24.04-server.iso` |
| NFS | `nfs://192.168.1.50/exports/isos/rhel-9.3-dvd.iso` |
| CIFS/SMB | `smb://fileserver/share/os-images/rhel-9.3-dvd.iso` |

HTTPS is preferred — it encrypts the image transfer between the iLO and the image server.

> **License requirement:** Virtual media features are gated behind the iLO Advanced license. If InsertMedia returns HTTP 400 or a `LicenseError`, verify the license is installed before continuing.

---

## Listing Virtual Media Slots

Check which slots exist and their current state:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia \
  | jq '.Members[]."@odata.id"'
```

Inspect a specific slot:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2 \
  | jq '{
      Name,
      MediaTypes,
      ConnectedVia,
      Inserted,
      Image,
      WriteProtected,
      TransferMethod,
      TransferProtocolType
    }'
```

Key fields:

| Field | Description |
|---|---|
| `Inserted` | `true` if media is currently mounted |
| `Image` | URL of the mounted image (empty if not mounted) |
| `ConnectedVia` | `"URI"` for URL-based mounts, `"Applet"` for browser-based HTML5 |
| `TransferProtocolType` | Protocol in use: `"HTTP"`, `"HTTPS"`, `"NFS"`, `"CIFS"` |
| `WriteProtected` | `true` for ISO (read-only); `false` for IMG (writable) |

---

## Mounting an ISO via Redfish

### Mount an ISO into the CD/DVD Slot

```bash
curl -sk -X POST \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.InsertMedia \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"Image": "https://iso-server.example.com/rhel-9.3-dvd.iso"}'
```

Expected response: `HTTP 200 OK` with an empty or minimal body.

### Mount with Explicit Options

Some iLO versions accept additional parameters:

```bash
curl -sk -X POST \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.InsertMedia \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{
    "Image": "https://iso-server.example.com/rhel-9.3-dvd.iso",
    "Inserted": true,
    "WriteProtected": true
  }'
```

### Verify the Mount

After inserting, confirm the image is active:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2 \
  | jq '{Inserted, Image, ConnectedVia, TransferProtocolType}'
```

Expected output when mounted:

```json
{
  "Inserted": true,
  "Image": "https://iso-server.example.com/rhel-9.3-dvd.iso",
  "ConnectedVia": "URI",
  "TransferProtocolType": "HTTPS"
}
```

If `Inserted` is `false` after the POST returned 200, wait a few seconds and re-check — iLO may still be initiating the connection to the image server.

---

## Setting One-Time Boot to Virtual CD

After mounting the ISO, configure the server to boot from the virtual CD on the next boot only. This does not change the persistent boot order.

```bash
curl -sk -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "Cd", "BootSourceOverrideEnabled": "Once"}}'
```

### Boot Override Values

| Field | Value | Meaning |
|---|---|---|
| `BootSourceOverrideTarget` | `"Cd"` | Boot from virtual CD/DVD |
| `BootSourceOverrideTarget` | `"UsbDrive"` | Boot from virtual floppy/USB (slot 1) |
| `BootSourceOverrideTarget` | `"Hdd"` | Boot from first local hard disk |
| `BootSourceOverrideTarget` | `"Pxe"` | Boot via PXE (network) |
| `BootSourceOverrideTarget` | `"None"` | Clear the override, use normal boot order |
| `BootSourceOverrideEnabled` | `"Once"` | Override applies to next boot only, then reverts |
| `BootSourceOverrideEnabled` | `"Continuous"` | Override applies every boot until cleared |
| `BootSourceOverrideEnabled` | `"Disabled"` | Override cleared, back to normal boot order |

### Verify Boot Override Is Set

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '.Boot | {
      BootSourceOverrideTarget,
      BootSourceOverrideEnabled,
      BootSourceOverrideTargetAllowableValues
    }'
```

---

## Ejecting Virtual Media

Always eject virtual media after the OS installation is complete and the server has rebooted from local disk. Leaving a virtual ISO mounted can cause the server to re-enter the installer on subsequent reboots if boot order changes.

```bash
curl -sk -X POST \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.EjectMedia \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{}'
```

Verify the slot is empty after ejecting:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2 \
  | jq '{Inserted, Image}'
```

Expected: `"Inserted": false`, `"Image": ""`.

---

## Virtual Media via iLOrest

iLOrest provides a `virtualmedia` command as an alternative to raw curl calls:

```bash
# List all virtual media slots
ilorest virtualmedia --url https://192.168.1.101 -u admin -p password

# Mount an ISO into slot 2 (CD/DVD)
ilorest virtualmedia 2 --url https://192.168.1.101 -u admin -p password \
  --image https://iso-server.example.com/rhel-9.3-dvd.iso

# Mount with boot-on-next-reset flag
ilorest virtualmedia 2 --url https://192.168.1.101 -u admin -p password \
  --image https://iso-server.example.com/rhel-9.3-dvd.iso \
  --bootnextreset

# Eject slot 2
ilorest virtualmedia 2 --url https://192.168.1.101 -u admin -p password \
  --remove
```

The `--bootnextreset` flag on iLOrest combines the InsertMedia and boot override PATCH into a single command.

---

## OS Provisioning Workflow

This workflow covers the agent-automatable portions of bare-metal OS provisioning via virtual media. Step 7 (the actual OS installation) requires human or OS-stack automation — it is outside the scope of this skill.

### Step 1: Verify Server Health

Before any operation, confirm the server is in a healthy state:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '{PowerState, Health: .Status.Health, HealthRollup: .Status.HealthRollup}'
```

All values should be `"OK"` before proceeding. If health is `"Warning"` or `"Critical"`, address the hardware issue first — see `skills/operations/health-check/README.md`.

### Step 2: Verify iLO Advanced License

Virtual media requires the Advanced license:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService \
  | jq '.Members[]."@odata.id"' | head -1 | xargs -I{} \
  curl -sk -u admin:password https://192.168.1.101{} | jq '{License: .Name, Tier: .Oem.Hpe.LicenseType}'
```

If no Advanced license is present, install one before proceeding — see `skills/operations/licensing/README.md`.

### Step 3: Mount the OS ISO

```bash
curl -sk -X POST \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.InsertMedia \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"Image": "https://iso-server.example.com/rhel-9.3-dvd.iso"}'
```

Confirm `Inserted: true` before moving to step 4.

### Step 4: Set One-Time Boot to Virtual CD

```bash
curl -sk -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideTarget": "Cd", "BootSourceOverrideEnabled": "Once"}}'
```

### Step 5: Reboot the Server

Use a graceful restart if the OS is running, or a direct power-on if the server is off:

```bash
# Server is currently on — graceful restart
curl -sk -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "GracefulRestart"}'

# Server is currently off — power on
curl -sk -X POST \
  https://192.168.1.101/redfish/v1/Systems/1/Actions/ComputerSystem.Reset \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"ResetType": "On"}'
```

See `skills/operations/power-mgmt/README.md` for full power management details.

### Step 6: OS Installer Boots from Virtual ISO

The server POST sequence will pick up the virtual CD for this boot cycle. The boot override is consumed on the first boot and automatically clears — `BootSourceOverrideEnabled` reverts to `"Disabled"` and `BootSourceOverrideTarget` reverts to `"None"` once the server boots.

Monitor boot progress via the iLO HTML5 remote console (see below).

### Step 7: Complete OS Installation

**This step is outside the scope of this skill.**

The OS installer runs interactively or via kickstart/preseed/cloud-init automation. The composing OS-specific stack is responsible for:
- Providing a kickstart/preseed/Ignition file (typically via HTTP)
- Monitoring installation progress
- Handling installer prompts if interactive

An agent cannot interact with the remote console to drive the installer — see the HTML5 Remote Console section for the browser-only limitation.

### Step 8: Disconnect Virtual Media

After the OS installation completes and the server reboots from local disk:

```bash
curl -sk -X POST \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.EjectMedia \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{}'
```

### Step 9: Confirm Boot Order Is Back to Local Disk

After the one-time override was consumed in step 6, verify the server will boot normally:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Systems/1 \
  | jq '.Boot | {BootSourceOverrideTarget, BootSourceOverrideEnabled}'
```

Expected: `"BootSourceOverrideTarget": "None"`, `"BootSourceOverrideEnabled": "Disabled"`.

If the values have not cleared (e.g., `"Continuous"` was used instead of `"Once"`), explicitly clear the override:

```bash
curl -sk -X PATCH \
  https://192.168.1.101/redfish/v1/Systems/1 \
  -u Administrator:password \
  -H "Content-Type: application/json" \
  -d '{"Boot": {"BootSourceOverrideEnabled": "Disabled"}}'
```

### Step 10: Hand Off to OS Stack

The server is now running its newly installed OS and booting from local disk. Pass control to the OS-specific configuration stack (Ansible, cloud-init, etc.) for post-install provisioning.

---

## HTML5 Remote Console

iLO's HTML5 remote console provides a browser-based virtual KVM — full keyboard, video, and mouse access to the server regardless of OS or network state. This is the primary tool for watching POST, interacting with BIOS setup, debugging boot failures, and observing OS installer progress.

### How to Access

1. Open a browser and navigate to `https://<ilo-ip>`
2. Log in with iLO credentials (Administrator or Operator role)
3. Navigate to **Remote Console & Media > HTML5 Console**
4. Click **Launch** — the console opens in a new browser tab

Requires the **iLO Advanced license**. The legacy Java-based remote console is removed in iLO 6.

### What the Console Is Good For

| Use Case | Notes |
|---|---|
| Watching POST and hardware initialization | Useful for diagnosing early boot failures before the OS loads |
| Entering BIOS setup (F9 during POST) | Required for one-time BIOS changes when Redfish BIOS settings are not available |
| Observing OS installer progress | Verify the virtual ISO booted correctly and the installer started |
| Debugging a hung OS | See kernel panics, fsck prompts, grub rescue shells |
| Entering RBSU/UEFI shell commands | Direct hardware configuration that has no Redfish equivalent |

### Agent Limitation

**Agents cannot interact with the HTML5 remote console.** The console is a browser-rendered WebSocket stream — there is no Redfish API or CLI equivalent for sending keystrokes or reading screen content programmatically.

Agent scope for remote access:
- Can mount/eject virtual media via Redfish API
- Can set boot order via Redfish API
- Can power the server on/off/restart via Redfish API
- Cannot type, click, or read the console screen

If the OS installer requires interactive input (e.g., disk layout confirmation, language selection without kickstart), a human operator must use the HTML5 console. Design automated provisioning workflows to use fully unattended installers (kickstart, preseed, Ignition, autoYaST) to avoid this dependency.

---

## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| InsertMedia returns HTTP 400 with `LicenseError` | iLO Advanced license not installed | Install license — see `skills/operations/licensing/README.md` |
| InsertMedia returns 200 but `Inserted` stays `false` | Image URL unreachable from iLO network | Verify the ISO server is accessible from iLO's network segment; check firewall rules and URL |
| InsertMedia returns 400 with `PropertyValueFormatError` | Malformed URL in `Image` field | Confirm the URL is well-formed and the file exists on the server |
| Server does not boot from virtual CD after reboot | Boot override was not set, or was set after reboot was triggered | Verify `BootSourceOverrideEnabled` is `"Once"` and `BootSourceOverrideTarget` is `"Cd"` before issuing reset |
| Boot override set but server boots local disk | Previous OS intercepted boot and ignored override | Ensure the server fully POST'd and reached BIOS boot selection; use `"Continuous"` for stubborn cases |
| EjectMedia returns 400 | Media is not currently inserted | Check `Inserted` field first; if already `false`, eject is a no-op |
| NFS mount fails silently | NFS share permissions or export options block iLO access | Ensure the NFS export allows access from the iLO IP; NFS v3 is more broadly supported than v4 |
| HTTPS mount fails with certificate error | iLO cannot verify the ISO server's TLS certificate | Use a certificate from a trusted CA, or use HTTP on a trusted internal network |
| Virtual CD slot unavailable | iLO firmware too old, or feature not supported on this server | Check iLO firmware version; Gen8 and older have limited virtual media support |

---

## Scope Boundary

This skill covers the iLO-level operations for virtual media and boot configuration:

**In scope:**
- Listing, mounting, and ejecting virtual media via Redfish
- Setting one-time and persistent boot override via Redfish
- Verifying virtual media state and boot configuration
- Understanding the HTML5 remote console and its agent limitations

**Out of scope:**
- OS installation itself — that is the composing OS-specific stack's responsibility
- OS-level configuration after install (Ansible, cloud-init, etc.)
- Kickstart/preseed/Ignition file authoring
- Physical KVM or out-of-band management hardware other than iLO
