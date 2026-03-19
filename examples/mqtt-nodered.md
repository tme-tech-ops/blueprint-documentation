# MQTT + Node-RED Example

This is the most complex example in the collection. It combines VM provisioning (from the [Simple VM](simple-vm.md) example) with application deployment using **Ansible playbooks**. The result is a single blueprint that creates a VM, installs Docker on it, and deploys MQTT and Node-RED containers -- all automated end to end.

This example demonstrates two advanced patterns:
1. **Composing sub-file sets** -- importing sub-files from both the `vm/` and `mqtt_nodered/` directories into a single blueprint
2. **Ansible plugin integration** -- using DSL definitions, inventory proxy settings, and run data to execute Ansible playbooks on NativeEdge-managed VMs

## Prerequisites

- [Blueprint Anatomy](../blueprint-structure/blueprint-anatomy.md) -- understand blueprint structure
- [Simple VM Example](simple-vm.md) -- this example reuses all VM sub-files; understand them first
- [Ansible Plugin](../ansible-plugin/overview.md) -- the `dell.nodes.ansible.Executor` node type
- [Edge Plugin: Virtual Machines](../edge-plugin/virtual-machines.md) -- VM provisioning
- [Utilities Plugin](../utilities-plugin/overview.md) -- SSH keys and cloud-init

---

## File Structure

```
mqtt_nodered_example_for_NED.yaml    <-- Entry point
vm/                                  <-- Reused from Simple VM example
  definitions.yaml                   <-- VM node templates (6 nodes)
  inputs.yaml                        <-- VM input declarations
  outputs.yaml                       <-- VM output capabilities
  scripts/                           <-- VM lifecycle scripts
  templates/                         <-- Netplan template
mqtt_nodered/                        <-- New: application-specific files
  definitions.yaml                   <-- Ansible node templates (2 nodes)
  inputs.yaml                        <-- Ansible and registry inputs
  outputs.yaml                       <-- MQTT and Node-RED URLs
  playbooks/
    install_docker.yaml              <-- Ansible playbook: Docker installation
    deploy_mqtt_nodered.yaml         <-- Ansible playbook: container deployment
```

The key insight is that the `vm/` directory is **shared** between this example and the Simple VM example. The files are identical. The `mqtt_nodered/` directory adds the application layer on top.

---

## The Entry Point

```yaml
tosca_definitions_version: dell_1_0

description: >
  This blueprint creates a Virtual Machine on a NativeEdge Endpoint and
  deploys MQTT and Node-Red containers and automates setting up a simple
  Node-Red flow.

imports:
  - dell/types/types.yaml
  - vm/definitions.yaml
  - vm/inputs.yaml
  - vm/outputs.yaml
  - mqtt_nodered/definitions.yaml
  - mqtt_nodered/inputs.yaml
  - mqtt_nodered/outputs.yaml
```

**Key concept -- composing multiple sub-file sets:**

This blueprint imports **seven** files: the base types, three files from `vm/`, and three files from `mqtt_nodered/`. The NativeEdge orchestrator merges all of them into a single blueprint. This means:

- All node templates from `vm/definitions.yaml` (ssh_keys, binary_image, prepare_config, cloudinit, vm, vm_info) are combined with node templates from `mqtt_nodered/definitions.yaml` (install_docker, deploy_mqtt_nodered).
- All inputs from both `vm/inputs.yaml` and `mqtt_nodered/inputs.yaml` are available.
- All outputs from both `vm/outputs.yaml` and `mqtt_nodered/outputs.yaml` are exposed.

This is the **composition pattern** -- you build complex blueprints by combining reusable sub-file sets. The VM sub-files do not need to know about the MQTT/Node-RED sub-files, and vice versa. They communicate only through node names and attributes.

### Input Groups

The input groups combine VM-related groups with Ansible-specific groups:

