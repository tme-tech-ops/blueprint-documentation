# Ansible Plugin Workflows

This document describes the two custom workflows provided by the NativeEdge Ansible Plugin. Workflows are operations you can trigger on a running deployment **after** it has been installed, without having to uninstall and reinstall.

## Prerequisites

- A deployment must already exist and be in an **active** state (fully installed or partially installed, depending on the workflow).
- The deployment must use the Ansible Plugin and contain at least one `dell.nodes.ansible.Executor` node.
- See [Ansible Plugin Overview](./overview.md) for plugin import instructions.
- See [Ansible Executor Node Type](./executor-node-type.md) for node type configuration.

---

## reload_ansible_playbook

### Purpose

The `reload_ansible_playbook` workflow re-runs an Ansible playbook on one or more node instances within a deployment. This is the primary way to **re-execute** a playbook after the initial install, for example:

- After changing configuration values and wanting to apply them.
- To retry a playbook that partially failed during install.
- To run the playbook with different parameters (tags, run_data, etc.) without modifying the blueprint.

### Definition

| Field | Value |
|---|---|
| Workflow name | `reload_ansible_playbook` |
| Display label | Reload Ansible Plugin |
| Implementation | `ansible.plugins_ansible.workflows.reload_playbook` |

### Availability Rules

This workflow is only available when:

- **Node instances are active:** At least some node instances in the deployment are in an active state (`all` or `partial`).
- **Required operation exists:** The target node instances must have the `ansible.reload` operation defined.

These rules are enforced by the NativeEdge orchestrator. If no node instances have the `ansible.reload` operation, the workflow will not appear in the UI.

### Parameters

The workflow accepts **all playbook configuration properties** (the same ones you set on the node template) plus two additional filtering parameters. Any parameter you provide here **overrides** the value stored in the node's properties for that execution only.

#### Playbook Configuration Parameters

