# Virtual Machines

## Prerequisites

Before using the VM node types you should be familiar with the general plugin concepts described in the [Edge Plugin Overview](overview.md). You will also need:

- A NativeEdge environment with at least one registered endpoint (ECE).
- The endpoint's **service tag** (used as the `location` property).
- A VM image already uploaded to the artifact catalog, or an external image URL. See [Binary Images](binary-images.md) for uploading images.

## Node Types

The edge plugin provides two node types for virtual machine management:

| Node Type | Description |
|---|---|
| `dell.nodes.nativeedge.template.NativeEdgeVM` | Full VM provisioning from an image reference with complete control over hardware, networking, cloud-init, and passthrough devices. |
| `dell.nodes.nativeedge.template.ApplicationVM` | Derives from `NativeEdgeVM` but deploys a VM from a pre-built **VM template** instead of a raw image. Uses a simplified property set. |

---

## NativeEdgeVM

**Full type name:** `dell.nodes.nativeedge.template.NativeEdgeVM`
**Derived from:** `dell.nodes.Root`

This is the primary node type for provisioning virtual machines on NativeEdge endpoints. It gives you fine-grained control over CPU, memory, storage, networking, passthrough ports, cloud-init configuration, and more.

### Properties

The node type exposes a single top-level property called `vm_config` whose value must conform to the `dell.types.nativeedge.VMDeployDefinition` data type (documented below).

| Property | Type | Required | Description |
|---|---|---|---|
| `vm_config` | `dell.types.nativeedge.VMDeployDefinition` | Yes | The complete VM deployment definition. |

### Lifecycle Operations

All operations live under the `dell.interfaces.lifecycle` interface.

| Operation | Implementation | Description |
|---|---|---|
| `precreate` | `edge.nativeedge_plugin.tasks.precreate` | Validates the VM configuration and prepares internal state before creation. |
| `create` | `edge.nativeedge_plugin.tasks.create` | Submits the VM deployment request to the endpoint. This is an asynchronous operation that retries up to **1000** times while the endpoint provisions the VM. |
| `start` | `edge.nativeedge_plugin.tasks.start` | Powers on the VM if it is currently stopped. |
| `poststart` | `edge.nativeedge_plugin.tasks.poststart` | Runs after the VM has started; typically used to collect runtime attributes such as IP addresses. |
| `stop` | `edge.nativeedge_plugin.tasks.stop` | Gracefully shuts down the VM. |
| `restart` | `edge.nativeedge_plugin.tasks.restart` | Restarts the VM (stop followed by start). |
| `suspend` | `edge.nativeedge_plugin.tasks.suspend` | Suspends the VM, saving its memory state. |
| `resume` | `edge.nativeedge_plugin.tasks.resume` | Resumes a previously suspended VM. |
| `check_drift` | `edge.nativeedge_plugin.tasks.check_drift` | Compares the current VM state on the endpoint with the expected state in the blueprint. Returns drift information if the two differ. |
| `update_config` | `edge.nativeedge_plugin.tasks.update_config` | Stages configuration changes for the VM without applying them yet. |
| `update_apply` | `edge.nativeedge_plugin.tasks.update_apply` | Applies staged configuration changes. Retries up to **360** times while the endpoint processes the update. |
| `delete` | `edge.nativeedge_plugin.tasks.delete` | Destroys the VM and releases all associated resources on the endpoint. |

---

## VMDeployDefinition

**Full type name:** `dell.types.nativeedge.VMDeployDefinition`

