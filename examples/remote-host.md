# Remote Host Example

This example demonstrates how to use NativeEdge blueprints to manage a host that exists **outside** the NativeEdge endpoint infrastructure. Instead of creating a VM on a NativeEdge endpoint or referencing an existing NativeEdge deployment, this blueprint connects directly to any SSH-accessible remote host and installs the same Nginx file server used in the [Nginx App](nginx-app.md) example.

The key concept here is **disabling the NativeEdge proxy** -- since the remote host is not behind a NativeEdge endpoint, there is no proxy tunnel to route through.

## Prerequisites

- [Blueprint Anatomy](../blueprint-structure/blueprint-anatomy.md) -- understand blueprint structure
- [Fabric Plugin](../fabric-plugin/overview.md) -- remote command execution over SSH
- [Nginx App Example](nginx-app.md) -- useful comparison (same app, different connectivity model)

---

## File Structure

```
remote_host_example.yaml    <-- Entry point
remotehost/
  definitions.yaml          <-- Node templates (1 node)
  inputs.yaml               <-- Input declarations (~15 inputs)
  outputs.yaml              <-- Output capabilities (1 URL)
```

This is the simplest blueprint in the collection. It has a single node template and no complex dependency chains.

---

## The Entry Point

```yaml
tosca_definitions_version: dell_1_0

description: >
  This blueprint creates a connection to a remote host and uses Fabric
  to install an NGINX file server

imports:
  - dell/types/types.yaml
  - remotehost/definitions.yaml
  - remotehost/inputs.yaml
  - remotehost/outputs.yaml
```

### Input Groups

```yaml
input_groups:
  remote_host:
    display_label: Remote host credentials
    collapsible: false
    index: 0
    inputs:
      - remote_host
      - remote_username
      - ssh_private_key
      - remote_port
  nginx_settings:
    display_label: Nginx Settings
    collapsible: true
    index: 1
    inputs:
      - nginx_htuser
      - nginx_htpass
      - enable_http_redirect
      - nginx_port
      - browser_title
      - body_title
      - nginx_debug
  certificate:
    display_label: Self-Signed Certificate
    collapsible: true
    index: 2
    inputs:
      - duration_days
      - common_name
      - country
      - state
      - location
      - organization
```

The first input group (`remote_host`) is non-collapsible because the user **must** provide the remote host connection details. There are no defaults for the host address, username, or SSH key -- these must be specified for every deployment.

### Labels

```yaml
labels:
  target_environment:
    values:
      - remote           # Not "nativeedge" -- this targets a remote host
  solution:
    values:
      - POC-REMOTE-HOST

blueprint_labels:
  env:
    values:
      - remote           # Not "ned" -- this is a remote environment
```

Notice that both `target_environment` and `env` are set to `remote` instead of `nativeedge`/`ned`. This distinguishes remote host deployments from NativeEdge endpoint deployments in the UI and in label-based filtering.

---

## Node Templates (definitions.yaml)

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin
  - plugin:utilities-plugin?version=>=3.1.3.0
  - plugin:fabric-plugin?version=>=3.3.2.0
```

The same three plugins as the Nginx App example. No Ansible plugin is needed.

### Dependency Chain

```
install_file_server    (single node, no dependencies)
```

There is only one node template. No dependency chain exists.

---

### Node: `install_file_server` -- Install Nginx on Remote Host

```yaml
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
              ## Self-Signed Certificate info
              DURATION_DAYS: { get_input: duration_days }
              ARTIFACT_COMMON_NAME: { get_input: common_name }
              COUNTRY: { get_input: country }
              STATE: { get_input: state }
              LOCATION: { get_input: location }
              ORGANIZATION: { get_input: organization }
              ## Basic Authentication info
              NGINX_HTUSER: { get_input: nginx_htuser }
              NGINX_HTPASS: { get_input: nginx_htpass }
              ## TCP Port and Server titles
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

**What it does:** Connects to a remote host via SSH and runs the same Nginx installation commands as the [Nginx App example](nginx-app.md).

The `commands`, `shell_env`, and overall structure are nearly identical to the Nginx App example's `install_file_server` node. The two critical differences are in how the SSH connection is established and how the proxy is handled.

---

### Difference 1: Direct SSH Connection via `get_input`

