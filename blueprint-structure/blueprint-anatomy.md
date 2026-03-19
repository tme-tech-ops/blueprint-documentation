# Blueprint Anatomy

This guide explains the structure and composition of NativeEdge blueprints. A blueprint is a YAML file that declaratively describes the infrastructure and applications you want to deploy. Understanding how blueprints are structured is fundamental to working with NativeEdge.

## Table of Contents

- [TOSCA Definitions Version](#tosca-definitions-version)
- [Description](#description)
- [Imports](#imports)
- [DSL Definitions](#dsl-definitions)
- [Inputs](#inputs)
- [Input Groups](#input-groups)
- [Node Types](#node-types)
- [Node Templates](#node-templates)
- [Relationships](#relationships)
- [Capabilities (Outputs)](#capabilities-outputs)
- [Labels and Blueprint Labels](#labels-and-blueprint-labels)
- [Intrinsic Functions](#intrinsic-functions)
- [Modular Blueprint Pattern](#modular-blueprint-pattern)

---

## TOSCA Definitions Version

Every blueprint begins with the TOSCA definitions version declaration. This tells the NativeEdge orchestrator which DSL (Domain Specific Language) version the blueprint uses.

```yaml
tosca_definitions_version: dell_1_0
```

This must be the very first line of your main blueprint file. The `dell_1_0` version is the standard for NativeEdge blueprints.

---

## Description

An optional human-readable description of what this blueprint does.

```yaml
description: >
  This blueprint creates a Virtual Machine on a NativeEdge Endpoint
  and provides the VM login details.
```

The `>` character creates a folded scalar in YAML, joining multiple lines into a single string with spaces.

---

## Imports

The `imports` section tells the orchestrator which type definitions and plugin definitions to load before processing the blueprint. Without imports, the blueprint has no access to node types, data types, or plugin operations.

### Types of Imports

#### 1. Base Type Definitions

Every blueprint imports the base NativeEdge type system:

```yaml
imports:
  - dell/types/types.yaml
```

This provides foundational types like `dell.nodes.Root`, `dell.nodes.ApplicationModule`, `dell.relationships.depends_on`, and the standard lifecycle interfaces (`dell.interfaces.lifecycle`).

#### 2. Plugin Imports

Plugins are imported by their registered package name, optionally with a version constraint:

```yaml
imports:
  - plugin:edge-plugin
  - plugin:utilities-plugin?version=>=3.1.3.0
  - plugin:fabric-plugin?version=>=3.3.2.0
  - plugin:ansible-plugin?version=>=4.1.2.0
```

- `plugin:edge-plugin` -- imports the latest version of the edge plugin
- `plugin:utilities-plugin?version=>=3.1.3.0` -- imports utilities plugin version 3.1.3.0 or newer

Plugin imports make the plugin's node types, data types, relationships, and workflows available in the blueprint.

#### 3. File Imports

Blueprints can be split across multiple YAML files. Import other blueprint files by their relative path:

```yaml
imports:
  - dell/types/types.yaml
  - vm/definitions.yaml
  - vm/inputs.yaml
  - vm/outputs.yaml
```

This is the **modular blueprint pattern** -- the main entry point file imports sub-files that each contain a portion of the blueprint. See [Modular Blueprint Pattern](#modular-blueprint-pattern) for details.

### Import Order

Imports are processed in order. If two imported files define the same key (e.g., both define an input with the same name), the later import overrides the earlier one. Generally, import order is:

1. Base types first (`dell/types/types.yaml`)
2. Plugins next
3. Your own sub-files last

---

## DSL Definitions

DSL (Domain Specific Language) definitions let you define reusable YAML fragments using **YAML anchors**. This avoids repetition when multiple nodes share the same configuration.

### Syntax

```yaml
dsl_definitions:

  # Define a reusable block with &anchor_name
  ansible_env_vars: &ansible_default_env_vars
    ANSIBLE_HOST_KEY_CHECKING: "False"
    ANSIBLE_INVALID_TASK_ATTRIBUTE_FAILED: "False"
    ANSIBLE_STDOUT_CALLBACK: json

  # Define another reusable block
  ansible_common_properties: &ansible_common_properties
    ansible_external_venv: /opt/ansible
    log_stdout: true
    store_facts: false
    timeout: '30'
    debug_level: 2
    number_of_attempts: 10
```

### Using Anchors

Reference a defined anchor using `*anchor_name` (alias) or `<<: *anchor_name` (merge key):

```yaml
node_templates:
  my_ansible_node:
    type: dell.nodes.ansible.Executor
    properties:
      # Merge all properties from the anchor into this block
      <<: *ansible_common_properties
      # Override or add specific properties
      playbook_path: my_playbook.yaml
      ansible_env_vars:
        <<: *ansible_default_env_vars
```

The `<<:` merge key copies all key-value pairs from the anchor into the current mapping. You can override individual keys by specifying them after the merge.

### Why Use DSL Definitions?

- **Reduce duplication** -- define SSH settings, Ansible env vars, or common properties once
- **Consistency** -- change a value in one place to update all nodes that reference it
- **Readability** -- long property lists are defined separately from the node templates

---

## Inputs

Inputs are parameters that users provide when deploying a blueprint. They make blueprints configurable without modifying the YAML directly.

### Basic Input Structure

```yaml
inputs:
  vm_hostname:
    type: string
    display_label: VM Hostname
    description: |
      Hostname of the Virtual Machine.
      Cannot contain characters other than letters, numbers, or hyphens.
    default: pochost
    hidden: false
    allow_update: true
    required: true
```

### Input Properties

| Property | Purpose |
|----------|---------|
| `type` | Data type: `string`, `integer`, `boolean`, `list`, `dict`, `secret_key` |
| `display_label` | Human-readable label shown in the NativeEdge UI |
| `description` | Detailed explanation of the input |
| `default` | Default value if the user doesn't provide one |
| `hidden` | If `true`, the input is not shown in the UI (used for internal/computed values) |
| `allow_update` | If `true`, the input can be changed after initial deployment |
| `required` | If `true`, the user must provide a value (defaults to `true`) |
| `constraints` | Validation rules (patterns, valid values, ranges) |
| `display` | UI display configuration (group, index for ordering) |
| `only_with` | Only show this input when the referenced boolean input is `true` |
| `exclusive_with` | Only show this input when the referenced boolean input is `false` |
| `item_type` | For `list` type inputs, specifies the type of list elements |

### Constraints

Constraints validate user input before deployment begins:

```yaml
inputs:
  vm_hostname:
    type: string
    constraints:
      # Regex pattern matching
      - pattern: ^(?!-)[a-zA-Z0-9-]{1,63}(?<!-)$
        error_message: Must be letters, numbers, or hyphens. Max 64 characters.
      # Maximum length
      - max_length: 64

  vcpus:
    type: integer
    constraints:
      # Minimum value
      - greater_or_equal: 2

  os_type:
    type: string
    constraints:
      # Enumerated valid values
      - valid_values:
          - "UBUNTU22.04"
          - "UBUNTU24.04"
          - "CENTOS9"
          - "RHEL9"

  disk_wrapper:
    type: string
    constraints:
      # Dynamic valid values from device inventory using JMESPath
      - valid_values:
          jmespath:
            - extra.hw_core.storage.dataStoreOrVolumes[?status=='ONLINE'].mountPath
            - get_inventory:
                get_environment_capability: ece_service_tag
```

### Secret Key Inputs

Inputs of type `secret_key` reference secrets stored in the NativeEdge secret store:

```yaml
inputs:
  vm_password:
    type: secret_key
    display_label: VM Password
    description: Secret key containing the VM password.
    constraints:
      - type: password
```

The value is a secret name. Use `get_secret` to retrieve the actual secret value elsewhere in the blueprint.

### Conditional Visibility

Inputs can be shown or hidden based on other boolean inputs:

```yaml
inputs:
  use_dhcp:
    type: boolean
    default: true

  # Only shown when use_dhcp is false
  static_ip:
    type: string
    exclusive_with: use_dhcp

  add_second_nic:
    type: boolean
    default: false

  # Only shown when add_second_nic is true
  vnic_1_segment_name:
    type: string
    only_with: add_second_nic
```

### Computed/Hidden Inputs

Hidden inputs let you define internal values that are computed from other inputs or secrets:

```yaml
inputs:
  # User provides a secret name
  poc_os_image_secret:
    type: secret_key
    display_label: OS Image Secret

  # Hidden input computed from the secret
  binary_image_url:
    type: string
    hidden: true
    default:
      get_secret:
        - get_input: poc_os_image_secret
        - binary_image_url
```

---

## Input Groups

Input groups organize inputs into collapsible sections in the NativeEdge UI. They control the visual layout but have no effect on blueprint logic.

```yaml
input_groups:
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
  network:
    display_label: Network Configuration
    collapsible: true
    index: 3
    inputs:
      - vnic_0_segment_name
      - use_dhcp
      - static_ip
```

| Property | Purpose |
|----------|---------|
| `display_label` | Section heading in the UI |
| `collapsible` | Whether the section can be collapsed |
| `index` | Display order (lower = higher on page) |
| `inputs` | List of input names belonging to this group |

---

## Node Types

Node types define the **schema** for a category of resources -- what properties they accept, what lifecycle operations they support, and what runtime properties they produce. Think of node types as "classes" in object-oriented programming.

### Structure of a Node Type

```yaml
node_types:
  dell.nodes.nativeedge.template.NativeEdgeVM:
    derived_from: dell.nodes.Root
    properties:
      vm_config:
        type: dell.types.nativeedge.VMDeployDefinition
        description: "VM deployment definition"
        required: true
      config:
        type: dell.types.nativeedge.Config
        description: "Deployment config"
    interfaces:
      dell.interfaces.lifecycle:
        precreate:
          implementation: edge.nativeedge_plugin.tasks.precreate
        create:
          implementation: edge.nativeedge_plugin.tasks.create
          max_retries: 1000
        start:
          implementation: edge.nativeedge_plugin.tasks.start
        delete:
          implementation: edge.nativeedge_plugin.tasks.delete
```

### Key Concepts

- **`derived_from`**: Inheritance. `dell.nodes.Root` is the base type for most nodes. A derived type inherits all properties and interfaces from its parent.
- **`properties`**: The configuration inputs the node type accepts. Each property has a type, description, and optionally a default value and required flag.
- **`interfaces`**: Named groups of operations. `dell.interfaces.lifecycle` is the standard lifecycle with operations like `create`, `start`, `stop`, `delete`.
- **`implementation`**: Points to the plugin function that executes the operation, in the format `plugin_name.module.function`.
- **`runtime_properties`**: Values that are set during execution and can be read by other nodes using `get_attribute`.

### Lifecycle Operations

The standard lifecycle follows this order during **install**:

1. `precreate` -- validation and preparation
2. `create` -- create the resource
3. `configure` -- configure the resource
4. `start` -- start/activate the resource
5. `poststart` -- post-start actions (e.g., store facts, gather info)

During **uninstall**, the reverse operations execute:

1. `prestop` -- pre-stop actions
2. `stop` -- stop the resource
3. `delete` -- delete/remove the resource
4. `postdelete` -- cleanup

Additional operations like `check_drift`, `update`, `restart`, `suspend`, and `resume` can be triggered via workflows outside the install/uninstall cycle.

### Custom Node Types in Blueprints

Blueprints don't typically define new node types -- they use the types provided by plugins. However, you can define custom types if needed:

```yaml
node_types:
  my.custom.Type:
    derived_from: dell.nodes.Root
    properties:
      my_setting:
        type: string
        default: "hello"
```

---

## Node Templates

Node templates are **instances** of node types -- the actual resources you want to create. This is where you configure specific values for properties and wire nodes together with relationships.

### Basic Structure

```yaml
node_templates:
  my_vm:
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
    relationships:
      - type: dell.relationships.depends_on
        target: cloudinit
      - type: dell.relationships.depends_on
        target: binary_image
```

### Key Concepts

- **`type`**: References a node type (from a plugin or defined in the blueprint)
- **`properties`**: Provides values for the node type's property schema. Values can be literals or intrinsic functions.
- **`relationships`**: Defines dependencies on other node templates. See [Relationships](#relationships).
- **`interfaces`**: Optionally override or extend the node type's interface operations with custom implementations or inputs.

### Overriding Interfaces

You can override a node type's default interface implementations:

```yaml
node_templates:
  binary_image:
    type: dell.nodes.nativeedge.template.BinaryImage
    properties:
      binary_image_config:
        artifact:
          path: { get_input: binary_image_url }
    interfaces:
      dell.interfaces.lifecycle:
        # Override precreate with explicit implementation
        precreate:
          implementation: edge.nativeedge_plugin.tasks.validate_binary_image_config
        # Add a delete operation not in the default type
        delete:
          implementation: edge.nativeedge_plugin.tasks.delete_binary
```

### Inline Script Implementations

Operations can use inline Python scripts instead of plugin references:

```yaml
node_templates:
  prepare_compose_yaml:
    type: dell.nodes.Root
    interfaces:
      dell.interfaces.lifecycle:
        start: |
          import sys
          from nativeedge import ctx

          filename = ctx.download_resource("container_based/compose/container_compose.yaml")
          with open(filename) as f:
            content = f.read()

          ctx.instance.runtime_properties["compose_yaml"] = content
          ctx.instance.update()
```

### External Script Implementations

Operations can also reference external script files bundled with the blueprint:

```yaml
node_templates:
  prepare_config:
    type: dell.nodes.ApplicationModule
    interfaces:
      dell.interfaces.lifecycle:
        precreate:
          implementation: vm/scripts/prepare_network_settings.py
          executor: central_deployment_agent
          inputs:
            network_segments:
              - { get_input: vnic_0_segment_name }
```

Note the `executor: central_deployment_agent` is required when using external scripts to specify where the script runs.

---

## Relationships

Relationships define dependencies and data flow between node templates. They control the **order** in which nodes are created and allow nodes to exchange data.

### `dell.relationships.depends_on`

The most common relationship. It ensures the target node is fully created before the source node begins its lifecycle.

```yaml
node_templates:
  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      # Uses data from cloudinit and binary_image nodes
      vm_config:
        image: { get_attribute: [binary_image, binary_details, extra, artifact_id] }
        cloudinit: { get_attribute: [cloudinit, cloud_config] }
    relationships:
      - type: dell.relationships.depends_on
        target: cloudinit
      - type: dell.relationships.depends_on
        target: binary_image
      - type: dell.relationships.depends_on
        target: prepare_config
```

In this example, the `vm` node will not begin creating until `cloudinit`, `binary_image`, and `prepare_config` have all completed their lifecycle. This is essential because the VM needs data produced by those nodes (the cloud-init config string, the uploaded image ID, and the network settings).

### `dell.relationships.connected_to`

Indicates a logical connection between two nodes, establishing dependency ordering similar to `depends_on`.

### Custom Relationships

Plugins can define relationships with their own interface operations. For example, the utilities plugin provides:

```yaml
# Loads configuration from a ConfigurationLoader into the source node's runtime properties
- type: dell.relationships.load_from_config
  target: configuration_loader

# Reserves an item from a resource list
- type: dell.relationships.resources.reserve_list_item
  target: resource_list
```

### Relationship Interfaces

Relationships can have their own lifecycle operations that execute during the relationship establishment:

- `preconfigure` -- runs on the target node before the source node's `configure`
- `postconfigure` -- runs after configure
- `establish` -- runs during relationship establishment
- `unlink` -- runs when the relationship is removed

---

## Capabilities (Outputs)

Capabilities (also called outputs) expose values from the deployment that can be viewed by users or consumed by other deployments. They are defined under the `capabilities` key.

```yaml
capabilities:
  service_tag:
    description: Service Tag
    value: { get_property: [vm, vm_config, location] }

  vm_name:
    description: Virtual Machine Name
    value: { get_attribute: [vm, vm_details, name] }

  vm_ip:
    description: Virtual Machine IP
    value: { get_attribute: [vm_info, capabilities, vm_public_ip] }

  vm_ssh_private_key:
    description: Virtual Machine SSH Private Key
    value:
      concat:
        - get_attribute: [ssh_keys, secret_key_name]
        - "_private"
```

### Consuming Capabilities from Other Deployments

Other blueprints can reference a deployment's capabilities using `dell.nodes.SharedResource`:

```yaml
node_templates:
  vm_platform:
    type: dell.nodes.SharedResource
    properties:
      resource_config:
        deployment:
          id: { get_input: vm_deployment_id }
```

Then access the parent deployment's capabilities:

```yaml
host_string: { get_attribute: [vm_platform, capabilities, vm_name] }
user: { get_attribute: [vm_platform, capabilities, vm_user_name] }
```

This pattern lets you build **layered deployments** -- one blueprint creates the VM, another deploys applications onto it.

### Dynamic Outputs with JMESPath

Capabilities can use JMESPath expressions to query NativeEdge inventory data:

```yaml
capabilities:
  node_red_url:
    description: Node-Red URL
    value:
      concat:
        - "http://"
        - jmespath:
            - extra.hw_core.network.hostNetwork[?isDefault==`true`].ipAddress | [0] | to_string(@)
            - get_inventory:
                get_environment_capability: ece_service_tag
        - ":1880"
```

---

## Labels and Blueprint Labels

Labels are key-value metadata attached to deployments for categorization, filtering, and search.

### Deployment Labels

Applied to each deployment instance created from the blueprint:

```yaml
labels:
  csys-obj-type:
    values:
      - environment
  target_environment:
    values:
      - nativeedge
  vendor:
    values:
      - DELL
  solution:
    values:
      - POC-VM
  version:
    values:
      - "3.2.0"
```

### Blueprint Labels

Applied to the blueprint definition itself:

```yaml
blueprint_labels:
  env:
    values:
      - ned
```

Labels can be used in the NativeEdge UI to filter and organize blueprints and deployments.

---

## Intrinsic Functions

Intrinsic functions are special expressions evaluated by the orchestrator at deployment time. They allow dynamic values that are resolved from inputs, secrets, other nodes, and the platform.

### `get_input`

Retrieves the value of a blueprint input:

```yaml
name: { get_input: vm_hostname }
```

### `get_secret`

Retrieves a value from the NativeEdge secret store:

```yaml
# Simple secret retrieval
password: { get_secret: my_secret_name }

# Nested secret retrieval (secret contains a dict)
url:
  get_secret:
    - get_input: poc_os_image_secret
    - binary_image_url
```

### `get_property`

Retrieves a property value from a node template (resolved at deployment planning time):

```yaml
value: { get_property: [vm, vm_config, location] }
```

### `get_attribute`

Retrieves a runtime attribute from a node instance (resolved during execution):

```yaml
image: { get_attribute: [binary_image, binary_details, extra, artifact_id] }
cloud_config: { get_attribute: [cloudinit, cloud_config] }
```

`get_attribute` can traverse nested structures using a list of keys.

### `get_environment_capability`

Retrieves a capability from the deployment's environment (the NativeEdge endpoint):

```yaml
location: { get_environment_capability: ece_service_tag }
```

### `get_sys`

Retrieves system-level information:

```yaml
name: { get_sys: [deployment, name] }
```

### `concat`

Concatenates strings:

```yaml
value:
  concat:
    - get_attribute: [ssh_keys, secret_key_name]
    - "_private"
```

### `merge`

Merges dictionaries:

```yaml
resource_config:
  merge:
    - { get_attribute: [SELF, resource_config] }
    - { get_input: cloudinit }
```

### `jmespath`

Queries structured data using JMESPath expressions (commonly used with `get_inventory`):

```yaml
constraints:
  - valid_values:
      jmespath:
        - extra.hw_core.network.virtualSegment[?type=='NAT'||type=='BRIDGE'].name
        - get_inventory:
            get_environment_capability: ece_service_tag
```

### `get_inventory`

Retrieves inventory data for a NativeEdge endpoint, identified by its service tag:

```yaml
get_inventory:
  get_environment_capability: ece_service_tag
```

Typically used inside `jmespath` expressions to query device hardware, network, and storage information.

---

## Modular Blueprint Pattern

NativeEdge blueprints commonly use a **modular file structure** where the main entry point imports sub-files, each responsible for a portion of the blueprint. This keeps large blueprints manageable and enables reuse.

### Typical File Layout

```
my_blueprint/
  blueprint.yaml          # Main entry point
  vm/
    definitions.yaml      # node_templates (the resources)
    inputs.yaml           # inputs (user parameters)
    outputs.yaml          # capabilities (deployment outputs)
    scripts/              # Helper scripts referenced by nodes
    templates/            # Jinja/YAML templates
  app/
    definitions.yaml
    inputs.yaml
    outputs.yaml
    playbooks/            # Ansible playbooks
```

### How It Works

The main entry point imports sub-files and adds blueprint-level configuration:

```yaml
tosca_definitions_version: dell_1_0

description: >
  Deploy a VM with an application.

imports:
  - dell/types/types.yaml
  - vm/definitions.yaml      # node_templates for VM creation
  - vm/inputs.yaml           # inputs for VM configuration
  - vm/outputs.yaml          # outputs exposing VM details
  - app/definitions.yaml     # node_templates for application
  - app/inputs.yaml          # inputs for application
  - app/outputs.yaml         # outputs exposing application details

input_groups:
  # UI grouping for all inputs from both sub-files
  vm:
    display_label: Virtual Machine
    inputs:
      - vm_hostname
      - vcpus
  app:
    display_label: Application
    inputs:
      - app_setting

labels:
  csys-obj-type:
    values:
      - environment
```

### Important Rules

- Each sub-file must also declare `tosca_definitions_version: dell_1_0`
- Sub-files that define `node_templates` must import the plugins they use
- The `inputs` key is merged across files -- input names must be unique
- The `capabilities` key is merged across files -- capability names must be unique
- The `node_templates` key is merged across files -- node template names must be unique

### Benefits

- **Reuse**: The `vm/` files can be imported by multiple blueprints
- **Separation of concerns**: Inputs, definitions, and outputs are in separate files
- **Composability**: The MQTT + Node-RED example imports both `vm/` and `mqtt_nodered/` sub-files to compose a VM with applications
- **Maintainability**: Changes to VM logic only need to happen in one place

---

## Next Steps

- Explore the [plugin documentation](../index.md) for details on each plugin's node types, data types, and operations
- Walk through the [blueprint examples](../examples/overview.md) to see these concepts applied in practice
