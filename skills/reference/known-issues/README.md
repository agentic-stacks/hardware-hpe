# Reference: Known Issues

Version-specific bugs, firmware defects, and platform quirks with exact workarounds.

---

## When to Use This Reference

Check these files **before every operation** (Critical Rule #6). A firmware version that appears healthy may carry a known defect that will surface under your specific workload. Search by:

- iLO firmware version you are running or targeting
- Platform generation (Gen10 / Gen11)
- Symptom keyword (e.g., "session", "virtual media", "cache", "console")

---

## Files in This Directory

| File | Contents |
|------|----------|
| [ilo5.md](ilo5.md) | iLO 5 firmware known issues (Gen10 / Gen10 Plus) |
| [ilo6.md](ilo6.md) | iLO 6 firmware known issues (Gen11) |
| [gen10.md](gen10.md) | Gen10 platform-specific hardware and BIOS issues |
| [gen11.md](gen11.md) | Gen11 platform-specific hardware and automation issues |

---

## Entry Format

Every entry follows this standard structure. When adding new issues, match it exactly.

```
### [Short Description]
**Symptom:** What the operator sees — error messages, unexpected behavior, failure modes.
**Cause:** Why it happens — firmware bug, race condition, removed feature, API change.
**Workaround:** Exact steps to resolve or avoid the issue. Use numbered steps for multi-step procedures.
**Affected versions:** firmware x.y.z through x.y.w  (or "all versions before x.y.w")
**Status:** Open | Fixed in x.y.w | Vendor advisory: <ID>
```

**Rules:**
- "Affected versions" must name a specific range or state "all known versions".
- "Status" must be one of: `Open`, `Fixed in <version>`, or `Vendor advisory: <ID>`.
- Do not leave any field blank — write "Unknown" if the information is not available.
- Link to the HPE advisory or community thread in the entry body when available.

---

## How to Search

```bash
# Search all known-issues files for a keyword
grep -r "virtual media" skills/reference/known-issues/

# Search for a specific firmware version
grep -r "3\.06\|3\.05" skills/reference/known-issues/

# Search for open issues only
grep -r "Status: Open" skills/reference/known-issues/
```

---

## Adding New Issues

1. Identify the correct file by component: iLO firmware → `ilo5.md` or `ilo6.md`; platform/BIOS/hardware → `gen10.md` or `gen11.md`.
2. Add the entry at the top of the relevant version section (newest first).
3. Include the date discovered and source (HPE advisory ID, community link, or internal ticket).
4. Update the `Status` field when a fix is released.

---

## Status Legend

| Status | Meaning |
|--------|---------|
| `Open` | No fix available; workaround only |
| `Fixed in x.y.w` | Resolved by upgrading to that firmware version |
| `Vendor advisory: <ID>` | HPE has published a formal advisory; check the linked document |
| `Won't fix` | HPE has closed the issue without a fix; workaround is permanent |