In the **Nginx App** example, connection details come from a SharedResource (another deployment's capabilities):

```yaml
# Nginx App -- connection from SharedResource
fabric_env:
  host_string: get_attribute: [vm_platform, capabilities, vm_name]
  user: get_attribute: [vm_platform, capabilities, vm_user_name]
  key: get_secret: { get_attribute: [vm_platform, capabilities, vm_ssh_private_key] }
  port: 22
```

In the **Remote Host** example, connection details come directly from user inputs:

```yaml
# Remote Host -- connection from user inputs
fabric_env:
  host_string: { get_input: remote_host }
  user: { get_input: remote_username }
  key: { get_secret: { get_input: ssh_private_key } }
  port: { get_input: remote_port }
```

This is a fundamental difference. The Nginx App example is tightly coupled to the NativeEdge ecosystem (it needs a deployed VM with known capabilities). The Remote Host example is generic -- it can connect to any SSH-accessible host: a cloud VM, a bare-metal server, a Raspberry Pi, or any other Linux system.

The `host_string` is the IP address or FQDN of the remote host (provided by the user). The `key` uses the same `get_secret` pattern to retrieve the SSH private key from the NativeEdge secret store -- the user provides the secret name, and the blueprint retrieves the key content.

---

### Difference 2: Disabled Proxy -- `proxy_settings: disable: true`

```yaml
proxy_settings:
  disable: true
```

This is the most important distinction. In all the NativeEdge endpoint examples, the `proxy_settings` uses `get_attribute: [SELF, connection_proxy_settings]` or `get_attribute: [vm_info, connection_proxy_settings]` to route SSH traffic through the NativeEdge endpoint agent.

Here, `disable: true` tells the Fabric plugin to **skip the NativeEdge proxy entirely** and connect directly to the remote host. This is necessary because:

1. The remote host is not behind a NativeEdge endpoint -- there is no agent to proxy through.
2. The NativeEdge manager must be able to reach the remote host directly over the network.

If you omit the `proxy_settings` entirely, the Fabric plugin would attempt to use the default NativeEdge proxy behavior, which would fail for a host that is not managed by a NativeEdge endpoint.

---

## Inputs (inputs.yaml)

### Remote Host Connection Inputs

```yaml
inputs:

  remote_host:
    type: string
    hidden: false
    allow_update: true
    display_label: Remote host IP or FQDN
    description: |
      Remote host IP or FQDN where Ansible playbook will run

  remote_username:
    type: string
    hidden: false
    allow_update: true
    display_label: Username for remote host login
    description: |
      Remote host's username for ssh login

  ssh_private_key:
    type: secret_key
    hidden: false
    display_label: SSH Private key for ssh login
    description: |
      Secret key containing the private key of the remote host for SSH access

  remote_port:
    type: integer
    hidden: false
    allow_update: true
    display_label: Remote host SSH port
    description: |
      TCP Port for ssh connection to remote host
    default: 22
```

These four inputs define the SSH connection to the remote host:

- **`remote_host`** -- the IP address or fully qualified domain name. No default is provided -- the user must specify this.
- **`remote_username`** -- the SSH username. No default -- the user must specify this.
- **`ssh_private_key`** -- a `secret_key` type input. The user selects a secret from the NativeEdge secret store that contains the SSH private key. The key must already be stored as a secret before deploying this blueprint.
- **`remote_port`** -- the SSH port, defaulting to 22. Users can change this if their host uses a non-standard SSH port.

All connection inputs have `allow_update: true`, meaning you can change the target host after deployment (e.g., to re-run the installation on a different host).

### Nginx and Certificate Inputs

The remaining inputs are identical to those in the Nginx App example:

```yaml
# Self-Signed Certificate (same as Nginx App)
duration_days:
  type: integer
  default: 365
common_name:
  type: string
  default: "artifacts.edge.lab"
# ... country, state, location, organization

# Nginx Settings (same as Nginx App)
nginx_htuser:
  type: string
  default: "admin"
nginx_htpass:
  type: string
  default: "Password123!"
# ... enable_http_redirect, nginx_port, browser_title, body_title, nginx_debug
```

---

## Outputs (outputs.yaml)

```yaml
capabilities:

  nginx_url:
    description: NGINX web server URL
    value: { concat: ['https://', { get_input: remote_host }, ':', 443] }
```

The output constructs the Nginx URL from the user-provided `remote_host` input and a hardcoded port 443.

Compare this with the Nginx App example's output:

```yaml
# Nginx App output -- uses SharedResource capability
nginx_url:
  value:
    concat:
      - 'https://'
      - { get_attribute: [ vm_platform, capabilities, vm_ip ] }
      - ':'
      - { get_input: nginx_port }
```

The Nginx App example reads the IP from a SharedResource capability and the port from an input. The Remote Host example reads the host from an input and uses a fixed port. Both produce a URL the user can click to access the server.

Note that the Remote Host example only exposes the URL -- it does not expose the username and password as separate capabilities (unlike the Nginx App example).

---

## Execution Flow

1. **`install_file_server`** connects directly to the remote host via SSH (no proxy), clones the Nginx repository, and runs the installation script with the configured environment variables.
2. **Output** is computed from the user's remote host input.

This is the simplest execution flow in the collection -- a single node with a single lifecycle phase.

---

## Side-by-Side Comparison: Nginx App vs. Remote Host

Both examples install the same Nginx file server using the same commands and environment variables. The differences are entirely in how they establish the SSH connection:

| Aspect | Nginx App | Remote Host |
|---|---|---|
| Target host | NativeEdge VM (from SharedResource) | Any SSH-accessible host |
| Connection source | `get_attribute: [vm_platform, capabilities, ...]` | `get_input: remote_host` |
| SSH key source | Secret name from VM deployment capabilities | Secret name from user input |
| Proxy settings | `get_attribute: [vm_info, connection_proxy_settings]` | `disable: true` |
| Prerequisites | Deployed Simple VM instance | SSH access and credentials |
| Node count | 3 (vm_platform + vm_info + install) | 1 (install only) |
| Labels | `target_environment: nativeedge` | `target_environment: remote` |
| Outputs | URL + username + password | URL only |

### When to Use Each

Use the **Nginx App** pattern when:
- Your target is a VM managed by NativeEdge
- You want to build on top of an existing NativeEdge deployment
- You need the NativeEdge proxy to reach the VM
- You want label-based deployment selection in the UI

Use the **Remote Host** pattern when:
- Your target is a host outside the NativeEdge infrastructure
- The NativeEdge manager can reach the host directly over the network
- You want a simple, standalone deployment without cross-deployment dependencies
- You are managing existing infrastructure that was not provisioned by NativeEdge

---

## Adapting This Pattern

The Remote Host pattern is the easiest to adapt for your own use cases. To install a different application:

1. Replace the `commands` list with your own installation commands.
2. Modify the `shell_env` to pass the environment variables your scripts need.
3. Update the inputs to match your application's configuration parameters.
4. Update the outputs to expose the relevant URLs or endpoints.

The SSH connection block (`host_string`, `user`, `key`, `port`, and `proxy_settings: disable: true`) stays the same for any remote host target.