This is the main data type that describes everything needed to deploy a VM.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `location` | string | Yes | -- | ECE service tag identifying which endpoint to deploy the VM on. Only single-endpoint deployment is supported. |
| `name` | string | Yes | -- | Name of the VM as it will appear on the endpoint. |
| `retries` | integer | No | `3` | Number of retry attempts for provisioning failures. |
| `image` | string | No | -- | Reference to a catalog / artifact image ID. Typically obtained from a `BinaryImage` node via `get_attribute`. |
| `image_location` | `dell.types.nativeedge.VMImageLocation` | No | -- | Configuration for provisioning a VM using an external image source instead of the catalog. |
| `os_type` | string | Yes | -- | Operating system type (e.g., `linux`, `windows`). |
| `resource_constraints` | `dell.types.nativeedge.VMDeployConstraints` | Yes | -- | CPU, memory, storage, and disk controller settings. |
| `main_disk` | `dell.types.nativeedge.VMMainDisk` | No | -- | Configuration for the VM's main (boot) disk. |
| `hardware_options` | `dell.types.nativeedge.VMDeployHardwareOptions` | No | -- | Hardware security features: vTPM, secure boot, firmware type. |
| `enable_management` | boolean | No | `false` | When `true`, creates a tap interface on the infrastructure management segment. |
| `network_settings` | list of `dell.types.nativeedge.VMDeployNetworkSettings` | No | -- | Virtual NIC definitions (up to 10). |
| `ports` | `dell.types.nativeedge.VMDeployPorts` | No | -- | Passthrough port assignments (USB, serial, GPU, video, PCIe). |
| `cloudinit` | string | No | -- | Cloud-init cloud-config YAML or script. When provided, `ssh_keys` and `custom_parameters` are ignored. |
| `ssh_keys` | string | No | -- | Public SSH key(s) for root access. Ignored if `cloudinit` is specified. |
| `custom_parameters` | dict | No | -- | Key-value pairs passed to the VM. Ignored if `cloudinit` is specified. |
| `iso_files` | list of string | No | -- | Catalog references to ISO files to download before provisioning. |
| `iso_files_ref` | list | No | -- | ISO file references (alternative format). |
| `boot_order` | list | No | -- | Ordered list of boot devices. |
| `additional_disks` | list | No | -- | Additional disk images to download before provisioning. |
| `additional_disks_ref` | list | No | -- | Additional disk references (alternative format). |
| `download_timeout` | integer | No | `0` | Custom download timeout in seconds. `0` means use the system default. |
| `advanced_hypervisor_parameters` | dict | No | -- | Low-level hypervisor tuning parameters (key-value pairs). |
| `deployment_type` | string | No | `"create"` | Deployment type indicator. Default is `create`. |

---

## VMDeployConstraints

**Full type name:** `dell.types.nativeedge.VMDeployConstraints`

Defines the compute resources allocated to the VM.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `cpu` | integer | No | `2` | Number of virtual CPUs. |
| `disk` | string | No | -- | Datastore path on the target ECE. Retrieve available datastores from the inventory service. |
| `memory` | string | No | `2GB` | Memory allocation with unit. Accepted units: KB, MB, GB, TB, PB, EB, ZB, YB. |
| `storage` | string | No | `4GB` | Storage size with unit. Accepted units: KB, MB, GB, TB, PB, EB, ZB, YB. |
| `controller` | string | No | -- | Disk controller type. One of: `VIRTIO`, `SCSI`, `SATA`. |

---

## VMMainDisk

**Full type name:** `dell.types.nativeedge.VMMainDisk`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | string | No | `main_disk` | Unique name identifying the main disk. |
| `disconnect_flag` | boolean | No | `false` | When `true`, disconnects the main disk from the VM (useful for PXE boot scenarios). |

---

## VMDeployHardwareOptions

**Full type name:** `dell.types.nativeedge.VMDeployHardwareOptions`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `vTPM` | boolean | No | `false` | Enable virtual Trusted Platform Module. **Required** for Windows 11. |
| `secure_boot` | boolean | No | `false` | Enable UEFI Secure Boot. **Required** for Windows 11. |
| `firmware_type` | string | No | `"BIOS"` | Firmware type. One of: `BIOS`, `UEFI`. Must be `UEFI` for Windows 11. |

---

## VMDeployNetworkSettings

**Full type name:** `dell.types.nativeedge.VMDeployNetworkSettings`

Each entry in the `network_settings` list defines one virtual NIC (VNIC).

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | string | Yes | -- | Name of the virtual NIC. Use `VNIC1` through `VNIC10`. |
| `segment_name` | string | No | -- | Network segment to attach to. Uses the default segment if omitted. |
| `vm_mac` | string | No | -- | Explicit MAC address assignment. Auto-generated if omitted. |
| `model_type` | string | No | -- | NIC model type. |
| `port_fwd_rules` | list of `dell.types.nativeedge.VMDeployPortforwardRules` | No | -- | Port forwarding rules for this NIC. |

---

## VMDeployPortforwardRules

**Full type name:** `dell.types.nativeedge.VMDeployPortforwardRules`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `service_type` | string | Yes | -- | Service identifier. Common values: `SSH`, `HTTP`, or any custom string. |
| `protocol` | string | Yes | -- | Transport protocol. One of: `TCP`, `UDP`. |
| `host_ip` | string | Yes | -- | IP address on the host side. |
| `host_port` | integer | Yes | -- | Port number on the host side. |
| `vm_ip` | string | No | -- | IP address inside the VM. Typically not available at deployment time. |
| `vm_port` | integer | No | -- | Port number inside the VM. |

---

