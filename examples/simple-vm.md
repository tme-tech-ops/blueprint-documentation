# Simple VM Example

This is the most important example in the collection. It demonstrates the complete lifecycle of provisioning a virtual machine on a NativeEdge endpoint -- from generating SSH keys and uploading an OS image, through cloud-init configuration, to creating the VM and gathering its connection details. Every other example either builds on this one or references a deployment created from it.

## Prerequisites

- [Blueprint Anatomy](../blueprint-structure/blueprint-anatomy.md) -- understand the overall structure of a NativeEdge blueprint
- [Edge Plugin: Virtual Machines](../edge-plugin/virtual-machines.md) -- the `NativeEdgeVM` and `BinaryImage` node types
- [Utilities Plugin: SSH Keys](../utilities-plugin/ssh-keys.md) -- the `RSAKey` node type
- [Utilities Plugin: Cloud-Init](../utilities-plugin/cloud-init.md) -- the `CloudInit.CloudConfig` node type
- [Fabric Plugin](../fabric-plugin/overview.md) -- remote script execution over SSH

---

## File Structure

```
simple_vm_example_for_NED.yaml    <-- Entry point
vm/
  definitions.yaml                <-- Node templates (6 nodes)
  inputs.yaml                     <-- Input declarations (~50 inputs)
  outputs.yaml                    <-- Output capabilities (6 outputs)
  scripts/
    prepare_network_settings.py   <-- Prepares network segment config
    prepare_serial_ports.py       <-- Prepares serial port passthrough config
    prepare_passwd.py             <-- Hashes the VM password
    prepare_netplan_config.py     <-- Renders the netplan template
    get_target_id.py              <-- Resolves the NativeEdge target endpoint
    get_vm_info.sh                <-- Gathers VM IP and other info via SSH
  templates/
    netplan_config.yaml           <-- Jinja-style template for netplan
```

---

## The Entry Point

The file `simple_vm_example_for_NED.yaml` is the blueprint's main file. When you upload this blueprint to NativeEdge, this is the file you point to. It does not contain any node templates or logic itself -- it imports everything from sub-files.

```yaml
tosca_definitions_version: dell_1_0

description: >
  This blueprint creates a Virtual Machine on a NativeEdge Endpoint
  and provides the VM login details.

imports:
  - dell/types/types.yaml       # Base NativeEdge type definitions
  - vm/definitions.yaml          # Node templates
  - vm/inputs.yaml               # Input declarations
  - vm/outputs.yaml              # Output capabilities
```

### TOSCA Definitions Version

Every NativeEdge blueprint begins with `tosca_definitions_version: dell_1_0`. This tells the NativeEdge orchestrator which DSL (Domain Specific Language) version to use for parsing the blueprint. All examples in this collection use `dell_1_0`.

### Imports

The `imports` section lists files that the orchestrator merges into this blueprint. The merge is additive -- node templates, inputs, outputs, and type definitions from all imported files are combined into a single logical blueprint.

- `dell/types/types.yaml` is the base types library, always required. It defines the foundational node types, relationship types, and data types.
- The three `vm/` files contain the actual blueprint logic, split by concern.

### Input Groups

Input groups control how inputs are presented in the NativeEdge UI. They do not affect blueprint execution -- they are purely for user experience.

```yaml
input_groups:
  artifact:
    display_label: Deployment Artifacts
    collapsible: true
    index: 0
    inputs:
      - poc_os_image_secret
  vm:
    display_label: NativeEdge Virtual Machine
    collapsible: true
    index: 1
    inputs:
      - vm_hostname
      - vm_user_name
      - vm_password
      - vcpus
      - memory_size
      - os_disk_size
      - disk_wrapper
  device_passthrough:
    display_label: Device Passthrough
    collapsible: true
    index: 2
    inputs:
      - use_passthrough
      - pcie
      - usb
      - serial_port_wrapper
      - gpu
      - video
  network:
    display_label: Network Configuration
    collapsible: true
    index: 3
    inputs:
      - vnic_0_segment_name
      - use_dhcp
      - static_ip
      # ... additional network inputs
```

Each group has:
- `display_label` -- the heading shown in the UI
- `collapsible` -- whether the section can be collapsed
- `index` -- the display order (lower numbers appear first)
- `inputs` -- the list of input names that belong to this group

