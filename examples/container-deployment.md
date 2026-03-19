# Container Deployment Example

This example demonstrates how to deploy a multi-container application stack directly on a NativeEdge endpoint, without creating a virtual machine first. It uses **NativeEdgeCompose** -- the NativeEdge equivalent of Docker Compose -- to deploy Node-RED, InfluxDB, and Grafana containers as a single unit.

## Prerequisites

- [Blueprint Anatomy](../blueprint-structure/blueprint-anatomy.md) -- understand blueprint structure
- [Edge Plugin: Containers](../edge-plugin/containers.md) -- the `NativeEdgeCompose` node type
- [Blueprint Examples Overview](overview.md) -- the modular file structure pattern

---

## File Structure

```
container_based_example_for_NED.yaml    <-- Entry point
container_based/
  definitions.yaml                      <-- Node templates (2 nodes)
  inputs.yaml                           <-- Input declarations (6 inputs)
  outputs.yaml                          <-- Output capabilities (3 URLs)
  compose/
    container_compose.yaml              <-- Docker Compose template
```

This example introduces a new directory element: `compose/`, which contains the Docker Compose template used by NativeEdgeCompose. The template is not a static file -- it uses environment variable substitution to inject deployment-specific values.

---

## The Entry Point

```yaml
tosca_definitions_version: dell_1_0

description: >
  This deploys nodered, influxdb, and grafana containers on a
  NativeEdge Endpoint using NativeEdgeCompose

imports:
  - dell/types/types.yaml
  - container_based/definitions.yaml
  - container_based/inputs.yaml
  - container_based/outputs.yaml
```

The entry point follows the same pattern as the Simple VM example: it imports the base types and three sub-files.

### Input Groups

```yaml
input_groups:
  nodered:
    display_label: NodeRed Container
    collapsible: true
    index: 0
    inputs:
      - nodered_name
      - nodered_url
  influxdb:
    display_label: InfluxDB Container
    collapsible: true
    index: 1
    inputs:
      - influxdb_name
      - influxdb_url
  grafana:
    display_label: Device Passthrough
    collapsible: true
    index: 2
    inputs:
      - grafana_name
      - grafana_url
```

The inputs are organized by container -- each container gets its own collapsible section in the NativeEdge UI with a name and image URL.

### Labels

```yaml
labels:
  csys-obj-type:
    values:
      - environment
  target_environment:
    values:
      - nativeedge
  solution:
    values:
      - POC-CONTAINER
```

Note the `solution: POC-CONTAINER` label, which distinguishes this from the VM-based examples (`POC-VM`, `POC-MQTT-NR`, `POC-NGINX`).

---

## Node Templates (definitions.yaml)

This definitions file is notably compact compared to the Simple VM example. It only imports the edge plugin (no utilities or fabric needed) and defines just two node templates:

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin
```

### Dependency Chain

```
prepare_compose_yaml ──> nativeedge_compose
```

The compose deployment depends on the YAML preparation step -- the compose template must be loaded before it can be deployed.

---

### Node 1: `prepare_compose_yaml` -- Load the Compose Template

```yaml
prepare_compose_yaml:
  type: dell.nodes.Root
  interfaces:
    dell.interfaces.lifecycle:
      start: |
        import sys
        from nativeedge import ctx

        filename = ctx.download_resource(
            "container_based/compose/container_compose.yaml"
        )
        if filename is None or filename == "":
          ctx.logger.info("MFG filename not found")
          sys.exit()

        with open(filename) as f:
          content = f.read()

        ctx.instance.runtime_properties["compose_yaml"] = content
        ctx.instance.update()
```

**What it does:** Reads the Docker Compose template file from the blueprint package and stores its content as a runtime property.

**Key concept -- inline Python script:**

Unlike the Simple VM example where scripts are in separate `.py` files, this node embeds a Python script directly in the YAML using a YAML block scalar (`|`). The script:

1. Uses `ctx.download_resource()` to retrieve the compose template file from the blueprint archive. Blueprint files are packaged as a zip archive; `download_resource` extracts the specified file to a temporary location and returns its path.
2. Reads the file content as a string.
3. Stores the content in `runtime_properties["compose_yaml"]` so other nodes can access it via `get_attribute`.
4. Calls `ctx.instance.update()` to persist the runtime property change.

**Key concept -- `ctx` (context object):**

The `ctx` object is the NativeEdge context, available in all Python scripts. It provides:
- `ctx.download_resource(path)` -- retrieves a file from the blueprint package
- `ctx.instance.runtime_properties` -- a dictionary of runtime properties for this node instance
- `ctx.instance.update()` -- persists changes to runtime properties
- `ctx.logger` -- a logger for writing to deployment logs

**Runtime attributes produced:**
- `compose_yaml` -- the raw content of the Docker Compose template as a string

---

### Node 2: `nativeedge_compose` -- Deploy the Container Stack

```yaml
nativeedge_compose:
  type: dell.nodes.nativeedge.template.NativeEdgeCompose
  properties:
    compose_config:
      service_tag: { get_environment_capability: ece_service_tag }
      definition:
        name: { get_sys: [deployment, name] }
        compose: { get_attribute: [ prepare_compose_yaml, compose_yaml ] }
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

**What it does:** Deploys a Docker Compose application on the NativeEdge endpoint.

**The `compose_config` property:**

- **`service_tag`** -- identifies the target NativeEdge endpoint, using `get_environment_capability: ece_service_tag` (same pattern as the VM example)
- **`definition.name`** -- names the compose project after the deployment (using `get_sys: [deployment, name]`)
- **`definition.compose`** -- the compose YAML content, retrieved from the `prepare_compose_yaml` node's runtime property
- **`definition.environment_variables`** -- key-value pairs that will be substituted into the compose template. These correspond to `${VARIABLE_NAME}` placeholders in the compose YAML file.

