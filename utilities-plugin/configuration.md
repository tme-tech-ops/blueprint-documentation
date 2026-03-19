# Configuration Loader

The `dell.nodes.ConfigurationLoader` node type loads JSON configuration data into a node's runtime properties, making it available for other nodes to consume. This is a foundational pattern in NativeEdge blueprints for centralizing configuration: you define all your parameters in one place, and other nodes pull what they need through relationships.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.
- Understanding of NativeEdge runtime properties and the `get_attribute` intrinsic function.

## Node Type: dell.nodes.ConfigurationLoader

**Derived from:** `dell.nodes.ApplicationServer`

Unlike most other node types in this plugin (which derive from `dell.nodes.Root`), the ConfigurationLoader derives from `dell.nodes.ApplicationServer`. This places it at a higher level in the NativeEdge type hierarchy, which can affect how it interacts with containment relationships and lifecycle ordering.

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `parameters_json` | string/dict | No | `''` | The configuration data to load. This can be a JSON string or a YAML dictionary. It represents the full set of configuration parameters that this node will make available to other nodes. |

### Runtime Properties

| Runtime Property | Type | Description |
|---|---|---|
| `params` | string/dict | The loaded configuration parameters. After the `configure` operation runs, this runtime property contains the parsed configuration data from `parameters_json`. Other nodes can access individual values using `get_attribute: [config_loader_node, params, key_name]`. |

### Lifecycle Operations

| Operation | Implementation | Inputs | Description |
|---|---|---|---|
| `configure` | `configuration.plugins_configuration.tasks.load_configuration` | `parameters` (default: `{ get_property: [SELF, parameters_json] }`), `merge_dicts` (default: `false`) | Loads the configuration data into the `params` runtime property. |

**Operation Inputs:**

| Input | Type | Default | Description |
|---|---|---|---|
| `parameters` | string/dict | `{ get_property: [SELF, parameters_json] }` | The configuration data to load. By default, this reads from the node's `parameters_json` property, but you can override it to load from a different source. |
| `merge_dicts` | boolean | `false` | When `true`, if `params` already contains data (from a previous operation), the new parameters are merged into the existing data rather than replacing it. When `false`, the new parameters completely replace any existing data. |

### Example: Centralizing Configuration

```yaml
node_templates:
  configuration_loader:
    type: dell.nodes.ConfigurationLoader
    properties:
      parameters_json:
        db_host: database.internal
        db_port: 5432
        db_name: myapp
        cache_ttl: 300
        log_level: info
        features:
          enable_notifications: true
          enable_analytics: false

  app_server:
    type: dell.nodes.ApplicationServer
    relationships:
      - type: dell.relationships.load_from_config
        target: configuration_loader
```

In this example:
1. The `configuration_loader` node holds all configuration parameters.
2. The `app_server` node uses the `dell.relationships.load_from_config` relationship to automatically load configuration from the loader.
3. During the `preconfigure` phase of the relationship, the configuration data is copied from the loader's `params` runtime property into the `app_server`'s runtime properties.

### The Configuration Pattern

The ConfigurationLoader is designed to work with the `dell.relationships.load_from_config` relationship (see [Relationships](relationships.md)). The typical pattern is:

1. Define a `ConfigurationLoader` node with all your parameters.
2. Connect other nodes to it using `dell.relationships.load_from_config`.
3. The relationship automatically copies the parameters into each connected node's runtime properties during the `preconfigure` phase.

This pattern centralizes configuration management: if you need to change a parameter, you change it in one place (the ConfigurationLoader), and all connected nodes pick up the change.

### Merging Configuration

The `merge_dicts` input to the `configure` operation controls how configuration updates are handled:

- **`merge_dicts: false` (default):** The new parameters completely replace any existing `params` value. This is a clean slate approach.
- **`merge_dicts: true`:** The new parameters are merged with existing `params`. This is useful when configuration comes from multiple sources or is built up incrementally across lifecycle stages.

This merging behavior is also used by the `configuration_update` workflow (see [Workflows](workflows.md#configuration_update)), which can update configuration values on a running deployment without reinstalling it.

### Using Configuration Values

After the ConfigurationLoader runs, other nodes can access its parameters:

```yaml
# Direct access via get_attribute
some_node:
  type: dell.nodes.Root
  interfaces:
    dell.interfaces.lifecycle:
      create:
        inputs:
          db_host: { get_attribute: [configuration_loader, params, db_host] }
          db_port: { get_attribute: [configuration_loader, params, db_port] }
  relationships:
    - type: dell.relationships.depends_on
      target: configuration_loader
```

Or through the `load_from_config` relationship, which automatically copies the configuration into the node's runtime properties:

```yaml
# Automatic loading via relationship
some_node:
  type: dell.nodes.ApplicationServer
  relationships:
    - type: dell.relationships.load_from_config
      target: configuration_loader
```

See [Relationships](relationships.md) for details on how `dell.relationships.load_from_config` works.
