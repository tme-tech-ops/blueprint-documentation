# Nginx Application Example

This example demonstrates how to deploy an application onto an **existing** virtual machine that was created by a previous blueprint deployment. The key concept here is the **SharedResource** pattern -- a way for one blueprint to reference and consume capabilities from another deployment.

Instead of creating its own VM, this blueprint connects to a VM that was deployed using the [Simple VM](simple-vm.md) example and installs an Nginx web server configured to serve static files with HTTPS and basic authentication.

## Prerequisites

- [Blueprint Anatomy](../blueprint-structure/blueprint-anatomy.md) -- understand blueprint structure
- [Simple VM Example](simple-vm.md) -- you must understand this first, as this example depends on it
- [Fabric Plugin](../fabric-plugin/overview.md) -- remote command execution over SSH
- [Utilities Plugin](../utilities-plugin/overview.md) -- SharedResource type for cross-deployment references

---

## File Structure

```
nginx_app_example_for_NED.yaml    <-- Entry point
nginx_app/
  definitions.yaml                <-- Node templates (3 nodes)
  inputs.yaml                     <-- Input declarations (~15 inputs)
  outputs.yaml                    <-- Output capabilities (3 outputs)
vm/
  scripts/
    get_target_id.py              <-- Shared script (also used by Simple VM)
    get_vm_info.sh                <-- Shared script (also used by Simple VM)
```

Notice that this blueprint reuses scripts from the `vm/scripts/` directory. The `get_target_id.py` and `get_vm_info.sh` scripts are the same ones used in the Simple VM example. This is possible because both blueprints are part of the same blueprint package.

---

## The Entry Point

```yaml
tosca_definitions_version: dell_1_0

description: >
  This blueprint Deploys an NGINX server to serve static files
  on an existing POC VM deployment

imports:
  - dell/types/types.yaml
  - nginx_app/definitions.yaml
  - nginx_app/inputs.yaml
  - nginx_app/outputs.yaml
```

### Input Groups

```yaml
input_groups:
  deployment_target:
    display_label: Deployment Target
    collapsible: false
    index: 0
    inputs:
      - vm_deployment_id
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

The first input group (`deployment_target`) is non-collapsible (`collapsible: false`) because it is the most critical input -- the user must select which existing VM deployment to target. The nginx settings and certificate settings are collapsible because they have sensible defaults.

### Labels

```yaml
labels:
  solution:
    values:
      - POC-NGINX
```

The `solution: POC-NGINX` label distinguishes this deployment type from the VM deployment (`POC-VM`).

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

This blueprint uses three plugins: edge, utilities, and fabric -- the same set as the Simple VM example, but for different purposes.

### Dependency Chain

```
vm_platform ──> vm_info ──> install_file_server
```

The flow is linear: first establish the connection to the existing VM, then gather its info, then install the application.

---

### Node 1: `vm_platform` -- Reference an Existing Deployment

```yaml
vm_platform:
  type: dell.nodes.SharedResource
  properties:
    resource_config:
      deployment:
        id: { get_input: vm_deployment_id }
```

**What it does:** Creates a reference to an existing deployment (the Simple VM deployment) without creating any new resources. This is the **SharedResource** pattern.

**Key concept -- `dell.nodes.SharedResource`:**

A SharedResource node does not create, start, or delete anything. It establishes a connection to another deployment and exposes that deployment's capabilities (outputs) as attributes on this node. Once the SharedResource is established, you can access the target deployment's capabilities using:

```yaml
get_attribute: [vm_platform, capabilities, <capability_name>]
```

The `resource_config.deployment.id` specifies which deployment to reference. Here it comes from the `vm_deployment_id` input, which has a special type:

```yaml
vm_deployment_id:
  type: deployment_id
  constraints:
  - labels:
    - csys-obj-type: environment
    - solution: POC-VM
