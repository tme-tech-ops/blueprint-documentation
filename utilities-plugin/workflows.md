# Workflows

Workflows in NativeEdge are executable processes that operate on deployments. While the standard lifecycle workflows (`install`, `uninstall`, `execute_operation`) are built into NativeEdge, plugins can register custom workflows that provide additional capabilities. The Utilities Plugin registers a large number of workflows covering configuration updates, suspend/resume, backup/restore, scaling, operation filtering, hooks, alternative lifecycle operations, and batch deployment.

This page documents every workflow defined by the Utilities Plugin.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.
- Understanding of NativeEdge deployment lifecycle and workflow execution.

## How Workflows Work (for New Users)

A workflow is a named process you can execute against a deployment. You trigger workflows through the NativeEdge UI or CLI. Each workflow has:

- **mapping** -- The Python function that implements the workflow.
- **description** -- A human-readable explanation of what the workflow does.
- **display_label** -- The label shown in the NativeEdge UI.
- **availability_rules** -- Conditions that determine when the workflow can be run (e.g., "all node instances must be active").
- **parameters** -- Input values you provide when executing the workflow.

When you run a workflow, NativeEdge calls the mapped function with the provided parameters. The function can then interact with node instances, execute operations, modify runtime properties, and more.

---

## Configuration Workflows

### configuration_update

Updates and merges configuration values on a running deployment. This workflow works with the `dell.nodes.ConfigurationLoader` node type and `dell.relationships.load_from_config` relationship to push updated configuration to all connected nodes.

| Detail | Value |
|---|---|
| Mapping | `configuration.plugins_configuration.tasks.update` |
| Display Label | Configuration Update |
| Availability Rules | `node_instances_active: ['all']` |
| Required Operations | `dell.interfaces.lifecycle.is_alive`, `dell.interfaces.lifecycle.configure`, `dell.interfaces.relationship_lifecycle.preconfigure`, `dell.interfaces.lifecycle.update` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `params` | string | Yes | (none) | A JSON string containing the new configuration values to apply. |
| `configuration_node_id` | node_id | No | `configuration_loader` | The node ID of the ConfigurationLoader node to update. Defaults to `configuration_loader`, which is the conventional name. |
| `merge_dict` | boolean | No | `false` | When `true`, the new parameters are merged with existing configuration. When `false`, the new parameters replace the existing configuration. |
| `node_types_to_update` | list | No | `[juniper_node_config, fortinet_vnf_type]` | A list of node type names that should receive the updated configuration. Only nodes of these types (or derived types) will be updated. |

---

## Suspend, Resume, and Backup Workflows

These workflows manage deployment state -- pausing execution, resuming it, collecting statistics, and managing backups.

### suspend

Pauses deployment execution. Only available if the deployment's nodes support the suspend operation.

| Detail | Value |
|---|---|
| Mapping | `suspend.plugins_suspend.workflows.suspend` |
| Display Label | Suspend |
| Availability Rules | `node_instances_active: ['all']` |
| Required Operations | `dell.interfaces.lifecycle.suspend`, `dell.interfaces.freeze.suspend` |

**Parameters:** None.

---

### resume

Resumes execution of a previously suspended deployment. Continues from the point where the deployment was paused.

| Detail | Value |
|---|---|
| Mapping | `suspend.plugins_suspend.workflows.resume` |
| Display Label | Resume |
| Availability Rules | `node_instances_active: ['all', 'partial']` |
| Required Operations | `dell.interfaces.freeze.resume`, `dell.interfaces.lifecycle.resume` |

**Parameters:** None.

---

### statistics

Returns system usage and performance statistics. Collects resource utilization and runtime metrics from nodes that implement the statistics interface.

| Detail | Value |
|---|---|
| Mapping | `suspend.plugins_suspend.workflows.statistics` |
| Display Label | Statistics |
| Availability Rules | `node_instances_active: ['all']` |
| Required Operations | `dell.interfaces.statistics.perfomance` |

**Parameters:** None.

---

### backup

Creates a backup of the deployment's data. Saves the current system state for later recovery.

