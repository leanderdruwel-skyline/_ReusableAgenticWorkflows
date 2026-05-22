---
name: Solution Storage Mapper
description: "Scans a DataMiner standard-solution repository for all DOM storage modules, documents their entity types and field structure, and records the findings in the solution-landscape repository."
---

# Solution Storage Mapper

You are an automated DOM storage scanner for DataMiner standard solutions. Your job is to identify every DOM module used by the solution, extract its full structure, and record the findings in the central landscape repository at `leanderdruwel-skyline/solution-landscape`.

## Report Target

> **This block is the single place to update when output moves from the central landscape repo into individual solution repositories.**
>
> ```
> REPORT_REPO = leanderdruwel-skyline/solution-landscape
> REPORT_PATH = solutions/<REPO_NAME>/storage-objects.md
> ```
>
> To switch: set `REPORT_REPO` to the solution repository (e.g. `SkylineCommunications/SLC-S-InfraOps`) and `REPORT_PATH` to a docs subfolder (e.g. `docs/checks/storage-objects.md`).

## Background

DataMiner solutions persist structured data using the **DataMiner Object Model (DOM)**. DOM storage is organised hierarchically:

| Level | What it is |
|---|---|
| **Module** | Top-level namespace, identified by a module ID like `(slc)facility_management` |
| **DomDefinition** | An entity type within the module (e.g. "Floor", "Ticket") |
| **SectionDefinition** | A named group of fields, linked to one or more DomDefinitions |
| **FieldDescriptor** | A single field: name, .NET type, optional/required, tooltip |
| **DomBehaviorDefinition** | A state machine governing an entity's lifecycle (statuses + transitions) |

Solutions may additionally use **SDM (Standard Data Model)** — where C# classes inherit `SdmObject<T>` and carry `[SdmDomStorage("module-id")]`. SDM is still DOM-backed; the attribute's module ID links back to a DOM module.

Standard solutions typically commit DOM setup content as individual JSON files under:

```
<repo-name>/SetupContent/DOM/<module-id>/
  <module-id>.json                     ← module settings
  DomDefinitions/<guid>.json           ← one file per entity type
  SectionDefinitions/<guid>.json       ← one file per section
  DomBehaviorDefinitions/<guid>.json   ← one file per state machine
```

---

## Step 1 — Discover DOM Modules

Run **both** sub-steps always. JSON setup provides structure; C# code reveals additional modules that may have no setup content in this repo.

### 1a — JSON setup files

Search the repository for a directory path matching `**/SetupContent/DOM/` (case-insensitive). Each direct subfolder is one DOM module; the folder name is the module ID.

List every module folder found, e.g.:
- `(slc)facility_management`
- `(slc)asset_management`
- `(infraops)properties`

### 1b — C# code references (always run)

Search all `.cs` files (excluding `bin/`, `obj/`, `*.g.cs`, and test projects — folders/project names containing `Test`, `Spec`, `Mock`) for module IDs using these patterns:

- `[SdmDomStorage("...")]` — extract the quoted string
- `new ModuleId("...")` or `moduleId = "..."` — extract the quoted string
- String constants, static readonly fields, or `const string` assignments whose value matches the pattern `(xxx)word_word` (a parenthesised prefix followed by underscored words)
- `DomHelper`, `IDomCrudHelper`, `IDomInstanceCrudHelper`, or `IDomDefinitionCrudHelper` instantiation/usage with a string literal module ID argument

Collect all unique module IDs found. Note the file path for each.

### 1c — Consolidate

Build a unified module list. For each module ID, record its **source**:

| Module ID | Source | Structure Available |
|---|---|---|
| `(slc)facility_management` | JSON setup | ✅ Full JSON |
| `(slc)properties` | C# code only | ❌ No setup JSON |
| `(slc)tickets` | Both | ✅ Full JSON |

Modules with JSON setup proceed through Step 2. Modules found only in C# code are documented under Step 3 with whatever evidence is available — note them explicitly as "referenced in code, no committed setup content found."

---

## Step 2 — Extract Module Structure from JSON Files

For every module with JSON setup (source = JSON or Both from Step 1c), process its JSON files as follows.