```

The `type: deployment_id` tells the NativeEdge UI to present a dropdown of existing deployments. The `labels` constraint filters this dropdown to only show deployments that have both the `csys-obj-type: environment` and `solution: POC-VM` labels -- which matches deployments created by the Simple VM blueprint. This is why the labels in the Simple VM blueprint matter.

**Capabilities available from the VM deployment:**

From the [Simple VM outputs](simple-vm.md#outputs-outputsyaml), the following capabilities are accessible through `vm_platform`:

- `service_tag` -- the NativeEdge endpoint service tag
- `vm_name` -- the VM name on the endpoint
- `vm_hostname` -- the VM hostname
- `vm_ip` -- the VM IP address
- `vm_user_name` -- the SSH username
- `vm_ssh_private_key` -- the name of the secret containing the SSH private key

---

### Node 2: `vm_info` -- Gather VM Information

```yaml
vm_info:
  type: dell.nodes.Root
  interfaces:
    dell.interfaces.lifecycle:
      create:
        implementation: vm/scripts/get_target_id.py
        executor: central_deployment_agent
        inputs:
          service_tag: { get_attribute: [vm_platform, capabilities, service_tag] }
      start:
        implementation: fabric.fabric_plugin.tasks.run_script
        inputs:
          script_path: vm/scripts/get_vm_info.sh
          fabric_env:
            host_string:
              get_attribute: [vm_platform, capabilities, vm_name]
            user:
              get_attribute: [vm_platform, capabilities, vm_user_name]
            key:
              get_secret: { get_attribute: [vm_platform, capabilities, vm_ssh_private_key] }
            port: 22
          proxy_settings:
            get_attribute: [ SELF, connection_proxy_settings ]
  relationships:
     - type: dell.relationships.depends_on
       target: vm_platform
```

**What it does:** This node is functionally identical to the `vm_info` node in the Simple VM example, but instead of reading from nodes within the same blueprint, it reads from the SharedResource's capabilities.

**Key difference from Simple VM's `vm_info`:**

In the Simple VM example:
```yaml
host_string: get_attribute: [vm, vm_details, name]
user: get_input: vm_user_name
key: get_secret: { concat: [get_attribute: [ssh_keys, secret_key_name], "_private"] }
```

In this example:
```yaml
host_string: get_attribute: [vm_platform, capabilities, vm_name]
user: get_attribute: [vm_platform, capabilities, vm_user_name]
key: get_secret: { get_attribute: [vm_platform, capabilities, vm_ssh_private_key] }
```

All values now flow through the `vm_platform` SharedResource instead of local nodes. The `vm_ssh_private_key` capability from the Simple VM blueprint is the secret name (e.g., `poc_key_private`), so `get_secret` retrieves the actual private key content.

**Proxy settings pattern:**

```yaml
proxy_settings:
  get_attribute: [ SELF, connection_proxy_settings ]
```

The `connection_proxy_settings` attribute is automatically populated by NativeEdge based on the target endpoint. It configures SSH tunneling through the NativeEdge agent running on the endpoint. Without this, the NativeEdge manager cannot reach the VM because VMs on NativeEdge endpoints are not directly accessible from the manager network.

**The `create` phase** runs `get_target_id.py` on the central agent, passing the `service_tag` from the VM deployment's capabilities. This resolves the NativeEdge endpoint's target ID, which is needed for the proxy settings.

**The `start` phase** uses Fabric to SSH into the VM through the proxy and run `get_vm_info.sh` to collect the VM's current IP address and other runtime details.

---

### Node 3: `install_file_server` -- Install Nginx

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

**What it does:** SSHs into the target VM and runs a sequence of shell commands to install and configure an Nginx static file server.

**Key concept -- `fabric.fabric_plugin.tasks.run_commands`:**

Unlike the `run_script` implementation used by `vm_info`, this node uses `run_commands`. The difference:
- `run_script` -- uploads and executes a single script file from the blueprint package
- `run_commands` -- executes a list of shell commands sequentially on the remote host

The three commands:
1. Clone the Nginx configuration repository from GitHub
2. Make the install script executable
3. Run the install script with `sudo -E` (the `-E` flag preserves environment variables)

**Key concept -- `shell_env` for passing configuration:**

```yaml
fabric_env:
  shell_env:
    DURATION_DAYS: { get_input: duration_days }
    ARTIFACT_COMMON_NAME: { get_input: common_name }
    # ... more environment variables
