# Decision Guide: NIC Configuration Patterns

Use this guide to choose the right iLO NIC mode, teaming/bonding configuration, and network boot method for an HPE server deployment.

---

## iLO NIC Selection: Dedicated vs Shared

| Mode | Management Traffic Path | Recommended For | Avoid When |
|------|------------------------|-----------------|------------|
| Dedicated iLO NIC | Separate physical port, fully independent of OS | All production deployments | Never — always prefer dedicated if available |
| Shared NIC (NC-SI) | Shares a data NIC with the OS via NC-SI sideband | Lab / test / cost-constrained | Server is in production or management access must survive OS failure |

### When to Use Dedicated NIC

Always use the dedicated iLO NIC in production. Benefits:

- Management traffic is completely isolated from data network
- iLO remains accessible if the OS hangs, panics, or is being reinstalled
- No dependency on OS NIC driver state
- Allows iLO VLAN tagging independent of OS network config

The dedicated NIC is typically labeled **iLO** on the rear panel, physically separate from the numbered data ports.

### When Shared NIC Is Acceptable

Shared NIC (NC-SI mode) is acceptable only when:

- The server is in a lab or development environment
- A separate management network switch port is not available
- The tradeoff (losing iLO access if the OS NIC driver fails) is understood and accepted

To configure: iLO web UI > Network > iLO Dedicated Network Port vs Shared Network Port.

---

## OS-Level NIC Teaming / Bonding

These modes are configured at the OS or switch level, not in iLO. The hardware stack should advise on which mode matches the environment.

| Mode | How It Works | Switch Requirement | Bandwidth | Failover |
|------|-------------|-------------------|-----------|---------|
| Active/Standby | One NIC active, second waits; failover on link loss | None (standard ports) | Single NIC speed | Yes |
| LACP / 802.3ad | Both NICs active, switch aggregates the bond | LACP-capable switch, portchannel config | Aggregated (per-flow hashing) | Yes |
| Active/Active (balance-alb) | Both ports active, TX load balanced per destination | None | Near-aggregated | Yes |

### Active/Standby

Simplest configuration. One NIC carries all traffic; the standby takes over only on failure. No switch configuration required. Use when high availability is needed but the switch does not support LACP, or when simplicity is prioritized over throughput.

### LACP / 802.3ad

Both NICs active simultaneously. The switch treats both ports as a single logical port channel. Provides both redundancy and bandwidth aggregation. Requires the switch to have LACP configured on the corresponding ports. Note: a single TCP flow is hashed to one physical link — you get aggregated throughput across multiple flows, not faster single-stream transfers.

### Active/Active (balance-alb, Linux only)

Both NICs transmit independently using adaptive load balancing. Does not require switch configuration. Useful when LACP is unavailable but dual-NIC throughput is desired. Less deterministic than LACP.

---

## Network Boot Method Selection

| Method | Protocol | BIOS Mode | When to Use | Advantages | Requirements |
|--------|----------|-----------|-------------|------------|--------------|
| Legacy PXE | TFTP | Legacy BIOS | Existing legacy environments | Universal compatibility | Legacy BIOS mode, TFTP server |
| UEFI PXE | TFTP | UEFI | Modern environments, Secure Boot required | Works with UEFI Secure Boot, standard tooling | UEFI mode, TFTP server |
| UEFI HTTP Boot | HTTP / HTTPS | UEFI | Modern provisioning at scale | Fast transfers, VLAN-friendly, HTTPS support | UEFI mode, HTTP/HTTPS server |

### Legacy PXE

Uses TFTP to transfer bootloader and kernel. Works in Legacy BIOS boot mode. Choose this only when the deployment environment requires legacy BIOS or when existing infrastructure cannot be updated. TFTP is slow for large images and does not traverse VLANs easily without relay configuration.

### UEFI PXE

Same TFTP-based protocol as legacy PXE but runs in UEFI mode. Supports UEFI Secure Boot, which legacy PXE cannot. The preferred PXE method for any new UEFI deployment. Requires a DHCP server to hand out the TFTP server address and bootfile name (option 66/67 or DHCPv6 equivalents).

### UEFI HTTP Boot

Downloads the bootloader over HTTP or HTTPS. Significantly faster than TFTP for large images (no block-size limitations, full TCP windowing). Works across VLANs without special relay configuration. HTTPS support enables boot chain integrity verification. Recommended for modern provisioning systems (MaaS, Foreman, HPE iLO Amplifier). Requires a web server serving the boot artifacts and DHCP option 210 (or iPXE chain) to direct the client.

---

## VLAN Configuration Patterns

### Management Network Isolation

- Place iLO dedicated NIC on a dedicated management VLAN (commonly named MGMT or OOB)
- This VLAN should be routed only to management workstations, jump hosts, and automation systems — not to the general corporate network
- Segment management VLAN from data VLANs at the switch level (separate SVI / L3 boundary)

### iLO VLAN Tagging

iLO 5 and iLO 6 support 802.1Q VLAN tagging on the dedicated NIC. Configure via iLO web UI > Network > IPv4/IPv6 settings > VLAN. Set the VLAN ID matching the management VLAN. The port on the switch must be configured as an access port on that VLAN (not a trunk), or as a trunk with the native VLAN set to the management VLAN ID.

### Data NIC VLANs

Data NIC VLAN assignments are OS-controlled (via network interface configuration or VM switch). The hardware stack does not manage data VLANs directly — surface this to the OS/network configuration layer.

---

## HPE-Specific Notes

- iLO 5 / iLO 6: NIC selection is under iLO web UI > Network > iLO Dedicated Network Port
- FlexibleLOM and OCP adapters: teaming capabilities depend on the specific adapter firmware and OS driver
- HPE Virtual Connect (blade chassis): NIC teaming is often handled at the Virtual Connect module level, not at the server OS — check with the Virtual Connect profile before configuring OS-level bonding
- For iLO network boot settings: iLO web UI > System Configuration > BIOS/Platform Configuration > Network Boot Options
