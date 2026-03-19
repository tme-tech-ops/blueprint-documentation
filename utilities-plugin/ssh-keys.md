# SSH Keys

The Utilities Plugin provides the `dell.nodes.keys.RSAKey` node type for generating RSA SSH key pairs directly within your NativeEdge blueprint. SSH keys are fundamental to secure access: they allow you to log into virtual machines without passwords, and they are the standard method for authenticating with Linux servers.

When you declare an RSAKey node, the plugin generates a private key and a public key. It can optionally store both keys as NativeEdge secrets (encrypted values in the NativeEdge secret store), making them available to other nodes, blueprints, and deployments without exposing them in plain text.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.
- Familiarity with SSH key pairs (public key / private key concepts).

## Node Type: dell.nodes.keys.RSAKey

**Derived from:** `dell.nodes.Root`

### Data Type: dell.datatypes.key

This data type defines the configuration for key generation. You use it as the value of the `resource_config` property on the RSAKey node.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `private_key_path` | string | No | (none) | File system path where the generated private key should be saved. If not provided, the key is generated in memory and not written to disk. |
| `public_key_path` | string | No | `'~/.ssh/id_rsa.pub'` | File system path where the generated public key should be saved. |
| `key_name` | string | No | (none) | A name for the key. This name is used as the base name for secrets when storing key material in the NativeEdge secret store. For example, if `key_name` is `"poc_key"`, the secrets will be named `poc_key_public` and `poc_key_private`. |
| `algorithm` | string | No | `'RSA'` | The cryptographic algorithm to use for key generation. The default and most common value is `RSA`. |
| `bits` | integer | No | `2048` | The key size in bits. Larger values (e.g., 4096) provide stronger security but take slightly longer to generate. 2048 is the standard minimum for RSA keys. |
| `comment` | string | No | (none) | An optional comment to embed in the public key. This typically identifies the key owner or purpose (e.g., `"admin@mycompany.com"`). |
| `passphrase` | string | No | (none) | An optional passphrase to encrypt the private key. If set, the private key file will be encrypted and require this passphrase to use. |
| `openssh_format` | boolean | No | (none) | Whether to generate the key in OpenSSH format. When `true`, the key is in the newer OpenSSH format rather than the traditional PEM format. Set this to `true` for compatibility with modern SSH implementations. |
| `unvalidated` | any | No | (none) | Unvalidated parameters. This is an escape hatch for passing additional parameters that are not covered by the other properties. |

### Properties

These are the properties you set on the RSAKey node in your blueprint:

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `use_secret_store` | boolean | No | `true` | Whether to store the generated key material in the NativeEdge secret store. When `true` (the default), the public and private keys are saved as secrets, making them accessible via `get_secret` from any node or blueprint. |
| `use_secrets_if_exist` | boolean | No | `false` | This flag only takes effect when `use_secret_store` is `true`. When `true`, the plugin first checks if secrets with the expected names already exist. If they do, it uses the existing key material instead of generating new keys. If the secrets do not exist, new keys are generated and stored. This is useful when multiple deployments should share the same SSH keys. |
| `key_name` | string | No | (none) | **Deprecated.** Use `resource_config.key_name` instead. This property exists for backward compatibility. |
| `resource_config` | dell.datatypes.key | Yes | (none) | The key generation configuration. See the `dell.datatypes.key` data type table above. |

### Runtime Properties

After the node is created, these properties are available on the node instance:

| Runtime Property | Type | Description |
|---|---|---|
| `secret_key_name` | string | The base name used for secrets in the NativeEdge secret store. For example, if this value is `"poc_key"`, the public key secret is named `"poc_key_public"` and the private key secret is named `"poc_key_private"`. |
| `secrets_key_owner` | boolean | Indicates whether this node instance created the secrets. If `use_secrets_if_exist` was `true` and the secrets already existed, this will be `false` (meaning the node is using someone else's keys). If the node generated new keys, this will be `true`. |
| `private_key_path` | string | The file system path to the private key file, if it was written to disk. |
| `public_key_path` | string | The file system path to the public key file, if it was written to disk. |

### Lifecycle Operations

| Operation | Implementation | Inputs | Description |
|---|---|---|---|
| `create` | `keys.plugins_ssh_key.operations.create` | `store_public_key_material` (boolean, default `true`), `store_private_key_material` (boolean, default `false`) | Generates the RSA key pair. Optionally stores the key material as runtime properties (controlled by the inputs) and as secrets (controlled by `use_secret_store`). |
| `delete` | `keys.plugins_ssh_key.operations.delete` | (none) | Removes the key material. If the node owns the secrets (`secrets_key_owner` is `true`), the secrets are deleted from the store. |

**About the create inputs:**

- `store_public_key_material`: When `true`, the public key content is stored as a runtime property on the node instance. This is recommended because it allows other nodes to access the public key via `get_attribute`. Default is `true`.
- `store_private_key_material`: When `true`, the private key content is stored as a runtime property. This is **not recommended** for manager deployments because runtime properties are stored in the database and may be visible to users with deployment access. Default is `false`. The private key is still accessible via `get_secret` if `use_secret_store` is `true`.

## Example: Generating SSH Keys

The following example is from a real VM provisioning blueprint:

```yaml
node_templates:
  ssh_keys:
    type: dell.nodes.keys.RSAKey
    properties:
      use_secret_store: true
      use_secrets_if_exist: true
      key_name: "poc_key"
      resource_config:
        openssh_format: true
        key_name: "poc_key"
    interfaces:
      dell.interfaces.lifecycle:
        create:
          implementation: keys.ne_ssh_key.operations.create
          inputs:
            store_public_key_material: false
            store_private_key_material: false
        delete: {}
```

### What this does:

1. **Key generation:** On `create`, the plugin generates an RSA key pair in OpenSSH format.
2. **Secret storage:** Because `use_secret_store` is `true`, the keys are stored in the NativeEdge secret store as:
   - `poc_key_public` -- the public key
   - `poc_key_private` -- the private key
3. **Reuse existing keys:** Because `use_secrets_if_exist` is `true`, if secrets named `poc_key_public` and `poc_key_private` already exist (from a previous deployment or another blueprint), the plugin will use those existing keys instead of generating new ones.
4. **No runtime property storage:** Both `store_public_key_material` and `store_private_key_material` are set to `false`, so the actual key content is not stored as runtime properties. The keys are only accessible via `get_secret`.
5. **On delete:** The `delete` operation is defined with `{}` (empty), meaning it uses the default implementation which will clean up the secrets if this node instance owns them.

## The Secret Key Name Pattern

Understanding how SSH keys are stored and retrieved is critical for using them in your blueprints.

### Storage

When the RSAKey node creates keys with `use_secret_store: true`, it creates two secrets:

- `{key_name}_public` -- contains the public key material
- `{key_name}_private` -- contains the private key material

The `key_name` comes from `resource_config.key_name`. The base name is also stored as the `secret_key_name` runtime property.

### Retrieval

To retrieve the public key from another node, use this pattern:

```yaml
ssh_authorized_keys:
  - get_secret:
      concat:
        - get_attribute: [ssh_keys, secret_key_name]
        - "_public"
```

This works in three steps:
1. `get_attribute: [ssh_keys, secret_key_name]` resolves to the base key name (e.g., `"poc_key"`).
2. `concat` appends `"_public"` to get `"poc_key_public"`.
3. `get_secret` retrieves the secret value -- the actual public key content.

For the private key, use `"_private"` instead:

```yaml
key:
  get_secret:
    concat:
      - get_attribute: [ssh_keys, secret_key_name]
      - "_private"
```

### Full Example: Using SSH Keys in Cloud-Init and Fabric

Here is how the keys flow through a typical blueprint:

```yaml
node_templates:
  ssh_keys:
    type: dell.nodes.keys.RSAKey
    properties:
      use_secret_store: true
      use_secrets_if_exist: true
      resource_config:
        openssh_format: true
        key_name: "poc_key"

  cloudinit:
    type: dell.nodes.CloudInit.CloudConfig
    properties:
      resource_config:
        users:
          - name: admin
            ssh_authorized_keys:
              # Inject the public key into the VM's authorized_keys
              - get_secret:
                  concat:
                    - get_attribute: [ssh_keys, secret_key_name]
                    - "_public"
    relationships:
      - type: dell.relationships.depends_on
        target: ssh_keys

  vm_info:
    type: dell.nodes.Root
    interfaces:
      dell.interfaces.lifecycle:
        start:
          implementation: fabric.fabric_plugin.tasks.run_script
          inputs:
            script_path: scripts/get_vm_info.sh
            fabric_env:
              host_string: { get_attribute: [vm, vm_details, name] }
              user: admin
              # Use the private key to SSH into the VM
              key:
                get_secret:
                  concat:
                    - get_attribute: [ssh_keys, secret_key_name]
                    - "_private"
              port: 22
    relationships:
      - type: dell.relationships.depends_on
        target: ssh_keys
```

In this flow:
1. `ssh_keys` generates a key pair and stores it as secrets.
2. `cloudinit` puts the public key into the VM's `authorized_keys` file via cloud-init.
3. `vm_info` uses the private key to SSH into the VM via the Fabric plugin to run scripts after the VM boots.

The `depends_on` relationships ensure that `ssh_keys` is created before any node that needs the keys.