```yaml
input_groups:
  artifact:
    display_label: Deployment Artifacts
    collapsible: false
    index: 0
    inputs:
      - poc_os_image_secret
  registry:
    display_label: Registry Configuration
    collapsible: true
    index: 1
    inputs:
      - use_docker_authentication
      - registry_url
      - registry_credentials
  vm:
    display_label: NativeEdge Virtual Machine
    collapsible: true
    index: 2
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
    index: 3
    inputs:
      # ... same as Simple VM
  network:
    display_label: Network Configuration
    collapsible: true
    index: 4
    inputs:
      # ... same as Simple VM
  ansible_settings:
    display_label: Ansible Settings
    collapsible: true
    index: 5
    inputs:
      - ansible_debug_level
      - ansible_timeout
      - ansible_log_stdout
```

Two new groups appear that are not in the Simple VM example:
- **Registry Configuration** -- Docker registry authentication settings (for pulling container images from private registries)
- **Ansible Settings** -- debug level, timeout, and log output settings for Ansible execution

---

## Node Templates (mqtt_nodered/definitions.yaml)

This is the core of the MQTT + Node-RED example. It defines two Ansible executor nodes and uses several advanced YAML and blueprint patterns.

### Plugin Imports

```yaml
imports:
  - dell/types/types.yaml
  - plugin:edge-plugin
  - plugin:utilities-plugin?version=>=3.1.3.0
  - plugin:fabric-plugin?version=>=3.3.2.0
  - plugin:ansible-plugin?version=>=4.1.2.0
```

This is the only example that imports all four plugins. The `ansible-plugin` is new here, providing the `dell.nodes.ansible.Executor` node type.

### DSL Definitions

DSL definitions are YAML anchors and aliases that allow you to define reusable configuration blocks. They are not NativeEdge-specific -- they use standard YAML anchor (`&`) and alias (`*`) syntax.

```yaml
dsl_definitions:

  ansible_env_vars: &ansible_default_env_vars
    ANSIBLE_HOST_KEY_CHECKING: "False"
    ANSIBLE_INVALID_TASK_ATTRIBUTE_FAILED: "False"
    ANSIBLE_STDOUT_CALLBACK: json
    ANSIBLE_DEPRECATION_WARNINGS: "False"

  ansible_common_properties: &ansible_common_properties
    ansible_external_venv: /opt/ansible
    log_stdout: { get_input: ansible_log_stdout }
    store_facts: false
    timeout: { get_input: ansible_timeout }
    debug_level: { get_input: ansible_debug_level }
    number_of_attempts: 10

  ansible_host_common: &ansible_host_common
    ansible_host: { get_attribute: [SELF, inventory_proxy_settings, vm, ip] }
    ansible_port: { get_attribute: [SELF, inventory_proxy_settings, vm, port] }
    ansible_user: { get_input: vm_user_name }
    ansible_ssh_common_args: -o StrictHostKeyChecking=no -o ControlPersist=yes
    ansible_ssh_private_key_file:
      get_secret:
        concat:
          - get_attribute: [ ssh_keys, secret_key_name ]
          - "_private"
    ansible_become: True
```

Let's break down each DSL definition:

**`ansible_default_env_vars`** -- environment variables passed to the Ansible process:
- `ANSIBLE_HOST_KEY_CHECKING: "False"` -- disables SSH host key verification (necessary because VMs are dynamically created)
- `ANSIBLE_INVALID_TASK_ATTRIBUTE_FAILED: "False"` -- prevents failures on unrecognized task attributes
- `ANSIBLE_STDOUT_CALLBACK: json` -- outputs Ansible results in JSON format for parsing
- `ANSIBLE_DEPRECATION_WARNINGS: "False"` -- suppresses deprecation warnings

**`ansible_common_properties`** -- properties shared by all Ansible executor nodes:
- `ansible_external_venv: /opt/ansible` -- path to the Ansible virtual environment on the NativeEdge manager
- `log_stdout` -- whether to log Ansible output to deployment logs (controlled by user input)
- `store_facts` -- whether to store Ansible gathered facts (disabled for performance)
- `timeout` -- maximum execution time in seconds (default 3600 = 1 hour)
- `debug_level` -- Ansible verbosity level (0-3)
- `number_of_attempts: 10` -- retry count for failed tasks