## VMDeploySerialPort

**Full type name:** `dell.types.nativeedge.VMDeploySerialPort`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `port` | string | Yes | -- | Serial port name, `COM-1` through `COM-10`. |
| `mode` | string | No | -- | Serial port mode. One of: `RS-232`, `RS-422`, `RS-485`. |
| `serial_console` | boolean | No | `true` | Enable console log streaming from the VM (ttyS0) on this serial port. |

---

## VMDeployPorts

**Full type name:** `dell.types.nativeedge.VMDeployPorts`

Groups all passthrough port assignments.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `usb` | list of string | No | `[]` | USB passthrough devices by logical name. Example: `['USB-1']`. |
| `serial_port` | list of `dell.types.nativeedge.VMDeploySerialPort` | No | `[]` | Serial port passthrough. Example: `[{'port': 'COM-1', 'mode': 'RS-232'}]`. |
| `gpu` | list of string | No | `[]` | GPU passthrough by device name. Example: `['NVIDIA Corporation L4']`. |
| `video` | list of string | No | `[]` | Video passthrough. Only accepted value: `['onboarded-controller']`. |
| `pcie` | list of string | No | `[]` | PCIe passthrough by device name. Example: `['SUNIX Co., Ltd. Multiport serial controller']`. |

---

## VMDeployFilesRef

**Full type name:** `dell.types.nativeedge.VMDeployFilesRef`

Reference to an image file for ISO or additional disk attachment.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `filename` | string | No | -- | Name of the file. |
| `name` | string | No | -- | Logical disk name. |
| `controller` | string | No | -- | Virtual disk controller type. One of: `VIRTIO`, `SCSI`, `SATA`. |

---

## VMDeployAdditionalDisksRef

**Full type name:** `dell.types.nativeedge.VMDeployAdditionalDisksRef`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `image` | `dell.types.nativeedge.VMDeployFilesRef` | Yes | -- | Image file reference for this additional disk. |
| `storage` | string | No | -- | Storage size in bytes or with unit (KB, MB, GB, etc.). |
| `disk` | string | No | -- | Datastore path on the target ECE. |

---

## VMImageLocation

**Full type name:** `dell.types.nativeedge.VMImageLocation`

Use this data type when provisioning a VM from an external image source instead of the built-in artifact catalog.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `secret_name` | string | No | `""` | Name of the NativeEdge secret that holds credentials for the image source. |
| `method` | string | No | `"none"` | Authentication method. `"none"` for unauthenticated access. |
| `protocol` | string | No | `""` | Protocol used to retrieve the image (e.g., `https`, `s3`). |
| `resource_path` | string | No | `""` | Path or URL to the image resource. |
| `metadata` | dict | No | -- | Additional metadata key-value pairs for the image location. |

---

## Blueprint Example -- NativeEdgeVM

The following example is drawn from a production-style blueprint. It shows how to define a `NativeEdgeVM` node template with cloud-init, network settings, passthrough ports, and relationships to helper nodes.

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin
  - plugin:utilities-plugin?version=>=3.1.3.0
  - plugin:fabric-plugin?version=>=3.3.2.0

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

  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: { get_environment_capability: ece_service_tag }
        name: { get_sys: [deployment, name] }
        image: { get_attribute: [binary_image, binary_details, extra, artifact_id] }
        os_type: { get_input: os_type }
        resource_constraints:
          cpu: { get_input: vcpus }
          memory: { get_input: memory_size }
          storage: { get_input: os_disk_size }
          disk: { get_input: disk }
          controller: { get_input: disk_controller }
        hardware_options:
          vTPM: { get_input: hardware_options.vTPM }
          secure_boot: { get_input: hardware_options.secure_boot }
          firmware_type: { get_input: hardware_options.firmware_type }
        enable_management: { get_input: enable_management }
        network_settings: { get_attribute: [prepare_config, network_settings] }
        ports:
          usb: { get_input: usb }
          serial_port: { get_attribute: [prepare_config, serial_ports_list] }
          gpu: { get_input: gpu }
          video: { get_input: video }
          pcie: { get_input: pcie }
        cloudinit: { get_attribute: [cloudinit, cloud_config] }
        iso_files: { get_input: iso_files }
        boot_order: { get_input: boot_order }
        additional_disks: { get_input: additional_disks }
    relationships:
      - type: dell.relationships.depends_on
        target: cloudinit
      - type: dell.relationships.depends_on
        target: binary_image
      - type: dell.relationships.depends_on
        target: prepare_config
