# Secrets Management

NativeEdge has a built-in secret store -- an encrypted key-value store where you can save sensitive data such as passwords, API keys, tokens, and certificates. The Utilities Plugin provides two node types for interacting with this store directly from your blueprints:

- **`dell.nodes.secrets.Writer`** -- Creates (writes) secrets into the store.
- **`dell.nodes.secrets.Reader`** -- Reads existing secrets from the store and exposes them as runtime properties.

These node types are useful when your blueprint needs to programmatically create secrets (e.g., generating credentials during deployment) or when you want to read multiple secrets into a single node's runtime properties for easy access by other nodes.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.
- Understanding of the NativeEdge secret store (`get_secret` intrinsic function).

## Node Type: dell.nodes.secrets.Writer

**Derived from:** `dell.nodes.Root`

The Writer node creates one or more secrets in the NativeEdge secret store during the `create` lifecycle operation. It can also update and delete those secrets as the deployment is managed.

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `entries` | dict | No | `{}` | A dictionary of key-value pairs to store as secrets. Each key becomes a secret name, and its value becomes the secret value. For example, `{ db_password: "s3cret", api_key: "abc123" }` creates two secrets. |
| `do_not_delete` | boolean | No | `false` | When `true`, the secrets created by this node will not be deleted when the node is deleted (during uninstall). This is useful when secrets should persist beyond the lifetime of the deployment that created them. |
| `variant` | string | No | (none) | An optional variant identifier. This can be used to differentiate between different sets of secrets or to control how the secret names are constructed. |
| `separator` | string | No | (none) | An optional separator string used when constructing secret names from hierarchical keys. For example, if separator is `"."` and you have nested entries, the keys are joined with dots. |
| `logs_secrets` | boolean | No | `false` | When `true`, secret values are included in log output. **Use with caution** -- this is intended for debugging only and should never be enabled in production deployments, as it exposes sensitive values in logs. |
| `update_if_exists` | boolean | No | `false` | When `true`, if a secret with the same name already exists, its value is updated. When `false` (the default), attempting to create a secret that already exists may result in an error. |

### Runtime Properties

| Runtime Property | Type | Description |
|---|---|---|
| `data` | dict | The values of the entries that were written. This is a dictionary mirroring the `entries` property, allowing other nodes to access the secret names and values via `get_attribute`. |
| `do_not_delete` | boolean | The current value of the `do_not_delete` setting. This can change if updated during the deployment lifecycle. |

### Lifecycle Operations

| Operation | Interface | Implementation | Description |
|---|---|---|---|
| `create` | `dell.interfaces.lifecycle` | `secrets.plugins_secrets.tasks.create` | Creates all secrets defined in `entries`. Stores the entries as runtime properties in `data`. |
| `check_drift` | `dell.interfaces.lifecycle` | `secrets.plugins_secrets.tasks.check_drift` | Checks whether the secrets in the store still match the expected values. Used for drift detection. |
| `update` | `dell.interfaces.lifecycle` | `secrets.plugins_secrets.tasks.update` | Updates the secret values in the store. Called during the update lifecycle. |
| `delete` | `dell.interfaces.lifecycle` | `secrets.plugins_secrets.tasks.delete` | Deletes the secrets from the store, unless `do_not_delete` is `true`. |
| `update` | `dell.interfaces.operations` | `secrets.plugins_secrets.tasks.update` | An additional update operation available through the operations interface. This allows triggering updates outside of the standard lifecycle, such as from workflows. |

### Example: Writing Secrets

```yaml
node_templates:
  my_secrets:
    type: dell.nodes.secrets.Writer
    properties:
      entries:
        database_password: { get_input: db_password }
        api_token: { get_input: api_token }
        service_url: "https://api.example.com"
      do_not_delete: false
      update_if_exists: true
```

This creates three secrets in the NativeEdge secret store:
- `database_password` with the value from the `db_password` blueprint input
- `api_token` with the value from the `api_token` blueprint input
- `service_url` with the literal value `"https://api.example.com"`

Other nodes or blueprints can then retrieve these values using `{ get_secret: database_password }`.

---

## Node Type: dell.nodes.secrets.Reader

**Derived from:** `dell.nodes.Root`

The Reader node retrieves existing secrets from the NativeEdge secret store and makes their values available as runtime properties. This is useful when you need to read multiple secrets at once and make them accessible via `get_attribute` for other nodes in the blueprint.

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `keys` | list | No | `[]` | A list of secret names to read from the NativeEdge secret store. Each name in the list corresponds to an existing secret. The values are read and stored in the `data` runtime property. |
| `variant` | string | No | (none) | An optional variant identifier, matching the variant used when the secrets were written. |
| `separator` | string | No | (none) | An optional separator string, matching the separator used when the secrets were written. |

### Runtime Properties

| Runtime Property | Type | Description |
|---|---|---|
| `data` | dict | A dictionary of the secret values that were read. The keys are the secret names (from the `keys` property) and the values are the secret contents retrieved from the store. |

### Lifecycle Operations

| Operation | Interface | Implementation | Description |
|---|---|---|---|
| `create` | `dell.interfaces.lifecycle` | `secrets.plugins_secrets.tasks.read` | Reads all secrets specified in `keys` and stores them in the `data` runtime property. |
| `check_drift` | `dell.interfaces.lifecycle` | `secrets.plugins_secrets.tasks.reader_check_drift` | Checks whether the secret values in the store have changed since the last read. |
| `update` | `dell.interfaces.lifecycle` | `secrets.plugins_secrets.tasks.read` | Re-reads all secrets and updates the `data` runtime property. |
| `update` | `dell.interfaces.operations` | `secrets.plugins_secrets.tasks.read` | An additional read operation available through the operations interface, allowing re-reads triggered by workflows. |

### Example: Reading Secrets

```yaml
node_templates:
  read_credentials:
    type: dell.nodes.secrets.Reader
    properties:
      keys:
        - database_password
        - api_token
        - service_url
```

After the `create` operation runs, the `data` runtime property will contain:

```json
{
  "database_password": "s3cret",
  "api_token": "abc123",
  "service_url": "https://api.example.com"
}
```

Other nodes can access individual values:

```yaml
some_other_node:
  type: dell.nodes.Root
  interfaces:
    dell.interfaces.lifecycle:
      create:
        inputs:
          db_pass: { get_attribute: [read_credentials, data, database_password] }
  relationships:
    - type: dell.relationships.depends_on
      target: read_credentials
```

## Writer and Reader Together

A common pattern is to use a Writer in one deployment and a Reader in another:

**Deployment A (creates secrets):**
```yaml
node_templates:
  shared_creds:
    type: dell.nodes.secrets.Writer
    properties:
      entries:
        shared_db_host: "db.internal.example.com"
        shared_db_port: "5432"
        shared_db_password: { get_input: db_password }
      do_not_delete: true  # Keep secrets after uninstall
```

**Deployment B (reads secrets):**
```yaml
node_templates:
  creds_reader:
    type: dell.nodes.secrets.Reader
    properties:
      keys:
        - shared_db_host
        - shared_db_port
        - shared_db_password
```

This pattern allows secrets to be shared across deployments in a controlled way. The `do_not_delete: true` flag ensures the secrets persist even if Deployment A is uninstalled.
