# Cloud Init

Cloud-init is an industry-standard system for automating the initialization of virtual machines on first boot. When a VM starts for the first time, cloud-init reads a configuration (called a "cloud-config") and uses it to set the hostname, create users, write files, run commands, configure SSH keys, and perform many other setup tasks -- all without manual intervention.

The `dell.nodes.CloudInit.CloudConfig` node type in the Utilities Plugin lets you define this cloud-init configuration directly in your NativeEdge blueprint. During deployment, the plugin renders your configuration into a cloud-config string, which you then pass to the VM node so it is available at boot time.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.
- Familiarity with YAML syntax and NativeEdge blueprint structure.

## Node Type: dell.nodes.CloudInit.CloudConfig

**Derived from:** `dell.nodes.Root`

This node type takes your cloud-init configuration as input properties, processes it, and produces a rendered cloud-config string as a runtime property. Other nodes in your blueprint (such as a VM node) can then reference this output to inject the cloud-config into the virtual machine.

### Data Type: dell.datatypes.CloudInit.SecretProperty

Before looking at the node properties, you need to understand this data type. It controls whether the generated cloud-init configuration values are also stored as NativeEdge secrets (encrypted key-value pairs stored in the NativeEdge secret store).

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `enabled` | boolean | No | `false` | Whether to store the generated cloud-init values in the NativeEdge secret store. When `true`, the cloud-config, JSON config, and resource config are saved as secrets so other deployments or components can retrieve them securely. |
| `prefix` | string | No | `''` (empty) | The prefix for the names of the secrets. If left empty, the node instance ID is used as the prefix. For example, if the prefix is `"myvm"`, the secret names will be `myvm_resource_config`, `myvm_client_config`, and `myvm_json_config`. |
| `overrides` | dict | No | `{}` | Extra parameters to pass to the NativeEdge REST client when creating the secrets. This is an advanced option for controlling secret creation behavior. |

### Properties

These are the properties you set on the node in your blueprint:

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `header` | string | No | `'#cloud-config'` | The header line placed at the top of the generated cloud-config output. The standard cloud-init header is `#cloud-config`. You should not change this unless you have a specific reason. Cloud-init uses this header to identify the file format. |
| `encode_base64` | boolean | No | `false` | Whether to base64-encode the final cloud-config output. Some VM platforms require the cloud-config to be base64-encoded. Set this to `true` if your target platform expects encoded input. |
| `secrets` | dell.datatypes.CloudInit.SecretProperty | No | (see data type defaults) | Controls whether and how the cloud-init configuration is stored in the NativeEdge secret store. See the data type table above. |
| `resource_config` | dict | No | `{}` | The actual cloud-init configuration content. This is a dictionary that follows the cloud-init cloud-config specification. You define hostname, users, write_files, runcmd, ssh settings, and any other cloud-init modules here. |

### Runtime Properties

After the node's lifecycle operations run, these properties are available on the node instance. Other nodes can read them using `get_attribute`.

| Runtime Property | Type | Description |
|---|---|---|
| `cloud_config` | string | The rendered cloud-config as a string, including the `#cloud-config` header. This is what you pass to a VM node. It is the complete, ready-to-use cloud-init payload. |
| `json_config` | string | A dictionary (JSON-like) representation of the cloud-config. This is the same data as `cloud_config` but in a structured format rather than a YAML string. |
| `resource_config` | string | The current version of the `resource_config` node property. This reflects the configuration as it exists after all lifecycle operations have processed it. |

### Lifecycle Operations

The CloudConfig node has a unique lifecycle pattern: the `create`, `configure`, `start`, and `stop` operations all call the same underlying implementation (`update`). This means the cloud-config is regenerated at each lifecycle stage, picking up any changes from relationships or merged inputs.

| Operation | Implementation | Description |
|---|---|---|
| `create` | `cloudinit.plugins_cloudinit.tasks.update` | Generates (or regenerates) the cloud-config. |
| `configure` | `cloudinit.plugins_cloudinit.tasks.update` | Regenerates the cloud-config. |
| `start` | `cloudinit.plugins_cloudinit.tasks.update` | Regenerates the cloud-config. |
| `stop` | `cloudinit.plugins_cloudinit.tasks.update` | Regenerates the cloud-config. |
| `delete` | `cloudinit.plugins_cloudinit.tasks.delete` | Cleans up the cloud-config data. |

This pattern is intentional. Because the cloud-config may depend on runtime properties of other nodes (which may not be available at `create` time), the repeated `update` calls ensure the configuration is always current by the time the VM node needs it.

## Example: VM Cloud-Init Configuration

