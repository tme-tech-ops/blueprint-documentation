# Blueprint Examples Overview

This guide introduces the NativeEdge blueprint examples included in the `blueprint_example/infra-dev-apps/` directory. These examples demonstrate real-world patterns for provisioning virtual machines, deploying containers, configuring applications, and managing remote hosts using the NativeEdge plugin ecosystem.

If you are new to NativeEdge blueprints, start here. Each example builds on the same foundational concepts but introduces new plugins and patterns incrementally.

## Prerequisites

Before diving into the examples, make sure you are familiar with:

- [Blueprint Anatomy](../blueprint-structure/blueprint-anatomy.md) -- the structure of a NativeEdge blueprint
- [Edge Plugin: Virtual Machines](../edge-plugin/virtual-machines.md) -- VM provisioning on NativeEdge endpoints
- [Fabric Plugin](../fabric-plugin/overview.md) -- remote command execution over SSH
- [Ansible Plugin](../ansible-plugin/overview.md) -- Ansible playbook execution
- [Utilities Plugin](../utilities-plugin/overview.md) -- SSH key generation, cloud-init, and helper utilities

---

## The Modular File Structure Pattern

Every example in this collection follows the same organizational pattern: a single **entry point YAML file** that imports modular sub-files from a dedicated subdirectory.

```
simple_vm_example_for_NED.yaml      <-- Entry point
vm/
  definitions.yaml                  <-- Node templates (the resources to create)
  inputs.yaml                       <-- Input parameter declarations
  outputs.yaml                      <-- Output/capability declarations
```

The entry point file is intentionally lightweight. It declares:

- `tosca_definitions_version` -- always `dell_1_0` for NativeEdge blueprints
- `description` -- a human-readable summary
- `imports` -- references to the base types library (`dell/types/types.yaml`) plus the sub-files
- `input_groups` -- how inputs are organized in the NativeEdge UI (groups with display labels, collapsibility, and ordering)
- `labels` and `blueprint_labels` -- metadata tags for categorization and filtering

The sub-files are where the real work happens:

| Sub-file | Purpose |
|---|---|
| `definitions.yaml` | Declares the `node_templates` -- the actual infrastructure resources, their types, properties, interfaces (lifecycle operations), and relationships (dependency ordering). Also declares plugin imports. |
| `inputs.yaml` | Declares all `inputs` -- the parameters users provide when deploying the blueprint. Includes types, defaults, constraints, display metadata, and visibility settings (`hidden: true/false`). |
| `outputs.yaml` | Declares `capabilities` -- the values the blueprint exposes after deployment (e.g., IP addresses, URLs, credentials). Other blueprints can consume these via `SharedResource`. |

This separation keeps each concern in its own file and makes it possible to **reuse sub-files across multiple blueprints**. The MQTT + Node-RED example demonstrates this directly: it imports the `vm/` sub-files from the Simple VM example and adds its own `mqtt_nodered/` sub-files on top.

---

## Example Inventory

There are five examples in total. They fall into two categories:

1. **NativeEdge endpoint examples** -- deploy resources onto a NativeEdge-managed edge device (require a NativeEdge endpoint with a service tag)
2. **Remote host examples** -- connect to any SSH-accessible host (no NativeEdge endpoint required)

### The Build-On Pattern

The examples are designed to be learned in order. The **Simple VM** example is the foundation. Other examples either extend it or depend on a deployment created from it:

```
Simple VM (base)
  |
  +-- MQTT + Node-RED (extends: imports vm/ sub-files and adds mqtt_nodered/ sub-files)
  |
  +-- Nginx App (depends on: references a deployed Simple VM instance via SharedResource)

Container Deployment (standalone -- deploys containers directly on endpoint)

Remote Host (standalone -- connects to any SSH host, no NativeEdge VM needed)
```

---

## Example Reference Table

| Example | Entry Point | Plugins Used | What It Demonstrates |
|---|---|---|---|
| [Simple VM](simple-vm.md) | `simple_vm_example_for_NED.yaml` | Edge, Utilities, Fabric | Full VM lifecycle: SSH key generation, OS image upload, cloud-init configuration, VM creation, and post-creation info gathering via SSH |
| [Container Deployment](container-deployment.md) | `container_based_example_for_NED.yaml` | Edge | Deploying multi-container applications (Node-RED, InfluxDB, Grafana) on a NativeEdge endpoint using NativeEdgeCompose with a Docker Compose template |
| [Nginx App](nginx-app.md) | `nginx_app_example_for_NED.yaml` | Edge, Utilities, Fabric | Deploying an application onto an existing VM using SharedResource to reference a parent deployment and Fabric to execute remote shell commands |
| [MQTT + Node-RED](mqtt-nodered.md) | `mqtt_nodered_example_for_NED.yaml` | Edge, Utilities, Fabric, Ansible | End-to-end: VM creation plus application deployment using Ansible playbooks to install Docker and deploy containers, with DSL definitions for reusable configuration |
| [Remote Host](remote-host.md) | `remote_host_example.yaml` | Edge, Utilities, Fabric | Connecting to a remote host outside NativeEdge, with proxy disabled, using Fabric for direct SSH command execution |

---

## Recommended Reading Order

1. **[Simple VM](simple-vm.md)** -- Start here. This is the most detailed example and covers all the foundational concepts: node templates, relationships, intrinsic functions, input patterns, and the VM provisioning lifecycle.

2. **[Container Deployment](container-deployment.md)** -- Learn how NativeEdgeCompose works for deploying containers directly on endpoints without a VM.

3. **[Nginx App](nginx-app.md)** -- Understand the SharedResource pattern for building layered deployments where one blueprint depends on another.

4. **[MQTT + Node-RED](mqtt-nodered.md)** -- See how to compose multiple sub-file sets into a single blueprint and how the Ansible plugin integrates with NativeEdge proxy settings.

5. **[Remote Host](remote-host.md)** -- Learn how blueprints work outside the NativeEdge endpoint model, connecting directly to any SSH host.

---

## Common Patterns Across All Examples

As you read through the examples, you will encounter several recurring patterns:

- **`get_input`** -- retrieves a value the user provided at deployment time
- **`get_secret`** -- retrieves a value from the NativeEdge secret store
- **`get_attribute`** -- retrieves a runtime attribute from another node (resolved during execution)
- **`get_environment_capability`** -- retrieves a capability from the NativeEdge environment (e.g., the endpoint service tag)
- **`get_sys`** -- retrieves system-level values (e.g., the deployment name)
- **`concat`** -- concatenates strings together
- **`merge`** -- merges two maps/dictionaries together
- **`dell.relationships.depends_on`** -- establishes execution order between node templates
- **`proxy_settings` / `connection_proxy_settings`** -- configures SSH connectivity through the NativeEdge proxy (or disables it for direct connections)
- **`labels` and `blueprint_labels`** -- metadata used for filtering, categorization, and cross-blueprint references
