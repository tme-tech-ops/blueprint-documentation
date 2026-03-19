# Relationships

Relationships in NativeEdge define how nodes interact with each other. They establish dependencies (ensuring one node is created before another) and can execute operations when the relationship is established or removed. The Utilities Plugin defines two custom relationship types that automate common patterns: loading configuration and reserving resources.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.
- Understanding of NativeEdge relationships, lifecycle phases, and the `source`/`target` concept.

## Understanding Relationships (for New Users)

In a NativeEdge blueprint, a relationship connects two nodes:

- The **source** node is the one that declares the relationship (in its `relationships` list).
- The **target** node is the one being referenced.

Relationships can have operations that run at specific phases:
- **preconfigure** -- Runs after both nodes are created but before they are configured. This is when initial data exchange happens.
- **postconfigure** -- Runs after both nodes are configured.
- **establish** -- Runs when the relationship is fully established.
- **unlink** -- Runs when the relationship is being removed (during uninstall).

Relationship operations can run on either the **source** or the **target** node's interfaces:
- `source_interfaces` -- Operations that run in the context of the source node.
- `target_interfaces` -- Operations that run in the context of the target node (but can modify the source node).

---

## dell.relationships.load_from_config

**Derived from:** `dell.relationships.depends_on`

This relationship loads configuration parameters from a `ConfigurationLoader` target node into the source node's runtime properties. It is the companion relationship to the `dell.nodes.ConfigurationLoader` node type (see [Configuration Loader](configuration.md)).

### How It Works

When this relationship is established, during the `preconfigure` phase, the plugin reads the `params` runtime property from the TARGET node (the ConfigurationLoader) and copies it into the SOURCE node's runtime properties. This means the source node automatically receives all configuration values without needing explicit `get_attribute` calls.

### Target Interface Operations

| Phase | Implementation | Inputs | Description |
|---|---|---|---|
| `preconfigure` | `configuration.plugins_configuration.tasks.load_configuration_to_runtime_properties` | `source_config` (default: `{ get_attribute: [TARGET, params] }`) | Reads the configuration from the target's `params` attribute and loads it into the source node's runtime properties. |

### Operation Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `source_config` | dict | `{ get_attribute: [TARGET, params] }` | The configuration data to load. By default, this reads from the target's `params` runtime property. You can override this to load from a different attribute or provide static values. |

### Example

```yaml
node_templates:
  configuration_loader:
    type: dell.nodes.ConfigurationLoader
    properties:
      parameters_json:
        db_host: database.internal
        db_port: 5432
        app_mode: production

  web_server:
    type: dell.nodes.ApplicationServer
    relationships:
      - type: dell.relationships.load_from_config
        target: configuration_loader

  api_server:
    type: dell.nodes.ApplicationServer
    relationships:
      - type: dell.relationships.load_from_config
        target: configuration_loader
```

In this example:
1. `configuration_loader` holds all configuration parameters.
2. Both `web_server` and `api_server` use the `load_from_config` relationship.
3. During deployment, the `preconfigure` operation copies all parameters (`db_host`, `db_port`, `app_mode`) into both `web_server` and `api_server` runtime properties.
4. Each server node can then access its configuration directly from its own runtime properties.

### The Pattern: ConfigurationLoader to Other Nodes

The `load_from_config` relationship implements a "fan-out" configuration pattern:

```
                    +--> web_server    (receives params)
                    |
configuration_loader+--> api_server    (receives params)
                    |
                    +--> worker_node   (receives params)
```

One central configuration node distributes parameters to many consumer nodes. This pattern makes it easy to update configuration in one place and have it propagate to all consumers.

---

## dell.relationships.resources.reserve_list_item

**Derived from:** `dell.relationships.connected_to`

This relationship automates the reservation and return of resources from a `dell.nodes.resources.List` node. When the relationship is established, a resource is reserved from the list. When the relationship is removed, the resource is returned.

### How It Works

1. **On preconfigure (relationship establishment):** The plugin calls the `reserve_list_item` task on the target (the List node). This removes one item from the list's `free_resources` and assigns it to the source node under its instance ID in `reservations`.

2. **On unlink (relationship removal):** The plugin calls the `return_list_item` task, which moves the resource back from `reservations` to `free_resources`.

3. **On update:** The reservation can be re-evaluated and updated.

4. **On check_drift:** The plugin checks whether the reservation is still valid.

### Target Interface Operations

| Phase | Implementation | Inputs | Description |
|---|---|---|---|
| `preconfigure` | `resources.plugins_resources.tasks.reserve_list_item` | `resources_list_node_id` (string, default `''`) | Reserves one resource from the target list for the source node. |
| `check_drift` | `resources.plugins_resources.tasks.check_drift_for_reserve_list_item_rel` | (none) | Checks whether the reservation is still valid and consistent. |
| `update` | `resources.plugins_resources.tasks.reserve_list_item` | `resources_list_node_id` (string, default `''`) | Updates the reservation. |
| `unlink` | `resources.plugins_resources.tasks.return_list_item` | (none) | Returns the reserved resource back to the free pool. |

### Operation Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `resources_list_node_id` | string | `''` | An optional identifier for the resource list node. When empty, the target node of the relationship is used. This input allows you to specify a different resource list if needed. |

### Example: Reserving IP Addresses

```yaml
node_templates:
  ip_pool:
    type: dell.nodes.resources.List
    properties:
      resource_config:
        - "10.0.1.10"
        - "10.0.1.11"
        - "10.0.1.12"
        - "10.0.1.13"
        - "10.0.1.14"

  server_a:
    type: dell.nodes.Root
    relationships:
      - type: dell.relationships.resources.reserve_list_item
        target: ip_pool

  server_b:
    type: dell.nodes.Root
    relationships:
      - type: dell.relationships.resources.reserve_list_item
        target: ip_pool
```

**What happens during deployment:**

1. `ip_pool` is created with `free_resources: ["10.0.1.10", "10.0.1.11", "10.0.1.12", "10.0.1.13", "10.0.1.14"]` and `reservations: {}`.

2. When `server_a` is created, the relationship `preconfigure` runs. It reserves `"10.0.1.10"` (the first free resource).
   - `free_resources` becomes `["10.0.1.11", "10.0.1.12", "10.0.1.13", "10.0.1.14"]`
   - `reservations` becomes `{"server_a_instance_id": "10.0.1.10"}`

3. When `server_b` is created, it reserves `"10.0.1.11"`.
   - `free_resources` becomes `["10.0.1.12", "10.0.1.13", "10.0.1.14"]`
   - `reservations` becomes `{"server_a_instance_id": "10.0.1.10", "server_b_instance_id": "10.0.1.11"}`

4. When `server_a` is uninstalled, the relationship `unlink` runs. It returns `"10.0.1.10"` to the free pool.
   - `free_resources` becomes `["10.0.1.12", "10.0.1.13", "10.0.1.14", "10.0.1.10"]`
   - `reservations` becomes `{"server_b_instance_id": "10.0.1.11"}`

### Why connected_to Instead of depends_on?

This relationship derives from `dell.relationships.connected_to` rather than `dell.relationships.depends_on`. In NativeEdge, `connected_to` implies a logical connection between nodes (not just a creation order dependency). It provides additional lifecycle phases (`establish` and `unlink`) that are not available with `depends_on`. The `unlink` phase is essential for the return pattern -- it ensures resources are returned when the relationship is dissolved.