These are identical to the playbook configuration properties documented in the [Executor Node Type](./executor-node-type.md#playbook-configuration-properties). They are listed here for quick reference:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `ansible_external_venv` | string | `""` | Path to an existing Ansible Python virtual environment. |
| `ansible_playbook_executable_path` | string | `""` | Full path to a custom `ansible-playbook` executable. |
| `extra_packages` | list | `[]` | Python packages to install before running. |
| `galaxy_collections` | list | `[]` | Ansible Galaxy collections to install before running. |
| `roles` | list | `[]` | Ansible roles to install before running. |
| `module_path` | string | `""` | Path where Ansible modules are installed. |
| `playbook_source_path` | string | `""` | Path or URL to the playbook source directory/archive. |
| `playbook_path` | string | `""` | Path to the playbook entry file. |
| `additional_playbook_files` | list | `[]` | Additional blueprint resources to download. |
| `sources` | dict | *(none)* | Ansible inventory (YAML dict or file path). |
| `run_data` | dict | `{}` | Variables passed to the playbook. |
| `sensitive_keys` | list | `["ansible_password"]` | Keys to obscure in logs. |
| `options_config` | dict | `{}` | Command-line options for `ansible-playbook`. |
| `ansible_env_vars` | dict | *(see plugin defaults)* | Environment variables for the playbook run. |
| `debug_level` | integer | `2` | Ansible verbosity level. |
| `log_stdout` | boolean | `true` | Whether to log playbook stdout. |
| `additional_args` | string | `""` | Extra CLI arguments for `ansible-playbook`. |
| `start_at_task` | string | `""` | Start playbook at a named task. |
| `scp_extra_args` | string | `""` | Extra arguments for `scp`. |
| `sftp_extra_args` | string | `""` | Extra arguments for `sftp`. |
| `ssh_common_args` | string | `""` | Common arguments for `ssh`/`scp`/`sftp`. |
| `ssh_extra_args` | string | `""` | Extra arguments for `ssh`. |
| `timeout` | string | `"10"` | SSH connection timeout in seconds. |
| `remerge_sources` | boolean | `false` | Re-evaluate sources before running. |
| `ansible_become` | boolean | `false` | Run with elevated privileges. |
| `tags` | list | `[]` | Ansible tags to execute. |
| `auto_tags` | boolean | `false` | Auto-generate tags from the playbook. |
| `number_of_attempts` | integer | `3` | Number of retry attempts on failure. |
| `store_facts` | boolean | `true` | Store Ansible facts as runtime properties. |
| `facts_mapping` | dict | `{}` | Mapping of run_data keys to facts keys for drift detection. |

#### Filtering Parameters

These parameters control **which** node instances the workflow targets:

| Parameter | Type | Default | Description |
|---|---|---|---|
| `node_instance_ids` | list (of node_instance) | `[]` | A list of specific node instance IDs to target. An empty list means no filtering -- all eligible node instances are included. |
| `node_ids` | list (of node_id) | `[]` | A list of node IDs (node template names). The workflow runs on all instances of these nodes. An empty list means no filtering. Only nodes that have the `ansible.reload` operation are valid targets. |

**How filtering works:**

- If both `node_instance_ids` and `node_ids` are empty, the workflow runs on **all** node instances that have the `ansible.reload` operation.
- If `node_instance_ids` is provided, only those specific instances are targeted.
- If `node_ids` is provided, all instances of those node templates are targeted.
- If both are provided, the intersection is used.

### When to Use This Workflow

| Scenario | Example |
|---|---|
| Apply configuration changes | You changed a variable in `run_data` and want to re-apply the playbook. |
| Retry after failure | The initial install succeeded partially, and you want to re-run the playbook on failed nodes. |
| Run specific tags | You want to execute only a subset of the playbook by providing `tags`. |
| Update a single node | You have multiple Ansible nodes but only want to re-run one of them. |

### Example: Invoke via Blueprint Workflow Section

You do not need to define the workflow in your blueprint -- it is automatically available from the plugin. However, here is how you would invoke it using the NativeEdge CLI or API:

```yaml
# Example: CLI invocation (conceptual)
# nativeedge executions start \
#   -d my_deployment \
#   -w reload_ansible_playbook \
#   -p parameters.yaml

# parameters.yaml
node_ids:
  - deploy_mqtt_nodered
run_data:
  use_docker_authentication: true
  registry_url: "https://registry.example.com"
  registry_username: "deploy_user"
  registry_password: "s3cret"
number_of_attempts: 5
```

This would re-run the playbook only on the `deploy_mqtt_nodered` node, passing updated `run_data` variables and allowing up to 5 retries.

### Example: Override Tags

```yaml
# parameters.yaml
node_ids:
  - install_docker
tags:
  - install
  - configure
debug_level: 3
```

This re-runs the `install_docker` node's playbook, executing only tasks tagged with `install` or `configure`, at verbosity level 3 (`-vvv`).

---

## update_playbook_venv

### Purpose

The `update_playbook_venv` workflow installs additional Python packages, Ansible Galaxy collections, or Ansible roles into the virtual environment used by the Ansible nodes -- **without re-running any playbooks**.

This is useful when:

- A playbook requires a new Python library that was not included in the original deployment.
- You need to add an Ansible Galaxy collection to support a new module or role.
- You want to install or update Ansible roles.

### Definition

| Field | Value |
|---|---|
| Workflow name | `update_playbook_venv` |
| Display label | Update Playbook Virtual Environment |
| Implementation | `ansible.plugins_ansible.workflows.update_playbook_venv` |

### Availability Rules

This workflow has no explicit availability rules defined in the plugin, so it is available whenever the deployment is active.

### Parameters

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `extra_packages` | list | No | `[]` | A list of Python package names to install via `pip` into the Ansible virtual environment. Each entry should be a string in pip format (for example `"requests>=2.28.0"` or `"pywinrm"`). |
| `galaxy_collections` | list | No | `[]` | A list of Ansible Galaxy collection names to install (for example `"community.general"` or `"ansible.posix:>=1.5.0"`). |
| `roles` | list | No | `[]` | A list of Ansible role names to install from Galaxy or other sources. |

### When to Use This Workflow

| Scenario | Example |
|---|---|
| Add a Python dependency | Your playbook now uses a module that requires `pymongo` -- install it without redeploying. |
| Add a Galaxy collection | You need `community.docker` for Docker-related Ansible modules. |
| Update a role | A newer version of a role fixes a bug -- install it into the existing venv. |
| Prepare for reload | Install dependencies first with `update_playbook_venv`, then run `reload_ansible_playbook` with a new playbook that uses them. |

### Example: Install Python Packages

```yaml
# parameters.yaml
extra_packages:
  - "pymongo>=4.0"
  - "boto3"
  - "requests>=2.28.0"
```

### Example: Install Galaxy Collections and Roles

```yaml
# parameters.yaml
galaxy_collections:
  - "community.docker"
  - "community.general>=6.0.0"
roles:
  - "geerlingguy.docker"
  - "geerlingguy.nginx"
```

### Example: Combined Update

```yaml
# parameters.yaml
extra_packages:
  - "pywinrm"
galaxy_collections:
  - "ansible.windows"
roles:
  - "ansible.windows.win_updates"
```

---

## Workflow Comparison

| Feature | `reload_ansible_playbook` | `update_playbook_venv` |
|---|---|---|
| Runs the playbook | Yes | No |
| Installs packages/collections/roles | Yes (via `extra_packages`, `galaxy_collections`, `roles` parameters) | Yes |
| Can target specific nodes | Yes (via `node_ids`, `node_instance_ids`) | No (applies to all nodes) |
| Can override playbook parameters | Yes (all playbook_config parameters) | No |
| Availability requirement | Node instances must be active; `ansible.reload` operation required | Deployment must be active |
| Typical use case | Re-apply or update configuration | Prepare the environment for future runs |

---

## Typical Workflow Sequence

A common pattern when updating a running deployment:

1. **Update the virtual environment** (if new dependencies are needed):

   ```yaml
   # Step 1: Install new dependencies
   # Workflow: update_playbook_venv
   extra_packages:
     - "new-python-library"
   galaxy_collections:
     - "community.new_collection"
   ```

2. **Reload the playbook** (with updated parameters):

   ```yaml
   # Step 2: Re-run the playbook with new configuration
   # Workflow: reload_ansible_playbook
   node_ids:
     - my_ansible_node
   run_data:
     new_setting: "updated_value"
   ```

This two-step approach ensures that dependencies are in place before the playbook runs, avoiding failures due to missing modules or libraries.

---

## See Also

- [Ansible Plugin Overview](./overview.md) -- introduction to the plugin and DSL definitions.
- [Ansible Executor Node Type](./executor-node-type.md) -- full reference for node properties and lifecycle.
