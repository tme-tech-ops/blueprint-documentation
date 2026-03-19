# Resource Management

The Utilities Plugin provides three node types for managing resources:

- **`dell.nodes.resources.List`** -- Manages a pool of resources that can be reserved and returned. Implements a resource allocation pattern.
- **`dell.nodes.resources.ListItem`** -- Represents a single item that can be associated with a resource list.
- **`dell.nodes.resources.ModifiedIso`** -- Creates modified ISO images by adding new files and directories to an existing ISO file.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.

---

## Node Type: dell.nodes.resources.List

**Derived from:** `dell.nodes.Root`

This node type implements a resource pool pattern. You define a list of resources (e.g., IP addresses, port numbers, hostnames, or any other values), and other nodes can reserve items from the list during deployment. When a node is deleted, its reserved item is returned to the pool, making it available for future use.

This pattern is commonly used for:
- Allocating IP addresses from a predefined range
- Assigning unique identifiers to deployed instances
- Managing limited resources that need to be shared across multiple deployments or node instances

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `resource_config` | list | No | `[]` | A list of available resources. Each element in the list is one allocatable resource. The elements can be strings, numbers, dictionaries, or any other YAML-compatible value. |

### Runtime Properties

| Runtime Property | Type | Description |
|---|---|---|
| `resource_config` | list | The full list of resources as defined in the node property. This is a copy of the property stored at runtime for reference. |
| `free_resources` | list | Resources that have not been assigned to any node. Initially, this is the same as `resource_config`. As resources are reserved, they move from this list to `reservations`. |
| `reservations` | dict | A dictionary of resources that have been assigned. The keys are reservation IDs (typically node instance IDs) and the values are the reserved resources. |
| `reservation` | string | The most recently queued reservation. This is a temporary holding property used during the reservation process. |

### Lifecycle Operations

| Operation | Interface | Implementation | Description |
|---|---|---|---|
| `create` | `dell.interfaces.lifecycle` | `resources.plugins_resources.tasks.create_list` | Initializes the resource list. Populates `resource_config` and `free_resources` runtime properties with the values from the node property. |
| `delete` | `dell.interfaces.lifecycle` | `resources.plugins_resources.tasks.delete_list` | Cleans up the resource list. |
| `check_drift` | `dell.interfaces.lifecycle` | `resources.plugins_resources.tasks.check_drift_for_list` | Checks whether the resource list state is consistent (e.g., no resources are missing from both free and reserved lists). |
| `update` | `dell.interfaces.lifecycle` | `resources.plugins_resources.tasks.create_list` | Re-initializes the resource list. Uses the same implementation as `create`. |

### Custom Operations

In addition to the standard lifecycle, the List node provides operations for resource reservation:

| Operation | Interface | Implementation | Inputs | Description |
|---|---|---|---|---|
| `reserve` | `dell.interfaces.operations` | `resources.plugins_resources.tasks.reserve_list_item` | `reservation_id` (string, default `''`) | Reserves one resource from the `free_resources` list and moves it to `reservations`. If `reservation_id` is provided, it is used as the key in the reservations dictionary. If empty, a default ID is generated. |
| `return` | `dell.interfaces.operations` | `resources.plugins_resources.tasks.return_list_item` | `reservation_id` (string, default `''`) | Returns a previously reserved resource back to the `free_resources` list. The `reservation_id` must match the ID used when the resource was reserved. |

### The Reserve/Return Pattern

Here is how resource allocation works:

1. **Initialization:** On `create`, all resources in `resource_config` are placed in `free_resources`. The `reservations` dictionary is empty.

2. **Reservation:** When a node calls the `reserve` operation (typically through the `dell.relationships.resources.reserve_list_item` relationship), one item is removed from `free_resources` and added to `reservations` under the requesting node's instance ID.

3. **Return:** When the relationship is unlinked (during uninstall), the `return` operation moves the resource back from `reservations` to `free_resources`.

This ensures that each resource is assigned to at most one consumer at a time.

### Example: IP Address Pool

```yaml
node_templates:
  ip_pool:
    type: dell.nodes.resources.List
    properties:
      resource_config:
        - "10.0.1.10"
        - "10.0.1.11"
        - "10.0.1.12"
        - "10.0.1.13"
        - "10.0.1.14"

  server:
    type: dell.nodes.Root
    relationships:
      - type: dell.relationships.resources.reserve_list_item
        target: ip_pool
```

When `server` is created, it automatically reserves an IP address from `ip_pool`. The reserved IP is available on the `server` node's runtime properties. When `server` is deleted, the IP is returned to the pool.

---

## Node Type: dell.nodes.resources.ListItem

