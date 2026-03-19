# Cluster Endpoints

## Prerequisites

Before working with cluster endpoints you should be familiar with the general plugin concepts described in the [Edge Plugin Overview](overview.md).

## Overview

A **cluster endpoint** in NativeEdge represents a group of edge endpoints that work together as a logical cluster (for example, a Kubernetes cluster or a high-availability pair). The `OutcomeClusterEndpoint` node type allows you to register (add) a new cluster endpoint and update (patch) its configuration, including adding member endpoints and setting cluster metadata.

---

## Node Type

**Full type name:** `dell.nodes.nativeedge.template.OutcomeClusterEndpoint`
**Derived from:** `dell.nodes.Root`

### Properties

The properties for this node type combine two DSL definitions: `add_outcome_cluster_endpoint` (used during creation) and `patch_outcome_cluster_endpoint` (used during patching). All properties are optional so you can supply only the ones relevant to the operation you are performing.

#### Add Properties (used by the `create` operation)

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | string | No | -- | Name of the cluster outcome endpoint. |
| `description` | string | No | -- | Human-readable description of the cluster. |
| `type` | string | No | -- | Type of the cluster endpoint. |
| `cluster_kind` | string | No | -- | Kind of cluster (e.g., `kubernetes`, `swarm`). |
| `hostname` | string | No | -- | Hostname of the cluster endpoint. Can be an FQDN or IP address. |
| `ip` | string | No | -- | IP address of the cluster endpoint. |
| `service_tag` | string | No | -- | Service tag of the primary endpoint in the cluster. |
| `oxy_id` | string | No | -- | Proxy (Oxy) ID associated with the cluster endpoint. |
| `secret_key` | string | No | -- | Name of a NativeEdge secret containing credentials for the cluster. |
| `port` | string | No | -- | Port number for the cluster endpoint API. |

#### Patch Properties (used by the `patch` operation)

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `service_tag` | string | No | -- | Service tag of the cluster endpoint to patch. |
| `cluster_manager` | string | No | -- | URL of the cluster manager. |
| `cluster_version` | string | No | -- | Version string of the cluster. |
| `endpoints` | list of `dell.types.nativeedge.UpdateOutcomeClusterEndpoints` | No | `[]` | List of member endpoints to add to the cluster. |
| `status` | string | No | -- | Status of the cluster (e.g., `active`, `degraded`). |

### Lifecycle Operations

All operations live under the `dell.interfaces.lifecycle` interface.

| Operation | Implementation | Description |
|---|---|---|
| `create` | `edge.nativeedge_plugin.tasks.add_outcome_cluster_endpoint` | Registers a new outcome cluster endpoint with the NativeEdge system using the "add" properties. |
| `patch` | `edge.nativeedge_plugin.tasks.patch_outcome_cluster_endpoint` | Updates an existing cluster endpoint using the "patch" properties. Typically called after the cluster is initially created to add member endpoints or update the cluster version. |

---

## Data Types

### UpdateOutcomeClusterEndpoints

**Full type name:** `dell.types.nativeedge.UpdateOutcomeClusterEndpoints`

Each entry in the `endpoints` list during a patch operation describes a member endpoint to add to the cluster.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `service_tag` | string | Yes | -- | Service tag of the member endpoint. |
| `connection` | string | Yes | -- | Connection status of the endpoint (e.g., `connected`, `disconnected`). |
| `management_ip` | string | Yes | -- | Management IP address of the endpoint. |

---

## Blueprint Example

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin

node_templates:

  cluster_endpoint:
    type: dell.nodes.nativeedge.template.OutcomeClusterEndpoint
    properties:
      name: "my-edge-cluster"
      description: "Production edge cluster"
      type: "kubernetes"
      cluster_kind: "kubernetes"
      hostname: "cluster.example.com"
      ip: "10.0.0.100"
      service_tag: { get_input: primary_service_tag }
      port: "6443"
      secret_key: "k8s_cluster_creds"
      cluster_manager: "https://cluster.example.com:6443"
      cluster_version: "1.28.0"
      endpoints:
        - service_tag: { get_input: node1_service_tag }
          connection: "connected"
          management_ip: "10.0.0.101"
        - service_tag: { get_input: node2_service_tag }
          connection: "connected"
          management_ip: "10.0.0.102"
      status: "active"
```

### Explanation

1. **Create phase** -- When the deployment is installed, the `create` operation registers the cluster endpoint using the "add" properties: `name`, `description`, `type`, `cluster_kind`, `hostname`, `ip`, `service_tag`, `port`, and `secret_key`.
2. **Patch phase** -- The `patch` operation can then be called (via a workflow or update) to set the `cluster_manager`, `cluster_version`, `endpoints` list, and `status`.
3. The `endpoints` list adds two member endpoints to the cluster, each identified by their service tag, connection status, and management IP.
