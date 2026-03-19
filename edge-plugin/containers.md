# Containers (Compose)

## Prerequisites

Before deploying containers you should be familiar with the general plugin concepts described in the [Edge Plugin Overview](overview.md). You will also need:

- A NativeEdge environment with at least one registered endpoint (ECE).
- The endpoint's **service tag**.
- A valid Docker Compose YAML file that defines your container workload.
- (Optional) Credentials for a private container registry if your images are not publicly available.

## Overview

The edge plugin can deploy Docker Compose workloads directly onto NativeEdge endpoints. This is useful for lightweight, containerized applications that do not need a full virtual machine. The plugin sends your Compose YAML to the endpoint, which runs the containers using its built-in container runtime.

Two node types are involved:

| Node Type | Description |
|---|---|
| `dell.nodes.nativeedge.template.NativeEdgeCompose` | Deploys and manages a Docker Compose application on an endpoint. |
| `dell.nodes.nativeedge.template.RegistryLogin` | Authenticates the endpoint to a private container image registry before pulling images. |

---

## NativeEdgeCompose

**Full type name:** `dell.nodes.nativeedge.template.NativeEdgeCompose`
**Derived from:** `dell.nodes.Root`

### Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `compose_config` | `dell.types.nativeedge.NativeEdgeComposeConfig` | Yes | Configuration block containing the target endpoint and the Compose definition. |

### Lifecycle Operations

All operations live under the `dell.interfaces.lifecycle` interface.

| Operation | Implementation | Description |
|---|---|---|
| `precreate` | `edge.nativeedge_plugin.tasks.precreate_compose` | Validates the Compose configuration (checks that required fields are present and the YAML is well-formed). |
| `create` | `edge.nativeedge_plugin.tasks.create_compose` | Sends the Compose definition to the endpoint and starts the containers. |
| `delete` | `edge.nativeedge_plugin.tasks.delete_compose` | Stops and removes the Compose deployment and its associated containers/networks/volumes on the endpoint. |
| `start` | `edge.nativeedge_plugin.tasks.start_compose` | Starts a previously stopped Compose deployment. |
| `stop` | `edge.nativeedge_plugin.tasks.stop_compose` | Stops all containers in the Compose deployment without removing them. |
| `restart` | `edge.nativeedge_plugin.tasks.restart_compose` | Restarts all containers in the Compose deployment. |
| `update` | `edge.nativeedge_plugin.tasks.update_compose` | Applies an updated Compose definition (for example, a new image tag or changed environment variable). |
| `check_drift` | `edge.nativeedge_plugin.tasks.compose_check_drift` | Compares the running Compose state on the endpoint with the expected state in the blueprint. Returns drift information if they differ. |
| `poststart` | `edge.nativeedge_plugin.tasks.poststart_compose` | Runs after the Compose deployment has started; collects runtime attributes such as container IDs and port mappings. |

---

## Data Types

### NativeEdgeComposeConfig

**Full type name:** `dell.types.nativeedge.NativeEdgeComposeConfig`

This is the top-level configuration object passed as `compose_config`.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `service_tag` | string | Yes | -- | ECE service tag identifying which endpoint will run the containers. |
| `definition` | `dell.types.nativeedge.NativeEdgeComposeDefinition` | Yes | -- | The Compose deployment definition containing the name, YAML payload, and optional environment variables. |

### NativeEdgeComposeDefinition

**Full type name:** `dell.types.nativeedge.NativeEdgeComposeDefinition`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | string | No | -- | A human-readable name for the Compose deployment. If omitted, the system may generate one. Typically set to the deployment name via `{ get_sys: [deployment, name] }`. |
| `compose` | string | Yes | -- | The full Docker Compose YAML content as a string. This is the equivalent of a `docker-compose.yaml` file. You can load it from a blueprint resource file using a helper node (see the example below). |
| `environment_variables` | dict | No | -- | Key-value pairs that are substituted into the Compose YAML. Any `${VARIABLE}` or `$VARIABLE` placeholder in the Compose file will be replaced with the corresponding value from this dictionary. |

---

## RegistryLogin

**Full type name:** `dell.nodes.nativeedge.template.RegistryLogin`
**Derived from:** `dell.nodes.Root`

Use this node type when your Docker Compose workload needs to pull images from a private container registry. The node authenticates the endpoint with the registry so that subsequent `docker pull` commands succeed.

### Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `registry_config` | `dell.types.nativeedge.RegistryConfig` | Yes | Registry credentials configuration. |
| `service_tag` | string | Yes | ECE service tag identifying which endpoint to authenticate. |

### Lifecycle Operations

| Operation | Implementation | Description |
|---|---|---|
| `precreate` | `edge.nativeedge_plugin.tasks.validate_registry_config` | Validates that the registry configuration is well-formed. |
| `create` | `edge.nativeedge_plugin.tasks.registry_login` | Sends the registry credentials to the endpoint and performs `docker login`. |

### RegistryConfig

**Full type name:** `dell.types.nativeedge.RegistryConfig`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `credentials` | list | No | `[]` | A list of registry credential objects. Each object in the list should contain the keys: `registry_url` (string), `username` (string), `password` (string), and optionally `tls_verify` (boolean, defaults to `true`). You can authenticate to multiple registries by adding multiple entries. |

**Credential object format:**

```yaml
credentials:
  - registry_url: "https://registry.example.com"
    username: "my-user"
    password: "my-password"
    tls_verify: true
```

---

## Blueprint Example -- Compose Deployment

The following example shows how to deploy a Docker Compose application with environment variable substitution. It uses a helper node to load the Compose YAML from a file bundled with the blueprint.

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin

node_templates:

  prepare_compose_yaml:
    type: dell.nodes.Root
    interfaces:
      dell.interfaces.lifecycle:
        start: |
          import sys
          from nativeedge import ctx

          filename = ctx.download_resource(
              "container_based/compose/container_compose.yaml")
          if filename is None or filename == "":
            ctx.logger.info("Compose file not found")
            sys.exit()

          with open(filename) as f:
            content = f.read()

          ctx.instance.runtime_properties["compose_yaml"] = content
          ctx.instance.update()

  nativeedge_compose:
    type: dell.nodes.nativeedge.template.NativeEdgeCompose
    properties:
      compose_config:
        service_tag: { get_environment_capability: ece_service_tag }
        definition:
          name: { get_sys: [deployment, name] }
          compose: { get_attribute: [prepare_compose_yaml, compose_yaml] }
          environment_variables:
            NODERED_NAME: { get_input: nodered_name }
            NODERED_URL: { get_input: nodered_url }
            INFLUXDB_NAME: { get_input: influxdb_name }
            INFLUXDB_URL: { get_input: influxdb_url }
            GRAFANA_NAME: { get_input: grafana_name }
            GRAFANA_URL: { get_input: grafana_url }
    relationships:
      - type: dell.relationships.depends_on
        target: prepare_compose_yaml
```

### Explanation

1. **prepare_compose_yaml** -- A helper node that reads the `container_compose.yaml` file from the blueprint archive and stores its content as a runtime property called `compose_yaml`.
2. **nativeedge_compose** -- The Compose deployment node:
   - `service_tag` targets the endpoint from the environment profile.
   - `name` is set to the NativeEdge deployment name.
   - `compose` reads the YAML content from the helper node via `get_attribute`.
   - `environment_variables` injects deployment-time inputs into the Compose YAML (for example, container names and URLs).
3. The **relationship** ensures the Compose YAML is loaded before the Compose node is created.

---

## Blueprint Example -- With Registry Login

If your Compose application pulls from a private registry, add a `RegistryLogin` node and make the Compose node depend on it:

```yaml
node_templates:

  registry_login:
    type: dell.nodes.nativeedge.template.RegistryLogin
    properties:
      registry_config:
        credentials:
          - registry_url: { get_input: registry_url }
            username: { get_input: registry_username }
            password: { get_secret: { get_input: registry_password } }
      service_tag: { get_environment_capability: ece_service_tag }

  nativeedge_compose:
    type: dell.nodes.nativeedge.template.NativeEdgeCompose
    properties:
      compose_config:
        service_tag: { get_environment_capability: ece_service_tag }
        definition:
          name: { get_sys: [deployment, name] }
          compose: { get_attribute: [prepare_compose_yaml, compose_yaml] }
    relationships:
      - type: dell.relationships.depends_on
        target: prepare_compose_yaml
      - type: dell.relationships.depends_on
        target: registry_login
```

The `registry_login` node runs first, authenticating the endpoint to the private registry. Then when `nativeedge_compose` creates the deployment, the endpoint can successfully pull private images.