```

The `shell_env` section within `fabric_env` sets environment variables on the remote host before executing the commands. The install script (`install_nginx.sh`) reads these environment variables to configure Nginx. This is a clean way to pass configuration to scripts without modifying command-line arguments.

The environment variables control:
- **Certificate settings** -- duration, common name, country, state, location, organization (for generating a self-signed TLS certificate)
- **Authentication** -- HTTP basic auth username and password
- **Server settings** -- port number, HTTP redirect behavior, page titles
- **Debug mode** -- enables verbose logging

**Key concept -- proxy through `vm_info`:**

```yaml
proxy_settings:
  get_attribute: [ vm_info, connection_proxy_settings ]
```

Note that the proxy settings come from `vm_info`, not `SELF`. This is because `vm_info` already resolved the target endpoint's proxy configuration during its `create` phase. The `install_file_server` node reuses that resolved proxy configuration.

---

## Inputs (inputs.yaml)

### The Deployment Target Input

```yaml
vm_deployment_id:
  type: deployment_id
  hidden: false
  allow_update: false
  display_label: POC VM Deployment Instance
  description: |
    POC VM Deployment Instance
  constraints:
  - labels:
    - csys-obj-type: environment
    - solution: POC-VM
```

This is the most important input. The `type: deployment_id` makes the NativeEdge UI render a deployment selector dropdown. The `labels` constraint filters the dropdown to only show deployments labeled as `POC-VM` environments. The user selects which VM deployment to install Nginx onto.

### Certificate and Nginx Inputs

The remaining inputs are standard string, integer, and boolean types for configuring the Nginx server:

```yaml
# Certificate inputs with defaults
duration_days:
  type: integer
  default: 365

common_name:
  type: string
  default: "artifacts.edge.lab"

# Nginx inputs with defaults
nginx_htuser:
  type: string
  default: "admin"

nginx_htpass:
  type: string
  default: "Password123!"

nginx_port:
  type: integer
  default: 443

nginx_debug:
  type: integer
  default: 0
  description: |
    Set to 0 or 1 to enable or disable debug logging
```

All inputs have sensible defaults so the user only needs to select the target VM deployment to get a working Nginx server.

---

## Outputs (outputs.yaml)

```yaml
capabilities:

  nginx_url:
    description: NGINX web server URL
    value:
      concat:
        - 'https://'
        - { get_attribute: [ vm_platform, capabilities, vm_ip ] }
        - ':'
        - { get_input: nginx_port }

  nginx_username:
    description: NGINX web server username
    value: { get_input: nginx_htuser }

  nginx_password:
    description: NGINX web server password
    value: { get_input: nginx_htpass }
```

The outputs provide everything needed to access the deployed Nginx server:

- **`nginx_url`** -- constructed using `concat` to combine `https://`, the VM's IP address (read from the parent VM deployment's capabilities through the SharedResource), and the configured port
- **`nginx_username`** and **`nginx_password`** -- the basic auth credentials, echoed from the inputs

---

## Execution Flow

1. **`vm_platform`** establishes a reference to the target Simple VM deployment, making its capabilities available.
2. **`vm_info`** resolves the endpoint target ID and SSHs into the VM to gather current info (IP address, etc.).
3. **`install_file_server`** SSHs into the VM and runs the Nginx installation commands with the configured environment variables.
4. **Outputs** are computed, providing the Nginx URL and credentials.

On uninstall, the deployment reference is released, but the Nginx installation on the VM is not automatically removed (there is no `delete` lifecycle operation defined on `install_file_server`). To clean up the VM entirely, you would uninstall the parent Simple VM deployment.

---

## SharedResource Pattern Summary

The SharedResource pattern enables a **layered deployment model**:

```
Layer 0: NativeEdge Endpoint (physical hardware)
Layer 1: Simple VM deployment (creates the VM)
Layer 2: Nginx App deployment (installs app on the VM)
```

Each layer is an independent deployment with its own lifecycle. You can:
- Deploy multiple applications onto the same VM (multiple Layer 2 deployments referencing the same Layer 1)
- Update the application independently of the VM
- Replace the application without affecting the underlying VM

This pattern is the recommended approach when you want to separate infrastructure provisioning from application deployment, which mirrors real-world operations where different teams manage infrastructure and applications.