**Derived from:** `dell.nodes.Root`

Represents a single item within a resource list management system. This node type is used as a companion to `dell.nodes.resources.List` and provides lifecycle operations for individual resource items.

### Lifecycle Operations

| Operation | Implementation | Description |
|---|---|---|
| `create` | `resources.plugins_resources.tasks.create_list_item` | Creates and registers the list item. |
| `delete` | `resources.plugins_resources.tasks.delete_list_item` | Removes and unregisters the list item. |
| `check_drift` | `resources.plugins_resources.tasks.return_drift_true` | Always returns that drift exists. This is a conservative approach that flags the item for review. |
| `update` | `resources.plugins_resources.tasks.always_return_true` | Always returns true, indicating the update was accepted. |

---

## Node Type: dell.nodes.resources.ModifiedIso

**Derived from:** `dell.nodes.Root`

This node type creates modified ISO images by adding new files and directories to an existing ISO file. This is useful for creating custom installation media, seed ISOs for cloud-init, or any scenario where you need to inject files into an ISO image.

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `iso_path` | string | Yes | (none) | The file system path to the original ISO file on the NativeEdge manager. This is the source ISO that will be modified. |
| `output_iso_path` | string | No | (none) | The file system path where the modified ISO should be written. If not provided, the plugin determines an appropriate output path. |
| `new_directories` | list | No | `[]` | A list of new directories to add to the ISO filesystem. Each entry is a dictionary with keys that depend on the ISO format. See the directory specification below. |
| `new_files` | list | No | `[]` | A list of new files to add to the ISO filesystem. Each entry is a dictionary with keys that depend on the ISO format. See the file specification below. |

### Directory Specification

Each entry in `new_directories` must include the following keys (depending on the ISO format):

| Key | Required | Description |
|---|---|---|
| `iso_path` | Yes | The path within the ISO filesystem where the directory should be created. |
| `rr_name` | Rock Ridge only | The Rock Ridge name for the directory (used in Rock Ridge extensions). |
| `joliet_path` | Joliet only | The Joliet path for the directory (used in Joliet extensions). |
| `file_mode` | Rock Ridge only | The file mode (permissions) for the directory in Rock Ridge format. |
| `udf_path` | UDF only | The UDF path for the directory (used in UDF filesystems). |

### File Specification

Each entry in `new_files` must include the following keys:

| Key | Required | Description |
|---|---|---|
| `iso_path` | Yes | The path within the ISO filesystem where the file should be created. |
| `file_context` | Yes | The content of the file to add (note: the plugin uses "file_context" to mean "file content"). |
| `rr_name` | Rock Ridge only | The Rock Ridge name for the file. |
| `joliet_path` | Joliet only | The Joliet path for the file. |
| `file_mode` | Rock Ridge only | The file mode (permissions) for the file. |
| `udf_path` | UDF only | The UDF path for the file. |

### Runtime Properties

| Runtime Property | Type | Description |
|---|---|---|
| `modified_iso_path` | string | The file system path to the generated modified ISO file. |
| `iso_path` | string | The path to the original source ISO. |
| `output_iso_path` | string | The path to the output ISO. |
| `new_directories` | list | The directories that were added. |
| `new_files` | list | The files that were added. |

### Lifecycle Operations

| Operation | Implementation | Description |
|---|---|---|
| `create` | `iso.plugins_iso.tasks.modify_iso` | Reads the original ISO, adds the specified directories and files, and writes the modified ISO to the output path. |
| `delete` | `iso.plugins_iso.tasks.delete_iso` | Deletes the modified ISO file from the filesystem. |
| `check_drift` | `iso.plugins_iso.tasks.check_drift` | Checks whether the modified ISO still exists and matches the expected configuration. |
| `update` | `iso.plugins_iso.tasks.modify_iso` | Regenerates the modified ISO. Uses the same implementation as `create`. |

### Example: Creating a Seed ISO for Cloud-Init

```yaml
node_templates:
  seed_iso:
    type: dell.nodes.resources.ModifiedIso
    properties:
      iso_path: /opt/isos/base_seed.iso
      output_iso_path: /opt/isos/custom_seed.iso
      new_files:
        - iso_path: /user-data
          file_context: |
            #cloud-config
            hostname: my-server
            users:
              - name: admin
                sudo: ALL=(ALL) NOPASSWD:ALL
          rr_name: user-data
        - iso_path: /meta-data
          file_context: |
            instance-id: my-server-001
            local-hostname: my-server
          rr_name: meta-data
```

This creates a modified ISO that contains cloud-init `user-data` and `meta-data` files, which can be attached to a VM as a seed ISO for first-boot initialization.
