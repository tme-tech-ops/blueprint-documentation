# Licensing

## Prerequisites

Before working with licensing you should be familiar with the general plugin concepts described in the [Edge Plugin Overview](overview.md).

## Overview

The edge plugin provides two node types for managing licenses in the NativeEdge system:

| Node Type | Description |
|---|---|
| `dell.nodes.nativeedge.template.License` | Applies a static or dynamic license to the NativeEdge platform. |
| `dell.nodes.nativeedge.template.RegisterPrivateCloudLicense` | Manages the full lifecycle of a private cloud license: register, expand, reduce, and deregister. |

---

## License

**Full type name:** `dell.nodes.nativeedge.template.License`
**Derived from:** `dell.nodes.Root`

Use this node type to apply a license to the NativeEdge system. The license can be either **static** (a pre-generated license file) or **dynamic** (retrieved automatically from a licensing service).

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `is_dynamic` | boolean | No | `false` | Set to `true` for a dynamic license (fetched from a licensing service at runtime). Set to `false` for a static license file. |
| `license_file` | string | No | `""` | The content of the static license file. **Required** when `is_dynamic` is `false`. Ignored when `is_dynamic` is `true`. |
| `entitlements` | dict | No | `{}` | A dictionary that will be populated with the license application result. Typically left empty at deployment time. |

### Lifecycle Operations

The License node uses the `nativeedge.interfaces.validation` interface (not the standard `dell.interfaces.lifecycle`).

| Operation | Implementation | Description |
|---|---|---|
| `create` | `edge.nativeedge_plugin.tasks.apply_license` | Applies the license to the NativeEdge system. For static licenses, the `license_file` content is submitted. For dynamic licenses, the system contacts the licensing service to obtain and apply the license. The result (entitlements) is stored as a runtime attribute. |

### Blueprint Example -- Static License

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin

node_templates:

  license:
    type: dell.nodes.nativeedge.template.License
    properties:
      is_dynamic: false
      license_file: { get_secret: nativeedge_license }
```

### Blueprint Example -- Dynamic License

```yaml
node_templates:

  license:
    type: dell.nodes.nativeedge.template.License
    properties:
      is_dynamic: true
```

---

## RegisterPrivateCloudLicense

**Full type name:** `dell.nodes.nativeedge.template.RegisterPrivateCloudLicense`
**Derived from:** `dell.nodes.Root`

This node type manages the full lifecycle of a **private cloud license**. A private cloud license is associated with a cluster of NativeEdge endpoints and supports registration, expansion (adding nodes), reduction (removing nodes), and deregistration.

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `cswid` | string | Yes | -- | CSWID (Concise Software Identification) of the cluster assigned during registration. Pass an empty string (`""`) when registering for the first time. |
| `component_name` | string | No | -- | Component name of the outcome endpoint. Optional during initial registration. |
| `outcome` | string | No | -- | Outcome associated with the cluster outcome endpoint. |
| `entitlement_ids` | list of string | No | -- | List of entitlement IDs to consume. **Required** for register, expand, and reduce operations. |
| `service_tags` | list of string | No | -- | List of service tags for the cluster endpoints. **Required** for register, expand, and reduce operations. |
| `license_file` | string | No | -- | Static license file content. **Required** if `entitlement_ids` is not provided. |

### Lifecycle Operations

All operations live under the `dell.interfaces.lifecycle` interface.

| Operation | Implementation | Description |
|---|---|---|
| `create` | `edge.nativeedge_plugin.tasks.register_private_cloud` | Registers a new private cloud cluster with the licensing system. Consumes entitlements and associates the specified service tags with the cluster. |
| `expand` | `edge.nativeedge_plugin.tasks.expand_private_cloud_node` | Adds nodes to an existing private cloud cluster. Requires updated `service_tags` and `entitlement_ids`. |
| `reduce` | `edge.nativeedge_plugin.tasks.reduce_private_cloud_node` | Removes nodes from an existing private cloud cluster. |
| `delete` | `edge.nativeedge_plugin.tasks.deregister_private_cloud` | Deregisters the private cloud cluster from the licensing system, releasing all consumed entitlements. |
| `validate_register` | `edge.nativeedge_plugin.tasks.validate_register_private_cloud` | Validates that the registration parameters are correct before performing the actual registration. |
| `validate_expand` | `edge.nativeedge_plugin.tasks.validate_expand_private_cloud` | Validates that the expansion parameters are correct. |
| `validate_reduce` | `edge.nativeedge_plugin.tasks.validate_reduce_private_cloud` | Validates that the reduction parameters are correct. |

### Lifecycle Flow

A typical private cloud license lifecycle follows these steps:

1. **validate_register** -- Verify the registration inputs.
2. **create** (register) -- Register the cluster and consume entitlements.
3. **expand** -- Add new nodes to the cluster as it grows.
4. **reduce** -- Remove nodes from the cluster when scaling down.
5. **delete** (deregister) -- Remove the cluster registration entirely.

### Blueprint Example

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin

node_templates:

  private_cloud_license:
    type: dell.nodes.nativeedge.template.RegisterPrivateCloudLicense
    properties:
      cswid: ""
      entitlement_ids:
        - { get_input: entitlement_id }
      service_tags:
        - { get_input: service_tag_1 }
        - { get_input: service_tag_2 }
```

### Explanation

1. `cswid` is set to an empty string because this is a new registration.
2. `entitlement_ids` lists the license entitlements to consume.
3. `service_tags` lists the endpoints that will form the private cloud cluster.
4. After the `create` operation runs, the node's runtime attributes will contain the assigned `cswid` for use in subsequent expand or reduce operations.
