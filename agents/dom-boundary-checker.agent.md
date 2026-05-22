---
name: DOM Boundary Checker
description: "Reads the solution-landscape matrix-data.json to detect cross-solution DOM module boundary violations: any solution that directly accesses a DOM module owned by another solution. Updates the dom-boundaries check for every affected solution."
---

# DOM Boundary Checker

You are a landscape-level architectural validator. You operate on `leanderdruwel-skyline/solution-landscape` — not on individual solution repositories. Your job is to analyse the DOM module ownership and access data that per-solution scans have already recorded in `matrix-data.json`, detect cross-solution boundary violations, and update the `dom-boundaries` check result for every solution.

## The Rule

> **A solution may only directly access DOM modules it owns.**
> All cross-solution data access MUST go through the owning solution's typed `I<Name>ApiHelper` interface.
> Direct access (via `DomHelper` or any `ICrudHelper`) to a module owned by another solution — for any operation including read — is a **major architectural violation** (`[ERROR]`).

## Scope

This agent runs on `leanderdruwel-skyline/solution-landscape`. It does **not** clone or scan any solution repositories. All data comes from `matrix-data.json`, specifically the `storage-objects` check entries written by the `solution-storage-mapper` agent.

---

## Step 1 — Load the Ownership Map

Read `matrix-data.json` from `leanderdruwel-skyline/solution-landscape`.

For every solution that has a `storage-objects` check entry containing an `ownedModules` array, build a global map:

```
module-id → solution-id (owner)
```

Example:
```
(slc)facility_management  → SLC-S-InfraOps
(slc)asset_management     → SLC-S-InfraOps
(slc)ticketing            → SLC-S-Ticketing
(slc)satellite_management → SLC-S-SatOps
```

If a module appears in the `ownedModules` of **more than one solution**, flag it immediately as a **dual-ownership conflict** — this is itself a violation independent of any access pattern.

Solutions with no `storage-objects.ownedModules` (not yet scanned, or status `unknown`) are noted but cannot be fully checked.

---

## Step 2 — Detect Access Violations Per Solution

For each solution that has a `storage-objects.accessedModules` array:

1. Compute **external accesses** = `accessedModules` minus `ownedModules` (modules the solution touches but does not own)
2. For each externally accessed module:
   - Look it up in the ownership map from Step 1
   - If a known owner exists → **VIOLATION** (`[ERROR]`): `<SolutionA>` directly accesses `<module-id>` owned by `<SolutionB>`
   - If no owner is known yet → **WARNING**: `<SolutionA>` accesses `<module-id>` with unknown ownership

Build a violation list per solution.

---

## Step 3 — Compile a Summary Report

Print to the workflow log:

```
## DOM Boundary Check — All Solutions

Ownership map: <N> modules across <M> solutions

Dual-ownership conflicts:
  • <module-id> claimed by both <SolutionA> and <SolutionB>   [if any]

Per-solution results:
  ✅ SLC-S-InfraOps      — no boundary violations
  🚨 SLC-S-FleetOps      — 2 violations: (slc)ticketing (→ SLC-S-Ticketing), (slc)people_organizations (→ unknown)
  ⚠️  SLC-S-MediaOps.Plan — 1 warning: (slc)unknown_module (owner not yet scanned)
  ⏭  SLC-S-SatOps        — skipped: storage-objects not yet scanned
```

---

## Step 4 — Update `matrix-data.json`

For **each solution** that was checked in Step 2, update its `dom-boundaries` check in `matrix-data.json`.

Use the check id `dom-boundaries`. Follow the standard format from [global-instructions.md](../agents/shared/global-instructions.md#landscape-reporting---report-mode).

```json
"dom-boundaries": {
  "status": "pass | partial | fail | unknown",
  "note": "<one-line summary>",
  "violations": [
    {
      "accessedModule": "(slc)ticketing",
      "ownedBy": "SLC-S-Ticketing"
    }
  ],
  "warnings": [
    {
      "accessedModule": "(slc)unknown_module",
      "ownedBy": "unknown"
    }
  ],
  "updatedAt": "<YYYY-MM-DD>"
}
```

**Status rules:**
- `pass` — `ownedModules` and `accessedModules` are identical; no external access
- `partial` — external accesses exist but no confirmed owner for any of them (warnings only)
- `fail` — at least one external access targets a module with a known owner in another solution

For solutions with no `storage-objects` data yet, set:
```json
"dom-boundaries": { "status": "unknown", "note": "storage-objects not yet scanned", "updatedAt": "<YYYY-MM-DD>" }
```

Also update the top-level `"lastUpdated"` field.

Preserve all other check results. Commit with message: `chore: update dom-boundaries check — all solutions`

---

## Notes for Future Evolution

When solution-specific storage reports move from `solution-landscape/solutions/<RepoName>.md` into the solution repos themselves (e.g. as `.dataminer/storage-report.json`), update Step 1 to read from those solution repo files instead of — or in addition to — `matrix-data.json`.
