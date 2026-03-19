# Utilities Plugin Overview

The NativeEdge Utilities Plugin is a multi-purpose plugin that bundles a wide range of infrastructure and configuration tools into a single package. Rather than installing separate plugins for cloud-init, SSH key generation, REST API calls, terminal commands, file management, and more, you import one plugin and gain access to all of these capabilities.

This page introduces the plugin, explains how it is structured, and provides a reference to every node type, relationship, data type, and workflow it offers.

## Plugin Information

| Detail | Value |
|---|---|
| Package Name | `utilities-plugin` |
| Version | `3.2.0.0` |
| Executor | `central_deployment_agent` |

The `central_deployment_agent` executor means that all operations defined by this plugin run on the NativeEdge manager itself, not on a remote host. This is important to understand: when you use this plugin, the manager is the machine that generates SSH keys, renders cloud-init configs, makes REST calls, and so on.

## How to Import

To use the Utilities Plugin in your blueprint, add the following line to the `imports` section of your blueprint YAML file:

```yaml
imports:
  - plugin:utilities-plugin?version=>=3.1.3.0
```

This tells NativeEdge to load the Utilities Plugin at version 3.1.3.0 or higher. The `plugin:` prefix is a special NativeEdge directive that resolves to an installed plugin package. The `?version=>=3.1.3.0` part is a version constraint -- it ensures that the plugin version loaded is at least 3.1.3.0 but will accept newer versions (such as 3.2.0.0).

You place this import alongside any other plugins or type definitions your blueprint needs. For example:

```yaml
imports:
  - dell/types/types.yaml
  - plugin:edge-plugin
  - plugin:utilities-plugin?version=>=3.1.3.0
  - plugin:fabric-plugin?version=>=3.3.2.0
```

## How the Plugin is Structured: Internal Aliases

When you look at the plugin.yaml source, you will notice that the `plugins` section defines many names -- but they all point to the same underlying package (`utilities-plugin` version `3.2.0.0`). This is an important design pattern to understand.

Here are all the internal plugin aliases registered by the Utilities Plugin:

| Alias Name | Purpose |
|---|---|
| `plugins_util` | The primary alias (anchor definition) |
| `plugins_files` | File management operations |
| `plugins_ftp` | FTP file transfer operations |
| `plugins_custom_workflow` | Custom batch deployment workflows |
| `plugins_hooks_workflow` | Hook-triggered workflow execution |
| `cloudinit` | Cloud-init configuration generation |
| `configuration` | Configuration loading and merging |
| `keys` | SSH key generation and management |
| `suspend` | Suspend, resume, backup, and restore workflows |
| `terminal` | Raw terminal (SSH) command execution |
| `rest` | REST API request execution |
| `scalelist` | Scale up/down list-based workflows |
| `secrets` | Secret reading and writing |
| `lifecycle_operations` | Alternative lifecycle operation workflows |
| `resources` | Resource list management and reservation |
| `iso` | ISO file modification |

**Why does this matter?** When you see an operation implementation like `cloudinit.plugins_cloudinit.tasks.update` or `keys.plugins_ssh_key.operations.create`, the first part before the dot (e.g., `cloudinit`, `keys`) is the plugin alias name. NativeEdge uses this alias to know which plugin package contains the code. Since all aliases point to the same `utilities-plugin` package, the code all lives in one place -- but different modules within that package handle different concerns.

This design means you only need one `import` line in your blueprint to access all of these capabilities.

## Node Types

The plugin provides the following node types. Each one represents a different kind of resource or configuration object you can declare in your blueprint.

| Node Type | Description | Details |
|---|---|---|
| [`dell.nodes.CloudInit.CloudConfig`](cloud-init.md) | Generates cloud-init configuration for VM initialization. Produces a cloud-config string that can be passed to virtual machines at boot time. | [Cloud Init](cloud-init.md) |
| [`dell.nodes.keys.RSAKey`](ssh-keys.md) | Generates RSA SSH key pairs and optionally stores them as NativeEdge secrets. Used to provide SSH access to VMs. | [SSH Keys](ssh-keys.md) |
| [`dell.nodes.secrets.Writer`](secrets.md) | Creates and manages NativeEdge secrets. Writes key-value entries to the secret store. | [Secrets Management](secrets.md) |
| [`dell.nodes.secrets.Reader`](secrets.md) | Reads existing secrets from the NativeEdge secret store and exposes them as runtime properties. | [Secrets Management](secrets.md) |
| [`dell.nodes.rest.Requests`](rest-requests.md) | Executes REST API calls using Jinja2 templates. Supports full HTTP lifecycle (create, configure, start, stop, delete). | [REST Requests](rest-requests.md) |
| [`dell.nodes.rest.BunchRequests`](rest-requests.md) | Executes multiple REST API call templates in a single batch. | [REST Requests](rest-requests.md) |
| [`dell.nodes.File`](files-and-ftp.md) | Manages files on the local system -- creates, deletes, and checks drift for files defined in the blueprint. | [Files and FTP](files-and-ftp.md) |
| [`dell.nodes.ftp`](files-and-ftp.md) | Uploads files to an FTP server. Supports TLS connections. | [Files and FTP](files-and-ftp.md) |
| [`dell.nodes.terminal.Raw`](terminal.md) | Sends raw commands to network devices or servers via SSH terminal sessions. | [Terminal Operations](terminal.md) |
| [`dell.nodes.resources.List`](resources.md) | Manages a list of resources that can be reserved and returned. Implements a resource pool pattern. | [Resource Management](resources.md) |
| [`dell.nodes.resources.ListItem`](resources.md) | Represents a single item within a resource list. | [Resource Management](resources.md) |
| [`dell.nodes.resources.ModifiedIso`](resources.md) | Creates modified ISO images by adding new files and directories to an existing ISO. | [Resource Management](resources.md) |
| [`dell.nodes.ConfigurationLoader`](configuration.md) | Loads JSON configuration data and exposes it as runtime properties for other nodes to consume. | [Configuration Loader](configuration.md) |

