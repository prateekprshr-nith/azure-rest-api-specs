# Handoff: EdgeMachine reportedProperties parity changes

**Audience:** another GitHub Copilot instance that will implement the TypeSpec changes.
**Provider:** `Microsoft.AzureStackHCI`
**Primary file to edit:** [models.tsp](models.tsp)
(`specification/azurestackhci/resource-manager/Microsoft.AzureStackHCI/StackHCI/models.tsp`)
**Branch:** `user/pprasher/spec/edgeDevice-edgeMachine`
**Companion analysis (read-only, do NOT edit):** `EdgeDevice-to-EdgeMachine-gap-analysis.md`

---

## 0. How to work

- **You MUST use the `azure-typespec-author` skill** for these edits — every change is to `.tsp`
  files. Do not hand-edit generated OpenAPI/swagger.
- Do **not** modify `EdgeDevice-to-EdgeMachine-gap-analysis.md` or this handoff doc.
- All target models already live in the single file `models.tsp`. All models you need to reference
  **already exist** in that same file (see §6) — you are only adding a handful of properties.
- After editing, run TypeSpec validation/compile and confirm no new errors (see §7).

---

## 1. Objective

Bring four EdgeDevice (HCI) reported subfields into the EdgeMachine reported surface so that
`EdgeMachineReportedProperties` has parity with `HciReportedProperties`. This is **4 property
additions total** across **3 models**, all reusing models that already exist in `models.tsp`.

---

## 2. Scope — do exactly these (and nothing else)

| # | Gap | Action | Target model | Add property |
|---|---|---|---|---|
| 2 | `confidentialVmProfile` | **ADD** | `EdgeMachineReportedProperties` | `confidentialVmProfile?: ConfidentialVmProfile` |
| 3 | `hostNetwork` | **ADD** | `EdgeMachineNetworkProfile` | `hostNetwork?: HciEdgeDeviceHostNetwork` |
| 4 | `sdnProperties` | **ADD** | `EdgeMachineNetworkProfile` | `sdnProperties?: SdnProperties` |
| 5 | `storageProfile.disks[]` | **ADD** | `StorageProfile` | `disks?: EdgeDeviceDisks[]` |

### Explicitly DO NOT implement

| # | Gap | Decision |
|---|---|---|
| 1 | `deviceState` | **Skip** — do not add. |
| 6 | `deviceConfiguration.nicDetails[]` | **Skip** — do not add. |
| 7 | `deviceConfiguration.deviceMetadata` | **Skip** — do not add. |

---

## 3. Design decision: reuse existing models (do not duplicate)

All four additions **reuse models that already exist** in `models.tsp`. Do **not** create new
`EdgeMachine*`-prefixed copies — reference the existing types directly. This keeps the reported
shape identical to EdgeDevice and avoids drift.

> **SDK-naming check (required):** after editing, open
> [client.tsp](client.tsp) and confirm the reused models (`ConfidentialVmProfile`,
> `HciEdgeDeviceHostNetwork` and its subtree, `SdnProperties`, `EdgeDeviceDisks`) still produce
> acceptable SDK names now that they are also reachable from EdgeMachine. If the C#/other SDK names
> read poorly, add `@@clientName(...)` entries in `client.tsp` (do not rename the models
> themselves).

---

## 4. Exact edits

> Each edit shows the **current** model and the **desired** model. Apply via the typespec-author
> skill. Keep all existing decorators/visibility. Every new property needs a `/** ... */` doc
> comment (Azure `documentation-required` rule).

### Gap 2 — add `confidentialVmProfile` to `EdgeMachineReportedProperties`

Insert the new property immediately after the existing `extensionProfile` property.

**Current:**

```tsp
  /**
   * Extension details for edge machine.
   */
  @visibility(Lifecycle.Read)
  extensionProfile?: ExtensionProfile;

  /**
   * Read-only Hyper-V VM inventory reported by the device management extension for VMs visible on this node.
   */
  @visibility(Lifecycle.Read)
  @Azure.ResourceManager.identifiers(#["workloadId"])
  workloadInventory?: EdgeMachineWorkloadInventoryItem[];
```

**Desired:**

```tsp
  /**
   * Extension details for edge machine.
   */
  @visibility(Lifecycle.Read)
  extensionProfile?: ExtensionProfile;

  /**
   * CVM (Confidential VM) support details for the edge machine.
   */
  @visibility(Lifecycle.Read)
  confidentialVmProfile?: ConfidentialVmProfile;

  /**
   * Read-only Hyper-V VM inventory reported by the device management extension for VMs visible on this node.
   */
  @visibility(Lifecycle.Read)
  @Azure.ResourceManager.identifiers(#["workloadId"])
  workloadInventory?: EdgeMachineWorkloadInventoryItem[];
```

### Gaps 3 & 4 — add `hostNetwork` and `sdnProperties` to `EdgeMachineNetworkProfile`

**Current:**

```tsp
/**
 * NetworkProfile of edge machine.
 */
@added(Versions.v2026_05_01_preview)
model EdgeMachineNetworkProfile {
  /**
   * List of Network Interface Card (NIC) Details of edge machine.
   */
  @visibility(Lifecycle.Read)
  @Azure.ResourceManager.identifiers(#["adapterName"])
  nicDetails?: EdgeMachineNicDetail[];

  /**
   * List of switch Details of edge machine.
   */
  @visibility(Lifecycle.Read)
  @Azure.ResourceManager.identifiers(#["switchName"])
  switchDetails?: SwitchDetail[];
}
```

**Desired:**