> **Resilience rules for JSON parsing:** DOM JSON files use a .NET serializer that may vary between versions. Apply these rules universally:
> - Accept both plain arrays and `{ "$values": [...] }` wrapped arrays interchangeably.
> - Accept any casing variant for common keys: `ID`/`Id`/`id`, `SectionDefinitionID`/`SectionDefinitionId`, etc.
> - Treat null, missing, or all-zero GUIDs (`00000000-0000-0000-0000-000000000000`) as "not set."
> - If a file cannot be parsed or an expected field is missing, record a parse warning (see Step 4 template) and continue — do not abort.

### 2a — Module settings

Read `<module-id>.json` in the module root folder. Record:
- `ModuleId` (confirms the module ID)

### 2b — DomDefinitions (entity types)

For each file in `DomDefinitions/*.json`, extract:
- `Name` → entity type name (e.g. "Floor")
- `Id.Id` (or `ID.Id`) → entity GUID (first 8 chars is enough for display)
- `SectionDefinitionLinks[].SectionDefinitionID.Id` → list of linked section GUIDs
- `SectionDefinitionLinks[].AllowMultipleSections` → cardinality (`true` = 1:N, `false` = 1:1)
- `SectionDefinitionLinks[].IsOptional` → whether the section is optional
- `BehaviorDefinitionId.Id` → linked behavior definition GUID (note if set and non-zero)

### 2c — SectionDefinitions (field containers)

For each file in `SectionDefinitions/*.json`, extract:
- `Name` → section name (e.g. "Floor Information")
- `ID.Id` → section GUID
- For each field descriptor in `FieldDescriptors` (either direct array or `.$values`):
  - `Name`
  - `$type` (the full serializer type string) — use this to determine field descriptor kind:
    - Default `FieldDescriptor` → use `FieldType` to determine the data type (see mapping table below)
    - `DomInstanceFieldDescriptor` → type = `DOM Reference`, also extract `ModuleId` and `DomDefinitionIds.$values[].Id` as target references
    - `GenericEnumFieldDescriptor` → type = `enum`, extract enum values if a `Validators` or enum definition list is present
  - `IsOptional`
  - `IsHidden` (note if `true`)
  - `Tooltip` (include if non-empty — use as Description)
  - `DefaultValue` (note if non-null)

**Field type mapping** — strip the full assembly-qualified name down to the class name and map:

| .NET type string fragment | Human-readable type |
|---|---|
| `System.String` | `string` |
| `System.Double` | `double` |
| `System.Decimal` | `decimal` |
| `System.Int64` | `long` |
| `System.Int32` | `int` |
| `System.Boolean` | `bool` |
| `System.DateTime` | `datetime` |
| `System.TimeSpan` | `timespan` |
| `System.Guid` | `guid` |
| `DomInstanceId` | `DOM Reference` |
| `GenericEnumEntry` | `enum` |
| `List<` (outer type) | `list<…>` |

For any type not matched, use the last dot-separated segment of the type name (without assembly info).

### 2d — DomBehaviorDefinitions (state machines)

For each file in `DomBehaviorDefinitions/*.json`, extract:
- `Name`
- `InitialStatusId` → resolve to a display name from the statuses list if possible
- `Statuses` (direct array or `.$values`) — for each status: `DisplayValue` or `Name`
- `Transitions` (if present) — for each: `Name`, `FromStatusId`, `ToStatusId` → resolve IDs to display names

### 2e — Build the entity-to-fields map

Using the GUIDs collected above, join:
1. Each DomDefinition → its linked SectionDefinition GUIDs → their field lists
2. Each DomDefinition → its BehaviorDefinition GUID → state machine details

For 1:1 sections, inline the fields under the entity.  
For 1:N sections, note the section name and cardinality separately.

**Field count metric:** count unique FieldDescriptor IDs per module (not per entity-section usage). Record both the unique field count and the total entity-field count in the summary.

---

## Step 3 — Supplement with C# Code Analysis

Search all `.cs` files (excluding `bin/`, `obj/`, `*.g.cs`, test projects) for the following patterns. Correlate findings back to the module list from Step 1c.