## Data Types

The plugin defines custom data types used as property types in the node types above:

| Data Type | Used By | Description |
|---|---|---|
| `dell.datatypes.key` | `dell.nodes.keys.RSAKey` | SSH key configuration: algorithm, bits, paths, key name, passphrase, format |
| `dell.datatypes.ftp_auth` | `dell.nodes.ftp` | FTP server connection credentials: user, password, IP, port, TLS settings |
| `dell.datatypes.terminal_auth` | `dell.nodes.terminal.Raw` | SSH terminal connection credentials: user, password, IP, port, prompt patterns, error patterns |
| `dell.datatypes.File` | `dell.nodes.File` | File specification: source path in blueprint, destination path, owner, mode, template variables |
| `dell.datatypes.CloudInit.SecretProperty` | `dell.nodes.CloudInit.CloudConfig` | Controls whether cloud-init values are stored as secrets: enabled flag, prefix, overrides |

## Relationships

| Relationship | Description | Details |
|---|---|---|
| [`dell.relationships.load_from_config`](relationships.md) | Loads configuration parameters from a `ConfigurationLoader` node into the source node's runtime properties. | [Relationships](relationships.md) |
| [`dell.relationships.resources.reserve_list_item`](relationships.md) | Reserves an item from a `resources.List` node when a relationship is established, and returns it when the relationship is removed. | [Relationships](relationships.md) |

## Workflows

The plugin registers many workflows that can be executed against deployments. These are grouped by category:

### Configuration Workflows

| Workflow | Description | Details |
|---|---|---|
| [`configuration_update`](workflows.md#configuration_update) | Updates and merges configuration values | [Workflows](workflows.md) |

### Suspend, Resume, and Backup Workflows

| Workflow | Description | Details |
|---|---|---|
| [`suspend`](workflows.md#suspend) | Pauses deployment execution | [Workflows](workflows.md) |
| [`resume`](workflows.md#resume) | Resumes a previously suspended deployment | [Workflows](workflows.md) |
| [`statistics`](workflows.md#statistics) | Returns system usage and performance statistics | [Workflows](workflows.md) |
| [`backup`](workflows.md#backup) | Creates a backup of the deployment's data | [Workflows](workflows.md) |
| [`restore`](workflows.md#restore) | Restores from a previously created backup | [Workflows](workflows.md) |
| [`remove_backup`](workflows.md#remove_backup) | Deletes a saved backup | [Workflows](workflows.md) |

### Scale Workflows

| Workflow | Description | Details |
|---|---|---|
| [`scaleuplist`](workflows.md#scaleuplist) | Adds instances to specified nodes in bulk | [Workflows](workflows.md) |
| [`scaledownlist`](workflows.md#scaledownlist) | Removes instances from specified nodes in bulk | [Workflows](workflows.md) |

### Operation Workflows

| Workflow | Description | Details |
|---|---|---|
| [`update_operation_filtered`](workflows.md#update_operation_filtered) | Executes an operation on filtered node instances | [Workflows](workflows.md) |

### Hook Workflows

| Workflow | Description | Details |
|---|---|---|
| [`hook_workflow_run_filtered`](workflows.md#hook_workflow_run_filtered) | Runs a workflow triggered by a hook with filtering | [Workflows](workflows.md) |
| [`hook_workflow_rest`](workflows.md#hook_workflow_rest) | Executes REST calls as a hook-triggered workflow | [Workflows](workflows.md) |
| [`hook_workflow_terminal`](workflows.md#hook_workflow_terminal) | Runs terminal commands as a hook-triggered workflow | [Workflows](workflows.md) |

### Alternative Lifecycle Workflows

| Workflow | Description | Details |
|---|---|---|
| [`alt_start`](workflows.md#alt_start) | Alternative start operation workflow | [Workflows](workflows.md) |
| [`alt_stop`](workflows.md#alt_stop) | Alternative stop operation workflow | [Workflows](workflows.md) |
| [`alt_precreate`](workflows.md#alt_precreate) | Alternative pre-create operation workflow | [Workflows](workflows.md) |
| [`alt_create`](workflows.md#alt_create) | Alternative create operation workflow | [Workflows](workflows.md) |
| [`alt_configure`](workflows.md#alt_configure) | Alternative configure operation workflow | [Workflows](workflows.md) |
| [`alt_poststart`](workflows.md#alt_poststart) | Alternative post-start operation workflow | [Workflows](workflows.md) |
| [`alt_prestop`](workflows.md#alt_prestop) | Alternative pre-stop operation workflow | [Workflows](workflows.md) |
| [`alt_delete`](workflows.md#alt_delete) | Alternative delete operation workflow | [Workflows](workflows.md) |
| [`alt_postdelete`](workflows.md#alt_postdelete) | Alternative post-delete operation workflow | [Workflows](workflows.md) |

### Deprecated Workflows

| Workflow | Description | Details |
|---|---|---|
| [`rollback_deprecated`](workflows.md#rollback_deprecated) | Reverts changes (deprecated) | [Workflows](workflows.md) |

### Batch Workflows

| Workflow | Description | Details |
|---|---|---|
| [`batch_deploy_and_install`](workflows.md#batch_deploy_and_install) | Deploys and installs multiple deployments at once | [Workflows](workflows.md) |
| [`batch_deploy`](workflows.md#batch_deploy) | Deploys multiple deployments in one operation | [Workflows](workflows.md) |
| [`batch_install`](workflows.md#batch_install) | Installs multiple deployments in a single action | [Workflows](workflows.md) |
