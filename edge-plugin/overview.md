# Edge Plugin Overview

## What is the Edge Plugin?

The NativeEdge Edge Plugin is a server-side plugin that allows you to manage workloads and infrastructure on Dell NativeEdge endpoints (also called ECE -- Edge Computing Endpoints) through NativeEdge blueprints. An "endpoint" is a physical Dell edge device running the NativeEdge software stack. The plugin communicates with endpoint APIs from the NativeEdge orchestrator to provision virtual machines, deploy containers, upload binary images, manage software packages, handle licensing, register cluster endpoints, and configure network switches.

When you include this plugin in a blueprint, the orchestrator's **central deployment agent** executes every operation on your behalf -- you do not need to install any agent software on the endpoint itself.

## Plugin Identity

| Field | Value |
|---|---|
| Package name | `edge-plugin` |
| Package version | `3.3.22.0` |
| Executor | `central_deployment_agent` |
| Internal plugin alias | `edge` |

## How to Import the Plugin

Add the following line to the `imports` section of any blueprint that needs edge functionality:

```yaml
tosca_definitions_version: dell_1_0

imports:
  - plugin:edge-plugin
```

You can pin a minimum version if your blueprint depends on features introduced in a specific release:

```yaml
imports:
  - plugin:edge-plugin?version=>=3.3.22.0
```

## Node Types at a Glance

The plugin registers the following node types. Each type maps to a different kind of resource you can manage on a NativeEdge endpoint.

| Node Type | Purpose | Details |
|---|---|---|
| `dell.nodes.nativeedge.template.NativeEdgeVM` | Provision and manage a virtual machine on an endpoint | [Virtual Machines](virtual-machines.md) |
| `dell.nodes.nativeedge.template.ApplicationVM` | Deploy a VM from a pre-built VM template | [Virtual Machines](virtual-machines.md) |
| `dell.nodes.nativeedge.template.NativeEdgeCompose` | Deploy and manage Docker Compose workloads | [Containers (Compose)](containers.md) |
| `dell.nodes.nativeedge.template.RegistryLogin` | Authenticate to a private container registry before pulling images | [Containers (Compose)](containers.md) |
| `dell.nodes.nativeedge.template.BinaryImage` | Upload and manage binary images (VM images, ISOs) in the artifact catalog | [Binary Images](binary-images.md) |
| `dell.nodes.nativeedge.template.SoftwarePackages` | Download software packages to the orchestrator repository | [Software Packages](software-packages.md) |
| `dell.nodes.nativeedge.template.OxyServerSoftwarePackages` | Download software packages for an Oxy Server proxy | [Software Packages](software-packages.md) |
| `dell.nodes.nativeedge.template.ClientCredentials` | Generate client credentials and retrieve TLS certificates from the orchestrator | [Client Credentials](client-credentials.md) |
| `dell.nodes.nativeedge.template.License` | Apply a static or dynamic license to the NativeEdge system | [Licensing](licensing.md) |
| `dell.nodes.nativeedge.template.RegisterPrivateCloudLicense` | Register, expand, reduce, or deregister a private cloud license | [Licensing](licensing.md) |
| `dell.nodes.nativeedge.template.OutcomeClusterEndpoint` | Add or patch an outcome cluster endpoint | [Cluster Endpoints](cluster-endpoints.md) |
| `dell.nodes.nativeedge.template.NetworkSwitch` | Create or update (upsert) a network switch record | [Network Switch](network-switch.md) |

## Key Concepts for New Users

### Blueprints

A blueprint is a YAML file (or set of YAML files) that declaratively describes the desired state of your infrastructure and application workloads. The NativeEdge orchestrator reads a blueprint, resolves inputs and relationships, and executes the lifecycle operations defined by each node type.

### Node Types and Node Templates

A **node type** is a reusable class defined by a plugin (like the ones listed above). A **node template** is a concrete instance of a node type that you declare inside the `node_templates` section of your blueprint. For example:

```yaml
node_templates:
  my_vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: "ABCD123"
        name: "my-edge-vm"
        # ... additional properties
```

### Lifecycle Operations

Each node type defines a set of lifecycle operations (such as `create`, `start`, `stop`, `delete`). The orchestrator calls these operations in order when you install, update, or uninstall a deployment. The edge plugin maps each operation to a Python task that communicates with the NativeEdge endpoint API.

### Relationships

Nodes can depend on other nodes through relationships. For example, a VM node might depend on a `BinaryImage` node so that the image is uploaded before the VM is created. Relationships are declared with `dell.relationships.depends_on`.

### Inputs, Secrets, and Intrinsic Functions

Blueprints use intrinsic functions to make configurations dynamic:

- `{ get_input: my_input }` -- reads a value provided at deployment time.
- `{ get_secret: my_secret }` -- reads a value stored in the NativeEdge secret store.
- `{ get_attribute: [node_name, attribute] }` -- reads a runtime attribute set by another node.
- `{ get_environment_capability: capability_name }` -- reads a capability from the environment profile (commonly used for `ece_service_tag`).

### Service Tag

Every NativeEdge endpoint is identified by a unique **service tag** string. Many node types require you to specify a `location` or `service_tag` property that tells the plugin which endpoint to target.

## Further Reading

- [Virtual Machines](virtual-machines.md) -- full-featured VM provisioning
- [Containers (Compose)](containers.md) -- Docker Compose deployments
- [Binary Images](binary-images.md) -- artifact catalog management
- [Software Packages](software-packages.md) -- downloading packages to the orchestrator
- [Client Credentials](client-credentials.md) -- orchestrator credentials and certificates
- [Licensing](licensing.md) -- static and dynamic license management
- [Cluster Endpoints](cluster-endpoints.md) -- outcome cluster endpoint registration
- [Network Switch](network-switch.md) -- network switch inventory