### Labels and Blueprint Labels

Labels are metadata tags attached to the deployment. They serve two purposes: filtering deployments in the UI, and enabling cross-blueprint references (as seen in the Nginx App example).

```yaml
labels:
  csys-obj-type:
    values:
      - environment          # Marks this as an "environment" type deployment
  target_environment:
    values:
      - nativeedge           # Targets a NativeEdge endpoint
  vendor:
    values:
      - DELL
  solution:
    values:
      - POC-VM               # Identifies this as the POC-VM solution
  version:
    values:
      - "3.2.0"

blueprint_labels:
  env:
    values:
      - ned                  # NativeEdge environment
```

The `labels` section is set at deployment time and can vary per deployment instance. The `blueprint_labels` section is fixed for all deployments of this blueprint. The `solution: POC-VM` label is particularly important -- the Nginx App example uses it to filter which deployments can be selected as a target.

---

## Node Templates (definitions.yaml)

The `vm/definitions.yaml` file is where the infrastructure is defined. It contains six node templates that together provision a complete virtual machine. The file starts with its own imports:

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin
  - plugin:utilities-plugin?version=>=3.1.3.0
  - plugin:fabric-plugin?version=>=3.3.2.0
```

The `plugin:` import syntax loads a NativeEdge plugin by name. The `?version=>=3.1.3.0` suffix specifies a minimum version constraint. This blueprint uses three plugins:

- **edge-plugin** -- provides `NativeEdgeVM`, `BinaryImage`, and other edge-specific node types
- **utilities-plugin** -- provides `RSAKey`, `CloudInit.CloudConfig`, and other utilities
- **fabric-plugin** -- provides SSH-based remote execution capabilities

### Node Template Dependency Chain

Before diving into each node, here is the dependency chain that controls execution order:

```
ssh_keys ─────────────┐
                      v
prepare_config ──> cloudinit ──> vm ──> vm_info
                      ^          ^
binary_image ─────────┘──────────┘
```

Nodes connected by `dell.relationships.depends_on` execute in order: the target node must complete before the source node starts. Nodes without dependencies between them (like `ssh_keys`, `binary_image`, and `prepare_config`) can execute in parallel.

---

### Node 1: `ssh_keys` -- Generate SSH Key Pair

```yaml
ssh_keys:
  type: dell.nodes.keys.RSAKey
  properties:
    use_secret_store: true
    use_secrets_if_exist: true
    key_name: "poc_key"
    resource_config:
      openssh_format: true
      key_name: "poc_key"
  interfaces:
    dell.interfaces.lifecycle:
      create:
        implementation: keys.ne_ssh_key.operations.create
        inputs:
          store_public_key_material: false
          store_private_key_material: false
      delete: {}
