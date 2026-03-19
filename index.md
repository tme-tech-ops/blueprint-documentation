# NativeEdge Plugins Documentation

Welcome to the documentation for the NativeEdge plugin suite. These plugins work together to enable infrastructure automation on Dell NativeEdge devices, allowing you to create virtual machines, deploy containers, run remote commands, execute Ansible playbooks, and manage configuration -- all defined declaratively in YAML blueprints.

## What is NativeEdge?

NativeEdge is Dell's edge computing platform that provides orchestration and lifecycle management for workloads running on edge devices (referred to as ECEs -- Edge Compute Endpoints). Blueprints are YAML files that describe the desired state of your infrastructure and applications, and these plugins provide the actions that make that desired state real.

## Plugins at a Glance

| Plugin | Package Name | Purpose |
|--------|-------------|---------|
| [Edge Plugin](edge-plugin/overview.md) | `edge-plugin` | Create and manage VMs, containers, binary images, software packages, licensing, and device configuration on NativeEdge endpoints |
| [Ansible Plugin](ansible-plugin/overview.md) | `ansible-plugin` | Execute Ansible playbooks against NativeEdge VMs and remote hosts |
| [Fabric Plugin](fabric-plugin/overview.md) | `fabric-plugin` | Run shell commands and scripts on remote machines over SSH using Python Fabric |
| [Utilities Plugin](utilities-plugin/overview.md) | `utilities-plugin` | Cloud-init configuration, SSH key management, secrets, REST API calls, terminal operations, file management, and many workflow utilities |

## How the Plugins Work Together

A typical deployment uses several plugins in concert. For example, to deploy an application on a NativeEdge VM:

1. **Utilities Plugin** generates SSH keys (`dell.nodes.keys.RSAKey`) and prepares cloud-init configuration (`dell.nodes.CloudInit.CloudConfig`)
2. **Edge Plugin** uploads the OS image (`dell.nodes.nativeedge.template.BinaryImage`) and creates the VM (`dell.nodes.nativeedge.template.NativeEdgeVM`)
3. **Fabric Plugin** runs shell commands on the new VM over SSH (`fabric.fabric_plugin.tasks.run_commands`)
4. **Ansible Plugin** executes playbooks to install and configure software (`dell.nodes.ansible.Executor`)

## Documentation Guide

### Start Here

- **[Blueprint Anatomy](blueprint-structure/blueprint-anatomy.md)** -- Understand how blueprints are structured: imports, inputs, outputs, node_templates, node_types, relationships, and more. Read this first if you are new to NativeEdge blueprints.

### Plugin References

- **[Edge Plugin](edge-plugin/overview.md)** -- VM and container management on NativeEdge endpoints
  - [Virtual Machines](edge-plugin/virtual-machines.md)
  - [Containers](edge-plugin/containers.md)
  - [Binary Images](edge-plugin/binary-images.md)
  - [Software Packages](edge-plugin/software-packages.md)
  - [Client Credentials](edge-plugin/client-credentials.md)
  - [Licensing](edge-plugin/licensing.md)
  - [Cluster Endpoints](edge-plugin/cluster-endpoints.md)
  - [Network Switch](edge-plugin/network-switch.md)
- **[Ansible Plugin](ansible-plugin/overview.md)** -- Ansible playbook execution
  - [Executor Node Type](ansible-plugin/executor-node-type.md)
  - [Workflows](ansible-plugin/workflows.md)
- **[Fabric Plugin](fabric-plugin/overview.md)** -- Remote command and script execution
  - [Operations](fabric-plugin/operations.md)
- **[Utilities Plugin](utilities-plugin/overview.md)** -- Cloud-init, keys, secrets, REST, and more
  - [Cloud Init](utilities-plugin/cloud-init.md)
  - [SSH Keys](utilities-plugin/ssh-keys.md)
  - [Secrets](utilities-plugin/secrets.md)
  - [REST Requests](utilities-plugin/rest-requests.md)
  - [Files and FTP](utilities-plugin/files-and-ftp.md)
  - [Terminal](utilities-plugin/terminal.md)
  - [Resources](utilities-plugin/resources.md)
  - [Configuration](utilities-plugin/configuration.md)
  - [Relationships](utilities-plugin/relationships.md)
  - [Workflows](utilities-plugin/workflows.md)

### Blueprint Examples

- **[Examples Overview](examples/overview.md)** -- How the example blueprints are organized
- [Simple VM](examples/simple-vm.md) -- Create a VM on a NativeEdge endpoint
- [Container Deployment](examples/container-deployment.md) -- Deploy containers using NativeEdge Compose
- [Nginx App](examples/nginx-app.md) -- Deploy Nginx on an existing VM
- [MQTT + Node-RED](examples/mqtt-nodered.md) -- Create a VM and deploy MQTT/Node-RED with Ansible
- [Remote Host](examples/remote-host.md) -- Run commands on a remote host via SSH
