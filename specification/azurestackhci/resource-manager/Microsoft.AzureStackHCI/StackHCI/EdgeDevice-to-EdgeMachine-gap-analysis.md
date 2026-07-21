# EdgeDevice (HCI) → EdgeMachine — API Gap Analysis

**Provider:** `Microsoft.AzureStackHCI`
**Scope:** Transitioning the **HCI Edge Device** API surface to the **EdgeMachine** API surface.
**Goal:** Ensure `EdgeMachine` does not lose anything present in the HCI `EdgeDevice`.
**Source of truth:** `models.tsp`, `EdgeDevice.tsp`, `EdgeMachine.tsp`, and the EdgeMachine child-resource files (`EdgeMachineStorage.tsp`, `EdgeMachineNetworkAdapters.tsp`).

---

## 1. Methodology & scope

`EdgeDevice` is an abstract discriminated **extension resource**
(`Azure.ResourceManager.Legacy.DiscriminatedExtensionResource<DeviceKind>`).
The `DeviceKind` union defines exactly **one** variant — `HCI` — and the only model that
`extends EdgeDevice` is **`HciEdgeDevice`**. Therefore this analysis targets the fully
flattened `HciEdgeDevice` inheritance chain:

```
HciEdgeDevice ─▶ properties: HciEdgeDeviceProperties
                   ├── (inherited) EdgeDeviceProperties
                   └── reportedProperties: HciReportedProperties
                         └── (inherited) ReportedProperties
```

**Parity rules (confirmed 2026-07-21):**

1. **Reported parity:** every subfield of `HciReportedProperties` must exist **inside**
   `EdgeMachineReportedProperties`. Placement in a **child resource** (`EdgeMachineDisk`,
   `EdgeMachineNetworkAdapter`, …) or on the `EdgeMachineProperties` top level does **not** satisfy it.
2. **deviceConfiguration parity:** the non-reported, top-level writable `deviceConfiguration`
   (`nicDetails`, `deviceMetadata`) must also exist on **EdgeMachine** directly; its data living on a
   child resource does **not** satisfy it. Tracked as dedicated gaps.
3. `lastSyncTimestamp` → `lastUpdated` **rename is accepted** (not a gap).

**Total: 7 gaps** — 5 in `reportedProperties` (§5.2), 2 in `deviceConfiguration` (§5.1). See §7.

**Legend:** ✅ present / acceptable · ❌ gap (must be added) · ⚠️ differs (context only).

---

## 2. Resource envelope

| Aspect | HCI EdgeDevice | EdgeMachine | Status |
|---|---|---|---|
| Base type | `Legacy.DiscriminatedExtensionResource<DeviceKind>` (extension resource) | `TrackedResource<EdgeMachineProperties>` | ⚠️ scope model differs |
| Polymorphism | Discriminated by `kind` (`"HCI"` → `HciEdgeDevice`) | Not polymorphic; `edgeMachineKind` is a plain property (`Standard`/`Dedicated`) | ⚠️ no `HCI` value |
| Name key / segment | `edgeDeviceName` / `edgeDevices` | `edgeMachineName` / `edgeMachines` | ⚠️ |
| Name default | `"default"` | none | ⚠️ default dropped |
| Name pattern | `^[a-zA-Z0-9-]{3,24}$` | `^[a-zA-Z0-9-_]{3,63}$` | ⚠️ |
| Identity | none | `ManagedServiceIdentityProperty` | ✅ added |
| Location / tags | none (extension) | present (tracked) | ✅ added |

---

## 3. Operations

| EdgeDevice (`EdgeDevices`) | EdgeMachine (`EdgeMachines`) | Status |
|---|---|---|
| `get` — `Extension.Read` | `get` — `ArmResourceRead` | ✅ |
| `createOrUpdate` — `Extension.CreateOrReplaceAsync` | `createOrUpdate` — `ArmResourceCreateOrReplaceAsync` | ✅ |
| `delete` — `Extension.DeleteWithoutOkAsync` | `delete` — `ArmResourceDeleteWithoutOkAsync` | ✅ |
| `list` — `Extension.ListByTarget` (by scope) | `listByResourceGroup` + `listBySubscription` | ⚠️ list semantics differ |
| `validate` — `Extension.ActionAsync` | `validate` — `ArmResourceActionAsync` | ✅ |
| — | `update` — `ArmCustomPatchAsync` (PATCH) | ✅ added |