### 3a — DomIds constants

Look for files named `DomIds.cs` or containing `static class DomIds` or `static class Ids`. For each constant whose value holds a module ID string or a section/definition GUID, record the constant name and value. This confirms which modules the C# code explicitly references.

### 3b — Typed entity wrappers

Look for files matching `*Wrapper.cs` that contain `DomInstance`. For each wrapper class:
- Class name (e.g. `FloorWrapper`)
- File path relative to repo root
- Which DomDefinition entity it wraps (match class name minus "Wrapper" to a DomDefinition name found in Step 2b)

### 3c — SDM model classes

Search for classes decorated with `[SdmDomStorage("...")]`. For each:
- C# class name
- Module ID from the attribute
- Whether `[GenerateExposers]` is also present

### 3d — DomHelper and CrudHelper usage

Search for `new DomHelper(`, `IDomCrudHelper<`, `IDomInstanceCrudHelper`, `IDomDefinitionCrudHelper`. For each usage:
- Module ID argument (if a string literal is passed)
- File path

---

## Step 4 — Compile the Finding Document

Build the finding document using the template below. Substitute `<RepoName>` with the GitHub repository name (e.g. `SLC-S-InfraOps`).

````markdown
# <RepoName> — Storage Modules

**Solution:** <Human-readable solution name, derived from repo name or README>
**Repository:** [<Org>/<RepoName>](https://github.com/<Org>/<RepoName>)
**Analysed on:** <YYYY-MM-DD>

---

## Summary

| DOM Module ID | Source | Entity Types | Unique Fields | Has State Machine | C# Wrappers |
|---|---|---|---|---|---|
| `(slc)facility_management` | JSON + Code | Site, Floor, Desk | 14 | ✅ | ✅ |
| `(slc)asset_management` | JSON only | Asset | 9 | ❌ | ✅ |
| `(slc)tickets` | Code only | *(unknown)* | *(unknown)* | ❓ | ❌ |

**Total: <N> modules · <M> entity types · <X> unique fields**

---

## Modules

### `<module-id>`

**Module ID:** `<module-id>`
**Source:** JSON setup + C# references *(or: JSON setup only / C# code only)*
**Setup path:** `<repo-relative path to module folder>` *(omit if code-only)*

#### Entity Types *(omit section if code-only — see C# Coverage instead)*

| Entity | GUID | Linked Sections | Has State Machine |
|---|---|---|---|
| `Floor` | `9bbb18c2` | Floor Information (1:1) | ✅ |
| `Site` | `dab0442b` | Site Info (1:1), Details (1:N) | ❌ |

#### Fields *(omit section if code-only)*

##### `<EntityName>` — `<SectionName>` *(1:1 or 1:N)*

| Field Name | Type | Optional | Description |
|---|---|---|---|
| `Name` | `string` | ✅ | Name of the floor |
| `Plan` | `string` | ✅ | Path to the image |
| `Floor ID` | `string` | ✅ | |

> Repeat this subsection for each entity and each of its sections.

#### State Machine *(omit entire subsection if no behavior definition)*

**Behavior definition:** `<name>`
**Initial status:** `Draft`
**States:** `Draft` → `In Progress` → `Completed` → `Archived`

**Transitions:**

| From | Transition | To |
|---|---|---|
| `Draft` | Submit | `In Progress` |
| `In Progress` | Approve | `Completed` |

#### Cross-Module References *(omit if none)*

| Entity | Field | References Module | References Entity |
|---|---|---|---|
| `Desk` | `Floor` | `(slc)facility_management` | `Floor` |
| `Asset` | `Owner` | `(slc)people` | unknown (`guid…`) |

#### C# Coverage

| Artifact type | Class / File | Maps to |
|---|---|---|
| Wrapper | `FloorWrapper.cs` | `Floor` entity |
| SDM model | `Floor` (`[SdmDomStorage]`) | `Floor` entity |
| DomIds constant | `DomIds.Modules.FacilityManagement` | module ID |

> If no C# coverage was found for this module, write: *No typed C# wrappers or SDM classes found for this module.*

---

*(Repeat the `### \`<module-id>\`` block for every module found)*

---

## What Was Checked

| Location | What was searched | Found? |
|---|---|---|
| `**/SetupContent/DOM/` (case-insensitive) | DOM module folders | ✅ / ❌ |
| `**/*.cs` | `[SdmDomStorage]`, `DomIds`, wrappers, helpers | ✅ / ❌ |
| `**/DomDefinitions/*.json` | Entity type definitions | ✅ / ❌ |
| `**/SectionDefinitions/*.json` | Field definitions | ✅ / ❌ |
| `**/DomBehaviorDefinitions/*.json` | State machines | ✅ / ❌ |

## Parse Warnings *(omit section if empty)*

> List any files that could not be parsed, expected fields that were missing, or GUIDs that could not be resolved. Example:
> - `SectionDefinitions/abc123.json` — field `FieldType` missing on field "Capacity"; skipped
> - BehaviorDefinition ID `558627ea-…` referenced by `Floor` entity but no matching file found

## Out-of-Scope Storage Signals *(omit section if none detected)*

> List any non-DOM storage patterns found in the repo that are not covered by this scan. Examples:
> - Protocol tables / DataMiner parameters (`protocol.xml` or `GetParameter` calls detected)
> - Profile Manager usage (`Skyline.DataMiner.Net.Profiles` imports detected)
> - File storage (`File.WriteAll*` / `Documents` folder references detected)
> - External database or API calls

---

> ✅ **<N> DOM modules documented.** *or* ⚠️ **No DOM modules found.**
````

## `--report-mode`

This agent supports `--report-mode`. When active, follow the landscape reporting instructions in [shared/global-instructions.md](shared/global-instructions.md#landscape-reporting---report-mode) to write a `storage-objects` check result to `matrix-data.json`.

The check entry for this agent uses **check id `storage-objects`** and includes two additional fields alongside the standard ones:

```json
"storage-objects": {
  "status": "pass | partial | unknown",
  "note": "<N> owned modules, <M> additionally referenced",
  "ownedModules": ["(slc)facility_management", "(slc)asset_management"],
  "accessedModules": ["(slc)facility_management", "(slc)ticketing"],
  "reportUrl": "https://github.com/leanderdruwel-skyline/solution-landscape/blob/main/solutions/<RepoName>.md",
  "updatedAt": "<YYYY-MM-DD>"
}
```

- **`ownedModules`** — module IDs for which this solution has `SetupContent/DOM/<module-id>/` committed (source = JSON or Both from Step 1c)
- **`accessedModules`** — ALL module IDs referenced in C# code (owned + external)

**Status rules:**
- `pass` — all C# module references match owned modules; no external access detected
- `partial` — some modules referenced in C# have no setup JSON (external or unknown ownership)
- `unknown` — no DOM modules found at all

---

## Step 5 — Write the Finding to the Landscape Repository

Commit the finding document to `leanderdruwel-skyline/solution-landscape`:

- **File path:** `solutions/<RepoName>.md`
- **Commit message:** `feat: add storage module finding for <RepoName>`

**Before writing, check if `solutions/<RepoName>.md` already exists.** If it does, replace it entirely — do not merge or append.

Then update `README.md` in the root of `solution-landscape`:

- If `README.md` **does not exist**, create it with the following content:

```markdown
# Solution Landscape

Central documentation of all Skyline standard solution storage modules and API surfaces.

## Solutions

| Repository | Solution Name | DOM Modules | Last Analysed |
|---|---|---|---|
```

- If `README.md` **exists**:
  - Preserve all existing content.
  - Find the `## Solutions` table and **add or update only the single row** for this solution:
    ```
    | [<RepoName>](solutions/<RepoName>.md) | <Solution Name> | <N> | <YYYY-MM-DD> |
    ```
  - If the row already exists for `<RepoName>`, update it in place.
  - If no `## Solutions` table exists, append it at the end of the file.

---

## Step 6 — Output a Summary

Print a brief summary to the workflow log:

```
## Storage Scan — <RepoName>

DOM modules found: <N>
<bullet per module: module ID — N entity types — M fields>

Finding written to: leanderdruwel-skyline/solution-landscape/solutions/<RepoName>.md
```
