# Decision Guide: RAID Level Selection

Use this guide to choose the right RAID level for an HPE Smart Array or MR controller. Read the use-case recommendations first, then consult the comparison table for details.

---

## Quick Recommendations by Use Case

| Use Case | Recommended RAID | Notes |
|----------|-----------------|-------|
| Boot / OS drive | RAID 1 | Two SSDs. Fast rebuild, simple. |
| General file storage (3-5 drives) | RAID 5 | Good balance of capacity and protection. |
| General file storage (4+ drives, data critical) | RAID 6 | Tolerates 2 simultaneous failures. |
| Database / high IOPS | RAID 10 | Four or more SSDs. Best random I/O. |
| Large sequential I/O (video, backups) | RAID 5 with many drives | Or RAID 50 for very large arrays. |
| Maximum fault tolerance | RAID 6 or RAID 60 | Required when drives are large (>4 TB). |
| Temporary / scratch data only | RAID 0 | Never use for anything you cannot lose. |

---

## RAID Level Comparison

| RAID | Min Drives | Fault Tolerance | Usable Capacity | Read Perf | Write Perf | Rebuild Time | Best For |
|------|-----------|-----------------|-----------------|-----------|------------|--------------|----------|
| 0 | 1 (2 for striping) | None | 100% | Excellent | Excellent | N/A | Temp, scratch |
| 1 | 2 | 1 drive | 50% | Good | Good | Fast | Boot / OS |
| 5 | 3 | 1 drive | (N-1)/N | Good | Fair | Slow on large drives | General purpose |
| 6 | 4 | 2 drives | (N-2)/N | Good | Poor | Very slow | Critical data |
| 10 | 4 | 1 per mirror pair | 50% | Excellent | Good | Fast | Database / IOPS |
| 50 | 6 | 1 per span | ~83% (varies) | Excellent | Fair | Medium | Large arrays |
| 60 | 8 | 2 per span | ~75% (varies) | Excellent | Poor | Slow | Large + critical |

### Notes on Usable Capacity

- RAID 5 with 4 drives: 3/4 = 75% usable
- RAID 6 with 6 drives: 4/6 = 67% usable
- RAID 10 always uses 50% regardless of drive count
- RAID 50/60 usable capacity depends on span count and drives per span

---

## Detailed Level Guidance

### RAID 0 — Striping, No Parity

All drive capacity is usable. Any single drive failure loses all data. Appropriate only for data that can be regenerated or discarded: render scratch, temp build caches, test environments. Do not use on production systems storing persistent data.

### RAID 1 — Mirroring

Two drives hold identical copies. One drive can fail without data loss. Rebuild is fast because the surviving mirror is simply copied to the replacement. Use for OS/boot volumes where simplicity and fast recovery matter more than capacity efficiency.

### RAID 5 — Distributed Parity

One drive's worth of space is used for distributed parity across the array. Survives one drive failure. Write performance is limited by the parity calculation overhead. Rebuild time grows with drive size — a 10 TB HDD can take 24+ hours to rebuild, during which a second failure is catastrophic. Prefer SSD to reduce rebuild window.

### RAID 6 — Dual Distributed Parity

Two drives' worth of space is used for parity. Survives two simultaneous drive failures. Write penalty is higher than RAID 5 (two parity calculations per write). Required for arrays using large-capacity HDDs (>4 TB) where rebuild windows are long enough that a second failure during rebuild becomes a realistic risk.

### RAID 10 — Striped Mirrors

Pairs of mirrored drives are striped together. Survives one failure per mirror pair. Excellent random read and write performance — the preferred choice for databases, OLTP workloads, and any IOPS-sensitive application. Capacity cost is 50% regardless of drive count.

### RAID 50 — Striped RAID 5 Spans

Multiple RAID 5 arrays striped together. Better write performance than a single large RAID 5. One drive per span can fail. Minimum 6 drives (two spans of 3). Useful for large arrays where a single RAID 5 would have excessive rebuild time.

### RAID 60 — Striped RAID 6 Spans

Multiple RAID 6 arrays striped together. Two drives per span can fail. Minimum 8 drives (two spans of 4). Use for large, critical arrays that need maximum fault tolerance without sacrificing too much capacity.

---

## Drive Type Considerations

### SSD vs HDD

- Use SSD for IOPS-heavy workloads (databases, VMs, boot volumes)
- Use HDD for bulk capacity (backups, archives, large file storage)
- Mixed arrays (SSD + HDD in same logical drive) are not recommended on Smart Array controllers — tiering is handled separately via HPE SmartCache or dedicated tiers

### SAS vs SATA

- SAS: dual-port design, higher rated MTBF, 12 Gb/s, preferred for enterprise and high-reliability requirements
- SATA: single-port, lower cost, 6 Gb/s, acceptable for secondary storage or budget-constrained deployments
- Do not mix SAS and SATA drives in the same logical drive — controllers may allow it but performance and reliability are unpredictable

### NVMe

NVMe drives bypass the Smart Array/MR controller entirely and connect directly to the CPU via PCIe. Hardware RAID is not available in the traditional sense. Options:

- **Software RAID (mdadm / Windows Storage Spaces):** OS-managed, works with any NVMe drives
- **ZFS / Storage Spaces Direct:** Full-featured software-defined storage with its own redundancy
- **HPE SR Gen10/Gen11 controllers with NVMe:** Some controllers support NVMe logical drives — check the controller's data sheet and SPP compatibility matrix

---

## Can You Change RAID Level Later?

Generally no. Migrating between RAID levels on Smart Array controllers (online capacity expansion / RAID level migration) is supported on some controller generations but:

- It is slow (can take days on large arrays)
- The array is at elevated risk during migration (degraded parity state)
- Not all level combinations are supported (e.g., RAID 5 to RAID 6 may not be available)
- NVMe arrays managed by software RAID do not support online migration

**Plan the RAID level carefully before deployment.** When in doubt, use RAID 6 or RAID 10 — the capacity cost is lower than a data recovery event.

---

## HPE-Specific Notes

- Smart Array controllers (P-series, E-series) use the ssacli tool or iLO Storage configuration
- MegaRAID controllers (in some Gen10+) use storcli or iLO Storage
- The HPE SSA (Smart Storage Administrator) GUI in iLO 5/6 shows predicted usable capacity before committing
- Always enable the Array Accelerator (write cache) with a healthy battery/capacitor backup — without it, RAID 5/6 write performance degrades significantly
- Enable Predictive Spare Activation if the controller supports it to trigger automatic rebuild before a failing drive actually fails
