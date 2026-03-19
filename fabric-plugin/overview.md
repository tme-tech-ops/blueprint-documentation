# Fabric Plugin Overview

## What is the Fabric Plugin?

The NativeEdge Fabric Plugin enables you to execute shell commands and scripts on remote machines over SSH directly from your NativeEdge blueprints. It is built on the [Python Fabric 2](https://www.fabfile.org/) library, which itself uses [Paramiko](https://www.paramiko.org/) for SSH connectivity.

Use this plugin when you need to:

- Run one or more shell commands on a remote host (for example, installing packages, cloning repositories, or restarting services).
- Execute a shell script file that is bundled in your blueprint on a remote host.
- Run a Python task function on a remote host through an SSH connection.

The plugin handles all SSH connection management, key authentication, proxy tunneling, and output logging automatically.

## Plugin Details

| Property | Value |
|---|---|
| Package name | `fabric-plugin` |
| Plugin version | `3.3.3.0` |
| Executor | `central_deployment_agent` |
| Underlying library | Python Fabric 2 (with Paramiko) |

## How to Import the Plugin

Add the following line to the `imports` section at the top of your blueprint YAML file:

```yaml
imports:
  - plugin:fabric-plugin?version=>=3.3.2.0
```

The `?version=>=3.3.2.0` part is a version constraint. It tells NativeEdge to use any installed version of the fabric plugin that is 3.3.2.0 or newer. You can pin to an exact version if needed (for example, `?version=3.3.3.0`).

## Key Concept: No Node Types

Unlike most NativeEdge plugins, the Fabric Plugin does **not** define any `node_types`. If you look at the plugin's `plugin.yaml`, you will see that it only registers the plugin itself -- there are no type definitions:

```yaml
plugins:
  fabric:
    executor: central_deployment_agent
    package_name: fabric-plugin
    package_version: '3.3.3.0'
```

This is a critical conceptual difference from plugins like the Edge Plugin or Utilities Plugin, which define their own node types (such as `dell.nodes.nativeedge.template.NativeEdgeVM`).

Instead, the Fabric Plugin provides **operations** -- callable functions that you reference in the `interfaces` section of a node template that uses an existing node type. In practice, you will typically use a base type like `dell.nodes.Root` and then wire up the Fabric Plugin operations in the lifecycle interface of that node.

Here is a simplified example showing this pattern:

```yaml
node_templates:
  my_remote_task:
    type: dell.nodes.Root              # <-- uses an existing node type
    interfaces:
      dell.interfaces.lifecycle:
        start:
          implementation: fabric.fabric_plugin.tasks.run_commands   # <-- fabric operation
          inputs:
            commands:
              - echo "Hello from the remote host"
            fabric_env:
              host_string: 10.0.0.5
              user: ubuntu
              key_filename: /path/to/key.pem
```

The string `fabric.fabric_plugin.tasks.run_commands` tells NativeEdge: "use the `run_commands` function from the `fabric_plugin.tasks` module provided by the plugin named `fabric`." The plugin name `fabric` comes from the key in the `plugins` section of `plugin.yaml`.

## Available Operations

The Fabric Plugin provides four operations. Each operation is documented in detail in the [Operations](operations.md) page.

| Operation | Description |
|---|---|
| [`fabric.fabric_plugin.tasks.run_commands`](operations.md#run_commands) | Run a list of shell commands sequentially on a remote host. |
| [`fabric.fabric_plugin.tasks.run_script`](operations.md#run_script) | Upload and run a script file from your blueprint on a remote host. |
| [`fabric.fabric_plugin.tasks.run_task`](operations.md#run_task) | Run a Python task function loaded from a tasks file in your blueprint. |
| [`fabric.fabric_plugin.tasks.run_module_task`](operations.md#run_module_task) | Run a Python task function specified by a fully-qualified module path. |

## Prerequisites

Before using the Fabric Plugin, make sure you have:

1. **The Fabric Plugin installed** on your NativeEdge manager. The plugin package `fabric-plugin` version `3.3.3.0` (or compatible) must be uploaded.
2. **SSH access** to the target remote host. You need either an SSH private key or a password for authentication.
3. **Network connectivity** between the NativeEdge manager (or its proxy/oxy endpoint) and the target host on the SSH port (default 22).
4. **A base node type** to attach operations to. The most common choice is `dell.nodes.Root`, which is available from `dell/types/types.yaml`.

## Next Steps

- [Operations Reference](operations.md) -- detailed documentation for each operation, including all parameters, the `fabric_env` configuration object, proxy settings, and real-world YAML examples.
