# Fabric Plugin Operations

This page provides a complete reference for every operation exposed by the NativeEdge Fabric Plugin. Each operation is a Python function that you reference in the `interfaces` section of a node template. For an introduction to how the plugin works and why it does not define its own node types, see the [Overview](overview.md).

## Prerequisites

- The `fabric-plugin` package (version `3.3.3.0` or compatible) must be installed on your NativeEdge manager.
- Your blueprint must import the plugin:
  ```yaml
  imports:
    - plugin:fabric-plugin?version=>=3.3.2.0
  ```
- You need SSH credentials (key or password) and network access to the target host.
- A node type such as `dell.nodes.Root` must be available (typically from `dell/types/types.yaml`).

---

## Operations

- [`run_commands`](#run_commands) -- Run a list of shell commands on a remote host.
- [`run_script`](#run_script) -- Upload and execute a script file on a remote host.
- [`run_task`](#run_task) -- Run a Python task function from a tasks file.
- [`run_module_task`](#run_module_task) -- Run a Python task function from an installed module.

---

<a id="run_commands"></a>

## `fabric.fabric_plugin.tasks.run_commands`

Runs a list of shell commands **sequentially** on a remote host over SSH. Each command is executed in order; if any command fails, execution stops and a non-recoverable error is raised.

### Parameters

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `commands` | list of strings | Yes | -- | An ordered list of shell commands to execute on the remote host. Each entry is a single command string. |
| `fabric_env` | dict | Yes | -- | SSH connection configuration. See [The `fabric_env` Configuration Object](#the-fabric_env-configuration-object). |
| `use_sudo` | bool | No | `false` | If `true`, every command in the list is executed with `sudo`. The plugin calls `conn.sudo()` instead of `conn.run()`. If a sudo password is needed, provide it in `fabric_env` via `sudo_password`. |
| `proxy_settings` | dict | No | `{}` | Proxy/oxy tunneling configuration. See [The `proxy_settings` Configuration Object](#the-proxy_settings-configuration-object). |
| `hide_output` | list of strings | No | `[]` (show all) | Controls which output streams are suppressed. See [The `hide_output` Parameter](#the-hide_output-parameter). |

### How It Works

1. The plugin establishes an SSH connection to the remote host using the details in `fabric_env`.
2. For each command in the `commands` list:
   - If `fabric_env.shell_env` is provided, the plugin prepends `export KEY=VALUE;` statements to the command string so that environment variables are available to the command.
   - If `use_sudo` is `true`, the command is executed via `conn.sudo()` with a pseudo-terminal (`pty=True`).
   - Otherwise, the command is executed via `conn.run()` with `pty=True`.
3. Output is logged according to the `hide_output` setting.

### Example: Running Commands on a Remote Host with Proxy Disabled

This example is taken from a blueprint that installs an Nginx file server on a remote host that is directly reachable (not behind a NativeEdge endpoint). Note that `proxy_settings.disable` is set to `true`.

```yaml
# Source: remotehost/definitions.yaml
node_templates:
  install_file_server:
    type: dell.nodes.Root
    interfaces:
      dell.interfaces.lifecycle:
        create:
          executor: central_deployment_agent
          implementation: fabric.fabric_plugin.tasks.run_commands
          inputs:
            commands:
              - git clone -q https://github.com/Chubtoad5/nginx-static-files.git
              - sudo chmod +x nginx-static-files/install_nginx.sh
              - sudo -E ./nginx-static-files/install_nginx.sh
            fabric_env:
              shell_env:
                DURATION_DAYS: { get_input: duration_days }
                ARTIFACT_COMMON_NAME: { get_input: common_name }
                COUNTRY: { get_input: country }
                STATE: { get_input: state }
                LOCATION: { get_input: location }
                ORGANIZATION: { get_input: organization }
                NGINX_HTUSER: { get_input: nginx_htuser }
                NGINX_HTPASS: { get_input: nginx_htpass }
                ENABLE_HTTP_REDIRECT: { get_input: enable_http_redirect }
                NGINX_PORT: { get_input: nginx_port }
                BROWSER_TITLE: { get_input: browser_title }
                BODY_TITLE: { get_input: body_title }
                DEBUG: { get_input: nginx_debug }
              host_string: { get_input: remote_host }
              user: { get_input: remote_username }
              key: { get_secret: { get_input: ssh_private_key } }
              port: { get_input: remote_port }
            proxy_settings:
              disable: true
```

**What happens here:**

1. Three commands run in order: clone a Git repository, set execute permissions, and run an install script.
2. `shell_env` injects environment variables (like `NGINX_PORT`, `NGINX_HTUSER`, etc.) into each command so the install script can read them.
3. `proxy_settings.disable: true` tells the plugin to connect directly to the host without going through the NativeEdge proxy.

### Example: Running Commands Through the NativeEdge Proxy

This example installs the same Nginx server, but the target VM is behind a NativeEdge endpoint. The proxy settings are resolved automatically from the node's connection attributes.

```yaml
# Source: nginx_app/definitions.yaml (abbreviated)
  install_file_server:
    type: dell.nodes.Root
    interfaces:
      dell.interfaces.lifecycle:
        create:
          executor: central_deployment_agent
          implementation: fabric.fabric_plugin.tasks.run_commands
          inputs:
            commands:
              - git clone -q https://github.com/Chubtoad5/nginx-static-files.git
              - sudo chmod +x nginx-static-files/install_nginx.sh
              - sudo -E ./nginx-static-files/install_nginx.sh
            fabric_env:
              shell_env:
                DURATION_DAYS: { get_input: duration_days }
                ARTIFACT_COMMON_NAME: { get_input: common_name }
                # ... additional environment variables ...
              host_string:
                get_attribute: [vm_platform, capabilities, vm_name]
              user:
                get_attribute: [vm_platform, capabilities, vm_user_name]
              key:
                get_secret: { get_attribute: [vm_platform, capabilities, vm_ssh_private_key] }
              port: 22
            proxy_settings:
              get_attribute: [ vm_info, connection_proxy_settings ]
    relationships:
       - type: dell.relationships.depends_on
         target: vm_info
```

**What is different:** The `proxy_settings` are pulled from the `vm_info` node's `connection_proxy_settings` attribute at runtime. Since `disable` is not set to `true`, the plugin will automatically resolve a proxy endpoint to tunnel the SSH connection through the NativeEdge oxy service.

---

<a id="run_script"></a>

## `fabric.fabric_plugin.tasks.run_script`

Uploads a script file from your blueprint to a remote host and executes it over SSH. The script can be a shell script, a Python script, or any executable file. The plugin also sets up a `ctx` proxy on the remote host, allowing your script to call back into the NativeEdge context (for example, to set runtime properties).

### Parameters

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `script_path` | string | Yes | -- | Path to the script file, relative to the blueprint root. Can also be an HTTP/HTTPS URL to download the script from. |
| `fabric_env` | dict | Yes | -- | SSH connection configuration. See [The `fabric_env` Configuration Object](#the-fabric_env-configuration-object). |
| `process` | dict | No | `{}` | Advanced execution options. Supported keys: `env` (dict of extra environment variables), `args` (list of command-line arguments), `command_prefix` (string prepended to the command, e.g. `"python3"`), `cwd` (remote working directory), `base_dir` (remote base directory for uploaded files), `ctx_server_port` (port for the ctx proxy server). |
| `use_sudo` | bool | No | `false` | If `true`, the script is executed with `sudo`. |
| `proxy_settings` | dict | No | `{}` | Proxy/oxy tunneling configuration. See [The `proxy_settings` Configuration Object](#the-proxy_settings-configuration-object). |
| `hide_output` | list of strings | No | `[]` (show all) | Controls which output streams are suppressed. See [The `hide_output` Parameter](#the-hide_output-parameter). |

### How It Works

1. The plugin downloads the script from the blueprint archive (or from a URL if `script_path` starts with `http://` or `https://`).
2. It establishes an SSH connection to the remote host.
3. It uploads the script, a ctx proxy client, and a helper wrapper to a temporary directory on the remote host.
4. It generates an environment script that sets `PATH`, `PYTHONPATH`, and any variables from `process.env`, then marks all uploaded files as executable.
5. It starts a local HTTP ctx proxy server and opens a reverse SSH tunnel so the remote script can communicate back to NativeEdge.
6. It runs the script on the remote host (with `sudo` if `use_sudo` is `true`).
7. Output is logged according to the `hide_output` setting.
8. If the script used `ctx` to return a value, that value is returned from the operation.

### Example: Running a Script on a VM

This example runs a shell script that collects VM information. The script file `vm/scripts/get_vm_info.sh` is part of the blueprint archive.

```yaml
# Source: vm/definitions.yaml
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
    relationships:
       - type: dell.relationships.depends_on
         target: vm
```

**What happens here:**

1. During the `create` lifecycle event, a local Python script (`get_target_id.py`) runs on the NativeEdge manager to prepare some context.
2. During the `start` lifecycle event, the Fabric Plugin uploads `vm/scripts/get_vm_info.sh` to the remote VM and executes it.
3. The `host_string` is resolved from the VM's runtime attributes (the VM name, which NativeEdge can resolve to an IP).
4. Authentication uses an SSH private key retrieved from the NativeEdge secret store.
5. The `proxy_settings` are read from the node's own `connection_proxy_settings` attribute, allowing the NativeEdge platform to tunnel the SSH connection through its proxy if needed.

---

<a id="run_task"></a>

## `fabric.fabric_plugin.tasks.run_task`

Runs a specific Python task function from a tasks file that is part of your blueprint. The tasks file is loaded dynamically, and the named function is called with the SSH connection as the first argument (if decorated with `@task`) or without it (for legacy compatibility).

### Parameters

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `tasks_file` | string | Yes | -- | Path to a Python file within the blueprint archive that contains the task function. |
| `task_name` | string | Yes | -- | The name of the Python function to call within the tasks file. |
| `fabric_env` | dict | Yes | -- | SSH connection configuration. See [The `fabric_env` Configuration Object](#the-fabric_env-configuration-object). |
| `task_properties` | dict | No | `{}` | A dictionary of keyword arguments to pass to the task function when it is called. |
| `proxy_settings` | dict | No | `{}` | Proxy/oxy tunneling configuration. See [The `proxy_settings` Configuration Object](#the-proxy_settings-configuration-object). |

**Note:** The `hide_output` parameter is **not supported** for `run_task`. If provided, it will be logged as a debug message and ignored.

### How It Works

1. The plugin loads the Python file specified by `tasks_file` from the blueprint archive.
2. It looks up the function named `task_name` in that file.
3. It establishes an SSH connection to the remote host using `fabric_env`.
4. It calls the task function, passing `task_properties` as keyword arguments.
   - If the function is decorated with Fabric's `@task` decorator, it receives the SSH `Connection` object as its first argument. This is the recommended approach for new tasks.
   - If the function is **not** decorated with `@task`, the plugin wraps it automatically so the connection argument is skipped. This provides backward compatibility with older task files.

### Example

```yaml
node_templates:
  my_task_runner:
    type: dell.nodes.Root
    interfaces:
      dell.interfaces.lifecycle:
        start:
          implementation: fabric.fabric_plugin.tasks.run_task
          inputs:
            tasks_file: scripts/my_tasks.py
            task_name: configure_application
            fabric_env:
              host_string: 10.0.0.5
              user: admin
              key_filename: /home/admin/.ssh/id_rsa
            task_properties:
              app_port: 8080
              app_name: my-service
```

The corresponding `scripts/my_tasks.py` in the blueprint:

```python
from fabric2 import task

@task
def configure_application(conn, app_port, app_name):
    """Configure the application on the remote host.

    conn: the SSH connection object provided by fabric.
    """
    conn.run(f'echo "Configuring {app_name} on port {app_port}"')
    conn.run(f'sed -i "s/PORT=.*/PORT={app_port}/" /etc/my-service/config')
    conn.run('sudo systemctl restart my-service')
```

---

<a id="run_module_task"></a>

## `fabric.fabric_plugin.tasks.run_module_task`

Runs a Python task function specified by a fully-qualified Python module path (dot-separated). This is useful when the task function lives in an installed Python package rather than in a blueprint file.

### Parameters

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `task_mapping` | string | Yes | -- | Fully-qualified Python path to the task function, e.g. `"mypackage.mymodule.my_function"`. The last segment is the function name; everything before it is the module path. |
| `fabric_env` | dict | Yes | -- | SSH connection configuration. See [The `fabric_env` Configuration Object](#the-fabric_env-configuration-object). |
| `task_properties` | dict | No | `{}` | A dictionary of keyword arguments to pass to the task function. |
| `proxy_settings` | dict | No | `{}` | Proxy/oxy tunneling configuration. See [The `proxy_settings` Configuration Object](#the-proxy_settings-configuration-object). |

**Note:** The `hide_output` parameter is **not supported** for `run_module_task`. If provided, it will be logged as a debug message and ignored.

### How It Works

1. The plugin splits `task_mapping` on the last `.` to separate the module path from the function name.
2. It imports the module using Python's `importlib.import_module`.
3. It looks up the function by name in the imported module.
4. It establishes an SSH connection and calls the function, following the same `@task` decorator logic as `run_task`.

### Example

```yaml
node_templates:
  module_task_runner:
    type: dell.nodes.Root
    interfaces:
      dell.interfaces.lifecycle:
        start:
          implementation: fabric.fabric_plugin.tasks.run_module_task
          inputs:
            task_mapping: my_company.deployment_tasks.setup_monitoring
            fabric_env:
              host_string: 10.0.0.5
              user: admin
              key_filename: /home/admin/.ssh/id_rsa
            task_properties:
              monitoring_port: 9090
```

---

<a id="the-fabric_env-configuration-object"></a>

## The `fabric_env` Configuration Object

The `fabric_env` dictionary is the **most important input** for every Fabric Plugin operation. It defines how the plugin connects to the remote host via SSH.

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `host_string` | string | Yes (or `host`) | -- | The remote host address (IP address or hostname). This is the Fabric 1.x-style key. The plugin accepts either `host_string` or `host`. |
| `host` | string | Yes (or `host_string`) | -- | The remote host address. This is the Fabric 2.x-style key. Equivalent to `host_string`. Use one or the other, not both. |
| `user` | string | Yes | -- | The SSH username to authenticate as on the remote host. If not provided, the plugin falls back to the agent user configured in the NativeEdge bootstrap context. |
| `key` | string | No | `null` | The SSH private key content as a string (the full PEM-encoded key, including `-----BEGIN ... PRIVATE KEY-----` headers). Supports RSA, ECDSA, and Ed25519 keys. Use this when you store the key in a NativeEdge secret and retrieve it with `get_secret`. |
| `key_filename` | string | No | `null` | Path to an SSH private key file on the NativeEdge manager filesystem. Use this when the key file exists on the manager rather than in a secret. Supports `~` for home directory expansion. |
| `password` | string | No | `null` | SSH password for password-based authentication. Use this as an alternative to `key` or `key_filename`. |
| `port` | integer | No | `22` | The SSH port on the remote host. |
| `connect_timeout` | integer | No | `10` | Timeout in seconds for establishing the SSH connection. |
| `command_timeout` | integer | No | `null` (no timeout) | Timeout in seconds for individual command execution. If a command runs longer than this, it is terminated. |
| `always_use_pty` | bool | No | `false` | If `true`, always allocate a pseudo-terminal for commands. Some commands and programs require a PTY to function correctly. |
| `gateway` | string | No | `null` | An SSH gateway (jump host) to connect through. Specify as `user@host:port`. |
| `forward_agent` | bool | No | `null` | If `true`, enable SSH agent forwarding so the remote host can use your local SSH agent's keys for onward connections. |
| `no_agent` | bool | No | `false` | If `true`, disable the use of the local SSH agent for authentication. |
| `sudo_password` | string | No | `null` | The password to use for `sudo` operations when `use_sudo` is `true`. Also accessible via `fabric_env.sudo.password`. |
| `shell_env` | dict | No | `{}` | A dictionary of environment variables to set before running commands. Each key-value pair is converted to an `export KEY=VALUE;` statement prepended to every command. See the `run_commands` examples above for real-world usage. |
| `connect_kwargs` | dict | No | `{}` | A dictionary of additional keyword arguments passed directly to Paramiko's SSH connection. Advanced usage -- for example, you can pass `pkey` (a pre-loaded private key object) here. |

### Authentication Priority

The plugin resolves authentication credentials in this order:

1. `connect_kwargs.pkey` -- a pre-loaded Paramiko private key object
2. `key` -- private key content as a string (loaded by the plugin)
3. `key_filename` -- path to a private key file
4. `password` -- SSH password
5. Agent key path from the NativeEdge bootstrap context (fallback)

If none of these are provided, the plugin raises a `NonRecoverableError`.

### Minimal Example

```yaml
fabric_env:
  host_string: 192.168.1.100
  user: ubuntu
  key_filename: /home/ubuntu/.ssh/id_rsa
  port: 22
```

### Example with Secret-Based Key

```yaml
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
```

### Example with `shell_env`

```yaml
fabric_env:
  shell_env:
    NGINX_PORT: { get_input: nginx_port }
    BROWSER_TITLE: { get_input: browser_title }
    DEBUG: { get_input: nginx_debug }
  host_string: { get_input: remote_host }
  user: { get_input: remote_username }
  key: { get_secret: { get_input: ssh_private_key } }
  port: { get_input: remote_port }
```

When `shell_env` is used with `run_commands`, each command is prepended with export statements. For example, if the command is `echo $NGINX_PORT`, the plugin actually runs:

```
export NGINX_PORT=8080;export BROWSER_TITLE=MyServer;export DEBUG=true;echo $NGINX_PORT
```

---

<a id="the-proxy_settings-configuration-object"></a>

## The `proxy_settings` Configuration Object

The `proxy_settings` dictionary controls how the Fabric Plugin reaches the remote host through the NativeEdge proxy infrastructure.

### Background: The NativeEdge Proxy (Oxy)

In many NativeEdge deployments, the VMs and devices you manage are not directly reachable from the NativeEdge manager over the network. Instead, they sit behind a NativeEdge Edge Compute Endpoint (ECE). The NativeEdge platform provides a proxy tunneling service (sometimes called "oxy") that allows the manager to reach these hosts.

When the Fabric Plugin needs to connect to a host, it can automatically resolve the correct proxy endpoint so that the SSH connection is tunneled through the NativeEdge infrastructure. This happens transparently -- you do not need to configure tunnels manually.

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `disable` | bool | No | `false` | If `true`, the plugin connects directly to the host specified in `fabric_env` without using the NativeEdge proxy. Set this to `true` when the remote host is directly reachable from the manager. |
| `auto_resolve` | bool | No | `true` | If `true` (and `disable` is `false`), the plugin automatically resolves the proxy endpoint using the NativeEdge oxy service and the deployment ID. |

### When to Use Each Setting

**Proxy auto-resolved (default behavior):**
Use this when the target host is behind a NativeEdge endpoint. The `proxy_settings` are typically read from a node attribute that the platform populates automatically.

```yaml
# The proxy endpoint is resolved automatically at runtime.
proxy_settings:
  get_attribute: [ SELF, connection_proxy_settings ]
```

Or equivalently, from another node:

```yaml
proxy_settings:
  get_attribute: [ vm_info, connection_proxy_settings ]
```

**Proxy disabled:**
Use this when you are connecting to a host that is directly reachable from the NativeEdge manager (for example, a remote server on the same network, or a cloud VM with a public IP).

```yaml
# Connect directly -- no proxy tunneling.
proxy_settings:
  disable: true
```

### Full Comparison

The following table shows the difference side by side:

| Scenario | `proxy_settings` value | Effect |
|---|---|---|
| VM behind NativeEdge endpoint | `get_attribute: [ SELF, connection_proxy_settings ]` | Plugin resolves the proxy endpoint via oxy. The SSH connection is tunneled through the NativeEdge infrastructure. |
| Directly reachable remote host | `disable: true` | Plugin connects directly to the `host_string`/`host` address. No proxy is used. |
| Default (no `proxy_settings` provided) | `{ auto_resolve: true }` | Plugin attempts auto-resolution. If no proxy endpoint is found, the connection may fail. Always specify `proxy_settings` explicitly. |

---

<a id="the-hide_output-parameter"></a>

## The `hide_output` Parameter

The `hide_output` parameter controls which output streams from remote commands are suppressed in the NativeEdge logs. It is available for `run_commands` and `run_script` operations. It is **not supported** for `run_task` or `run_module_task`.

### Accepted Values

`hide_output` takes a **list** of group name strings. The plugin maps these group names to Fabric 2's `hide` parameter as follows:

| Group Name | Fabric 2 `hide` Value | Effect |
|---|---|---|
| `status` | `out` | Suppress standard output. |
| `aborts` | `out` | Suppress standard output. |
| `warnings` | `out` | Suppress standard output. |
| `running` | `out` | Suppress standard output. |
| `user` | `out` | Suppress standard output. |
| `stdout` | `out` | Suppress standard output. |
| `stderr` | `err` | Suppress standard error. |
| `everything` | `out` + `err` | Suppress all output (both stdout and stderr). |
| `both` | `out` + `err` | Suppress all output (both stdout and stderr). |
| `None` | `false` | Show all output (no suppression). |

The group names (like `status`, `aborts`, `warnings`, `running`, `user`) exist for backward compatibility with Fabric 1.x terminology. In Fabric 2, they all map to suppressing standard output.

### Resolution Logic

- If `hide_output` is an empty list or not provided, **all output is shown** (nothing is hidden).
- If the resolved groups contain only `out`, only stdout is hidden.
- If the resolved groups contain only `err`, only stderr is hidden.
- If the resolved groups contain both `out` and `err`, all output is hidden.
- If any value in the list is not one of the recognized group names, the plugin raises a `NonRecoverableError`.

### Examples

Show all output (default):

```yaml
# No hide_output specified -- all output is visible in logs.
inputs:
  commands:
    - echo "This will appear in NativeEdge logs"
  fabric_env:
    host_string: 10.0.0.5
    user: admin
    key_filename: /path/to/key
```

Hide all output:

```yaml
inputs:
  commands:
    - echo "This will NOT appear in NativeEdge logs"
  fabric_env:
    host_string: 10.0.0.5
    user: admin
    key_filename: /path/to/key
  hide_output:
    - everything
```

Hide only stderr:

```yaml
inputs:
  commands:
    - some_command_that_produces_stderr
  fabric_env:
    host_string: 10.0.0.5
    user: admin
    key_filename: /path/to/key
  hide_output:
    - stderr
```
