# Ansible Executor Node Type

This document is the complete reference for the `dell.nodes.ansible.Executor` node type provided by the NativeEdge Ansible Plugin. This node type allows you to run Ansible playbooks against remote hosts as part of a NativeEdge blueprint deployment.

## Prerequisites

- The Ansible Plugin must be imported in your blueprint (see [Overview](./overview.md)).
- You need an Ansible playbook (a YAML file) accessible to the blueprint, either bundled inside the blueprint archive or downloadable from a URL.
- Target hosts must be reachable from the NativeEdge orchestrator over SSH (Linux) or WinRM (Windows).

---

## Node Type Definition

```
dell.nodes.ansible.Executor
  derived_from: dell.nodes.Root
```

The `dell.nodes.ansible.Executor` type inherits from `dell.nodes.Root`, which is the base type for all NativeEdge node types. This means it participates in standard NativeEdge lifecycle operations (install, uninstall) and supports relationships, runtime properties, and all other standard NativeEdge features.

---

## Properties

The node type's properties come from two sources:

1. **Playbook configuration properties** (`playbook_config`) -- shared with the `reload_ansible_playbook` workflow.
2. **Node-specific properties** -- only available on the node type itself.

### Playbook Configuration Properties

These properties control how the Ansible playbook is discovered, configured, and executed. They originate from the `&playbook_config` DSL definition anchor in the plugin YAML.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `ansible_external_venv` | string | No | `""` | Path to an existing Ansible Python virtual environment with Ansible and any extra packages pre-installed. When set, the plugin uses this venv instead of creating its own. |
| `ansible_playbook_executable_path` | string | No | `""` | Full filesystem path to a custom `ansible-playbook` executable. When empty, the plugin uses the executable bundled with the plugin or the one found in `ansible_external_venv`. |
| `extra_packages` | list | No | `[]` | List of Python package names (pip-installable) to install into the controller virtual environment before running the playbook. Requires internet connectivity on the orchestrator. |
| `galaxy_collections` | list | No | `[]` | List of Ansible Galaxy collection names to install before running the playbook. Requires internet connectivity on the orchestrator. |
| `roles` | list | No | `[]` | List of Ansible role names to install before running the playbook. Requires internet connectivity on the orchestrator. |
| `module_path` | string | No | `""` | Filesystem path on the orchestrator where Ansible modules are (or will be) installed. If empty, the plugin creates this directory automatically. The `ansible-ctx` module is always installed by default. |
| `playbook_source_path` | string | No | `""` | A full filesystem path or URL pointing to a directory or archive that contains the playbook specified in `playbook_path`. If this is a URL to an archive, `playbook_path` is treated as a relative path inside the archive. |
| `playbook_path` | string | No | `""` | Path to the playbook entry file (for example `site.yaml` or `main.yaml`). This is relative to the blueprint root directory, or relative to `playbook_source_path` if that property is set. If `playbook_source_path` is a URL to an archive, this path is relative to the root of the extracted archive. |
| `additional_playbook_files` | list | No | `[]` | List of blueprint resource paths to download into the playbook directory before execution. If you use this, you must list every file you expect to be available. |
| `sources` | dict | No | *(none)* | Your Ansible inventory. Can be a YAML dictionary (inline inventory) or a path to an inventory file. If not provided, the inventory is taken from the `sources` runtime property. **Note:** Proxy resolution is not performed for inventories provided as a file path. |
| `run_data` | dict | No | `{}` | A dictionary of variable values passed to the playbook (equivalent to Ansible `--extra-vars`). |
| `sensitive_keys` | list | No | `["ansible_password"]` | A list of dictionary key names whose values should be obscured in logs and NativeEdge events. |
| `options_config` | dict | No | `{}` | A dictionary of Ansible command-line options such as `tags` or `skip_tags`. These are passed directly to the `ansible-playbook` command. |
| `ansible_env_vars` | dict | No | See below | A dictionary of environment variables to set when running the playbook. |
| `debug_level` | integer | No | `2` | Ansible verbosity level (maps to the `-v` flags; `2` means `-vv`). |
| `log_stdout` | boolean | No | `true` | Whether to include playbook stdout in the NativeEdge execution event log. Set to `false` to speed up long-running playbooks. |
| `additional_args` | string | No | `""` | Any additional command-line arguments to pass to `ansible-playbook`, for example `"-c local"`. |
| `start_at_task` | string | No | `""` | Name of the task to start the playbook at (maps to `--start-at-task`). |
| `scp_extra_args` | string | No | `""` | Extra arguments passed only to `scp` (for example `-l`). |
| `sftp_extra_args` | string | No | `""` | Extra arguments passed only to `sftp` (for example `-f`, `-l`). |
| `ssh_common_args` | string | No | `""` | Arguments passed to `sftp`, `scp`, and `ssh` (for example a `ProxyCommand`). |
| `ssh_extra_args` | string | No | `""` | Extra arguments passed only to `ssh` (for example `-R`). |
| `timeout` | string | No | `"10"` | SSH connection timeout in seconds. Overrides the default Ansible connection timeout. |
| `remerge_sources` | boolean | No | `false` | When `true`, re-evaluates and updates the `sources` inventory on the target node before running the playbook. |
| `ansible_become` | boolean | No | `false` | Whether to run playbook tasks with elevated (sudo) privileges on the remote host. |
| `tags` | list | No | `[]` | An ordered list of Ansible tags to execute. When provided, `auto_tags` is ignored. |
| `auto_tags` | boolean | No | `false` | When `true`, the plugin reads the playbook and automatically generates a list of tags to execute. This is ignored if `tags` is non-empty. |
| `number_of_attempts` | integer | No | `3` | Total number of times the plugin will attempt to execute the playbook if the exit code indicates unreachable hosts or failure. |
| `store_facts` | boolean | No | `true` | Whether to store Ansible gathered facts as NativeEdge runtime properties after playbook execution. |
| `facts_mapping` | dict | No | `{}` | A dictionary mapping `run_data` keys to Ansible facts keys, used to enable drift detection. The key is the `run_data` key; the value is the corresponding facts key. |

