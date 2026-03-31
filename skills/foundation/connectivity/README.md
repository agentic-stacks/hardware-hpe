# Connectivity: iLO, OneView, and iLOrest Setup

Establishing management access to HPE ProLiant servers. Every other skill in this stack assumes this connectivity is already working. This skill covers three access paths: iLOrest CLI, direct Redfish API over curl, and HPE OneView for fleet management.

---

## Prerequisites Checklist

Before attempting any connection, verify:

| Requirement | What to Check | How to Satisfy |
|---|---|---|
| iLOrest installed | `ilorest --version` returns a version | Install via package or pip (see below) |
| Network path to iLO management port | `ping 192.168.1.101` from your workstation | Connect iLO dedicated NIC to management VLAN |
| iLO credentials | Default: `Administrator` + printed tag password | Check physical pull-out tab on server front panel |
| HTTPS reachable | `curl -sk https://192.168.1.101/redfish/v1/ \| jq .` returns JSON | Verify TLS port 443 is not blocked by firewall |
| For OneView: appliance address | `ping oneview.example.com` | Deploy OneView VA or confirm address with team |
| For in-band: CHIF driver | `ls /dev/hpilo*` on Linux (or check Device Manager on Windows) | Install `hpe-ilo-chif` driver package |

### Installing iLOrest

**From HPE GitHub Releases (preferred — always current):**
```bash
# Linux RPM (RHEL/SUSE)
wget https://github.com/HewlettPackard/python-redfish-utility/releases/latest/download/ilorest.rpm
sudo rpm -ivh ilorest.rpm

# Linux DEB (Ubuntu/Debian)
wget https://github.com/HewlettPackard/python-redfish-utility/releases/latest/download/ilorest.deb
sudo dpkg -i ilorest.deb

# Verify install
ilorest --version
```

**From pip (for development/scripting use):**
```bash
pip install python-ilorest-library
# Note: pip installs the library, not the ilorest CLI binary.
# For the CLI binary, use the RPM/DEB packages above.
```

**ESXi:**
```bash
# ESXi 7.0
/opt/tools/ilorest --version

# ESXi 8.0
/opt/ilorest/bin/ilorest.sh --version
```

**Supported iLO versions:** iLO 4 (2.10+), iLO 5 (2.10+), iLO 6 (1.10+). The same iLOrest binary works against all three generations.

---

## Finding the iLO IP Address

Before you can connect, you need the iLO IP. Four ways to find it:

### 1. Physical LCD / Front Panel
Gen10/Gen11 servers have a small LCD or indicator panel on the front. Press the iLO information button (the curved arrow icon) to cycle through the iLO IP address, hostname, and MAC address.

### 2. UEFI System Utilities at POST
Power on the server and press **F9** during POST to enter UEFI System Utilities. Navigate to **System Configuration > iLO 5 Configuration Utility** (or iLO 6). The current IP is displayed and configurable here.

### 3. iLOrest In-Band Query (from the OS on the server itself)
If you already have OS access, iLOrest can query the local iLO without a network path:
```bash
ilorest login
ilorest get IPv4Addresses --select EthernetInterface. --filter Name="Manager Dedicated Network Interface"
ilorest logout
```

### 4. DHCP Lease Table
If iLO is configured for DHCP (the default), check your DHCP server lease table. iLO hostnames are typically `ilo-<serial>` or `ILO<serial>`. The MAC address on the server pull-out tab belongs to the iLO dedicated NIC.

---

## iLO Network Configuration

### Dedicated NIC vs Shared NIC Mode

| Mode | Description | When to Use |
|---|---|---|
| **Dedicated NIC** | iLO uses its own physical port (labeled `iLO` on the rear panel) | Preferred — full bandwidth, always-on management regardless of server NIC state |
| **Shared NIC (NCSI)** | iLO shares one of the server's data NICs via Network Controller Sideband Interface | When only one cable is practical; management traffic competes with data traffic |

Dedicated NIC is strongly preferred. It keeps management traffic on a separate VLAN and ensures iLO remains reachable even if the server's primary NICs fail or the OS is hung.

Configure via UEFI System Utilities (F9 at POST) or via Redfish:
```bash
# Check current NIC mode via iLOrest
ilorest login 192.168.1.101 -u Administrator -p 'changeme'
ilorest get NICSupportsIPv6 SharedNetworkPortActive --select Manager.
ilorest logout
```

