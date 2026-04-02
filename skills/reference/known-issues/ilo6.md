# Known Issues: iLO 6 Firmware

Applies to HPE ProLiant Gen11 servers running iLO 6.
Latest iLO 6 release as of 2026-03: **v1.69+** (check https://servermanagementportal.ext.hpe.com/docs/redfishservices/ilos/ilo6/ilo6_changelog for current).

---

## Architecture Changes That Break iLO 5 Automation

Before reviewing version-specific bugs, note these permanent breaking changes introduced in iLO 6 / Gen11 that affect all scripts written for iLO 5.

| What Changed | iLO 5 Path | iLO 6 Path |
|---|---|---|
| Storage management | `/redfish/v1/Systems/1/SmartStorage/` (OEM) | `/redfish/v1/Systems/1/Storage/` (DMTF standard) |
| Network adapters | `/redfish/v1/Systems/1/BaseNetworkAdapters/` | `/redfish/v1/Chassis/1/NetworkAdapters/` |
| Network ports | `NetworkPort` schema | DMTF `Port` schema |
| Drive LED | `IndicatorLED` property | `LocationIndicatorActive` property |
| BIOS OEM config | `/redfish/v1/Systems/1/Bios/BaseConfigs` | `/redfish/v1/Systems/1/Bios/Oem/Hpe/BaseConfigs` |

See [gen11.md](gen11.md) for automation migration guidance.

---

## Version-Specific Issues

### SmartStorage OEM Namespace Completely Removed — Automation Breakage
**Symptom:** Scripts and tools using `HpeSmartStorage`, `HpeSmartStorageArrayController`, `HpeSmartStorageLogicalDrive`, `HpeSmartStorageDiskDrive`, or `HpeSmartStorageStorageEnclosure` endpoints return HTTP 404 on Gen11 iLO 6. Monitoring dashboards show "storage unknown". Automation that creates or modifies RAID volumes fails silently or with a 404.
**Cause:** HPE intentionally removed the proprietary HPE OEM SmartStorage data model in iLO 6. Gen11 servers use the DMTF Redfish standard `Storage` model exclusively. The MR (Tri-Mode) and SR (SAS/SATA) Gen11 controllers do not expose the legacy SmartStorage OEM URIs.
**Workaround:**
1. Migrate all storage queries to `/redfish/v1/Systems/1/Storage/` and enumerate controllers there.
2. For logical drive operations, use `POST /redfish/v1/Systems/1/Storage/{controllerId}/Volumes`.
3. For disk enumeration, query `/redfish/v1/Systems/1/Storage/{controllerId}/Drives/`.
4. Remove all references to `SmartStorageConfig` and `SmartStorage` from scripts.
5. Validate that the HPE iLO REST interface tool (`ilorest`) is version 4.0 or later, as older versions use OEM SmartStorage paths.
**Affected versions:** iLO 6 all versions (permanent architectural change)
**Status:** Won't fix — by design; migrate to standard Redfish Storage model

---

### CAC Smartcard Authentication Fails with TLS 1.3 on Chrome/Edge
**Symptom:** Users attempting to log into iLO 6 web interface or HTML5 remote console using CAC/PIV smartcard authentication on Chrome or Microsoft Edge receive an SSL handshake error or blank authentication prompt. Firefox works correctly. The issue is reproducible and consistent for all CAC users on those browsers.
**Cause:** iLO 6 1.57 and surrounding versions require TLS post-handshake authentication for CAC smartcards. Chrome and Microsoft Edge disabled post-handshake authentication support when TLS 1.3 is active (per RFC 8446 client requirements). Firefox retains the option to re-enable it.
**Workaround:**
1. Use Mozilla Firefox for CAC-authenticated iLO 6 sessions.
2. In Firefox, navigate to `about:config` and set `security.tls.enable_post_handshake_auth` to `true`.
3. As an alternative, configure a non-smartcard fallback admin account for emergency access from Chrome/Edge.
4. Monitor HPE advisories for a firmware fix that implements certificate-first TLS 1.3 mutual auth.
**Affected versions:** iLO 6 1.57 (confirmed); earlier 1.x versions also affected
**Status:** Open — browser vendor behavior; Firefox workaround available; Vendor advisory: a00099683en_us

---

### HTML5 Remote Console Interrupted by Idle Connection Timeout During OS Install
**Symptom:** While performing an OS installation via iLO 6 HTML5 Integrated Remote Console (IRC), the console session drops unexpectedly mid-install. The iLO web interface also logs out. Virtual media mounted for the installation is ejected, causing the OS installer to fail with "installation source not found" or similar.
**Cause:** iLO 6 enforces an Idle Connection Timeout on management sessions. During an OS installation that is progressing without operator keyboard/mouse input, the session appears idle to iLO and is terminated. This ejects the virtual media and closes the console.
**Workaround:**
1. Before starting any OS installation, increase the Idle Connection Timeout: iLO Web UI > Security > Access Settings > Idle Connection Timeout > set to a value longer than your expected install time (e.g., 7200 seconds or Infinite).
2. Restore the timeout to a secure value (e.g., 1800 seconds) after the installation completes.
3. Alternatively, trigger periodic mouse movement or keystrokes in the console to prevent idle detection.
4. For fully automated OS deployments, use iLO 6 Virtual Media scripted mount (`POST /redfish/v1/Managers/1/VirtualMedia/2/Actions/VirtualMedia.InsertMedia`) rather than browser-based virtual media, as scripted mounts are not subject to the session idle timeout.
**Affected versions:** iLO 6 all versions
**Status:** Open — by design; mitigated by timeout configuration

---

### Redfish Schema Version Changes Break Property Access in Automation Scripts
**Symptom:** Automation scripts that hard-code `@odata.type` version strings (e.g., `Chassis.v1_10_2`, `ComputerSystem.v1_10_0`) fail to match objects returned by iLO 6, causing schema validation errors or missing-property exceptions. Monitoring tools that compare schema versions report unexpected mismatches.
**Cause:** iLO 6 upgraded all core Redfish schemas to significantly newer DMTF versions compared to iLO 5 (e.g., `Chassis.v1_10_2` → `v1_19_2`, `ComputerSystem.v1_10_0` → `v1_17_0`, `Memory.v1_7_1` → `v1_14_0`). Scripts that rely on exact schema version strings or assume specific property positions for those versions will fail.
**Workaround:**
1. Remove all hard-coded schema version checks from automation scripts.
2. Access properties by key name, not by positional index or version-gated logic.
3. For schema-version-dependent code, check the major version only (e.g., `Chassis.v1_*`) and handle missing properties gracefully.
4. Use `GET /redfish/v1/$metadata` or the individual resource's `@odata.type` at runtime to discover the current schema version.
5. Test all automation on a non-production Gen11 system before deploying to production.
**Affected versions:** iLO 6 all versions (compared to iLO 5 scripts)
**Status:** Open — permanent; scripts must be updated for Gen11

---

### BaseNetworkAdapters Endpoint Deprecated — Network Adapter Queries Return Empty
**Symptom:** Scripts querying `/redfish/v1/Systems/1/BaseNetworkAdapters/` on iLO 6 receive an empty collection or HTTP 404. Network adapter inventory appears missing from monitoring tools. Scripts that read `SupportsFlexibleLOM` or `SupportsLOM` properties encounter "property not found" errors.
**Cause:** The HPE OEM `HpeBaseNetworkAdapter` schema and the `BaseNetworkAdapters` endpoint are deprecated in iLO 6 v1.10 and later. Properties have migrated to the standard DMTF Redfish `NetworkAdapters` collection under the Chassis resource. The `SupportsFlexibleLOM` and `SupportsLOM` OEM properties were removed entirely with no replacement.
**Workaround:**
1. Replace queries to `/redfish/v1/Systems/1/BaseNetworkAdapters/` with `/redfish/v1/Chassis/1/NetworkAdapters/`.
2. Enumerate ports using the standard DMTF `Port` schema under each `NetworkAdapter` resource.
3. Remove all logic depending on `SupportsFlexibleLOM` or `SupportsLOM` — use physical slot/port discovery instead.
4. For write operations (PATCH/POST) on network adapter settings, verify the Gen11 controller firmware supports the target operation — some write paths are not yet available in early iLO 6 versions.
**Affected versions:** iLO 6 1.10 and later (deprecated); iLO 6 all versions for OEM property removal
**Status:** Won't fix — deprecated in favor of standard DMTF path

---

### iLO 6 Missing Features Present in iLO 5 at Launch — Early Firmware Gaps
**Symptom:** Features available in iLO 5 (e.g., certain SNMP v1/v2c trap configurations, some iLO Federation actions, specific iLOREST commands) are not available or behave differently on iLO 6 1.05 through 1.20. Operators migrating from Gen10 to Gen11 encounter "feature not supported" errors.
**Cause:** iLO 6 was released alongside Gen11 as a ground-up re-implementation targeting the new hardware platform. The initial release prioritized security hardening (OpenSSL upgrade from 1.0.2 to 3.0.x) and new Gen11 hardware support over feature parity with iLO 5. Several legacy iLO 5 features were deferred to subsequent firmware releases.
**Workaround:**
1. Update iLO 6 to the latest available firmware before attempting Gen10-equivalent feature usage.
2. Check the iLO 6 User Guide for your specific firmware version to confirm feature availability.
3. For SNMP: iLO 6 1.57 added enhanced SNMP support for new Gen11 storage controllers — update to 1.57 minimum for full SNMP functionality.
4. For iLO Federation: verify the specific federation action is listed in the iLO 6 documentation for your version.
5. File a case with HPE support if a critical feature remains absent in the latest firmware.
**Affected versions:** iLO 6 1.05 through 1.20 (most gaps resolved by 1.40+)
**Status:** Partially fixed — most features restored by iLO 6 1.40; check release notes for specific features