```

### Explanation

1. **binary_image** -- Uploads the VM disk image to the artifact catalog. See [Binary Images](binary-images.md).
2. **vm** -- The VM node template. Key points:
   - `location` uses `get_environment_capability` to read the endpoint's service tag from the environment profile.
   - `name` is set to the deployment name using `get_sys`.
   - `image` references the artifact ID produced by the `binary_image` node.
   - `resource_constraints` defines CPU, memory, storage, datastore path, and disk controller.
   - `hardware_options` enables Windows 11-compatible hardware features when needed.
   - `network_settings` is built dynamically by a helper node (`prepare_config`).
   - `ports` assigns passthrough devices.
   - `cloudinit` injects a cloud-config generated by a CloudInit helper node.
3. **relationships** ensure that cloud-init config, the binary image, and network preparation all complete before the VM is created.

---

## ApplicationVM

**Full type name:** `dell.nodes.nativeedge.template.ApplicationVM`
**Derived from:** `dell.nodes.nativeedge.template.NativeEdgeVM`

The `ApplicationVM` node type is a specialization of `NativeEdgeVM` for deploying virtual machines from pre-built **VM templates**. Instead of specifying an image, OS type, and full resource configuration, you reference a template ID. The endpoint uses that template as the base configuration and overlays any properties you specify.

### How It Differs from NativeEdgeVM

| Aspect | NativeEdgeVM | ApplicationVM |
|---|---|---|
| Primary property | `vm_config` (VMDeployDefinition) | `app_vm_config` (AppVMDeploy) |
| Image source | `image` or `image_location` field | `template` ID field |
| OS type | Explicitly set via `os_type` | Inherited from the template |
| Disk images, ISO files | Configurable | Not applicable (managed by template) |

### Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `vm_config` | -- | No | Set to `null` by default. Not used for ApplicationVM. |
| `app_vm_config` | `dell.types.nativeedge.AppVMDeploy` | Yes | Template-based VM deployment definition. |

### Lifecycle Operations

ApplicationVM inherits all lifecycle operations from NativeEdgeVM, but overrides two:

| Operation | Implementation | Description |
|---|---|---|
| `precreate` | `edge.nativeedge_plugin.tasks.precreate_app_vm` | Validates the application VM template configuration. |
| `create` | `edge.nativeedge_plugin.tasks.create_app_vm` | Deploys the VM from the specified template. Retries up to **1000** times. |

All other operations (`start`, `stop`, `restart`, `suspend`, `resume`, `check_drift`, `update_config`, `update_apply`, `delete`) are inherited unchanged from NativeEdgeVM.

---

## AppVMDeploy

**Full type name:** `dell.types.nativeedge.AppVMDeploy`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `template` | string | Yes | -- | Template ID to deploy from. |
| `retries` | integer | No | `3` | Number of retry attempts. |
| `download_timeout` | integer | No | `0` | Download timeout in seconds (`0` = system default). |
| `description` | string | Yes | -- | Human-readable description for the deployment. |
| `location` | string | Yes | -- | ECE service tag. |
| `name` | string | Yes | -- | Deployment name. |
| `resource_constraints` | `dell.types.nativeedge.VMDeployConstraints` | Yes | -- | CPU, memory, storage configuration. |
| `hardware_options` | `dell.types.nativeedge.VMDeployHardwareOptions` | No | -- | vTPM, secure boot, firmware type. |
| `network_settings` | list of `dell.types.nativeedge.VMDeployNetworkSettings` | No | -- | Virtual NIC definitions. |
| `ports` | `dell.types.nativeedge.VMDeployPorts` | No | -- | Passthrough port assignments. |
| `cloudinit` | string | No | `""` | Cloud-init configuration. |
| `ssh_keys` | string | No | `""` | Public SSH key(s). Ignored if `cloudinit` is specified. |
| `custom_parameters` | dict | No | -- | Custom key-value pairs. Ignored if `cloudinit` is specified. |
| `advanced_hypervisor_parameters` | dict | No | -- | Low-level hypervisor tuning parameters. |

### ApplicationVM Example

```yaml
node_templates:
  app_vm:
    type: dell.nodes.nativeedge.template.ApplicationVM
    properties:
      app_vm_config:
        template: "my-template-id-123"
        location: { get_environment_capability: ece_service_tag }
        name: { get_sys: [deployment, name] }
        description: "Application VM deployed from template"
        resource_constraints:
          cpu: 4
          memory: 8GB
          storage: 40GB
        network_settings:
          - name: VNIC1
            segment_name: my-segment
        cloudinit: |
          #cloud-config
          hostname: my-app-vm
```