### Static IP vs DHCP

DHCP is the factory default. For production management networks, configure a static IP so the iLO address does not change.

```bash
# Set static IP via Redfish (replace values for your environment)
curl -sk -u Administrator:changeme \
  -X PATCH https://192.168.1.101/redfish/v1/Managers/1/EthernetInterfaces/1/ \
  -H "Content-Type: application/json" \
  -d '{
    "DHCPv4": {"DHCPEnabled": false},
    "IPv4StaticAddresses": [{"Address": "192.168.1.101", "SubnetMask": "255.255.255.0", "Gateway": "192.168.1.1"}],
    "StaticNameServers": ["192.168.1.10", "192.168.1.11"]
  }'
```

After patching, iLO applies the new IP immediately. Your session to the old IP will drop — reconnect to the new address.

---

## iLOrest CLI Connection

iLOrest maintains a session context between commands (stored in `~/.ilorest/`), so you login once, run multiple commands, then logout. The tool handles authentication tokens and schema validation automatically.

### Out-of-Band (Remote) — Standard Login
Connect from a workstation or jump host across the network to iLO's management IP:

```bash
ilorest login 192.168.1.101 -u Administrator -p 'changeme'
```

With IPv6:
```bash
ilorest login [fe80::1234:5678:abcd:ef01] -u Administrator -p 'changeme'
```

### In-Band (From the Server's Own OS) — CHIF Driver
When you are logged into the OS running on the managed server, iLOrest communicates with iLO via the CHIF (Channel Interface) driver — an internal bus that bypasses the network entirely. No credentials are required in standard security mode:

```bash
# No IP, no credentials — iLOrest detects local iLO automatically
ilorest login
```

Higher security modes (Production or High Security) require credentials even in-band:
```bash
ilorest login -u Administrator -p 'changeme'
```

The CHIF driver must be installed. On Linux it creates `/dev/hpilo0`. Install via:
```bash
# RHEL/SUSE — driver is bundled in the SPP or available from HPE SDR
sudo rpm -ivh hpe-ilo-chif-*.rpm
```

### Certificate-Based Authentication
For automation environments where storing passwords in scripts is unacceptable, iLO supports user certificate authentication (requires iLO 5/6 with certificate configured for the account):

```bash
ilorest login 192.168.1.101 \
  --usercert user.pem \
  --userkey user.key
```

If the key is passphrase-protected:
```bash
ilorest login 192.168.1.101 \
  --usercert user.pem \
  --userkey user.key \
  --userpassphrase 'keypassphrase'
```

### Virtual NIC Login
On Gen10/Gen11 servers, iLO exposes a Virtual NIC (USB network adapter) to the OS for an alternative in-band path:

```bash
ilorest login --force_vnic -u Administrator -p 'changeme'
```

### Session ID Authentication
If you already have a valid Redfish session token from another tool, you can hand it to iLOrest:

```bash
ilorest login 192.168.1.101 --sessionid 'abc123sessiontoken'
```

### Verifying the Connection

After login, confirm the connection and explore available resource types:

```bash
# List all Redfish resource types available on this iLO
ilorest types

# List full schema type names (useful for --select filters)
ilorest types --fulltypes

# Select the ComputerSystem resource and list all properties
ilorest select ComputerSystem.
ilorest list --json

# Quick health check
ilorest get Status PowerState --select ComputerSystem.
```

Expected output from `ilorest get Status PowerState --select ComputerSystem.`:
```
PowerState=On
Status={"Health": "OK", "HealthRollup": "OK", "State": "Enabled"}
```

### Logout

Always logout to invalidate the session token:
```bash
ilorest logout
```

Or combine logout with the last command:
```bash
ilorest get PowerState --select ComputerSystem. --logout
```

---

## Direct Redfish API via curl

Use curl when scripting without iLOrest installed, when you need raw API response control, or when integrating with other tools.

### Session Token Authentication (Preferred for Scripts)

Session tokens avoid sending credentials on every request and produce auditable login events in the iLO Event Log.