**Key concept -- environment variable substitution:**

The compose template file (`container_based/compose/container_compose.yaml`) contains placeholders like `${NODERED_NAME}`, `${NODERED_URL}`, etc. When NativeEdgeCompose deploys the stack, it replaces these placeholders with the values from the `environment_variables` map. This is the same mechanism Docker Compose uses with `.env` files, but here the values come from blueprint inputs.

This pattern allows a single compose template to be parameterized for different deployments. For example, you could change the container image URLs to use different registries or versions without modifying the template.

**Commented-out registry login:**

The definitions file includes a commented-out `registry_login` node template:

```yaml
# registry_login:
#   type: nativeedge.nodes.template.RegistryLogin
#   properties:
#     registry_config:
#       credentials: [{ "registry_url": ..., "username": ..., "password": ... }]
#     service_tag: { get_environment_capability: ece_service_tag }
```

This shows a pattern you might need for private container registries. If your images are in a private registry, uncomment this node and add a dependency from `nativeedge_compose` to `registry_login`. The registry login happens before the compose deployment, ensuring the endpoint can pull the private images.

---

## Inputs (inputs.yaml)

The inputs are straightforward -- each container has a name and an image URL:

```yaml
inputs:
  nodered_name:
    display_label: "Node-Red container name"
    description: "Name of the Node-Red container"
    type: string
    required: false
    default: "nodered"

  nodered_url:
    display_label: "Node-Red image URL and tag"
    description: "URL path and tag of Node-Red image"
    type: string
    required: false
    default: "mirror.gcr.io/nodered/node-red:latest"

  influxdb_name:
    display_label: "InfluxDB container name"
    type: string
    default: "influxdb"

  influxdb_url:
    display_label: "InfluxDB image URL and tag"
    type: string
    default: "mirror.gcr.io/influxdb:latest"

  grafana_name:
    display_label: "Grafana container name"
    type: string
    default: "grafana"

  grafana_url:
    display_label: "Grafana image URL and tag"
    type: string
    default: "mirror.gcr.io/grafana/grafana:latest"
```

All inputs have sensible defaults using `mirror.gcr.io` as the image registry. The `required: false` setting means users can deploy with defaults without changing anything.

The commented-out sections in the inputs file hint at additional capabilities you might want to enable:

- **Port overrides** -- custom port mappings for the containers
- **Resource limits** -- CPU and memory limits per container
- **Serial port access** -- for IoT use cases where containers need serial port access
- **Registry authentication** -- username, password, and URL for private registries

---

## Outputs (outputs.yaml)

The outputs construct URLs for each deployed service by dynamically looking up the endpoint's host IP:

```yaml
capabilities:

  node_red_url:
    description: Node-Red URL
    value:
      concat:
        - "http://"
        - jmespath:
            - extra.hw_core.network.hostNetwork[?isDefault==`true`].ipAddress | [0] | to_string(@)
            - get_inventory:
                get_environment_capability: ece_service_tag
        - ":1880"

  influxdb_url:
    description: InfluxDB URL
    value:
      concat:
        - "http://"
        - jmespath:
            - extra.hw_core.network.hostNetwork[?isDefault==`true`].ipAddress | [0] | to_string(@)
            - get_inventory:
                get_environment_capability: ece_service_tag
        - ":8086"

  grafana_url:
    description: Grafana URL
    value:
      concat:
        - "http://"
        - jmespath:
            - extra.hw_core.network.hostNetwork[?isDefault==`true`].ipAddress | [0] | to_string(@)
            - get_inventory:
                get_environment_capability: ece_service_tag
        - ":3000"
```

**Key concept -- JMESPath for inventory data:**

Each output uses the same pattern to build a URL:

1. `get_environment_capability: ece_service_tag` -- gets the endpoint's service tag
2. `get_inventory` -- retrieves the full inventory data for that endpoint
3. `jmespath` -- applies a JMESPath query to extract the default host network's IP address

The JMESPath expression `extra.hw_core.network.hostNetwork[?isDefault==\`true\`].ipAddress | [0] | to_string(@)` does the following:
- Navigates to the `hostNetwork` array in the inventory
- Filters for the entry where `isDefault` is `true`
- Extracts the `ipAddress` field
- Takes the first result (`[0]`)
- Converts it to a string

The `concat` function then assembles the full URL by prepending `http://` and appending the port number. The ports correspond to the default ports for each service:
- Node-RED: port 1880
- InfluxDB: port 8086
- Grafana: port 3000

After deployment, the NativeEdge UI displays these URLs, and users can click through to access each service.

---

## Execution Flow

1. **`prepare_compose_yaml`** executes first, loading the Docker Compose template from the blueprint package and storing it as a runtime property.
2. **`nativeedge_compose`** executes next, sending the compose definition with resolved environment variables to the NativeEdge endpoint, which pulls the container images and starts the services.
3. **Outputs** are computed by querying the endpoint's inventory for the host network IP and combining it with the service ports.

On uninstall, NativeEdgeCompose stops and removes the containers from the endpoint.

---

## Comparison with VM-Based Deployment

| Aspect | Container Deployment | Simple VM |
|---|---|---|
| Plugins needed | Edge only | Edge, Utilities, Fabric |
| Node count | 2 | 6 |
| Time to deploy | Seconds to minutes | Minutes (image upload + VM boot) |
| Resource overhead | Minimal (shared OS kernel) | Full OS per VM |
| Use case | Lightweight services, microservices | Full OS environments, legacy apps |
| Network model | Host network (endpoint IP + ports) | VM has its own IP address |

The container-based approach is significantly simpler and faster but provides less isolation than a full VM. Choose it when your application is already containerized and does not require a dedicated OS environment.
