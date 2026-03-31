# Operations: iLO Licensing

iLO license management for HPE ProLiant servers — checking current license tier, installing license keys, verifying feature activation, and recovering keys before decommissioning.

> **Why this matters:** The iLO license tier gates access to features used throughout every other skill. Virtual media, full AHS downloads, remote syslog, LDAP/AD authentication, and iLO Federation all require iLO Advanced or higher. When a feature is unexpectedly missing or returns a 403, check the license tier first.

---

## Prerequisites

- iLO connectivity established — see `skills/foundation/connectivity/README.md`
- `curl` and `jq` available on the workstation
- Credentials with **Administrator** role to install or modify a license
- **ReadOnly** role is sufficient to query the current license

---

## Key URIs

| Resource | URI | Methods |
|---|---|---|
| Manager (iLO) overview | `/redfish/v1/Managers/1` | GET |
| License service collection | `/redfish/v1/Managers/1/LicenseService` | GET |
| License resource | `/redfish/v1/Managers/1/LicenseService/1` | GET, DELETE |
| Install license key (activate) | `/redfish/v1/Managers/1/LicenseService` | POST |

> The license service lives under `Managers/1`, not under `Systems/1`. On iLO 5 and iLO 6, the path is consistent. Some older iLO 4 Gen8 servers expose license info only through the OEM extension at `Managers/1` — see the iLO 4 note in the Troubleshooting section.

---

## iLO License Tiers

HPE sells iLO licenses per-server. The license is locked to the iLO hardware, not to the operating system or the server serial number alone.

| Feature | Standard (free) | Advanced | Advanced Premium Security |
|---|---|---|---|
| Remote console | Text-only (SSH) | Full HTML5 graphical | Full HTML5 graphical |
| Virtual media | No | Yes | Yes |
| iLO Federation | No | Yes | Yes |
| Directory auth (LDAP / AD) | No | Yes | Yes |
| AHS download via iLO web | Limited (no full log) | Full | Full |
| Firmware update via iLO web | Basic | Full SPP support | Full SPP support |
| Remote syslog forwarding | No | Yes | Yes |
| Alertmail (email alerting) | No | Yes | Yes |
| SNMP traps | Basic (hardware only) | Full (configurable) | Full (configurable) |
| Power capping and regulation | No | Yes | Yes |
| iLO RESTful API | Yes | Yes | Yes |
| Redfish API | Yes | Yes | Yes |
| Two-factor authentication | No | No | Yes |
| FIPS / CNSA mode | No | No | Yes |
| Secure Recovery (RoT) | No | No | Yes |
| iLO Amplifier Pack integration | No | Yes | Yes |

**Standard** ships with every server at no cost. **Advanced** and **Advanced Premium Security** are purchased from HPE. The license string is a 25-character alphanumeric key delivered by email after purchase.

---

## Checking Current License

### Via Redfish

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService/1 \
  | jq '{
      Name,
      LicenseType: .Oem.Hpe.LicenseType,
      LicenseKey: .Oem.Hpe.LicenseKey,
      LicenseExpire: .Oem.Hpe.LicenseExpire,
      GracePeriodDays: .Oem.Hpe.GracePeriodDays
    }'
```

Sample output for an unlicensed server:

```json
{
  "Name": "iLO License",
  "LicenseType": "iLO Standard",
  "LicenseKey": "",
  "LicenseExpire": "No Expiration",
  "GracePeriodDays": 0
}
```

Sample output for a licensed server:

```json
{
  "Name": "iLO License",
  "LicenseType": "iLO Advanced",
  "LicenseKey": "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX",
  "LicenseExpire": "No Expiration",
  "GracePeriodDays": 0
}
```

Key fields to check:

| Field | Description |
|---|---|
| `LicenseType` | `"iLO Standard"`, `"iLO Advanced"`, or `"iLO Advanced Premium Security"` |
| `LicenseKey` | The installed license key (masked or full depending on iLO version) |
| `LicenseExpire` | `"No Expiration"` for perpetual keys; date string for eval/trial keys |
| `GracePeriodDays` | Days remaining in grace period if a timed license has expired |

### Via iLOrest

```bash
# Connect and get license info
ilorest ilolicense \
  --url https://192.168.1.101 -u admin -p password
```

iLOrest output shows the license type and key in a compact format. The `ilolicense` command is a convenience wrapper around the LicenseService resource.

### Checking License from the Manager OEM Section

The Manager resource also surfaces a brief license summary in its OEM extension:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1 \
  | jq '.Oem.Hpe.License'
```

This returns a short object like `{"LicenseType": "iLO Advanced", "LicenseKey": "XXXXX-..."}`. Use this for a quick one-line check without navigating to the full LicenseService resource.

---

## Installing a License Key

### Via Redfish (POST)