```

**What it does:** Generates an RSA key pair and stores both keys (public and private) as secrets in the NativeEdge secret store. The secret names are derived from `key_name` -- you will see them referenced later as `poc_key_public` and `poc_key_private`.

**Key properties:**
- `use_secret_store: true` -- store the generated keys as NativeEdge secrets
- `use_secrets_if_exist: true` -- if secrets named `poc_key_public` and `poc_key_private` already exist, reuse them instead of generating new ones. This is important for idempotency.
- `openssh_format: true` -- generate keys in OpenSSH format (required for cloud-init `ssh_authorized_keys`)

**Key behaviors:**
- `store_public_key_material: false` and `store_private_key_material: false` mean the key content is NOT stored as a runtime property on the node instance. It is only stored in the secret store. This is a security best practice.
- The `delete: {}` interface means the delete operation is defined but empty -- the secrets are cleaned up separately.

**Runtime attributes produced:**
- `secret_key_name` -- the base name of the secret (e.g., `poc_key`). Other nodes use this to construct the full secret names: `poc_key_public` and `poc_key_private`.

---

### Node 2: `binary_image` -- Upload OS Image

```yaml
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
```

**What it does:** Downloads an OS image from a repository and uploads it to the NativeEdge endpoint's image catalog. The VM node will reference this image when creating the virtual machine.

**Key properties:**
- `artifact.path` -- the URL of the OS image file
- `artifact.username` and `artifact.access_token` -- credentials for accessing the image repository
- `version` -- the image version string

**Lifecycle operations:**
1. `precreate` -- validates the image configuration before attempting the upload
2. `create` -- performs the actual binary upload to the endpoint
3. `delete` -- removes the image from the catalog when the deployment is uninstalled

**Intrinsic function pattern -- nested secret lookup:**

```yaml
access_token: { get_secret: { get_input: binary_image_access_token } }
```

This is a two-level lookup. First, `get_input: binary_image_access_token` resolves to a list (the secret path), then `get_secret` uses that list to retrieve the actual token from the secret store. This pattern allows the user to provide a single secret name that contains all image-related credentials as a structured secret.

**Runtime attributes produced:**
- `binary_details.extra.artifact_id` -- the catalog ID of the uploaded image, used by the `vm` node.

---

### Node 3: `prepare_config` -- Prepare Network, Serial Ports, and Password

```yaml
prepare_config:
  type: dell.nodes.ApplicationModule
  interfaces:
    dell.interfaces.lifecycle:
      precreate:
        implementation:
          vm/scripts/prepare_network_settings.py
        executor: central_deployment_agent
        inputs:
          network_segments:
            - { get_input: vnic_0_segment_name }
            - { get_input: vnic_1_segment_name }
          port_forwards:
            - { get_input: port_forward_rules }
            - { get_input: port_forward_rules2 }
      create:
        implementation: vm/scripts/prepare_serial_ports.py
        executor: central_deployment_agent
        inputs:
          serial_port: { get_input: serial_port }
      start:
        implementation: vm/scripts/prepare_passwd.py
        executor: central_deployment_agent
        inputs:
          vm_password: { get_secret: { get_input: vm_password } }
```

**What it does:** Runs three Python scripts in sequence (during the `precreate`, `create`, and `start` lifecycle phases) to prepare configuration data that other nodes will consume.

**Lifecycle phases used:**
1. `precreate` -- `prepare_network_settings.py` takes the network segment names and port forwarding rules and transforms them into the `network_settings` structure expected by the VM node.
2. `create` -- `prepare_serial_ports.py` transforms the serial port input into a list format expected by the VM.
3. `start` -- `prepare_passwd.py` takes the user's password from the secret store and hashes it (SHA-512) for use in cloud-init.

**Key concept -- `executor: central_deployment_agent`:**

This tells NativeEdge to run the script on the NativeEdge manager itself (the central deployment agent), not on the target VM. This is necessary because at this point the VM does not exist yet -- these are preparation scripts that run locally to compute configuration values.

**Key concept -- `dell.nodes.ApplicationModule`:**

This is a generic node type used as a "compute module" -- a place to run scripts that produce runtime properties consumed by other nodes. It does not represent a deployed resource itself.

**Runtime attributes produced:**
- `network_settings` -- the formatted network settings list for the VM
- `serial_ports_list` -- the formatted serial ports list
- `hashed_vm_passwd` -- the SHA-512 hashed password

---

### Node 4: `cloudinit` -- Build Cloud-Init Configuration

This is the most complex node in the blueprint. It builds a complete cloud-init configuration that will be injected into the VM at boot time.

```yaml
cloudinit:
  type: dell.nodes.CloudInit.CloudConfig
  properties:
    resource_config:
      hostname: { get_input: vm_hostname }
      runcmd:
        - netplan apply
        - systemctl restart sshd
      write_files:
        - content: { get_attribute: [SELF, netplan_encoded] }
          encoding: b64
          path: /etc/netplan/50-cloud-init.yaml
        - content: |
            ClientAliveInterval 1800
            ClientAliveCountMax 3
          path: /etc/ssh/sshd_config
          append: true
      disable_root_opts: >
        no-port-forwarding,no-agent-forwarding,no-X11-forwarding
      disable_root: false
      ssh_pwauth: true
      ssh_authorized_keys:
        - get_secret:
            concat:
              - get_attribute: [ssh_keys, secret_key_name]
              - "_public"
      users:
        - name: { get_input: vm_user_name }
          sudo: ['ALL=(ALL) NOPASSWD:ALL']
          groups: users, admin
          passwd: { get_attribute: [prepare_config, hashed_vm_passwd] }
          lock_passwd: false
          shell: /bin/bash
          ssh_authorized_keys:
            - get_secret:
                concat:
                  - get_attribute: [ ssh_keys, secret_key_name ]
                  - "_public"