**No operation is missing.** EdgeMachine is a superset. The only nuance is that EdgeDevice's
`list` enumerates devices attached to a **target scope**, whereas EdgeMachine lists by resource
group / subscription.

---

## 4. Validate models

| EdgeDevice | EdgeMachine | Status |
|---|---|---|
| `ValidateRequest.edgeDeviceIds: string[]` | `EdgeMachineValidateRequest.edgeMachineIds: armResourceIdentifier<[Microsoft.AzureStackHCI/edgeMachines]>[]` | ✅ typed (improvement) |
| `ValidateRequest.additionalInfo?: string` | `EdgeMachineValidateRequest.additionalInfo?: string` | ✅ |
| `ValidateResponse.status?: string` (read) | `EdgeMachineValidateResponse.status?: string` (read) | ✅ |

Equivalent — no field lost.

---

## 5. Property-level comparison

§5.1 covers the top-level `properties` (including the non-reported `deviceConfiguration`); §5.2 covers
`reportedProperties`. Gaps are marked ❌ and numbered to match §7.

### 5.1 `properties` (`EdgeDeviceProperties` / `HciEdgeDeviceProperties`)

| HCI EdgeDevice property | EdgeMachine | Status |
|---|---|---|
| `provisioningState` (read) | `EdgeMachineProperties.provisioningState` | ✅ same |
| `deviceConfiguration.nicDetails[]` (writable) | **no** top-level equivalent on EdgeMachine (data only on `EdgeMachineNetworkAdapter` child — does not satisfy parity) | ❌ gap 6 |
| `deviceConfiguration.deviceMetadata` (string) | **no** equivalent anywhere | ❌ gap 7 |
| `reportedProperties` (read) | `EdgeMachineProperties.reportedProperties` | see §5.2 |

### 5.2 `properties.reportedProperties` (`ReportedProperties` / `HciReportedProperties`)

| HCI EdgeDevice property | EdgeMachine | Status |
|---|---|---|
| `deviceState` (union) | **absent** (`connectivityStatus`/`machineState` are top-level, not reported) | ❌ gap 1 |
| `extensionProfile.extensions[]` (extensionName, state, errorDetails[].exception, extensionResourceId, typeHandlerVersion, managedBy) | `EdgeMachineReportedProperties.extensionProfile` (same `ExtensionProfile` model) | ✅ identical |
| `lastSyncTimestamp` (read) | `EdgeMachineReportedProperties.lastUpdated` (renamed) | ✅ accepted (rename) |
| `confidentialVmProfile` (igvmStatus, statusDetails[].code/message) | **absent** | ❌ gap 2 |
| `networkProfile.nicDetails[]` (16 fields) | `EdgeMachineNetworkProfile.nicDetails` (`EdgeMachineNicDetail`) | ✅ identical |
| `networkProfile.switchDetails[]` | `EdgeMachineNetworkProfile.switchDetails` (same `SwitchDetail`) | ✅ identical |
| `networkProfile.hostNetwork` | **absent** from `EdgeMachineNetworkProfile` | ❌ gap 3 (see §5.3) |
| `networkProfile.sdnProperties` (sdnStatus, sdnDomainName, sdnApiAddress) | **absent** from `EdgeMachineNetworkProfile` | ❌ gap 4 |
| `osProfile` (bootType, assemblyVersion) | `EdgeMachineReportedProperties.osProfile` (`OsProfile`) | ✅ superset |
| `sbeDeploymentPackageInfo` (code, message, sbeManifest) | same model | ✅ identical |
| `storageProfile` (`HciStorageProfile`) | `EdgeMachineReportedProperties.storageProfile` (`StorageProfile`) | ✅ **container present** |
| `storageProfile.poolableDisksCount` | `StorageProfile.poolableDisksCount` | ✅ same |
| `storageProfile.disks[]` (`EdgeDeviceDisks`) | **absent** from `StorageProfile` (only on `EdgeMachineDisk` child) | ❌ gap 5 |
| `hardwareProfile.processorType` | `HardwareProfile.processorType` | ✅ superset |