**`ansible_host_common`** -- Ansible inventory host variables shared by all hosts:
- `ansible_host` -- the IP address to connect to, read from the `inventory_proxy_settings` (explained below)
- `ansible_port` -- the SSH port, also from `inventory_proxy_settings`
- `ansible_user` -- the SSH username from blueprint inputs
- `ansible_ssh_common_args` -- disables strict host key checking and enables connection persistence
- `ansible_ssh_private_key_file` -- the SSH private key, retrieved from the secret store using the same dynamic name pattern as the Simple VM example
- `ansible_become: True` -- enables privilege escalation (sudo) for all tasks

### Dependency Chain

Including the VM nodes from `vm/definitions.yaml`, the full dependency chain is:

```
ssh_keys ───────────┐
                    v
prepare_config ──> cloudinit ──> vm ──> vm_info ──> install_docker ──> deploy_mqtt_nodered
                    ^            ^
binary_image ───────┘────────────┘
```

The two new Ansible nodes extend the Simple VM chain: `install_docker` runs after `vm_info` (after the VM is created and its info is gathered), and `deploy_mqtt_nodered` runs after Docker is installed.

---

### Node 1: `install_docker` -- Install Docker Engine

```yaml
install_docker:
  type: dell.nodes.ansible.Executor
  properties:
    <<: *ansible_common_properties
    playbook_path: mqtt_nodered/playbooks/install_docker.yaml
    inventory_proxy_settings:
      vm:
        ip: { get_attribute: [vm, vm_details, name] }
        port: 22
        proxy_settings:
            get_attribute: [ vm_info, connection_proxy_settings ]
    sources:
      all:
        hosts:
          vm: *ansible_host_common
    ansible_env_vars:
      <<: *ansible_default_env_vars
  relationships:
    - type: dell.relationships.depends_on
      target: vm_info
```

**What it does:** Runs an Ansible playbook that installs Docker Engine on the newly created VM.

**Key concept -- `dell.nodes.ansible.Executor`:**

This node type runs an Ansible playbook. It requires:
- `playbook_path` -- the path to the playbook file within the blueprint package
- `sources` -- the Ansible inventory (which hosts to target)
- `ansible_env_vars` -- environment variables for the Ansible process

**Key concept -- `<<:` merge key (YAML anchor expansion):**

```yaml
properties:
  <<: *ansible_common_properties
  playbook_path: mqtt_nodered/playbooks/install_docker.yaml
```

The `<<: *ansible_common_properties` syntax merges all key-value pairs from the `ansible_common_properties` anchor into this properties map. This is equivalent to writing out all the common properties (ansible_external_venv, log_stdout, store_facts, timeout, debug_level, number_of_attempts) explicitly. The `playbook_path` is then added as an additional property.

**Key concept -- `inventory_proxy_settings`:**

```yaml
inventory_proxy_settings:
  vm:
    ip: { get_attribute: [vm, vm_details, name] }
    port: 22
    proxy_settings:
        get_attribute: [ vm_info, connection_proxy_settings ]
```

This is specific to the Ansible plugin's integration with NativeEdge. The `inventory_proxy_settings` property configures the SSH proxy tunnel that Ansible will use to reach the VM. It is a map where each key (`vm` in this case) corresponds to a host in the Ansible inventory.

For each host:
- `ip` -- the target VM's name (used as the SSH target through the proxy)
- `port` -- the SSH port
- `proxy_settings` -- the NativeEdge proxy connection settings (from `vm_info`)

The Ansible plugin uses these settings to establish an SSH tunnel through the NativeEdge endpoint agent, then routes Ansible's SSH connections through that tunnel. The actual host/port that Ansible connects to are dynamically computed and exposed via `get_attribute: [SELF, inventory_proxy_settings, vm, ip]` and `get_attribute: [SELF, inventory_proxy_settings, vm, port]` -- which is exactly what the `ansible_host_common` DSL definition uses.

**Key concept -- `sources` (Ansible inventory):**

```yaml
sources:
  all:
    hosts:
      vm: *ansible_host_common
```

This defines the Ansible inventory in the standard Ansible format. It creates a host named `vm` in the `all` group with the variables from `ansible_host_common`. The `*ansible_host_common` alias expands to all the host variables defined in the DSL definitions.

The expanded inventory looks like:

```yaml
sources:
  all:
    hosts:
      vm:
        ansible_host: <proxy_ip>       # Resolved at runtime
        ansible_port: <proxy_port>     # Resolved at runtime
        ansible_user: pocuser
        ansible_ssh_common_args: -o StrictHostKeyChecking=no -o ControlPersist=yes
        ansible_ssh_private_key_file: <private_key_content>
        ansible_become: True
```

---

### Node 2: `deploy_mqtt_nodered` -- Deploy MQTT and Node-RED Containers

```yaml
deploy_mqtt_nodered:
  type: dell.nodes.ansible.Executor
  properties: &deploy_mqtt_nodered_inputs
    <<: *ansible_common_properties
    playbook_path: mqtt_nodered/playbooks/deploy_mqtt_nodered.yaml
    inventory_proxy_settings:
      vm:
        ip: { get_attribute: [vm, vm_details, name] }
        port: 22
        proxy_settings:
            get_attribute: [ vm_info, connection_proxy_settings ]
    sources:
      all:
        hosts:
          vm: *ansible_host_common
    ansible_env_vars:
      <<: *ansible_default_env_vars
    run_data:
      use_docker_authentication: { get_input: use_docker_authentication }
      registry_url: { get_input: registry_url }
      registry_username: { get_input: registry_username }
      registry_password: { get_input: registry_password }
  interfaces:
    dell.interfaces.lifecycle:
      start:
        implementation: ansible.plugins_ansible.tasks.run
        inputs:
          <<: *deploy_mqtt_nodered_inputs
  relationships:
    - type: dell.relationships.depends_on
      target: install_docker
```

**What it does:** Runs an Ansible playbook that deploys MQTT broker and Node-RED containers using Docker, with optional private registry authentication.

**Key concept -- `run_data` (passing variables to playbooks):**

```yaml
run_data:
  use_docker_authentication: { get_input: use_docker_authentication }
  registry_url: { get_input: registry_url }
  registry_username: { get_input: registry_username }
  registry_password: { get_input: registry_password }
```

The `run_data` property passes variables to the Ansible playbook as extra variables (`--extra-vars`). Inside the playbook, these are accessible as standard Ansible variables (e.g., `{{ use_docker_authentication }}`). This is how blueprint inputs flow into Ansible playbooks.

The registry-related variables enable the playbook to optionally log into a private Docker registry before pulling container images. The `registry_username` and `registry_password` are hidden inputs that derive their values from a structured secret:

```yaml
# From mqtt_nodered/inputs.yaml
registry_username:
  type: string
  hidden: true
  default:
    get_secret:
    - get_input: registry_credentials
    - username

registry_password:
  type: string
  hidden: true
  default:
    get_secret:
    - get_input: registry_credentials
    - password
```

This uses the same structured secret pattern as the Simple VM example's binary image credentials.

**Key concept -- explicit lifecycle implementation with YAML anchor reuse:**

```yaml
properties: &deploy_mqtt_nodered_inputs
  <<: *ansible_common_properties
  playbook_path: ...
  # ... all properties
interfaces:
  dell.interfaces.lifecycle:
    start:
      implementation: ansible.plugins_ansible.tasks.run
      inputs:
        <<: *deploy_mqtt_nodered_inputs
```

This node uses a YAML anchor (`&deploy_mqtt_nodered_inputs`) on the properties block and then references it in the `start` lifecycle interface inputs. This means the same configuration is used for both the default create operation and the explicit start operation.

The `implementation: ansible.plugins_ansible.tasks.run` tells the orchestrator to use the Ansible plugin's `run` task. While the `install_docker` node relies on the default lifecycle behavior of the `dell.nodes.ansible.Executor` type, this node explicitly defines the `start` interface. This pattern is useful when you want the Ansible playbook to run during a specific lifecycle phase.

---

## Inputs (mqtt_nodered/inputs.yaml)

The MQTT + Node-RED specific inputs fall into two categories:

### Ansible Settings

```yaml
ansible_debug_level:
  type: integer
  default: 0
  description: |
    Ansible Debug Level (0 to 3)

ansible_timeout:
  type: integer
  default: 3600
  description: |
    Ansible Timeout value (in seconds)

ansible_log_stdout:
  type: boolean
  default: false
  description: |
    Enable ansible log stdout output to deployment logs
```