```

**What it does:** Generates a cloud-init cloud-config YAML document that will be attached to the VM. Cloud-init runs inside the VM on first boot and applies this configuration.

**The `resource_config` property contains:**

- **`hostname`** -- sets the VM's hostname
- **`runcmd`** -- commands to run on first boot. Here it applies the netplan network configuration and restarts the SSH daemon.
- **`write_files`** -- files to write inside the VM:
  - The netplan configuration (base64-encoded, rendered by the `precreate` lifecycle step)
  - SSH keep-alive settings appended to the SSH server config
- **`disable_root`** -- set to `false`, allowing root login (with restrictions from `disable_root_opts`)
- **`ssh_pwauth: true`** -- enables SSH password authentication
- **`ssh_authorized_keys`** -- the public SSH key (from the `ssh_keys` node) added to the root account
- **`users`** -- creates the VM user account with:
  - `sudo` access without password
  - Membership in `users` and `admin` groups
  - The hashed password from `prepare_config`
  - The SSH public key for key-based authentication

**Key intrinsic function pattern -- dynamic secret name:**

```yaml
ssh_authorized_keys:
  - get_secret:
      concat:
        - get_attribute: [ssh_keys, secret_key_name]
        - "_public"
```

This constructs the secret name at runtime. `get_attribute: [ssh_keys, secret_key_name]` resolves to `poc_key`, then `concat` appends `_public` to produce `poc_key_public`, and finally `get_secret` retrieves the actual public key content from that secret. This pattern is used throughout the blueprint wherever SSH keys are referenced.

**Lifecycle -- Netplan Preparation and Config Merge:**

```yaml
interfaces:
  dell.interfaces.lifecycle:
    precreate:
      implementation: vm/scripts/prepare_netplan_config.py
      executor: central_deployment_agent
      inputs:
        template: vm/templates/netplan_config.yaml
        parameters:
          use_dhcp: { get_input: use_dhcp }
          static_ip: { get_input: static_ip }
          use_gateway: { get_input: use_gateway }
          gateway: { get_input: gateway }
          use_dns: { get_input: use_dns }
          dns: { get_input: dns }
          add_second_nic: { get_input: add_second_nic }
          use_dhcp2: { get_input: use_dhcp2 }
          static_ip2: { get_input: static_ip2 }
          use_dns2: { get_input: use_dns2 }
          dns2: { get_input: dns2 }
    create:
      inputs:
        resource_config:
          merge:
            - { get_attribute: [SELF, resource_config] }
            - { get_input: cloudinit }
```

The `precreate` phase runs `prepare_netplan_config.py` to render the netplan template with the user's network parameters. The rendered content is stored as the `netplan_encoded` runtime attribute (base64-encoded), which is then referenced in the `write_files` section above.

The `create` phase uses the `merge` intrinsic function to combine the node's `resource_config` (which contains all the cloud-init settings defined in `properties`) with any additional cloud-init configuration provided through the `cloudinit` hidden input. This allows advanced users to inject extra cloud-init directives without modifying the blueprint.

**Relationships:**

```yaml
relationships:
  - type: dell.relationships.depends_on
    target: ssh_keys
  - type: dell.relationships.depends_on
    target: prepare_config
```

The cloud-init node depends on both `ssh_keys` (for the public key) and `prepare_config` (for the hashed password and netplan data). These nodes must complete before `cloudinit` begins execution.

---

### Node 5: `vm` -- Create the Virtual Machine

```yaml
vm:
  type: dell.nodes.nativeedge.template.NativeEdgeVM
  properties:
    vm_config:
      location: { get_environment_capability: ece_service_tag }
      name: { get_sys: [deployment, name ] }
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

**What it does:** Creates the actual virtual machine on the NativeEdge endpoint. This is the central node that consumes outputs from all the preparation nodes.

**Key `vm_config` properties:**

