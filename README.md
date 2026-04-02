# HPE ProLiant Server Administration — Agentic Stack

Teaches AI agents how to administer HPE ProLiant Gen10 and Gen11 servers across the full lifecycle — discovery, configuration, firmware management, health monitoring, troubleshooting, and decommissioning.

## What This Stack Covers

- **HPE ProLiant Gen10** (iLO 5) and **Gen11** (iLO 6) rack and tower servers
- **Management interfaces:** iLO RESTful API (Redfish), HPE OneView, iLOrest CLI
- **Full lifecycle:** from unboxed hardware to decommissioning
- **OS-agnostic:** prepares hardware for any OS — compose with OS-specific stacks

## Usage

```bash
# Start a new operator project
agentic-stacks init agentic-stacks/hpe-hardware my-hpe-fleet
cd my-hpe-fleet

# Or add to an existing project
agentic-stacks pull hpe-hardware
```

## Composability

This stack focuses on hardware. It pairs with:
- OS provisioning stacks (RHEL, Ubuntu, ESXi, etc.)
- Kubernetes stacks (e.g., kubernetes-talos for bare metal K8s)
- OpenStack stacks (e.g., openstack-kolla)

## Required Tools

| Tool | Purpose |
|---|---|
| `ilorest` | HPE iLOrest CLI for Redfish operations |
| `curl` | Direct Redfish API calls |
| `jq` | JSON response parsing |

Optional: `python3` + `hpeOneView` SDK (OneView automation), `ansible` + `hpe.ilo` collection (fleet automation)

## Authoring

This stack follows the [Authoring a Stack](https://github.com/agentic-stacks/agentic-stacks/blob/main/docs/guides/authoring-a-stack.md) guide.