These control the Ansible execution environment. The defaults are suitable for most deployments:
- Debug level 0 (minimal output)
- 1-hour timeout (Docker installation can take several minutes)
- stdout logging disabled (enable for troubleshooting)

### Registry Configuration

```yaml
use_docker_authentication:
  type: boolean
  default: false
  description: |
    Use Docker Authentication

registry_url:
  type: string
  only_with: use_docker_authentication
  default: "docker.io"
  description: |
    Registry URL for pulling containers (default is docker.io)

registry_credentials:
  type: secret_key
  only_with: use_docker_authentication
  constraints:
  - type: docker_configuration_secret_name
  description: |
    Docker Configuration type secret containing the registry credentials.
```

The registry inputs use the `only_with` pattern -- the URL and credentials fields only appear when `use_docker_authentication` is enabled. The `registry_credentials` input has a `type: docker_configuration_secret_name` constraint, which ensures the selected secret has the Docker configuration structure (containing `username` and `password` keys).

---

## Outputs (mqtt_nodered/outputs.yaml)

```yaml
capabilities:

  mqtt_endpoint:
    description: The MQTT tcp Endpoint
    value:
      concat:
        - 'tcp://'
        - { get_attribute: [vm_info, capabilities, vm_public_ip] }
        - ':'
        - 1883

  nodered_url:
    description: The NodeRed URL
    value:
      concat:
        - 'http://'
        - { get_attribute: [vm_info, capabilities, vm_public_ip] }
        - ':'
        - 1880
```

These outputs are **additional** to the VM outputs from `vm/outputs.yaml`. After deployment, the user sees both the VM outputs (service_tag, vm_name, vm_ip, etc.) and these application outputs.

The MQTT endpoint uses the `tcp://` protocol scheme (MQTT's native transport) on port 1883. The Node-RED URL uses `http://` on port 1880. Both use the VM's public IP address from `vm_info`.

---

## Complete Dependency Chain and Execution Flow

Here is the full execution sequence combining both the VM and application nodes:

1. **Parallel start:** `ssh_keys`, `binary_image`, and `prepare_config` begin (from `vm/definitions.yaml`).
2. `ssh_keys` generates the RSA key pair.
3. `binary_image` uploads the OS image to the endpoint.
4. `prepare_config` prepares network settings, serial ports, and password hash.
5. **`cloudinit`** builds the cloud-init configuration (after ssh_keys and prepare_config complete).
6. **`vm`** creates the virtual machine (after cloudinit, binary_image, and prepare_config complete).
7. **`vm_info`** SSHs into the VM to gather its IP address (after vm completes).
8. **`install_docker`** runs the Docker installation Ansible playbook on the VM (after vm_info completes).
9. **`deploy_mqtt_nodered`** runs the container deployment Ansible playbook (after install_docker completes).
10. **Outputs** are computed from all nodes.

The total execution involves 8 sequential nodes (some running in parallel at the start), making this the longest deployment pipeline in the example collection.

---

## Comparing Approaches: Ansible vs. Fabric vs. NativeEdgeCompose

This example is a good point to compare the three methods for deploying applications:

| Aspect | Ansible (this example) | Fabric (Nginx example) | NativeEdgeCompose (Container example) |
|---|---|---|---|
| Target | VM on NativeEdge endpoint | VM on NativeEdge endpoint | NativeEdge endpoint directly |
| Mechanism | Ansible playbooks over SSH | Shell commands over SSH | Docker Compose via NativeEdge API |
| Idempotency | Built into Ansible | Must be coded in scripts | Built into Docker Compose |
| Complexity | High (DSL defs, inventory, proxy) | Medium (fabric_env, shell_env) | Low (compose template) |
| Best for | Complex multi-step installations | Simple command sequences | Containerized applications |
| Retry support | Built-in (`number_of_attempts`) | Manual | Automatic |
| Variable passing | `run_data` (extra vars) | `shell_env` (env vars) | `environment_variables` |

Choose the approach that matches your application's needs. If your application is already containerized and you do not need a VM, use NativeEdgeCompose. If you need a VM with a complex installation process (like installing Docker and then deploying containers), Ansible provides the most robust automation. For simple scripts that run once, Fabric is the lightest-weight option.