- **`location`** -- the NativeEdge endpoint's service tag, retrieved from the environment using `get_environment_capability: ece_service_tag`. This tells NativeEdge which physical edge device to create the VM on.
- **`name`** -- uses `get_sys: [deployment, name]` to name the VM after the deployment. This intrinsic function retrieves system-level values; here it gets the deployment's name.
- **`image`** -- references the uploaded OS image via `get_attribute: [binary_image, binary_details, extra, artifact_id]`. This is a deeply nested attribute access -- it navigates through the `binary_image` node's runtime properties to find the catalog artifact ID.
- **`resource_constraints`** -- CPU, memory, storage, disk path, and controller type
- **`hardware_options`** -- vTPM, secure boot, and firmware type (important for Windows 11)
- **`network_settings`** -- the prepared network configuration from `prepare_config`
- **`ports`** -- device passthrough configuration (USB, serial, GPU, video, PCIe)
- **`cloudinit`** -- the rendered cloud-init config from the `cloudinit` node, accessed via `get_attribute: [cloudinit, cloud_config]`

**Relationships:**

The VM depends on three nodes: `cloudinit`, `binary_image`, and `prepare_config`. All three must complete successfully before the VM creation begins.

**Runtime attributes produced:**
- `vm_details.name` -- the VM name on the endpoint (used for SSH connections)

---

### Node 6: `vm_info` -- Gather VM Information

```yaml
vm_info:
  type: dell.nodes.Root
  interfaces:
    dell.interfaces.lifecycle:
      create:
        implementation: vm/scripts/get_target_id.py
        executor: central_deployment_agent
      start:
        implementation: fabric.fabric_plugin.tasks.run_script
        inputs:
          script_path: vm/scripts/get_vm_info.sh
          fabric_env:
            host_string:
              get_attribute: [vm, vm_details, name]
            user:
              get_input: vm_user_name
            key:
              get_secret:
                concat:
                  - get_attribute: [ ssh_keys, secret_key_name ]
                  - "_private"
            port: 22
          proxy_settings:
            get_attribute: [ SELF, connection_proxy_settings ]
```

**What it does:** After the VM is created and booted, this node gathers information about it. It runs in two phases:

1. `create` -- runs `get_target_id.py` on the central agent to resolve the NativeEdge endpoint target ID, which is needed to establish the proxy connection.
2. `start` -- uses the **Fabric plugin** to SSH into the VM and run `get_vm_info.sh`, which collects the VM's public IP address and other details.

**Key concept -- Fabric plugin integration:**

The `fabric.fabric_plugin.tasks.run_script` implementation tells the orchestrator to use the Fabric plugin to execute a script over SSH. The `fabric_env` section provides the SSH connection details:

- `host_string` -- the VM name (resolved from the `vm` node's attributes)
- `user` -- the SSH username
- `key` -- the SSH private key (retrieved from the secret store using the dynamic name pattern)
- `port` -- SSH port (22)

**Key concept -- `proxy_settings`:**

```yaml
proxy_settings:
  get_attribute: [ SELF, connection_proxy_settings ]
```

NativeEdge VMs are not directly accessible from the NativeEdge manager. The manager connects through the NativeEdge endpoint agent, which acts as an SSH proxy. The `connection_proxy_settings` attribute is automatically populated by the NativeEdge platform based on the target endpoint, enabling SSH tunneling through the NativeEdge infrastructure.

**Update lifecycle:**

Notice the `preupdate` and `update` operations mirror the `create` and `start` operations. This means when you run a deployment update, the VM information is refreshed.

**Runtime attributes produced:**
- `capabilities.vm_public_ip` -- the VM's IP address, gathered from inside the VM

---

## Inputs (inputs.yaml)

The inputs file is the largest file in this example (~790 lines). It defines approximately 50 inputs organized into two categories: **user-visible inputs** and **hidden inputs**.

### User-Visible Inputs

These inputs appear in the NativeEdge UI when a user creates a deployment:

```yaml
inputs:

  # Artifact inputs
  poc_os_image_secret:
    type: secret_key
    hidden: false
    allow_update: true
    display_label: OS Image Secret for Virtual Machine creation
    description: |
      Secret containing url, username, access_token, and
      version of the os image.
    constraints:
    - type: binary_configuration
    display:
      group: artifacts
      index: 10
```

**Input properties explained:**
- `type` -- the data type. Common types include `string`, `integer`, `boolean`, `list`, `secret_key`, and `deployment_id`.
- `hidden` -- whether the input appears in the UI. `false` means visible.
- `allow_update` -- whether the value can be changed after initial deployment.
- `display_label` -- the label shown in the UI.
- `description` -- help text shown to the user.
- `constraints` -- validation rules (patterns, valid values, min/max).
- `display.group` -- which input group this input belongs to.
- `display.index` -- the display order within the group.

**Constraint patterns of note:**

Hostname validation with regex:
```yaml
vm_hostname:
  type: string
  constraints:
  - pattern: ^(?!-)[a-zA-Z0-9-]{1,63}(?<!-)$
    error_message: |
      Must be letters (a-z, A-Z), numbers (0-9), or hyphens (-).
      No more than 64 characters.
  - max_length: 64
  default: pochost
```

Memory size validation enforcing a unit suffix:
```yaml
memory_size:
  type: string
  constraints:
  - pattern: '\d+(\.\d+)?(KB|MB|GB|TB|PB|EB|ZB|YB)'
    error_message: |
      Memory size must be followed by unit [KB,MB,GB,TB,PB,EB,ZB,YB].
      Example: 4GB
  default: 4GB
```

Dynamic valid values from inventory using JMESPath:
```yaml
disk_wrapper:
  type: string
  constraints:
    - valid_values:
        jmespath:
          - extra.hw_core.storage.dataStoreOrVolumes[?((((type=='SSD'||type=='HDD')&&name.contains(@,'DataStore'))||(shared&&name=='/Shared_DataStore'))&&status=='ONLINE')].mountPath
          - get_inventory:
              get_environment_capability: ece_service_tag
```

This retrieves available datastore paths from the NativeEdge endpoint's inventory at deployment time, so users can only select from valid options. The `jmespath` constraint uses a JMESPath expression to filter the inventory data.

**Conditional visibility:**

```yaml
use_passthrough:
  type: boolean
  display_label: Enable device passthrough
  default: false

usb:
  type: list
  only_with: use_passthrough     # Only shown when use_passthrough is true
  display_label: USB Device list

static_ip:
  type: string
  exclusive_with: use_dhcp       # Only shown when use_dhcp is false
  display_label: Static IP and CIDR prefix
```

- `only_with` -- shows this input only when the referenced boolean input is `true`
- `exclusive_with` -- shows this input only when the referenced boolean input is `false`

### Hidden Inputs

Hidden inputs (`hidden: true`) are not shown in the UI. They provide internal defaults, computed values, or configuration that advanced users might override programmatically:

```yaml
  os_type:
    type: string
    hidden: true
    default: UBUNTU22.04
    constraints:
      - valid_values:
        - "UBUNTU22.04"
        - "UBUNTU24.04"
        - "CENTOS9"
        # ... many more OS types

  disk:
    type: string
    hidden: true
    default: { get_input: disk_wrapper }   # Copies value from the visible input
```

The `disk` input demonstrates a common pattern: a visible "wrapper" input (`disk_wrapper`) with dynamic constraints (JMESPath) feeds into a hidden input (`disk`) that is used by the node templates. This allows the UI to show a user-friendly dropdown while the internal logic uses a simpler input name.

### Structured Secret Pattern

The binary image inputs demonstrate how a single structured secret contains multiple related values:

```yaml
  binary_image_url:
    type: string
    hidden: true
    default:
      get_secret:
      - get_input: poc_os_image_secret    # The secret name
      - binary_image_url                   # The key within the secret

  binary_image_access_user:
    type: string
    hidden: true
    default:
      get_secret:
      - get_input: poc_os_image_secret
      - binary_image_access_user
```

The user provides one secret name (`poc_os_image_secret`), and the blueprint extracts multiple values from it using the `get_secret` list syntax: `[secret_name, key_path]`.

### Custom Data Types

The inputs file also defines a custom `port_forward_rules` data type used by the port forwarding inputs:

```yaml
data_types:
  port_forward_rules:
    properties:
      host_ip:
        type: string
        default: CHANGE TO HOST IP OF NAT VIRTUAL NETWORK SEGMENT
      host_port:
        type: string
      protocol:
        type: string
        description: Protocol ("TCP", "UDP")
      service_type:
        type: string
        description: Service Type ("SSH", "HTTP", "CUSTOM")
      vm_port:
        type: string
      vm_ip:
        type: string
```

This structured type ensures that port forwarding rules have a consistent shape across all inputs.

---

## Outputs (outputs.yaml)

Outputs are declared under the `capabilities` section. They expose values from the deployment that users (and other blueprints) can consume.

```yaml
capabilities:

  service_tag:
    description: Service Tag
    value: { get_property: [vm, vm_config, location] }

  vm_name:
    description: Virtual Machine Name
    value: { get_attribute: [vm, vm_details, name] }

  vm_hostname:
    description: Virtual Machine hostname
    value: { get_input: vm_hostname }

  vm_ip:
    description: Virtual Machine IP
    value:
      get_attribute: [ vm_info, capabilities, vm_public_ip ]

  vm_user_name:
    description: Virtual Machine User Name
    value: { get_input: vm_user_name }

  vm_ssh_private_key:
    description: Virtual Machine SSH Private Key
    value:
      concat:
        - get_attribute: [ ssh_keys, secret_key_name ]
        - "_private"
```

**Output sources:**

- `service_tag` uses `get_property` (reads the property definition, not a runtime attribute) to expose the endpoint service tag
- `vm_name` uses `get_attribute` to read the runtime VM name from the `vm` node
- `vm_hostname` echoes back the user's hostname input
- `vm_ip` reads the IP address gathered by the `vm_info` node's script
- `vm_user_name` echoes back the username input
- `vm_ssh_private_key` constructs the secret name (not the key itself) for the SSH private key. Other blueprints can then use `get_secret` with this name to retrieve the actual key.

These capabilities are what the **Nginx App** and **MQTT + Node-RED** examples consume when they reference a deployed VM instance.

---

## Key Intrinsic Functions Summary

| Function | Example | Purpose |
|---|---|---|
| `get_input` | `{ get_input: vm_hostname }` | Retrieves a user-provided input value |
| `get_secret` | `{ get_secret: poc_key_private }` | Retrieves a value from the secret store |
| `get_secret` (nested) | `{ get_secret: { get_input: vm_password } }` | Retrieves a secret whose name comes from an input |
| `get_secret` (structured) | `get_secret: [secret_name, key]` | Retrieves a specific key from a structured secret |
| `get_attribute` | `{ get_attribute: [vm, vm_details, name] }` | Reads a runtime attribute from a node |
| `get_attribute` (SELF) | `{ get_attribute: [SELF, netplan_encoded] }` | Reads a runtime attribute from the current node |
| `get_environment_capability` | `{ get_environment_capability: ece_service_tag }` | Reads a capability from the NativeEdge environment |
| `get_sys` | `{ get_sys: [deployment, name] }` | Reads a system value (deployment name, ID, etc.) |
| `get_property` | `{ get_property: [vm, vm_config, location] }` | Reads a property definition (not runtime) from a node |
| `concat` | `{ concat: [str1, str2] }` | Concatenates strings |
| `merge` | `{ merge: [map1, map2] }` | Merges two dictionaries |

---

## Complete Execution Flow

When you deploy this blueprint, the following happens in order:

1. **Parallel start:** `ssh_keys`, `binary_image`, and `prepare_config` begin executing simultaneously (they have no dependencies on each other).
2. `ssh_keys` generates the RSA key pair and stores it in the secret store.
3. `binary_image` validates and uploads the OS image to the endpoint catalog.
4. `prepare_config` runs three scripts sequentially (precreate, create, start) to prepare network settings, serial ports, and the password hash.
5. **`cloudinit` starts** once both `ssh_keys` and `prepare_config` have completed. It renders the netplan template and builds the full cloud-init configuration.
6. **`vm` starts** once `cloudinit`, `binary_image`, and `prepare_config` have all completed. It creates the virtual machine on the endpoint with all the prepared configuration.
7. **`vm_info` starts** once `vm` has completed. It resolves the target endpoint and SSHs into the VM to gather its IP address and other details.
8. **Outputs are computed** from the runtime attributes of all nodes.

On uninstall, the operations execute in reverse order: `vm_info` cleanup, VM deletion, image removal from catalog, and secret cleanup.