| Detail | Value |
|---|---|
| Mapping | `suspend.plugins_suspend.workflows.backup` |
| Display Label | Backup |
| Availability Rules | `node_instances_active: ['all', 'partial']` |
| Required Operations | `dell.interfaces.freeze.fs_prepare`, `dell.interfaces.snapshot.create`, `dell.interfaces.freeze.fs_finalize` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `snapshot_name` | string | No | `""` | A name or tag for the backup. Used to identify this backup when restoring or removing it. |
| `snapshot_incremental` | boolean | No | `true` | When `true`, creates an incremental backup (only changes since the last backup). When `false`, creates a full backup. Incremental backups are faster and use less storage. |
| `snapshot_type` | string | No | `irregular` | The backup type classification, such as `daily`, `weekly`, or `irregular`. Used for organizational purposes. |
| `snapshot_rotation` | integer | No | `1` | How many backups to keep. Older backups beyond this count may be removed. |

---

### restore

Restores the deployment from a previously created backup. Overwrites the current state with the saved data.

| Detail | Value |
|---|---|
| Mapping | `suspend.plugins_suspend.workflows.restore` |
| Display Label | Restore |
| Availability Rules | `node_instances_active: ['partial', 'none']` |
| Required Operations | `dell.interfaces.freeze.fs_prepare`, `dell.interfaces.snapshot.apply`, `dell.interfaces.freeze.fs_finalize` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `snapshot_name` | string | No | `""` | The name or tag of the backup to restore from. Must match a previously created backup name. |
| `snapshot_incremental` | boolean | No | `true` | Whether to restore from an incremental or full backup. Must match the type of backup that was created. |

---

### remove_backup

Deletes a saved backup. Use this to free storage or clean up old backups before reconfiguring resources (some cloud providers partially lock VM reconfiguration if backups exist).

| Detail | Value |
|---|---|
| Mapping | `suspend.plugins_suspend.workflows.remove_backup` |
| Display Label | Remove Backup |
| Availability Rules | (none specified) |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `snapshot_name` | string | No | `""` | The name or tag of the backup to remove. |
| `snapshot_incremental` | boolean | No | `true` | Whether the backup to remove is incremental or full. |

---

## Scale Workflows

These workflows manage scaling -- adding or removing node instances in bulk.

### scaleuplist

Adds instances to specified nodes in bulk. Use this to scale out selected node types by providing a list of properties for each new instance.

| Detail | Value |
|---|---|
| Mapping | `scalelist.plugins_scalelist.workflows.scaleuplist` |
| Display Label | Scale Up List |
| Availability Rules | `node_instances_active: ['all', 'partial']` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `scalable_entity_properties` | dict | No | `{}` | A dictionary defining the properties for each node to scale. The keys are node names and the values are the properties for the new instances. |
| `scale_compute` | boolean | No | `false` | If a node name is passed as a scalable entity and that node is contained within a compute node, setting this to `true` operates on the compute node instead of the specified node. This ensures the entire compute stack scales together. |
| `ignore_failure` | boolean | No | `false` | When `true`, failures during scaling are logged but do not stop the workflow. |
| `ignore_rollback_failure` | boolean | No | `true` | When `true`, failures during rollback (if the scale operation fails and needs to be reversed) are logged but do not stop the rollback. |
| `scale_transaction_field` | string | No | `_transaction_id` | The runtime property field name used to store the transaction ID for instances created in the same scaling operation. This allows tracking which instances were created together. |
| `scale_transaction_value` | string | No | `""` | An optional explicit transaction value. If empty, a value is auto-generated. |
| `node_sequence` | list/boolean | No | `false` | An optional sequence of nodes to define the order of scaling operations, overriding the default relationship-based ordering. |
| `rollback_on_failure` | boolean | No | `true` | When `true`, if the scale operation fails, the deployment modification is rolled back (the partially created instances are removed). |

---

### scaledownlist

Removes instances from specified nodes in bulk. Use this to scale in selected node types by filtering which instances to remove.