```bash
# Step 1: Create session — capture the X-Auth-Token from response headers
# The token appears in response headers; -D /dev/stderr dumps headers to stderr
TOKEN=$(curl -k -s -X POST https://192.168.1.101/redfish/v1/SessionService/Sessions/ \
  -H "Content-Type: application/json" \
  -d '{"UserName": "Administrator", "Password": "changeme"}' \
  -D /dev/stderr 2>&1 | grep -i "x-auth-token" | awk '{print $2}' | tr -d '\r')

echo "Token: $TOKEN"

# Step 2: Also capture the session URI from Location header (needed for logout)
SESSION_URI=$(curl -k -s -X POST https://192.168.1.101/redfish/v1/SessionService/Sessions/ \
  -H "Content-Type: application/json" \
  -d '{"UserName": "Administrator", "Password": "changeme"}' \
  -D /dev/stderr 2>&1 | grep -i "^location:" | awk '{print $2}' | tr -d '\r')

# Step 3: Use the token for all subsequent requests
curl -k -s \
  -H "X-Auth-Token: $TOKEN" \
  https://192.168.1.101/redfish/v1/Systems/1/ | jq '{Name, PowerState, Status, BiosVersion}'

# Step 4: Delete the session when done (clean logout)
curl -k -s -X DELETE \
  -H "X-Auth-Token: $TOKEN" \
  "https://192.168.1.101${SESSION_URI}"
```

### Basic Auth (Simple Queries)

Simpler to use for one-off queries. Each request creates and immediately destroys an implicit session:

```bash
# Check server health
curl -k -s -u Administrator:changeme \
  https://192.168.1.101/redfish/v1/Systems/1/ | jq '.Status'

# Check power state
curl -k -s -u Administrator:changeme \
  https://192.168.1.101/redfish/v1/Systems/1/ | jq '.PowerState'

# Get iLO firmware version
curl -k -s -u Administrator:changeme \
  https://192.168.1.101/redfish/v1/Managers/1/ | jq '.FirmwareVersion'
```

### Verify Redfish Root (No Auth Required)
The service root is publicly accessible and confirms the API is reachable:

```bash
curl -k -s https://192.168.1.101/redfish/v1/ | jq '{RedfishVersion, Product}'
```

Expected output:
```json
{
  "RedfishVersion": "1.6.0",
  "Product": "iLO 6"
}
```

### Authentication Method Decision

| Situation | Method | Reason |
|---|---|---|
| Single one-off query | Basic auth | Simplest; no token management |
| Script with 5+ requests to same iLO | Session token | Fewer auth events in logs; slightly faster |
| Automated pipeline, no password storage | Certificate auth via iLOrest | Credentials never in scripts or logs |
| Interactive debugging | Basic auth | Immediate, no state to track |

---

## OneView Appliance Connection

HPE OneView manages server fleets. Instead of connecting to individual iLO IPs, you connect to the OneView appliance and it orchestrates across all managed servers.

### Acquiring an API Token

OneView uses its own REST API (not Redfish). Authentication is via `POST /rest/login-sessions`:

```bash
# Acquire OneView session token
OV_TOKEN=$(curl -sk -X POST https://oneview.example.com/rest/login-sessions \
  -H "Content-Type: application/json" \
  -H "X-API-Version: 4200" \
  -d '{"userName": "administrator", "password": "changeme"}' \
  | jq -r '.sessionID')

echo "OneView token: $OV_TOKEN"
```

Use the token in subsequent OneView API calls:
```bash
# List all managed server hardware
curl -sk \
  -H "Auth: $OV_TOKEN" \
  -H "X-API-Version: 4200" \
  https://oneview.example.com/rest/server-hardware | jq '.members[] | {name, powerState, status}'
```

### X-API-Version Header

OneView requires `X-API-Version` on every request. The version number maps to OneView software releases. Check supported versions:
```bash
curl -sk https://oneview.example.com/rest/version | jq '{currentVersion, minimumVersion}'
```

Use `currentVersion` from this response as your `X-API-Version` value for maximum feature access.

### Server Discovery via OneView

OneView discovers servers in two ways:

1. **Manual add:** Provide an iLO IP and credentials — OneView connects to that iLO and imports the server.
2. **Scoped discovery:** OneView scans a subnet range for iLO endpoints.

```bash
# Add a server by iLO IP
curl -sk -X POST https://oneview.example.com/rest/server-hardware \
  -H "Auth: $OV_TOKEN" \
  -H "X-API-Version: 4200" \
  -H "Content-Type: application/json" \
  -d '{
    "hostname": "192.168.1.101",
    "username": "Administrator",
    "password": "changeme",
    "licensingIntent": "OneView"
  }'
```

