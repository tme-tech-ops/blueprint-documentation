# Terminal Operations

The Utilities Plugin provides the `dell.nodes.terminal.Raw` node type for sending raw commands to network devices or servers via SSH terminal sessions. Unlike the Fabric plugin (which executes individual commands over SSH), the Terminal node establishes an interactive SSH session and sends commands as if a human were typing them at a terminal prompt. This makes it particularly suited for network devices (routers, switches, firewalls) that use CLI-based configuration interfaces where commands must be sent sequentially in a session context.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.
- A target device or server reachable via SSH from the NativeEdge manager.

## Node Type: dell.nodes.terminal.Raw

**Derived from:** `dell.nodes.Root`

This node type opens an SSH connection to a remote device, sends commands, waits for expected prompts, and captures output. It is designed for devices that require interactive terminal sessions rather than discrete command execution.

### Data Type: dell.datatypes.terminal_auth

This data type defines the SSH connection credentials and session behavior. It is used as the type of the `terminal_auth` property.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `user` | string | No | `''` | The SSH username for logging into the remote device. |
| `password` | string | No | `''` | The SSH password. Used for password-based authentication. |
| `ip` | string | No | `''` | The IP address or hostname of the remote device. |
| `key_content` | string | No | `''` | The SSH private key content for key-based authentication. If provided, this is used instead of password authentication. You can retrieve this from the secret store using `get_secret`. |
| `port` | integer | No | `22` | The SSH port number. The standard SSH port is 22. |
| `store_logs` | boolean | No | `false` | When `true`, the full communication log (all sent commands and received responses) is saved. Useful for debugging but may produce large amounts of data. |
| `promt_check` | list | No | `[]` | A list of prompt strings that indicate the device is ready for the next command. For example, `["#", "$", ">"]`. When the plugin sees one of these strings at the end of the output, it knows the previous command has completed. If empty, the default prompts `#`, `$` are used. (Note: the property name `promt_check` contains a typo in the original plugin definition -- it is missing the "p" in "prompt".) |
| `warnings` | list | No | `[]` | A list of strings that indicate warning conditions in the device output. If any of these strings appear in the output (without a newline), the plugin logs a warning. |
| `errors` | list | No | `[]` | A list of strings that indicate error conditions in the device output. If any of these strings appear in the output, the plugin raises a recoverable error (which may trigger a retry). |
| `criticals` | list | No | `[]` | A list of strings that indicate critical error conditions. If any of these appear in the output, the plugin raises a non-recoverable error (which fails the operation immediately). |
| `exit_command` | string | No | `'exit'` | The command sent to close the SSH session. The default is `exit`, which works for most systems. Change this if your device uses a different command to disconnect (e.g., `logout`, `quit`). |
| `smart_device` | boolean | No | `false` | When `true`, enables shell extension mode for "smart" devices that may require special handling of the terminal session (e.g., devices that use paging, require enable mode, or have multi-level prompts). |

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `terminal_auth` | dell.datatypes.terminal_auth | Yes | (none) | The SSH connection and session configuration. See the data type table above. |

### Runtime Properties

| Runtime Property | Type | Description |
|---|---|---|
| `_finished_operations` | dict | A dictionary tracking which lifecycle operations have completed. The keys are operation names and the values indicate completion status. This is used internally to prevent duplicate execution. |

### Lifecycle Operations

All lifecycle operations call the same implementation (`terminal.plugins_terminal.tasks.run`). The commands to execute during each operation are provided as operation inputs in the blueprint.

| Operation | Implementation | Inputs | Description |
|---|---|---|---|
| `create` | `terminal.plugins_terminal.tasks.run` | `{}` (user-defined) | Opens an SSH session, sends commands, captures output. |
| `configure` | `terminal.plugins_terminal.tasks.run` | `{}` (user-defined) | Same as create -- sends commands for the configure phase. |
| `start` | `terminal.plugins_terminal.tasks.run` | `{}` (user-defined) | Same -- sends commands for the start phase. |
| `stop` | `terminal.plugins_terminal.tasks.run` | `{}` (user-defined) | Same -- sends commands for the stop phase. |
| `delete` | `terminal.plugins_terminal.tasks.run` | `{}` (user-defined) | Same -- sends commands for the delete phase. |

The `inputs: {}` in the plugin.yaml means that the operation accepts arbitrary inputs that you define in your blueprint. Typically, you pass a `calls` input containing the list of commands to execute.

### Example: Configuring a Network Device

```yaml
node_templates:
  router_config:
    type: dell.nodes.terminal.Raw
    properties:
      terminal_auth:
        user: admin
        password: { get_secret: router_password }
        ip: 192.168.1.1
        port: 22
        promt_check:
          - "#"
          - ">"
        errors:
          - "% Invalid"
          - "Error"
        exit_command: exit
    interfaces:
      dell.interfaces.lifecycle:
        create:
          inputs:
            calls:
              - action: enable
              - action: configure terminal
              - action: interface GigabitEthernet0/0
              - action: ip address 10.0.0.1 255.255.255.0
              - action: no shutdown
              - action: end
              - action: write memory
        delete:
          inputs:
            calls:
              - action: enable
              - action: configure terminal
              - action: interface GigabitEthernet0/0
              - action: shutdown
              - action: end
              - action: write memory
```

### How This Example Works

1. **On create:** The plugin SSHs into the router at `192.168.1.1`, enters enable mode, enters configuration mode, configures an interface with an IP address, brings it up, saves the configuration.

2. **On delete:** The plugin SSHs in again and shuts down the interface, reversing the configuration.

3. **Prompt checking:** After each command, the plugin waits for the output to end with `#` or `>`, indicating the device is ready for the next command.

4. **Error detection:** If the device responds with `"% Invalid"` or `"Error"`, the plugin recognizes this as an error condition and handles it according to the error handling policy.

## Key Concepts for New Users

### Terminal vs. Fabric Plugin

NativeEdge offers two ways to run commands on remote hosts:

- **Fabric plugin** -- Executes individual, discrete commands over SSH. Best for Linux servers where you run shell commands like `apt-get install`, `systemctl start`, etc.
- **Terminal node (this plugin)** -- Establishes an interactive SSH session and sends commands sequentially, waiting for prompts between commands. Best for network devices (Cisco, Juniper, Fortinet, etc.) that have CLI-based configuration interfaces with modes and prompts.

### The Calls Pattern

Commands are typically passed as a list of `calls` in the operation inputs. Each call has an `action` field containing the command string. The plugin sends each action in sequence, waiting for a prompt response between commands.

### Warning, Error, and Critical Detection

The `warnings`, `errors`, and `criticals` lists in `terminal_auth` let you define patterns that the plugin watches for in the device output:

- **Warnings** -- Logged but do not stop execution.
- **Errors** -- Raise a recoverable error, which may trigger a retry.
- **Criticals** -- Raise a non-recoverable error, which fails the operation immediately.

This pattern-matching approach is essential for network device automation, where the device communicates success or failure through text output rather than exit codes.