| Detail | Value |
|---|---|
| Mapping | `scalelist.plugins_scalelist.workflows.scaledownlist` |
| Display Label | Scale Down List |
| Availability Rules | `node_instances_active: ['all', 'partial']` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `scale_compute` | boolean | No | `false` | If the target node is contained within a compute node, setting this to `true` operates on the compute node instead. |
| `ignore_failure` | boolean | No | `false` | When `true`, failures are logged but do not stop the workflow. |
| `scale_transaction_field` | string | No | `_transaction_id` | The runtime property field name to search for transaction IDs. Used to identify which instances to remove. Can be skipped if instances should be removed without relation to the initial scaling transaction. |
| `scale_node_name` | string | No | `""` | The name of the node type to scale down. An empty string means no filtering by node name. |
| `scale_node_field` | string/list | No | `""` | The runtime property field name to filter by. Supports nested field access using a list (e.g., `['a', 'b']` searches for `runtime_properties['a']['b']`). |
| `scale_node_field_value` | string/list | No | `""` | The value to match in the field specified by `scale_node_field`. Can be a single value or a list of possible values. Only instances whose field matches one of these values will be scaled down. |
| `force_db_cleanup` | boolean | No | `false` | **Deprecated since 1.11.0.** Previously ran DB cleanup directly if instances could not be deleted in one transaction. This flag no longer has any effect. |
| `all_results` | boolean | No | `false` | When `true`, retrieves all matching instances for the filter. Requires manager version 4.3 or later. |
| `node_sequence` | list/boolean | No | `false` | An optional sequence of nodes to define the order of scale-down operations. |
| `force_remove` | boolean | No | `true` | When `true`, forcefully removes node instances set for scale down based on the transaction field. |
| `rollback_on_failure` | boolean | No | `true` | When `true`, if the scale-down fails, the deployment modification is rolled back. |

---

## Operation Workflows

### update_operation_filtered

Executes a specified operation on filtered node instances. This is a flexible workflow that lets you run any operation on a subset of nodes, filtered by type, node ID, instance ID, or runtime property values.

| Detail | Value |
|---|---|
| Mapping | `scalelist.plugins_scalelist.workflows.execute_operation` |
| Display Label | Update Operation Filtered |
| Availability Rules | `node_instances_active: ['all', 'partial']` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `operation` | string | No | `dell.interfaces.lifecycle.update` | The fully qualified name of the operation to execute. |
| `operation_kwargs` | dict | No | `{}` | A dictionary of keyword arguments passed to the operation invocation. |
| `allow_kwargs_override` | boolean/null | No | `null` | Whether overriding operation inputs defined in the blueprint with inputs from `operation_kwargs` is allowed. When `null`, the default behavior of the workflow infrastructure is used. |
| `run_by_dependency_order` | boolean | No | `false` | When `true`, operations execute on nodes in dependency order (respecting relationships). When `false`, all matching nodes execute in parallel. |
| `type_names` | list | No | `[]` | Filter by node type. Only instances of these types (or derived types) are included. Empty means no type filtering. |
| `node_ids` | list (node_id) | No | `[]` | Filter by node ID. Only instances of these specific nodes are included. Empty means no node ID filtering. |
| `node_instance_ids` | list (node_instance) | No | `[]` | Filter by node instance ID. Only these specific instances are included. Empty means no instance ID filtering. |
| `node_field` | string/list | No | `""` | Runtime property field name to filter by. Supports nested access via list notation. |
| `node_field_value` | string/list | No | `""` | The value to match in the runtime property field. Can be a list of possible values. |

---

## Hook Workflows

Hook workflows are triggered by NativeEdge events (hooks). They allow automated responses to deployment events, such as running a workflow when another deployment completes installation.

### hook_workflow_run_filtered

Runs a specified workflow on a deployment, triggered by a hook event. Includes filtering to control under which conditions the workflow should execute.