### OneView Logout

```bash
curl -sk -X DELETE https://oneview.example.com/rest/login-sessions \
  -H "Auth: $OV_TOKEN" \
  -H "X-API-Version: 4200"
```

---

## iLO Federation Groups

iLO Federation allows one iLO to query and manage a group of iLOs on the same network segment — without OneView. It is a built-in iLO 5/6 feature for small to medium deployments.

### How Federation Works

iLOs discover each other via multicast on the management network. You configure a Federation group name and shared key on each iLO. Once joined, any member iLO can query health, firmware, and power state across the entire group.

**Requirements:**
- iLO 5 or iLO 6 (iLO 4 has limited federation support)
- iLO Advanced license (Standard license supports view-only federation)
- All iLOs on the same multicast-capable network segment (same VLAN or with multicast routing)
- All members configured with the same group name and shared key

### Configuring Federation Membership

```bash
# Join a Federation group via Redfish
curl -sk -u Administrator:changeme \
  -X POST https://192.168.1.101/redfish/v1/Managers/1/FederationGroups/ \
  -H "Content-Type: application/json" \
  -d '{
    "Name": "DataCenter-A",
    "Key": "sharedgroupkey123"
  }'
```

### Querying Federation Peers

```bash
# List discovered federation peers (from any group member)
curl -sk -u Administrator:changeme \
  https://192.168.1.101/redfish/v1/Managers/1/FederationPeers/ | jq '.Members[].@odata.id'
```

### Federation vs OneView

| Factor | iLO Federation | OneView |
|---|---|---|
| Scale | Up to ~100 servers (practical) | Hundreds to thousands |
| Setup cost | Minimal — configure group key on each iLO | Deploy OneView VA (VM appliance) |
| License required | iLO Advanced for write operations | OneView license per server |
| Server profiles | No | Yes |
| Firmware baseline compliance | No | Yes |
| Use case | Small cluster, no OneView budget | Enterprise fleet management |

---

## Inventory Management

For automation that targets multiple servers, maintain an inventory file. This drives loops in all other skills.

### Manual Inventory Pattern

For deployments without OneView, maintain a YAML inventory file:

```yaml
# inventory.yaml — server fleet manifest
# Used by automation scripts to iterate over servers

servers:
  - name: server-01
    ilo_ip: 192.168.1.101
    ilo_user: Administrator
    ilo_pass_env: ILO_PASS_01      # reference env var, never inline the password
    generation: gen11
    role: compute
    location: rack-a-u12

  - name: server-02
    ilo_ip: 192.168.1.102
    ilo_user: Administrator
    ilo_pass_env: ILO_PASS_02
    generation: gen11
    role: compute
    location: rack-a-u14

  - name: storage-01
    ilo_ip: 192.168.1.110
    ilo_user: Administrator
    ilo_pass_env: ILO_PASS_STORAGE01
    generation: gen10plus
    role: storage
    location: rack-b-u6

oneview:
  address: oneview.example.com
  user: administrator
  pass_env: OV_PASS
  api_version: 4200
```

**Key conventions:**
- Store passwords in environment variables or a secrets manager (Vault, AWS Secrets Manager) — never inline in the inventory
- Include `generation` (gen10, gen10plus, gen11) — storage and NIC API paths differ by generation (see concepts skill)
- Include `role` — many operations are role-specific (e.g., RAID config for storage nodes)
- Include `location` — essential for physical identification when the UID LED is triggered

### OneView Auto-Discovery for Larger Fleets

With OneView, skip the manual inventory. Discover all managed servers at runtime:

```bash
# Get all server hardware as JSON — pipe to jq for the fields you need
curl -sk \
  -H "Auth: $OV_TOKEN" \
  -H "X-API-Version: 4200" \
  "https://oneview.example.com/rest/server-hardware?count=250" \
  | jq '.members[] | {name, mpHostInfo: .mpHostInfo.mpIpAddresses[0].address, status, powerState}'
```

This produces a live inventory of every server OneView manages, with current power state and health.

### When to Use Each Approach

| Scenario | Approach |
|---|---|
| 1–10 servers, no OneView | Manual `inventory.yaml` |
| 10–100 servers, no OneView | Manual `inventory.yaml` + iLO Federation for discovery |
| Any fleet with OneView deployed | OneView API for discovery; `inventory.yaml` as fallback for servers not yet added |
| CI/CD pipeline needing reproducibility | `inventory.yaml` checked into version control (no passwords) |