The following example is taken from a real blueprint that provisions a VM with cloud-init. It demonstrates how to configure hostname, users, SSH keys, file writing, and command execution.

```yaml
imports:
  - dell/types/types.yaml
  - plugin:edge-plugin
  - plugin:utilities-plugin?version=>=3.1.3.0

node_templates:
  # First, generate SSH keys (see ssh-keys.md for details)
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

  # Define the cloud-init configuration
  cloudinit:
    type: dell.nodes.CloudInit.CloudConfig
    properties:
      resource_config:
        # Set the VM hostname
        hostname: { get_input: vm_hostname }

        # Commands to run on first boot
        runcmd:
          - netplan apply
          - systemctl restart sshd

        # Files to write to the VM filesystem
        write_files:
          - content: { get_attribute: [SELF, netplan_encoded] }
            encoding: b64
            path: /etc/netplan/50-cloud-init.yaml
          - content: |
              ClientAliveInterval 1800
              ClientAliveCountMax 3
            path: /etc/ssh/sshd_config
            append: true

        # Root access settings
        disable_root_opts: >
          no-port-forwarding,no-agent-forwarding,no-X11-forwarding
        disable_root: false

        # SSH settings
        ssh_pwauth: true
        ssh_authorized_keys:
          - get_secret:
              concat:
                - get_attribute: [ssh_keys, secret_key_name]
                - "_public"

        # Create a user with sudo access
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

    interfaces:
      dell.interfaces.lifecycle:
        create:
          inputs:
            resource_config:
              merge:
                - { get_attribute: [SELF, resource_config] }
                - { get_input: cloudinit }
    relationships:
      - type: dell.relationships.depends_on
        target: ssh_keys
      - type: dell.relationships.depends_on
        target: prepare_config

  # The VM node consumes the cloud-config output
  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        # ... other VM configuration ...
        cloudinit: { get_attribute: [cloudinit, cloud_config] }
    relationships:
       - type: dell.relationships.depends_on
         target: cloudinit
```

### How This Example Works

1. **SSH key generation:** The `ssh_keys` node generates an RSA key pair and stores it in the NativeEdge secret store under a name like `poc_key_public` and `poc_key_private`. The runtime property `secret_key_name` holds the base name (`poc_key`).

2. **Cloud-init configuration:** The `cloudinit` node defines the VM's initialization:
   - `hostname` is set from a blueprint input.
   - `runcmd` lists shell commands to execute on first boot (applying network configuration and restarting SSH).
   - `write_files` writes a netplan network configuration file and appends SSH keepalive settings to the SSH server config.
   - `ssh_authorized_keys` at the top level sets authorized keys for the default user. It retrieves the public key from the secret store using `get_secret` with a dynamically constructed name: the `secret_key_name` runtime property from the `ssh_keys` node, concatenated with `_public`.
   - `users` creates a named user with full sudo access, a hashed password, and the same SSH public key.

3. **Merge pattern:** In the `create` operation inputs, the `resource_config` uses a `merge` intrinsic function to combine the node's own `resource_config` with additional cloud-init values provided via blueprint input (`get_input: cloudinit`). This allows users to inject extra cloud-init configuration at deployment time without modifying the blueprint.

4. **Consumption by the VM:** The `vm` node references `{ get_attribute: [cloudinit, cloud_config] }` in its `vm_config.cloudinit` property. This passes the fully rendered cloud-config string to the VM platform, which injects it into the VM at boot time.

5. **Dependency ordering:** The `relationships` section ensures that `ssh_keys` and `prepare_config` are created before `cloudinit`, and `cloudinit` is created before `vm`. This guarantees that all runtime properties (like the SSH key secret name) are available when the cloud-config is rendered.

## Key Concepts for New Users

### What is cloud-config?

A cloud-config is a YAML document that begins with `#cloud-config` on the first line. It is consumed by the cloud-init service running inside a VM on first boot. Cloud-init is pre-installed on most Linux cloud images (Ubuntu, CentOS, Debian, etc.).

### How does the cloud-config reach the VM?

The Utilities Plugin does not send the cloud-config to the VM directly. Instead, it generates the cloud-config string and stores it as a runtime property (`cloud_config`). The VM provisioning plugin (such as the edge plugin) reads this runtime property and passes it to the VM platform through the appropriate mechanism (e.g., a metadata service, a CD-ROM drive, or a user-data field).

### Why are all lifecycle operations the same?

The `create`, `configure`, `start`, and `stop` operations all call `update` because the cloud-config may depend on values from other nodes that are only available at later lifecycle stages. By regenerating at each stage, the cloud-config always reflects the latest state.