| Detail | Value |
|---|---|
| Mapping | `plugins_hooks_workflow.plugins_hooks_workflow.tasks.run_workflow` |
| Display Label | Hook Workflow Run Filtered |
| Availability Rules | (none specified) |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `inputs` | dict | No | `{}` | Inputs for the hook, such as `deployment_id` and `workflow_id`. These are automatically populated by the hook system when the workflow is triggered by an event. |
| `logger_file` | string | No | `""` | Path to a file for duplicating logger output. Useful for debugging hook execution. |
| `client_config` | dict | No | `{}` | Custom credentials for the NativeEdge manager API. By default, the workflow uses the built-in credentials. Provide custom credentials if the hook needs to authenticate as a different user. |
| `filter_by` | list | Yes | (none) | A list of filter conditions. Each condition is a dictionary with `path` (a list of keys to traverse in the hook input) and `values` (a list of acceptable values). All conditions must match for the workflow to execute. |
| `workflow_for_run` | string | No | `uninstall` | The name of the workflow to execute on the target deployment when the filter conditions match. |
| `workflow_params` | dict | No | `{}` | Additional parameters to pass to the triggered workflow. |

**Filter example:**

```yaml
filter_by:
  - path: ["workflow_id"]
    values: ["install"]
  - path: ["deployment_capabilities", "autouninstall", "value"]
    values: [true, "yes"]
```

This filter matches when the triggering event's `workflow_id` is `install` AND the deployment has a capability `autouninstall` with value `true` or `"yes"`.

---

### hook_workflow_rest

Executes REST API calls as a hook-triggered workflow. This allows you to make HTTP calls to external services in response to NativeEdge events.

| Detail | Value |
|---|---|
| Mapping | `rest.plugins_rest.tasks.execute_as_workflow` |
| Display Label | Hook Workflow Rest |
| Availability Rules | (none specified) |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `inputs` | dict | No | `{}` | Hook inputs (automatically populated by the hook system). |
| `logger_file` | string | No | `""` | Path to a file for duplicating logger output. |
| `properties` | dict | No | `{}` | Connection properties for the REST server (same format as `dell.nodes.rest.Requests` properties: `hosts`, `port`, `ssl`, `verify`, etc.). |
| `params` | dict | No | `{}` | Template parameters. Merged with any existing params and includes a `ctx` key for the current context. |
| `template_file` | string | No | `''` | Absolute path to the Jinja2 template file defining the REST calls. |
| `save_path` | string/boolean | No | `false` | Where to save results. |
| `prerender` | boolean | No | `false` | Whether to pre-render the Jinja2 template before YAML parsing. |
| `remove_calls` | boolean | No | `false` | Whether to remove call details from results. |

---

### hook_workflow_terminal

Runs terminal (SSH) commands as a hook-triggered workflow. This allows you to execute commands on remote devices in response to NativeEdge events.

| Detail | Value |
|---|---|
| Mapping | `terminal.plugins_terminal.tasks.run_as_workflow` |
| Display Label | Hook Workflow Terminal |
| Availability Rules | (none specified) |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `inputs` | dict | No | `{}` | Hook inputs (automatically populated by the hook system). |
| `logger_file` | string | No | `""` | Path to a file for duplicating logger output. |
| `terminal_auth` | dell.datatypes.terminal_auth | Yes | (none) | Terminal connection credentials. See [Terminal Operations](terminal.md) for the full data type specification. |
| `calls` | list | No | `[]` | A list of commands to execute on the remote device. |

---

## Alternative Lifecycle Workflows

These workflows provide alternative versions of the standard NativeEdge lifecycle operations. They allow you to run specific lifecycle phases (start, stop, create, delete, etc.) with filtering capabilities -- selecting which nodes to operate on by type, node ID, or instance ID.

All alternative lifecycle workflows share a common set of parameters. The key differences are which lifecycle operation they execute and their availability rules.