> **`deviceState` note:** `DeviceState` = `NotSpecified, Connected, Disconnected, Repairing,
> Draining, InMaintenance, Resuming, Processing`. EdgeMachine's `EdgeMachineConnectivityStatus`
> only covers `NotSpecified, Connected, Disconnected`; `EdgeMachineState` describes OS lifecycle,
> not device health. The states **Repairing, Draining, InMaintenance, Resuming, Processing** have
> no equivalent.

### 5.3 `networkProfile.hostNetwork` subtree (all absent in EdgeMachine)

`HciEdgeDeviceHostNetwork`:

- `intents[]` (`HciEdgeDeviceIntents`) — scope, intentType, isComputeIntentSet, isStorageIntentSet,
  isOnlyStorage, isManagementIntentSet, isStretchIntentSet, isOnlyStretch, isNetworkIntentType,
  intentName, intentAdapters, overrideVirtualSwitchConfiguration,
  `virtualSwitchConfigurationOverrides` (enableIov, loadBalancingAlgorithm), overrideQosPolicy,
  `qosPolicyOverrides` (priorityValue8021Action_Cluster, priorityValue8021Action_SMB,
  bandwidthPercentage_SMB), overrideAdapterProperty,
  `adapterPropertyOverrides` (jumboPacket, networkDirect, networkDirectTechnology)
- `storageNetworks[]` (`HciEdgeDeviceStorageNetworks`) — name, networkAdapterName, storageVlanId,
  `storageAdapterIPInfo[]` (physicalNode, ipv4Address, subnetMask)
- `storageConnectivitySwitchless`
- `enableStorageAutoIp`

---

## 6. Where each gap's data currently lives (informational)

> These child-resource diffs show where the data exists today. Per the parity rules this does **not**
> satisfy the requirement — the data must be brought onto EdgeMachine directly (gap 5 → into
> `StorageProfile`; gap 6 → onto EdgeMachine top-level `deviceConfiguration`).

### 6.1 Gap 6 — `deviceConfiguration.nicDetails` (data currently on `EdgeMachineNetworkAdapter`)

Child resource key `networkAdapterName`, segment `networkAdapters`. All 9 EdgeDevice `NicDetail`
fields exist, but four move from **writable** to **read-only**:

| `NicDetail` field | Location in `EdgeMachineNetworkAdapter` | Mutability |
|---|---|---|
| adapterName, ip4Address, subnetMask, defaultGateway, dnsServers | `NetworkAdapterConfiguration` | ✅ writable (unchanged) |
| interfaceDescription, componentId, driverVersion, defaultIsolationId | `NetworkAdapterReportedProperties` | ⚠️ read-only |

EdgeMachine adds writable `ipInterfaceType`, `vlanId`, `interfaceState`, `wifiConfiguration`, and
many read-only reported fields (macAddress, slot, switchName, interfaceSpeed, rdmaCapability, …).

### 6.2 Gap 5 — `storageProfile.disks[]` (data currently on `EdgeMachineDisk`)

`StorageProfile` and its `poolableDisksCount` field **are** present inline in
`EdgeMachineReportedProperties`. Only the inline per-disk `disks[]` **array** is not part of
`StorageProfile`; equivalent (richer) per-disk data is exposed via the `EdgeMachineDisk` child
resource (key `diskName`, segment `disks`). All 6 `EdgeDeviceDisks` fields are present there; disk
`id` folds into the resource identity:

| `EdgeDeviceDisks` field | `EdgeMachineDisk` (`DiskReportedProperties`) | Change |
|---|---|---|
| `id` (required) | resource identity (`diskName` key) — no value field | ⚠️ value → resource name |
| `sizeInBytes` | `sizeInBytes` | ✅ |
| `type` (string) | `diskType` (`DiskType` enum) | ⚠️ rename + enum |
| `model` | `model` | ✅ |
| `manufacturer` | `manufacturer` | ✅ |
| `isSupported` | `isSupported` | ✅ |
| — | + serialNumber, firmwareVersion, busLocation, unallocatedSizeInBytes, state, volumes[] | richer |

---

## 7. Consolidated gap list (authoritative)

**7 gaps** — 5 in `reportedProperties`, 2 in `deviceConfiguration`. Child resources and top-level
`EdgeMachineProperties` fields do **not** satisfy parity.

### reportedProperties — add to `EdgeMachineReportedProperties`

1. `deviceState` (`DeviceState`) — whole field absent; must cover all 8 states incl. `Repairing`, `Draining`, `InMaintenance`, `Resuming`, `Processing`.
2. `confidentialVmProfile` (`ConfidentialVmProfile`: `igvmStatus`, `statusDetails[].code/message`).
3. `networkProfile.hostNetwork` (`HciEdgeDeviceHostNetwork`: `intents[]` + virtualSwitch/QoS/adapter overrides, `storageNetworks[]` + `storageAdapterIPInfo`, `storageConnectivitySwitchless`, `enableStorageAutoIp`) — add to `EdgeMachineNetworkProfile`.
4. `networkProfile.sdnProperties` (`SdnProperties`: `sdnStatus`, `sdnDomainName`, `sdnApiAddress`) — add to `EdgeMachineNetworkProfile`.
5. `storageProfile.disks[]` (`EdgeDeviceDisks`: `id`, `sizeInBytes`, `type`, `model`, `manufacturer`, `isSupported`) — add **inline** to `StorageProfile` (the `EdgeMachineDisk` child does not satisfy parity).

### deviceConfiguration — add to EdgeMachine top-level

6. `deviceConfiguration.nicDetails[]` (writable `NicDetail`) — currently only on the `EdgeMachineNetworkAdapter` child.
7. `deviceConfiguration.deviceMetadata` (string) — no equivalent anywhere.

### Covered — no action

`extensionProfile`, `networkProfile.nicDetails`, `networkProfile.switchDetails`, `osProfile` (superset),
`sbeDeploymentPackageInfo`, `storageProfile` + `poolableDisksCount`, `hardwareProfile` (superset), and
`lastSyncTimestamp` → `lastUpdated` (accepted rename). Operations & validate models: no gaps (§3–§4).

---

## 8. Recommended actions

| # | Gap | Action | Target |
|---|---|---|---|
| 1 | deviceState | Add `deviceState` (`DeviceState`, all 8 states) | `EdgeMachineReportedProperties` |
| 2 | confidentialVmProfile | Add `confidentialVmProfile` (`ConfidentialVmProfile`) | `EdgeMachineReportedProperties` |
| 3 | hostNetwork | Add `hostNetwork` (`HciEdgeDeviceHostNetwork` equivalent) | `EdgeMachineNetworkProfile` |
| 4 | sdnProperties | Add `sdnProperties` (`SdnProperties`) | `EdgeMachineNetworkProfile` |
| 5 | disks | Add `disks[]` (`EdgeDeviceDisks` equivalent) **inline** | `StorageProfile` |
| 6 | nicDetails | Add writable `deviceConfiguration.nicDetails[]` (`NicDetail`) | `EdgeMachine` top-level |
| 7 | deviceMetadata | Add `deviceConfiguration.deviceMetadata` (string) | `EdgeMachine` top-level |

> **Accepted (no action):** `lastSyncTimestamp` → `lastUpdated` rename.
> **Out of scope:** EdgeMachine net-new capabilities (GPU, updates, volumes, jobs, WiFi, etc.).
