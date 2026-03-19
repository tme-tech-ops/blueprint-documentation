# Binary Images

## Prerequisites

Before working with binary images you should be familiar with the general plugin concepts described in the [Edge Plugin Overview](overview.md). You will also need:

- A NativeEdge environment with access to the artifact catalog service.
- The URL (or catalog reference) of the image you want to upload.
- (Optional) Credentials for an external repository if the image is hosted outside the NativeEdge system.

## Overview

A **binary image** is a disk image (such as a QCOW2, VMDK, or ISO file) stored in the NativeEdge artifact catalog. Before you can provision a virtual machine, the image it boots from must exist in the catalog. The `BinaryImage` node type handles uploading images from external repositories, validating configurations, checking for drift, and updating images when new versions become available.

Once a binary image is uploaded, other node types -- most commonly [NativeEdgeVM](virtual-machines.md) -- can reference it by its artifact ID.

---

## Node Type

**Full type name:** `dell.nodes.nativeedge.template.BinaryImage`
**Derived from:** `dell.nodes.Root`

### Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `binary_image_config` | `dell.types.nativeedge.BinaryImageConfig` | Yes | Configuration describing the image to upload, including its source URL, version, and metadata. |

### Lifecycle Operations

All operations live under the `dell.interfaces.lifecycle` interface.

| Operation | Implementation | Description |
|---|---|---|
| `precreate` | `edge.nativeedge_plugin.tasks.validate_binary_image_config` | Validates the binary image configuration. Checks that required fields are present, the URL is reachable, and the artifact type is valid. |
| `create` | `edge.nativeedge_plugin.tasks.upload_binary` | Uploads the binary image to the artifact catalog. This is an asynchronous operation that retries up to **1000** times while the upload and any server-side processing complete. |
| `check_drift` | `edge.nativeedge_plugin.tasks.binary_check_drift` | Compares the current state of the binary image in the catalog with the expected state in the blueprint. Reports drift if the version or other metadata has changed. |
| `update` | `edge.nativeedge_plugin.tasks.update_binary` | Updates the binary image in the catalog (for example, uploading a new version). |

---

## Data Types

### BinaryImageConfig

**Full type name:** `dell.types.nativeedge.BinaryImageConfig`

This is the top-level configuration for the binary image.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `artifact` | `dell.types.nativeedge.ArtifactConfig` | Yes | -- | Source artifact configuration (URL, credentials, type). |
| `version` | string | Yes | -- | Version string for this binary image (e.g., `"1.0.0"`, `"22.04"`). Used for catalog indexing and drift detection. |
| `artifact_type` | string | No | `"VM_IMAGE"` | The type of artifact. Default is `VM_IMAGE`. Other values may be supported depending on your NativeEdge version. |
| `description` | string | No | `""` | Human-readable description of the binary image. |
| `developer` | string | No | `"NativeEdge"` | Name of the developer or publisher. |
| `name` | string | No | `""` | Display name for the binary image. If omitted, the filename from the artifact URL is used. |
| `tags` | list of string | No | `[]` | Tags for organizing and searching binary images in the catalog. |

### ArtifactConfig

**Full type name:** `dell.types.nativeedge.ArtifactConfig`

Describes where to download the image from and how to authenticate.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `path` | string | Yes | -- | URL of the external repository or file location. This is the full download URL for the image. |
| `username` | string | No | `""` | Username for authenticating to the external repository. Leave empty for public URLs. |
| `access_token` | string | No | `""` | Access token or password for the external repository. Typically retrieved from the NativeEdge secret store using `get_secret`. |
| `description` | string | No | `""` | Description of the artifact source. |
| `artifact_type` | string | No | `"EXTERNAL"` | Whether the artifact is stored locally or externally. One of: `LOCAL`, `EXTERNAL`. |

---

## Blueprint Example

The following example shows how to upload a binary image from an external repository and then use it to provision a VM. This pattern is extracted from the VM blueprint example.

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin

node_templates:

  binary_image:
    type: dell.nodes.nativeedge.template.BinaryImage
    properties:
      binary_image_config:
        artifact:
          path: { get_input: binary_image_url }
          username: { get_input: binary_image_access_user }
          access_token: { get_secret: { get_input: binary_image_access_token } }
        version: { get_input: binary_image_version }
    interfaces:
      dell.interfaces.lifecycle:
        precreate:
          implementation:
            edge.nativeedge_plugin.tasks.validate_binary_image_config
        create:
          implementation:
            edge.nativeedge_plugin.tasks.upload_binary
        delete:
          implementation:
            edge.nativeedge_plugin.tasks.delete_binary

  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: { get_environment_capability: ece_service_tag }
        name: { get_sys: [deployment, name] }
        image: { get_attribute: [binary_image, binary_details, extra, artifact_id] }
        os_type: linux
        resource_constraints:
          cpu: 2
          memory: 4GB
          storage: 20GB
    relationships:
      - type: dell.relationships.depends_on
        target: binary_image
```

### Explanation

1. **binary_image** -- Uploads the VM disk image to the catalog:
   - `path` is the download URL for the image file (provided as a deployment input).
   - `username` and `access_token` authenticate to the external repository. The access token is stored as a NativeEdge secret.
   - `version` identifies the image version for catalog management.
   - The blueprint also explicitly maps a `delete` operation to `delete_binary`, which removes the image from the catalog when the deployment is uninstalled.
2. **vm** -- References the uploaded image via `get_attribute`:
   - `{ get_attribute: [binary_image, binary_details, extra, artifact_id] }` retrieves the catalog artifact ID that was set as a runtime attribute after the upload completed.
3. The **relationship** ensures the image upload finishes before the VM creation begins.

---

## Runtime Attributes

After a successful upload, the `BinaryImage` node sets runtime attributes that other nodes can reference:

- `binary_details` -- A dictionary containing image metadata, including:
  - `extra.artifact_id` -- The unique artifact ID in the catalog (used by VM nodes to reference the image).

Access these via `get_attribute`:

```yaml
{ get_attribute: [binary_image, binary_details, extra, artifact_id] }
```

---

## Drift Detection

The `check_drift` operation compares the binary image state in the catalog with what the blueprint expects. Drift can occur when:

- The image version in the catalog has been changed outside of the blueprint.
- The image file has been replaced or deleted.

If drift is detected, you can use the `update` operation to re-upload or reconcile the image.
