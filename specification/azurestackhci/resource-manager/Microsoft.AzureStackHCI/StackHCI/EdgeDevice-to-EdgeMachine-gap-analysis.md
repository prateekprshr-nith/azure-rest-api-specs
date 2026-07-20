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

Every leaf property in this chain was compared against:

- `EdgeMachine` resource envelope + `EdgeMachineProperties`
- `EdgeMachineReportedProperties`
- EdgeMachine **child resources** (`EdgeMachineNetworkAdapter`, `EdgeMachineDisk`) where
  data was relocated out of the inline property bag.

**Legend:** ✅ covered (same or superset) · ⚠️ present but shape/mutability changed ·
❌ absent everywhere in EdgeMachine.

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

## 5. Property-level comparison (every leaf)

### 5.1 `properties` (`EdgeDeviceProperties` / `HciEdgeDeviceProperties`)

| HCI EdgeDevice property | EdgeMachine | Status |
|---|---|---|
| `provisioningState` (read) | `EdgeMachineProperties.provisioningState` | ✅ same |
| `deviceConfiguration.nicDetails[]` (writable) | → child resource `EdgeMachineNetworkAdapter` | ⚠️ relocated (see §6.1) |
| `deviceConfiguration.deviceMetadata` (string) | — | ❌ **absent** |
| `reportedProperties` (read) | `EdgeMachineProperties.reportedProperties` | see §5.2 |

### 5.2 `properties.reportedProperties` (`ReportedProperties` / `HciReportedProperties`)

| HCI EdgeDevice property | EdgeMachine | Status |
|---|---|---|
| `deviceState` (union) | `connectivityStatus` + `machineState` | ⚠️ partial — see note |
| `extensionProfile.extensions[]` (extensionName, state, errorDetails[].exception, extensionResourceId, typeHandlerVersion, managedBy) | `EdgeMachineReportedProperties.extensionProfile` (same `ExtensionProfile` model) | ✅ identical |
| `lastSyncTimestamp` (read) | `EdgeMachineProperties.lastSyncTimestamp` + `reportedProperties.lastUpdated` | ✅ covered |
| `confidentialVmProfile` (igvmStatus, statusDetails[].code/message) | — | ❌ **absent** |
| `networkProfile.nicDetails[]` (16 fields) | `EdgeMachineNetworkProfile.nicDetails` (`EdgeMachineNicDetail`) | ✅ identical |
| `networkProfile.switchDetails[]` | `EdgeMachineNetworkProfile.switchDetails` (same `SwitchDetail`) | ✅ identical |
| `networkProfile.hostNetwork` | — | ❌ **absent** (see §5.3) |
| `networkProfile.sdnProperties` (sdnStatus, sdnDomainName, sdnApiAddress) | — | ❌ **absent** |
| `osProfile` (bootType, assemblyVersion) | `EdgeMachineReportedProperties.osProfile` (`OsProfile`) | ✅ superset |
| `sbeDeploymentPackageInfo` (code, message, sbeManifest) | same model | ✅ identical |
| `storageProfile.poolableDisksCount` | `StorageProfile.poolableDisksCount` | ✅ same |
| `storageProfile.disks[]` (`EdgeDeviceDisks`) | → child resource `EdgeMachineDisk` | ⚠️ relocated (see §6.2) |
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

## 6. Relocated child-resource diffs (data preserved)

### 6.1 `deviceConfiguration.nicDetails` → `EdgeMachineNetworkAdapter`

Child resource key `networkAdapterName`, segment `networkAdapters`. All 9 EdgeDevice `NicDetail`
fields exist, but four move from **writable** to **read-only**:

| `NicDetail` field | Location in `EdgeMachineNetworkAdapter` | Mutability |
|---|---|---|
| adapterName, ip4Address, subnetMask, defaultGateway, dnsServers | `NetworkAdapterConfiguration` | ✅ writable (unchanged) |
| interfaceDescription, componentId, driverVersion, defaultIsolationId | `NetworkAdapterReportedProperties` | ⚠️ read-only |

EdgeMachine adds writable `ipInterfaceType`, `vlanId`, `interfaceState`, `wifiConfiguration`, and
many read-only reported fields (macAddress, slot, switchName, interfaceSpeed, rdmaCapability, …).

### 6.2 `storageProfile.disks` (`EdgeDeviceDisks`) → `EdgeMachineDisk`

Child resource key `diskName`, segment `disks`. All 6 fields present; disk `id` folds into the
resource identity:

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

## 7. Consolidated findings

### A. Hard absences — nothing in EdgeMachine or its children carries these

1. `deviceConfiguration.deviceMetadata` (string)
2. `reportedProperties.deviceState` states: **Repairing, Draining, InMaintenance, Resuming, Processing**
3. `reportedProperties.confidentialVmProfile` (igvmStatus, statusDetails[].code/message)
4. `reportedProperties.networkProfile.hostNetwork` (intents[], storageNetworks[], storageConnectivitySwitchless, enableStorageAutoIp)
5. `reportedProperties.networkProfile.sdnProperties` (sdnStatus, sdnDomainName, sdnApiAddress)

### B. Relocated — present, but shape / mutability / identity changed

6. writable `deviceConfiguration.nicDetails` → `EdgeMachineNetworkAdapter` (4 fields become read-only)
7. `reportedProperties.storageProfile.disks` → `EdgeMachineDisk` (`id` → resource identity, `type` → `diskType` enum)

### C. Envelope / semantic differences

8. `name` default `"default"` dropped
9. `name` pattern `{3,24}` (no `_`) → `{3,63}` (with `_`)
10. `kind: "HCI"` discriminator → `edgeMachineKind` property (no `HCI` value)
11. extension resource → tracked resource
12. `list` by target scope → `listByResourceGroup` + `listBySubscription`

### D. Covered (same or superset — not gaps)

`provisioningState`, `extensionProfile`, `lastSyncTimestamp`, `networkProfile.nicDetails` (read),
`switchDetails`, `osProfile`, `sbeDeploymentPackageInfo`, `storageProfile.poolableDisksCount`,
`hardwareProfile`, and all `get` / `createOrUpdate` / `delete` / `validate` operations.

---

## 8. Recommended actions to make EdgeMachine a strict superset

| # | Action | Priority |
|---|---|---|
| 1 | Add `deviceMetadata` (string) to an EdgeMachine writable configuration surface | High |
| 2 | Extend `connectivityStatus`/`machineState` (or add a device-state field) to cover Repairing, Draining, InMaintenance, Resuming, Processing | High |
| 3 | Add `confidentialVmProfile` to `EdgeMachineReportedProperties` | High |
| 4 | Add `hostNetwork` to `EdgeMachineNetworkProfile` | High |
| 5 | Add `sdnProperties` to `EdgeMachineNetworkProfile` | High |
| 6 | Confirm the writable-→read-only shift for `interfaceDescription`, `componentId`, `driverVersion`, `defaultIsolationId` is acceptable (relocation to `EdgeMachineNetworkAdapter`) | Medium |
| 7 | Confirm disk `id` → resource identity and `type` → `diskType` mapping is acceptable (relocation to `EdgeMachineDisk`) | Medium |
| 8 | Confirm envelope/semantic changes (name default & pattern, `kind` discriminator, extension→tracked, list semantics) are intentional | Review |

> **Out of scope:** EdgeMachine's net-new capabilities (GPU, updates, volumes, jobs, WiFi, etc.)
> are not "missing from EdgeDevice" and are excluded from this analysis.
