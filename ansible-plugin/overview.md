# Ansible Plugin Overview

This document introduces the NativeEdge Ansible Plugin, which enables you to run [Ansible](https://www.ansible.com/) playbooks against NativeEdge-managed virtual machines and remote hosts directly from your NativeEdge blueprints.

## Prerequisites

Before using this plugin, you should be familiar with:

- **NativeEdge blueprints** -- YAML files that describe the desired state of your infrastructure and applications.
- **TOSCA DSL** -- the specification language NativeEdge blueprints are built on (`tosca_definitions_version: dell_1_0`).
- **Ansible basics** -- playbooks, inventories, roles, and collections.

Related documentation:

- [Ansible Executor Node Type](./executor-node-type.md) -- full reference for the `dell.nodes.ansible.Executor` node type.
- [Ansible Plugin Workflows](./workflows.md) -- reference for the two built-in workflows.

---

## What Is the Ansible Plugin?

The Ansible Plugin is a NativeEdge plugin that lets you:

1. Declare Ansible playbook executions as **node templates** inside a blueprint.
2. Configure inventory, variables, environment settings, and proxy/SSH options declaratively.
3. Execute playbooks during the NativeEdge **install** workflow (lifecycle) and re-execute them on demand with the **reload** workflow.
4. Manage the Ansible Python virtual environment (install extra pip packages, Galaxy collections, and roles) without leaving NativeEdge.

When NativeEdge processes a blueprint that uses this plugin, the **central deployment agent** on the NativeEdge orchestrator runs the `ansible-playbook` command on your behalf, targeting the hosts you specify in the inventory.

---

## Plugin Identity

| Field | Value |
|---|---|
| Package name | `ansible-plugin` |
| Package version | `4.1.7.0` |
| Executor | `central_deployment_agent` |

**Executor explained:** The `central_deployment_agent` executor means that every Ansible operation runs on the NativeEdge orchestrator itself (the "central" manager machine), not on the remote target host. The orchestrator reaches the target hosts over SSH (or WinRM) as configured in the Ansible inventory.

---

## Importing the Plugin

To use the Ansible Plugin in a blueprint, add an entry to the `imports` section at the top of your blueprint YAML file. For example:

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:ansible-plugin?version=>=4.1.2.0
```

The `?version=>=4.1.2.0` suffix is a version constraint. It tells NativeEdge to use any installed version of the plugin that is **4.1.2.0 or newer**. You can pin an exact version (`?version=4.1.7.0`) or omit the constraint entirely to use whichever version is installed.

---

## Node Type

The plugin provides a single node type:

### `dell.nodes.ansible.Executor`

This is the core building block. Each node template of this type represents **one Ansible playbook execution**. You configure it with a playbook path, an inventory (called `sources`), optional variables (`run_data`), environment variables, SSH/proxy settings, and more.

During the NativeEdge **install** workflow the node goes through these lifecycle stages:

| Stage | What happens |
|---|---|
| `precreate` | Prepares the Python virtual environment and downloads any required packages, collections, or roles. |
| `start` | Runs the Ansible playbook against the configured inventory. |
| `poststart` | Optionally stores Ansible facts as NativeEdge runtime properties. |
| `delete` | Cleans up temporary files and the virtual environment. |

The node type also exposes a custom `ansible.reload` interface operation that re-runs the playbook (used by the `reload_ansible_playbook` workflow).

See the full reference: [Ansible Executor Node Type](./executor-node-type.md).

---

## Workflows

The plugin registers two custom workflows on every deployment that uses it:

| Workflow | Display Label | Purpose |
|---|---|---|
| `reload_ansible_playbook` | Reload Ansible Plugin | Re-runs the Ansible playbook on one or more node instances, optionally overriding playbook configuration parameters. |
| `update_playbook_venv` | Update Playbook Virtual Environment | Installs additional Python packages, Galaxy collections, or roles into the Ansible virtual environment without re-running the playbook. |

See the full reference: [Ansible Plugin Workflows](./workflows.md).

---

## Understanding DSL Definitions (YAML Anchors)

A key pattern you will encounter in the plugin and in blueprints that use it is **DSL definitions with YAML anchors**. This is a standard YAML feature that NativeEdge leverages heavily for reuse.

### How It Works

In YAML you can define an **anchor** (a reusable block of content) and then **reference** it elsewhere with an **alias**:

```yaml
dsl_definitions:

  # Define an anchor named "my_defaults"
  my_defaults: &my_defaults
    timeout: 30
    debug_level: 2

node_templates:
  my_node:
    type: some.type
    properties:
      # Merge the anchor into this mapping
      <<: *my_defaults
      # You can still add or override properties
      timeout: 60
```

The `&my_defaults` creates the anchor; `*my_defaults` references it. The `<<:` syntax is a YAML merge key that copies all key-value pairs from the anchor into the current mapping.

### How the Ansible Plugin Uses This

The plugin's `plugin.yaml` defines two important anchors:

1. **`&playbook_config`** -- contains all of the playbook-related property definitions (paths, environment variables, SSH args, retry settings, and so on). This anchor is merged into:
   - The `dell.nodes.ansible.Executor` node type properties.
   - The `reload_ansible_playbook` workflow parameters.

2. **`&playbook_inputs`** -- maps each property to a `get_property: [SELF, ...]` intrinsic function call so that the lifecycle operation inputs automatically read their values from the node's properties.

Because both the node type and the workflow share the same `&playbook_config` anchor, you can use the exact same parameter names whether you are configuring a node template in a blueprint or invoking the reload workflow at runtime.

### DSL Definitions in Your Blueprints

You will frequently define your own anchors in the `dsl_definitions` section of your blueprint to avoid repeating common values. For example:

```yaml
dsl_definitions:

  ansible_env_vars: &ansible_default_env_vars
    ANSIBLE_HOST_KEY_CHECKING: "False"
    ANSIBLE_INVALID_TASK_ATTRIBUTE_FAILED: "False"
    ANSIBLE_STDOUT_CALLBACK: json
    ANSIBLE_DEPRECATION_WARNINGS: "False"

  ansible_common_properties: &ansible_common_properties
    ansible_external_venv: /opt/ansible
    store_facts: false
    timeout: "30"
    debug_level: 2
    number_of_attempts: 10

node_templates:
  install_step:
    type: dell.nodes.ansible.Executor
    properties:
      <<: *ansible_common_properties
      playbook_path: playbooks/install.yaml
      ansible_env_vars:
        <<: *ansible_default_env_vars
```

This keeps your blueprint DRY (Don't Repeat Yourself) and makes it easy to change a shared value in one place.

---

## Labels

Deployments created from blueprints that import this plugin are automatically tagged with the label:

```yaml
labels:
  obj-type:
    values:
      - ansible
```

This label can be used to filter and search for Ansible-based deployments in the NativeEdge UI.

---

## Next Steps

- [Ansible Executor Node Type](./executor-node-type.md) -- learn how to configure playbook execution in detail.
- [Ansible Plugin Workflows](./workflows.md) -- learn how to reload playbooks and update virtual environments at runtime.