```tsp
/**
 * NetworkProfile of edge machine.
 */
@added(Versions.v2026_05_01_preview)
model EdgeMachineNetworkProfile {
  /**
   * List of Network Interface Card (NIC) Details of edge machine.
   */
  @visibility(Lifecycle.Read)
  @Azure.ResourceManager.identifiers(#["adapterName"])
  nicDetails?: EdgeMachineNicDetail[];

  /**
   * List of switch Details of edge machine.
   */
  @visibility(Lifecycle.Read)
  @Azure.ResourceManager.identifiers(#["switchName"])
  switchDetails?: SwitchDetail[];

  /**
   * HostNetwork configuration reported for the edge machine.
   */
  @visibility(Lifecycle.Read)
  hostNetwork?: HciEdgeDeviceHostNetwork;

  /**
   * Software Defined Networking (SDN) properties reported for the edge machine.
   */
  @visibility(Lifecycle.Read)
  sdnProperties?: SdnProperties;
}
```

### Gap 5 — add `disks` to `StorageProfile`

**Current:**

```tsp
/**
 * StorageProfile of edge machine.
 */
@added(Versions.v2026_05_01_preview)
model StorageProfile {
  /**
   * Number of storage disks in the device with $CanPool as true.
   */
  @visibility(Lifecycle.Read)
  poolableDisksCount?: int64;
}
```

**Desired:**

```tsp
/**
 * StorageProfile of edge machine.
 */
@added(Versions.v2026_05_01_preview)
model StorageProfile {
  /**
   * Number of storage disks in the device with $CanPool as true.
   */
  @visibility(Lifecycle.Read)
  poolableDisksCount?: int64;

  /**
   * List of storage disks on the edge machine.
   */
  @visibility(Lifecycle.Read)
  @Azure.ResourceManager.identifiers(#["id"])
  disks?: EdgeDeviceDisks[];
}
```

> `EdgeDeviceDisks` has a required `id: string`; the `@identifiers(#["id"])` array key matches it.
> Either `@Azure.ResourceManager.identifiers(...)` or the bare `@identifiers(...)` alias resolves
> in this file — use `@Azure.ResourceManager.identifiers(...)` to match the surrounding EdgeMachine
> models.

---

## 5. Result after edits (for verification)

`EdgeMachineReportedProperties` should read: `lastUpdated`, `networkProfile`, `osProfile`,
`hardwareProfile`, `storageProfile`, `sbeDeploymentPackageInfo`, `extensionProfile`,
**`confidentialVmProfile`**, `workloadInventory`, `workloadInventoryLastUpdated`.

`EdgeMachineNetworkProfile` should read: `nicDetails`, `switchDetails`, **`hostNetwork`**,
**`sdnProperties`**.

`StorageProfile` should read: `poolableDisksCount`, **`disks`**.

---

## 6. Reused models — confirm they already exist (no need to define)

All are already declared in `models.tsp`; just reference them.

| Model | Notes / version |
|---|---|
| `ConfidentialVmProfile` | `@added(Versions.v2026_05_01_preview)`; contains `igvmStatus?: IgvmStatus`, `statusDetails?: IgvmStatusDetail[]`. |
| `IgvmStatusDetail`, `IgvmStatus` | Already defined (v2026_05_01_preview). |
| `HciEdgeDeviceHostNetwork` | Un-versioned (available all versions). Subtree: `HciEdgeDeviceIntents`, `HciEdgeDeviceVirtualSwitchConfigurationOverrides`, `QosPolicyOverrides`, `HciEdgeDeviceAdapterPropertyOverrides`, `HciEdgeDeviceStorageNetworks`, `HciEdgeDeviceStorageAdapterIPInfo` — all already defined. |
| `SdnProperties`, `SdnStatus` | `@added(Versions.v2026_05_01_preview)`. |
| `EdgeDeviceDisks` | `@added(Versions.v2026_04_30)`; fields `id` (required), `sizeInBytes`, `type`, `model`, `manufacturer`, `isSupported`. |

All reused models are available at `v2026_05_01_preview` (the version EdgeMachine is introduced in),
so no versioning conflicts.

---

## 7. Versioning & breaking-change notes

- EdgeMachine and all three target models are `@added(Versions.v2026_05_01_preview)` — they are new
  in that (preview) version. Adding optional, read-only properties **within the same preview version**
  is non-breaking, so **no new API version and no `@added` on the new properties is required**.
- **If `v2026_05_01_preview` has already been published/shipped**, do not mutate it: instead target
  the next preview version (add the properties with an `@added(Versions.<next-preview>)` decorator).
  Confirm the current released version before applying.

---

## 8. Validation & acceptance criteria

1. Run the typespec-author skill's validation (or `npx tsp compile .` in the StackHCI TypeSpec
   directory). **No new compile/lint errors.**
2. Confirm the generated OpenAPI (`stable`/`preview` swagger for `v2026_05_01_preview`) now shows
   `confidentialVmProfile`, `hostNetwork`, `sdnProperties` under the machine's reported properties and
   `disks` under `StorageProfile`.
3. Check `client.tsp` per §3 (SDK naming).
4. Gaps 1, 6, 7 are **not** implemented.
5. `EdgeDevice-to-EdgeMachine-gap-analysis.md` is unchanged.

---

## 9. Quick checklist

- [ ] Gap 2 — `confidentialVmProfile?: ConfidentialVmProfile` added to `EdgeMachineReportedProperties`
- [ ] Gap 3 — `hostNetwork?: HciEdgeDeviceHostNetwork` added to `EdgeMachineNetworkProfile`
- [ ] Gap 4 — `sdnProperties?: SdnProperties` added to `EdgeMachineNetworkProfile`
- [ ] Gap 5 — `disks?: EdgeDeviceDisks[]` added to `StorageProfile`
- [ ] Doc comments on all four new properties
- [ ] TypeSpec compiles clean
- [ ] `client.tsp` SDK-naming reviewed
- [ ] Gaps 1/6/7 left untouched; report left untouched