The license service collection accepts a POST with the license key string to activate a new license:

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ProductKey": "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"}' \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService
```

On success: `HTTP 200 OK` with an empty body or a success confirmation object. The license activates immediately — no reboot required.

On failure: `HTTP 400 Bad Request` with an error message in the body such as `"InvalidLicenseKey"` or `"LicenseKeyAlreadyInstalled"`.

### Via iLOrest

```bash
ilorest ilolicense --set XXXXX-XXXXX-XXXXX-XXXXX-XXXXX \
  --url https://192.168.1.101 -u admin -p password
```

### Verify Activation After Install

Immediately after installation, confirm the license type has changed:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService/1 \
  | jq '.Oem.Hpe.LicenseType'
```

Expected: `"iLO Advanced"` (or `"iLO Advanced Premium Security"` for APS keys).

Full verification sequence:

```bash
ILO=192.168.1.101
AUTH="-u admin:password"

# Install the license
curl -sk $AUTH \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ProductKey": "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"}' \
  https://$ILO/redfish/v1/Managers/1/LicenseService

# Confirm type changed
curl -sk $AUTH \
  https://$ILO/redfish/v1/Managers/1/LicenseService/1 \
  | jq '{LicenseType: .Oem.Hpe.LicenseType, Expire: .Oem.Hpe.LicenseExpire}'

# Confirm no expiration (perpetual license)
curl -sk $AUTH \
  https://$ILO/redfish/v1/Managers/1/LicenseService/1 \
  | jq 'if .Oem.Hpe.LicenseExpire == "No Expiration" then "OK - perpetual" else "WARNING - expires: \(.Oem.Hpe.LicenseExpire)" end'
```

---

## Feature Activation Verification

After installing a license, verify specific features are now accessible before relying on them in automation.

### Verify Virtual Media Access

Virtual media requires iLO Advanced. After licensing, check the virtual media collection:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/VirtualMedia \
  | jq '.Members[] | .["@odata.id"]'
```

If virtual media members are returned without a 403 error, the feature is active. Attempting to use virtual media on a Standard-licensed iLO returns `HTTP 403 Forbidden` with `"iLOAdvancedFeature"` in the error body.

### Verify Remote Console Access

Full graphical HTML5 remote console requires iLO Advanced. Check the console endpoint:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1 \
  | jq '.Oem.Hpe.Links.RemoteConsole'
```

The presence of a `RemoteConsole` link confirms console access is licensed. Standard license only exposes the SSH text console via the `NetworkProtocol` resource.

### Verify LDAP / Directory Authentication

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/SecurityService/DirectoryService \
  | jq '{DirectoryAuthType: .Oem.Hpe.DirectoryAuthType, ServerType: .Oem.Hpe.ServerType}'
```

If this returns data rather than a 403, directory authentication is accessible. Configure it via PATCH to this resource — see the connectivity skill for full AD/LDAP setup.

### Verify iLO Federation

Federation features (group management, multi-server health queries) are accessible when iLO Advanced is installed:

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/FederationGroups \
  | jq '.Members | length'
```

A Standard-licensed iLO returns 403 on this endpoint.

### Verify Remote Syslog

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/NetworkProtocol \
  | jq '.Oem.Hpe.RemoteSyslogEnabled, .Oem.Hpe.RemoteSyslogServer'
```

On Standard license, `RemoteSyslogEnabled` is absent or read-only and attempts to PATCH it return `HTTP 403`.

---

## License Recovery Before Decommissioning

Before wiping a server (OS reinstall, hardware return, or repurposing), retrieve the installed license key so it can be reused on another server.

> **Portability:** iLO licenses are tied to the iLO hardware chip, not the server chassis or serial number. When you replace a system board (which includes the iLO chip), the old license key is no longer valid for the new hardware — contact HPE for a reassignment. Licenses are generally reusable when moving to a different server, but HPE's licensing terms allow one active installation per key. Verify current HPE licensing terms for your license tier.

### Retrieve the License Key

```bash
curl -sk -u admin:password \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService/1 \
  | jq -r '.Oem.Hpe.LicenseKey'
```

> On some iLO versions the full key is masked after installation (shown as `XXXXX-XXXXX-...`). In that case, retrieve the key from the HPE iLO license portal at https://www.hpe.com/us/en/servers/integrated-lights-out-ilo.html using your HPE Passport account, where all purchased keys are stored against your entitlement.

### Full Decommission License Capture

```bash
ILO=192.168.1.101
AUTH="-u admin:password"

echo "=== License Summary Before Decommission ==="
curl -sk $AUTH https://$ILO/redfish/v1/Managers/1/LicenseService/1 \
  | jq '{
      LicenseType: .Oem.Hpe.LicenseType,
      LicenseKey: .Oem.Hpe.LicenseKey,
      LicenseExpire: .Oem.Hpe.LicenseExpire
    }'

echo "=== iLO Firmware Version ==="
curl -sk $AUTH https://$ILO/redfish/v1/Managers/1 \
  | jq '.FirmwareVersion'

echo "=== Server Serial Number ==="
curl -sk $AUTH https://$ILO/redfish/v1/Systems/1 \
  | jq '{Model, SerialNumber}'