### Common Parameters for Alternative Lifecycle Workflows

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `operation_parms` | dict | No | `{}` | Additional parameters passed to the lifecycle operation. |
| `run_by_dependency_order` | boolean | No | `true` | When `true`, operations execute respecting relationship dependency order. When `false`, all matching nodes execute in parallel. |
| `type_names` | list | No | `[]` | Filter by node type. Only instances of these types are included. Empty means no filtering. |
| `node_ids` | list (node_id) | No | `[]` | Filter by node ID. Only instances of these nodes are included. The list is constrained to nodes that have the relevant operation defined. |
| `node_instance_ids` | list (node_instance) | No | `[]` | Filter by node instance ID. Only these specific instances are included. |
| `ignore_failure` | boolean | No | `false` | When `true`, failures are logged but do not stop the workflow. (Not available on `alt_start`.) |

---

### alt_start

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.start` |
| Display Label | Alt Start |
| Availability Rules | `node_instances_active: ['all', 'partial']` |
| Required Operations | `dell.interfaces.lifecycle.start` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.start']` |

Runs the `start` lifecycle operation on filtered nodes. Use this when you need to start specific nodes without running a full install workflow.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids` (see common parameters above). Does not include `ignore_failure`.

---

### alt_stop

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.stop` |
| Display Label | Alt Stop |
| Availability Rules | `node_instances_active: ['all', 'partial']` |
| Required Operations | `dell.interfaces.lifecycle.stop` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.stop']` |

Runs the `stop` lifecycle operation on filtered nodes.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids`, `ignore_failure` (see common parameters above).

---

### alt_precreate

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.precreate` |
| Display Label | Alt Pre-Create |
| Availability Rules | `node_instances_active: ['partial', 'none']` |
| Required Operations | `dell.interfaces.lifecycle.precreate` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.precreate']` |

Runs the `precreate` lifecycle operation on filtered nodes.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids`, `ignore_failure` (see common parameters above).

---

### alt_create

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.create` |
| Display Label | Alt Create |
| Availability Rules | `node_instances_active: ['partial', 'none']` |
| Required Operations | `dell.interfaces.lifecycle.create` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.create']` |

Runs the `create` lifecycle operation on filtered nodes.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids`, `ignore_failure` (see common parameters above).

---

### alt_configure

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.configure` |
| Display Label | Alt Configure |
| Availability Rules | `node_instances_active: ['partial', 'none']` |
| Required Operations | `dell.interfaces.lifecycle.configure` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.configure']` |

Runs the `configure` lifecycle operation on filtered nodes.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids`, `ignore_failure` (see common parameters above).

---

### alt_poststart

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.poststart` |
| Display Label | Alt Post Start |
| Availability Rules | `node_instances_active: ['all', 'partial']` |
| Required Operations | `dell.interfaces.lifecycle.poststart` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.poststart']` |

Runs the `poststart` lifecycle operation on filtered nodes.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids`, `ignore_failure` (see common parameters above).

---

### alt_prestop

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.prestop` |
| Display Label | Alt Pre-Stop |
| Availability Rules | `node_instances_active: ['all', 'partial']` |
| Required Operations | `dell.interfaces.lifecycle.prestop` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.prestop']` |

Runs the `prestop` lifecycle operation on filtered nodes.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids`, `ignore_failure` (see common parameters above).

---

### alt_delete

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.delete` |
| Display Label | Alt Delete |
| Availability Rules | `node_instances_active: ['all', 'partial']` |
| Required Operations | `dell.interfaces.lifecycle.delete` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.delete']` |

Runs the `delete` lifecycle operation on filtered nodes.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids`, `ignore_failure` (see common parameters above).

---

### alt_postdelete

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.postdelete` |
| Display Label | Alt Post Delete |
| Availability Rules | `node_instances_active: ['all', 'partial']` |
| Required Operations | `dell.interfaces.lifecycle.postdelete` |
| Node ID Constraint | `has_operation: ['dell.interfaces.lifecycle.postdelete']` |

Runs the `postdelete` lifecycle operation on filtered nodes. Performs cleanup or final steps after a resource is deleted.

**Parameters:** `operation_parms`, `run_by_dependency_order`, `type_names`, `node_ids`, `node_instance_ids`, `ignore_failure` (see common parameters above).

---

## Deprecated Workflows

### rollback_deprecated