---

## Connectivity Verification

Run these checks to confirm all access paths are working before using any other skill.

### iLOrest Verification

```bash
# Full end-to-end iLOrest connectivity test
ilorest login 192.168.1.101 -u Administrator -p 'changeme' && \
ilorest types && \
ilorest select ComputerSystem. && \
ilorest get Status PowerState BiosVersion --json && \
ilorest logout
```

Expected: login succeeds, `types` lists ComputerSystem and other resource types, `get` returns power state and BIOS version, logout exits cleanly.

### Redfish curl Verification

```bash
# 1. Confirm API is reachable (no auth)
curl -k -s https://192.168.1.101/redfish/v1/ | jq '{RedfishVersion, Product}'

# 2. Confirm authentication works
curl -k -s -u Administrator:changeme \
  https://192.168.1.101/redfish/v1/Systems/1/ | jq '{Name, PowerState, Status}'

# 3. Confirm session creation works
curl -k -s -X POST https://192.168.1.101/redfish/v1/SessionService/Sessions/ \
  -H "Content-Type: application/json" \
  -d '{"UserName": "Administrator", "Password": "changeme"}' \
  -D - 2>&1 | grep -E "(HTTP|X-Auth-Token|Location)"
```

A successful session creation returns `HTTP/1.1 201 Created` with `X-Auth-Token` and `Location` headers.

### OneView Verification

```bash
# 1. Check API version
curl -sk https://oneview.example.com/rest/version | jq .

# 2. Authenticate and confirm token
OV_TOKEN=$(curl -sk -X POST https://oneview.example.com/rest/login-sessions \
  -H "Content-Type: application/json" \
  -H "X-API-Version: 4200" \
  -d '{"userName": "administrator", "password": "changeme"}' \
  | jq -r '.sessionID')
echo "Token acquired: ${OV_TOKEN:0:20}..."

# 3. List managed servers (confirm OneView can reach them)
curl -sk \
  -H "Auth: $OV_TOKEN" \
  -H "X-API-Version: 4200" \
  "https://oneview.example.com/rest/server-hardware?count=5" \
  | jq '.members | length'
```

### Common Connection Failures

| Symptom | Likely Cause | Fix |
|---|---|---|
| `curl: (7) Failed to connect` | No network path to iLO port | Check iLO NIC cable, VLAN, firewall port 443 |
| `HTTP 401 NoValidSession` | Wrong credentials or account locked | Verify username/password; check account lockout in iLO UI |
| `HTTP 401` on iLOrest login | Security state requires credentials for in-band too | Add `-u Administrator -p password` to in-band login |
| `ilorest login` hangs | CHIF driver not installed or service not running | Install `hpe-ilo-chif` package; check `systemctl status hpe-ilo-chif` |
| Certificate error in curl | iLO using self-signed cert | Add `-k` flag; or add iLO cert to trust store |
| `ilorest types` returns empty | Connected to wrong endpoint or iLO resetting | Logout and login again; verify iLO firmware is not updating |
| OneView `403 Forbidden` | API version mismatch | Check `X-API-Version` value against `/rest/version` |

---

## Key Decisions for Agents

1. **Always check connectivity first** — before any operation skill, run a quick `GET /redfish/v1/Systems/1/` and confirm `HTTP 200`. If it fails, stop and diagnose connectivity before proceeding.

2. **Prefer iLOrest for multi-step operations** — BIOS changes, firmware updates, and configuration tasks involve multiple reads, writes, and commits. iLOrest handles session state and schema validation automatically.

3. **Use curl for single reads and in integrations** — when you need one value or are integrating with a shell pipeline, curl with basic auth is simpler.

4. **Know your generation before using storage or NIC APIs** — check `BiosVersion` or `Oem.Hpe.Type` at `/redfish/v1/Systems/1/` to confirm Gen10 vs Gen11 before touching storage paths. See concepts skill for the differences.

5. **Set a static iLO IP in production** — DHCP iLO addresses break automation. If you find a server with DHCP-assigned iLO IP, configure static before building any inventory.

6. **Secure session cleanup** — always DELETE the session when done. Orphaned sessions accumulate (iLO has a session limit of ~30), and they appear in security audits. Use `ilorest logout` or `curl -X DELETE` on the session URI.