```

Record this output. The serial number confirms which physical server the license was removed from, which is required for HPE support cases involving license reassignment.

### Removing a License (Downgrade to Standard)

To remove an installed license and revert to Standard:

```bash
curl -sk -u admin:password \
  -X DELETE \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService/1
```

The server reverts to iLO Standard immediately. Features that require Advanced will stop working at the next iLO session — active sessions are not immediately terminated.

---

## iLO Evaluation / Trial Keys

HPE provides 60-day evaluation keys for iLO Advanced and iLO Advanced Premium Security. Trial keys allow full feature testing in a lab before purchasing.

**How to obtain:**

1. Go to https://www.hpe.com/us/en/servers/integrated-lights-out-ilo.html
2. Select "Evaluate" or "Free Trial" under the iLO Advanced section
3. Log in or create an HPE Passport account
4. Complete the evaluation request form — no purchase required
5. Receive the 25-character trial key by email within minutes

**Applying an evaluation key is identical to applying a purchased key:**

```bash
curl -sk -u admin:password \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"ProductKey": "EVAL-KEY-XXXXX-XXXXX-XXXXX"}' \
  https://192.168.1.101/redfish/v1/Managers/1/LicenseService
```

**Trial key behaviour:**

- `LicenseExpire` shows the expiry date (60 days from activation)
- `GracePeriodDays` counts down after expiry before features are locked
- On expiry, iLO reverts to Standard; no data is lost but Advanced features become inaccessible
- Eval keys cannot be extended — purchase a perpetual key before the trial expires to maintain continuity

---

## Common Scenarios

### Scenario: Automation Script Failing with 403 on Virtual Media

```bash
# Check if the license is the issue
ILO=192.168.1.101
AUTH="-u admin:password"

LICENSE=$(curl -sk $AUTH \
  https://$ILO/redfish/v1/Managers/1/LicenseService/1 \
  | jq -r '.Oem.Hpe.LicenseType')

echo "License: $LICENSE"

if [ "$LICENSE" = "iLO Standard" ]; then
  echo "REASON: Virtual media requires iLO Advanced license."
  echo "Install an Advanced license key or use a different provisioning method."
else
  echo "License is $LICENSE — virtual media should be accessible."
  echo "Check user account permissions (requires Operator role or higher)."
fi
```

### Scenario: Batch License Check Across Multiple Servers

```bash
for ILO in 192.168.1.101 192.168.1.102 192.168.1.103; do
  LICENSE=$(curl -sk -u admin:password \
    https://$ILO/redfish/v1/Managers/1/LicenseService/1 \
    | jq -r '.Oem.Hpe.LicenseType')
  printf "%-18s %s\n" "$ILO" "$LICENSE"
done
```

Output example:
```
192.168.1.101      iLO Advanced
192.168.1.102      iLO Standard
192.168.1.103      iLO Advanced Premium Security
```

### Scenario: Install License on Multiple Servers

```bash
KEY="XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"

for ILO in 192.168.1.101 192.168.1.102 192.168.1.103; do
  echo -n "Installing on $ILO ... "
  HTTP_CODE=$(curl -sk -u admin:password \
    -o /dev/null -w "%{http_code}" \
    -X POST \
    -H "Content-Type: application/json" \
    -d "{\"ProductKey\": \"$KEY\"}" \
    https://$ILO/redfish/v1/Managers/1/LicenseService)

  if [ "$HTTP_CODE" = "200" ]; then
    echo "OK"
  else
    echo "FAILED (HTTP $HTTP_CODE)"
  fi
done
```

---

## Troubleshooting

| Symptom | Likely Cause | Action |
|---|---|---|
| `GET /LicenseService/1` returns 404 | iLO 4 (Gen8) does not use the LicenseService path | Use `GET /redfish/v1/Managers/1` and read `.Oem.Hp.License` (note: `Hp` not `Hpe` on iLO 4) |
| POST to install key returns 400 `InvalidLicenseKey` | Key string is malformed, wrong tier, or wrong iLO generation | Verify the key from the HPE license portal; check if key is for iLO 5 vs iLO 6 |
| POST returns 400 `LicenseKeyAlreadyInstalled` | The same key is already active | Check current license; if you need to reinstall, DELETE first then POST |
| License type still shows `iLO Standard` after POST returns 200 | Key was accepted but is for a different iLO hardware | Contact HPE support — the key may have already been bound to a different iLO chip |
| Feature returns 403 after installing Advanced license | User account lacks the required role | Verify the iLO account has Operator or Administrator role, not just ReadOnly |
| `LicenseExpire` shows a past date | Trial license has expired | Purchase a perpetual key or request a new evaluation key; apply via POST |
| `GracePeriodDays` is counting down | Timed license is expired and in grace period | Install a replacement perpetual key before grace period ends to avoid feature lockout |
| Key masked as `XXXXX-XXXXX-...` in GET response | iLO masks keys post-installation for security on some firmware versions | Retrieve the original key from the HPE iLO license portal using your HPE Passport account |