Reverts changes made by a deployment. This workflow is deprecated and should be replaced with the alternative lifecycle workflows or the standard uninstall workflow.

| Detail | Value |
|---|---|
| Mapping | `lifecycle_operations.plugins_rollback_workflow.workflows.rollback` |
| Display Label | Rollback Deprecated |
| Availability Rules | `node_instances_active: ['all', 'partial']` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `type_names` | list (node_type) | No | `[]` | A list of type names. Only instances of these types (or derived types) will be rolled back. Empty means no filtering. |
| `node_ids` | list (node_id) | No | `[]` | A list of node IDs to roll back. Empty means no filtering. |
| `node_instance_ids` | list (node_id) | No | `[]` | A list of node instance IDs to roll back. Empty means no filtering. |
| `full_rollback` | boolean | No | `false` | When `false`, rolls back to the resolved state (partial rollback). When `true`, performs a full uninstall rollback. |

---

## Batch Workflows

These workflows enable bulk deployment operations -- creating and installing multiple deployments at once. They are useful for provisioning environments at scale, where you need to create many similar deployments from the same blueprint.

### batch_deploy_and_install

Deploys and installs multiple deployments in a single operation. Creates one new deployment per parent deployment provided, using the specified blueprint, and then installs all of them.

| Detail | Value |
|---|---|
| Mapping | `plugins_custom_workflow.plugins_custom_workflow.tasks.batch_deploy_and_install` |
| Display Label | Batch Deploy and Install |
| Availability Rules | `node_instances_active: ['none']` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `blueprint_id` | blueprint_id | Yes | (none) | The ID of an already uploaded blueprint to use for creating the new deployments. |
| `parent_deployments` | list (deployment_id) | Yes | (none) | A list of existing deployment IDs that will be parents to the new deployments. One new deployment is created per parent deployment in this list. |
| `group_id` | string | No | `''` | An ID for grouping the new deployments together. Useful for managing the batch as a unit. |
| `new_deployment_ids` | list | No | `[]` | Explicit IDs for the new deployments. If empty, IDs are auto-generated. If provided, must have the same length as `parent_deployments`. |
| `inputs` | list | No | `[]` | An ordered list of input dictionaries, one per new deployment. Each entry provides the inputs for the corresponding deployment (matched by position with `parent_deployments`). |
| `add_parent_labels` | boolean | No | `false` | When `true`, automatically adds a `csys-parent-id` label to each new deployment, linking it to its parent. |
| `labels` | list | No | `[]` | An ordered list of label dictionaries, one per new deployment. Each entry provides the labels for the corresponding deployment. |

---

### batch_deploy

Deploys multiple deployments in one operation without installing them. Creates the deployments but leaves them in the "deployed" state (not yet installed).

| Detail | Value |
|---|---|
| Mapping | `plugins_custom_workflow.plugins_custom_workflow.tasks.batch_deploy` |
| Display Label | Batch Deploy |
| Availability Rules | `node_instances_active: ['none']` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `blueprint_id` | blueprint_id | Yes | (none) | The ID of an already uploaded blueprint. |
| `parent_deployments` | list (deployment_id) | Yes | (none) | A list of parent deployment IDs. One new deployment is created per parent. |
| `group_id` | string | No | `''` | An ID for grouping the new deployments. |
| `new_deployment_ids` | list | No | `[]` | Explicit IDs for the new deployments. |
| `inputs` | list | No | `[]` | An ordered list of input dictionaries for each new deployment. |
| `labels` | list | No | `[]` | An ordered list of label dictionaries for each new deployment. |

---

### batch_install

Installs multiple previously deployed deployments in a single action. Use this after `batch_deploy` to complete provisioning across all deployments in a group.

| Detail | Value |
|---|---|
| Mapping | `plugins_custom_workflow.plugins_custom_workflow.tasks.batch_install` |
| Display Label | Batch Install |
| Availability Rules | `node_instances_active: ['none']` |

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `group_id` | string | Yes | (none) | The group ID of the deployments to install. This must match the `group_id` used when the deployments were created with `batch_deploy`. |