**Default `ansible_env_vars`:**

```yaml
ansible_env_vars:
  ANSIBLE_HOST_KEY_CHECKING: "False"
  ANSIBLE_INVALID_TASK_ATTRIBUTE_FAILED: "False"
  ANSIBLE_STDOUT_CALLBACK: json
```

These defaults disable SSH host key checking (which would block non-interactive execution), prevent Ansible from failing on invalid task attributes (a behavioral change in Ansible 2.8+), and set the stdout callback to JSON so the plugin can parse output reliably.

### Node-Specific Properties

These properties exist only on the `dell.nodes.ansible.Executor` node type and are **not** available as workflow parameters.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `installation_source` | string | No | `""` | If the plugin needs to install Ansible itself, this value is passed as the argument to `pip install`. For example, `"ansible==2.9.27"`. Leave empty to use the version bundled with the plugin. |
| `log_runner_stdout` | boolean | No | `false` | When `true`, logs all stdout from the Ansible runner (more verbose than `log_stdout`). |
| `connectivity_test_url` | string | No | `"https://galaxy.ansible.com"` | A URL the plugin attempts to reach before downloading any packages, collections, or roles. This verifies that the orchestrator has appropriate network connectivity. |
| `retry_process_exception` | boolean | No | `true` | Most process exceptions are fatal. When `true` and the exception is expected, the plugin will retry the operation. |
| `use_system_tmp` | boolean | No | `false` | When `true`, uses the system temporary directory instead of the deployment directory for temporary files. Recommended for large deployments that will be uninstalled and reinstalled. |
| `inventory_proxy_settings` | dict | No | `{}` | Maps inventory hostnames to connection details (IP/port) and proxy settings. The resolved values are stored in runtime properties so they can be referenced from the `sources` property using intrinsic functions. See [Inventory and Proxy Settings](#inventory-and-proxy-settings) for details. |
| `proxy_settings` | dell.datatypes.ansible.ProxySettings | No | *(none)* | Global proxy settings for the node. See [ProxySettings Data Type](#proxysettings-data-type). |

---

## ProxySettings Data Type

**Type name:** `dell.datatypes.ansible.ProxySettings`

This data type configures how the Ansible Plugin connects to target hosts through NativeEdge proxy infrastructure. It is used by the `proxy_settings` property on the Executor node type.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `disable` | boolean | No | `false` | When `true`, disables all proxy settings and uses a direct connection instead. |
| `interface_type` | string | Yes | `"OOB"` | The network interface to use for the connection. `OOB` = Out of Band (default). `INB` = In Band. |
| `enable_socks5` | boolean | No | `false` | When `true`, enables SOCKS5 proxy. When `false`, the plugin uses the legacy proxy or a direct connection. Defaults to `false` for backward compatibility. |
| `auto_resolve` | boolean | No | `true` | When `true`, the plugin automatically resolves proxy settings from the NativeEdge environment. When `false`, you must provide either `target_id` or `oxy_id`. |
| `target_id` | string | No | *(none)* | The ID of the target endpoint. Use this for **managed** (onboarded) assets. The value can be retrieved from target deployment labels. |
| `oxy_id` | string | No | *(none)* | The ID of the proxy server. Use this for **unmanaged** (external) endpoints. The ID can be found in the NativeEdge UI under Orchestrator > Administration > Proxy Servers. |

### Example

```yaml
proxy_settings:
  interface_type: OOB
  enable_socks5: true
  auto_resolve: true
```

---

## Runtime Properties

After the node's lifecycle operations execute, the following runtime properties are set. You can read these from other nodes using `get_attribute` intrinsic functions.

| Runtime Property | Type | Required | Description |
|---|---|---|---|
| `inventory_proxy_settings` | dict | No | The resolved mapping of each inventory host to its IP address, port, and proxy configuration. |
| `playbook_venv` | string | Yes | Filesystem path to the Python virtual environment where Ansible, modules, and collections are installed. |
| `__UPDATE_WINRM` | boolean | No | `true` if the plugin has already moved the Ansible WinRM module into place (Windows targets only). |
| `module_path` | string | No | Filesystem path where Ansible modules are installed. |
| `_blueprint_dir` | string | Yes | Filesystem path to the original blueprint files on the orchestrator. |
| `workspace` | string | No | Root folder for files created or downloaded during playbook preparation, execution, and completion. |
| `installed_ansible_venv_packages` | string | No | A list of Python packages installed in the `playbook_venv`. |
| `local_venv` | string | No | `true` if the node is using a deployment-specific virtual environment (as opposed to a shared one). |
| `___AVAILABLE_STEPS` | list | No | Tags from the `tags` node property that have not yet been executed. Used with `__COMPLETED_TAGS` to track incremental playbook execution. |
| `___COMPLETED_TAGS` | list | No | Tags that have been successfully executed. |
| `KRB5_CONFIG` | string | No | Filesystem path to the Kerberos configuration file generated from node properties (for Kerberos/Active Directory authentication). |
| `sources` | dict | No | The resolved contents of the Ansible inventory. |
| `connected` | boolean | Yes | `true` if the orchestrator was able to reach the `connectivity_test_url`. |
| `installed_galaxy_collections` | string | No | The stdout from `ansible-galaxy collection list`, showing all installed collections. |
| `installed_roles` | string | No | The stdout from `ansible-galaxy list`, showing all installed roles. |
| `COLLECTIONS_PACKAGES_LIST` | list | No | List of Galaxy collections that have been installed. |
| `VENV_PACKAGE_LIST` | list | No | List of Python modules installed in the virtual environment. |
| `VENV_PYTHON` | string | No | Path to the Python executable inside the virtual environment. |
| `VENV_SITE_PACKAGES` | string | No | Path to the `site-packages` directory of the virtual environment. |

---

## Lifecycle Interfaces

The node type implements the `dell.interfaces.lifecycle` interface and a custom `ansible` interface.

### `dell.interfaces.lifecycle`

This interface is executed automatically during the NativeEdge **install** and **uninstall** workflows.

#### `precreate`

**Implementation:** `ansible.plugins_ansible.tasks.precreate`

This operation runs before the node is "created." It:

1. Tests connectivity to `connectivity_test_url`.
2. Creates or locates the Python virtual environment.
3. Installs any packages listed in `extra_packages`.
4. Installs any Galaxy collections listed in `galaxy_collections`.
5. Installs any roles listed in `roles`.
6. Resolves `inventory_proxy_settings` and stores results in runtime properties.

No inputs are required; it reads from node properties directly.

#### `start`

**Implementation:** `ansible.plugins_ansible.tasks.run`

This is the main operation that actually runs the Ansible playbook. It:

1. Prepares the playbook files in a workspace directory.
2. Builds the inventory from `sources` (with proxy resolution if configured).
3. Constructs the `ansible-playbook` command with all configured options (environment variables, extra args, tags, timeout, etc.).
4. Executes the playbook up to `number_of_attempts` times.
5. Logs output to NativeEdge events (unless `log_stdout` is `false`).

**Inputs:** All playbook configuration properties are mapped as operation inputs via the `&playbook_inputs` DSL anchor (see [Playbook Inputs Mapping](#playbook-inputs-mapping)).

#### `poststart`

**Implementation:** `ansible.plugins_ansible.tasks.store_facts`

Runs after the playbook completes. When `store_facts` is `true`, this operation reads Ansible gathered facts from the playbook output and stores them as NativeEdge runtime properties.

**Inputs:** Same mapping as `start`.

#### `delete`

**Implementation:** `ansible.plugins_ansible.tasks.cleanup`

Runs during the **uninstall** workflow. Cleans up:

- Temporary workspace files.
- The deployment-specific virtual environment (if one was created).

No inputs are required.

### `ansible` Interface (Custom)

#### `reload`

**Implementation:** `ansible.plugins_ansible.tasks.run`

This is the same implementation as `start`. It allows you to re-run the playbook outside of the install workflow, typically invoked by the `reload_ansible_playbook` workflow.

**Inputs:** Same mapping as `start`.

---

## Playbook Inputs Mapping

The plugin YAML defines a `&playbook_inputs` DSL anchor that automatically maps node properties to operation inputs. This means you only need to set values once (on the node's `properties`) and they flow into every lifecycle operation automatically.

Here is a simplified view of how it works:

```yaml
dsl_definitions:
  playbook_inputs: &playbook_inputs
    playbook_path:
      default: { get_property: [SELF, playbook_path] }
    sources:
      default: { get_property: [SELF, sources] }
    run_data:
      default: { get_property: [SELF, run_data] }
    # ... one entry per playbook_config property
```

Each key under `&playbook_inputs` corresponds to a property name. Its `default` value uses the `get_property` intrinsic function to read the value from the current node (`SELF`). When the lifecycle operation fires, these inputs are populated automatically unless you override them.

### Overriding Inputs

You can override any of these inputs at the node template level by specifying your own `interfaces` block. This is useful when you want to pass different values for a specific operation. For example:

```yaml
node_templates:
  my_node:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: playbooks/main.yaml
      run_data:
        version: "1.0"
    interfaces:
      dell.interfaces.lifecycle:
        start:
          implementation: ansible.plugins_ansible.tasks.run
          inputs:
            # Override run_data for the start operation only
            run_data:
              version: "2.0"
              extra_flag: true
```

---

## Inventory and Proxy Settings

NativeEdge often manages VMs behind proxies or within private networks that are not directly reachable from the orchestrator. The `inventory_proxy_settings` property solves this by resolving proxy-aware connection details for each host in your inventory.

### How It Works

1. You define `inventory_proxy_settings` on the node, mapping a logical host name to its real IP address, SSH port, and proxy configuration.
2. During `precreate`, the plugin resolves these settings (including SOCKS5 proxy tunnels if configured) and stores the results in the `inventory_proxy_settings` runtime property.
3. In the `sources` property, you reference the resolved values using `get_attribute` intrinsic functions.

### Configuration Pattern

```yaml
node_templates:
  my_ansible_node:
    type: dell.nodes.ansible.Executor
    properties:
      # Step 1: Define proxy settings per host
      inventory_proxy_settings:
        my_host:
          ip: { get_attribute: [vm_node, vm_details, name] }
          port: 22
          proxy_settings:
            get_attribute: [vm_info_node, connection_proxy_settings]

      # Step 2: Reference resolved values in the inventory
      sources:
        all:
          hosts:
            my_host:
              ansible_host: { get_attribute: [SELF, inventory_proxy_settings, my_host, ip] }
              ansible_port: { get_attribute: [SELF, inventory_proxy_settings, my_host, port] }
              ansible_user: my_user
              ansible_ssh_common_args: >-
                -o StrictHostKeyChecking=no
              ansible_ssh_private_key_file: /path/to/key
              ansible_become: true
```

**Key points:**

- `get_attribute: [SELF, inventory_proxy_settings, my_host, ip]` reads the resolved IP from the current node's own runtime properties.
- `get_attribute: [SELF, inventory_proxy_settings, my_host, port]` reads the resolved port (which may be a proxy tunnel port rather than the original `22`).
- The proxy resolution happens transparently -- you do not need to know the proxy details in advance.

---

## Complete Example: MQTT and Node-RED Deployment

The following example is taken from a real NativeEdge blueprint that installs Docker on a VM and then deploys MQTT, Node-RED, and Nginx containers. It demonstrates all the patterns described above.

### DSL Definitions (Reusable Anchors)

```yaml
tosca_definitions_version: dell_1_0

description: >
  Blueprint installs docker engine, then deploys an
  mqtt, nodered, and nginx container.

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin
  - plugin:utilities-plugin?version=>=3.1.3.0
  - plugin:fabric-plugin?version=>=3.3.2.0
  - plugin:ansible-plugin?version=>=4.1.2.0

dsl_definitions:

  # Shared Ansible environment variables
  ansible_env_vars: &ansible_default_env_vars
    ANSIBLE_HOST_KEY_CHECKING: "False"
    ANSIBLE_INVALID_TASK_ATTRIBUTE_FAILED: "False"
    ANSIBLE_STDOUT_CALLBACK: json
    ANSIBLE_DEPRECATION_WARNINGS: "False"

  # Shared properties for all Ansible nodes
  ansible_common_properties: &ansible_common_properties
    ansible_external_venv: /opt/ansible
    log_stdout: { get_input: ansible_log_stdout }
    store_facts: false
    timeout: { get_input: ansible_timeout }
    debug_level: { get_input: ansible_debug_level }
    number_of_attempts: 10

  # Shared host connection settings
  ansible_host_common: &ansible_host_common
    ansible_host: { get_attribute: [SELF, inventory_proxy_settings, vm, ip] }
    ansible_port: { get_attribute: [SELF, inventory_proxy_settings, vm, port] }
    ansible_user: { get_input: vm_user_name }
    ansible_ssh_common_args: -o StrictHostKeyChecking=no -o ControlPersist=yes
    ansible_ssh_private_key_file:
      get_secret:
        concat:
          - get_attribute: [ssh_keys, secret_key_name]
          - "_private"
    ansible_become: True
```

**What each anchor does:**

- `&ansible_default_env_vars` -- Sets environment variables that every Ansible execution in this blueprint should use. Adding `ANSIBLE_DEPRECATION_WARNINGS: "False"` suppresses deprecation warnings that clutter the output.
- `&ansible_common_properties` -- Avoids repeating the same property block on every Ansible node. Uses `ansible_external_venv: /opt/ansible` (a pre-installed venv on the orchestrator), disables fact storage, and allows up to 10 retry attempts.
- `&ansible_host_common` -- Defines how to connect to the target VM. The `ansible_host` and `ansible_port` values are read from the resolved `inventory_proxy_settings` runtime property. The SSH key is retrieved from NativeEdge secrets.

### Node Template: install_docker

```yaml
node_templates:
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
            get_attribute: [vm_info, connection_proxy_settings]
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

**Explanation line by line:**

1. `<<: *ansible_common_properties` -- Merges the shared properties (venv path, timeouts, etc.).
2. `playbook_path` -- Points to the Docker installation playbook inside the blueprint archive.
3. `inventory_proxy_settings` -- Defines a single host named `vm`. The real IP is resolved from the `vm` node's `vm_details.name` attribute. The proxy settings come from the `vm_info` node.
4. `sources` -- The Ansible inventory with one host group (`all`) containing one host (`vm`). The host connection details come from `*ansible_host_common`.
5. `ansible_env_vars` -- Merges the shared environment variable anchor.
6. `relationships` -- Declares that this node depends on `vm_info`, ensuring that the VM information is available before the playbook runs.

### Node Template: deploy_mqtt_nodered

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
            get_attribute: [vm_info, connection_proxy_settings]
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

**Key differences from `install_docker`:**

- `run_data` -- Passes deployment-specific variables (Docker registry credentials) to the playbook as extra variables.
- `properties: &deploy_mqtt_nodered_inputs` -- The entire `properties` block is given an anchor name (`&deploy_mqtt_nodered_inputs`) so it can be reused.
- `interfaces` -- The `start` operation explicitly overrides its inputs with `<<: *deploy_mqtt_nodered_inputs`, which merges the same properties block as operation inputs. This pattern ensures that the operation inputs exactly match the node properties.
- `relationships` -- This node depends on `install_docker`, so Docker is guaranteed to be installed before the containers are deployed.

### Execution Order

Because of the relationships:

1. `vm_info` is created first (provides VM details and proxy settings).
2. `install_docker` runs its playbook (installs Docker on the VM).
3. `deploy_mqtt_nodered` runs its playbook (deploys MQTT, Node-RED, and Nginx containers).

---

## Quick Reference

### Minimal Node Template

```yaml
node_templates:
  run_my_playbook:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: playbooks/main.yaml
      sources:
        all:
          hosts:
            target_host:
              ansible_host: 192.168.1.100
              ansible_user: admin
              ansible_ssh_private_key_file: /path/to/key
```

### With External Venv and Retry

```yaml
node_templates:
  run_my_playbook:
    type: dell.nodes.ansible.Executor
    properties:
      ansible_external_venv: /opt/ansible
      playbook_path: playbooks/main.yaml
      number_of_attempts: 5
      timeout: "30"
      debug_level: 2
      log_stdout: true
      store_facts: false
      sources:
        all:
          hosts:
            target_host:
              ansible_host: 192.168.1.100
              ansible_user: admin
              ansible_ssh_private_key_file: /path/to/key
              ansible_become: true
      run_data:
        app_version: "2.1.0"
        enable_monitoring: true
      ansible_env_vars:
        ANSIBLE_HOST_KEY_CHECKING: "False"
        ANSIBLE_INVALID_TASK_ATTRIBUTE_FAILED: "False"
        ANSIBLE_STDOUT_CALLBACK: json
```

---

## See Also

- [Ansible Plugin Overview](./overview.md) -- introduction and DSL definitions.
- [Ansible Plugin Workflows](./workflows.md) -- how to reload playbooks and update virtual environments.
