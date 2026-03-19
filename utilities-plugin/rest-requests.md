# REST Requests

The Utilities Plugin provides two node types for making HTTP REST API calls from your blueprints:

- **`dell.nodes.rest.Requests`** -- Executes REST API calls using Jinja2 template files. Each lifecycle operation can run a different set of REST calls defined in a template.
- **`dell.nodes.rest.BunchRequests`** -- Executes multiple REST API call templates in a single batch, with shared authentication.

REST nodes are useful when your blueprint needs to interact with external APIs -- for example, registering a VM with a monitoring service, configuring a load balancer, creating DNS records, or calling any HTTP-based service as part of the deployment lifecycle.

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.
- A REST API endpoint to call.
- Understanding of HTTP methods (GET, POST, PUT, DELETE) and JSON.

## Node Type: dell.nodes.rest.Requests

**Derived from:** `dell.nodes.Root`

This node type executes REST API calls defined in Jinja2 template files. The template approach lets you define the full request (URL path, method, headers, body) in a separate file that supports variable substitution, conditionals, and loops.

### Properties

These properties configure the connection to the REST API server:

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `hosts` | list | No | `[]` | A list of host names or IP addresses of REST servers. When multiple hosts are provided, the plugin can failover between them. This property takes precedence over `host`. |
| `host` | string/list | No | `[]` | A single host name or IP address of the REST server. Use this when you only need to connect to one server. If `hosts` is also provided, `hosts` takes precedence and this value is ignored. |
| `port` | integer | No | `-1` | The port number for the REST server. When set to `-1` (the default), standard ports are used: port 80 for HTTP (`ssl: false`) and port 443 for HTTPS (`ssl: true`). |
| `ssl` | boolean | No | `false` | Whether to use HTTPS (`true`) or HTTP (`false`). When `true`, connections are encrypted with TLS/SSL. |
| `verify` | boolean/string | No | `true` | Controls TLS certificate verification. Accepted values: `true` (verify certificates, the default), `false` (skip verification -- insecure), a file path to a CA certificate file, or the certificate content as a string. |
| `cert` | string/null | No | `null` | Client certificate for mutual TLS authentication. Accepted values: `null` (no client certificate, the default), a file path to the client certificate, or the certificate content as a string. |
| `timeout` | integer/null | No | `null` | Timeout in seconds for HTTP requests. When `null`, the default timeout of the underlying HTTP library is used. |
| `proxies` | dict | No | `{}` | A dictionary of proxy URLs. Keys are protocols (e.g., `http`, `https`) and values are proxy URLs (e.g., `http://proxy.example.com:8080`). |
| `params` | dict | No | `{}` | Template parameters. These values are passed to the Jinja2 template and can be referenced inside the template using `{{ params.key_name }}`. |

### Operation Inputs

Every lifecycle operation (`create`, `configure`, `start`, `stop`, `delete`) accepts the same set of inputs. These inputs control what REST calls are made during each operation:

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `params` | dict | No | `{}` | Template parameters for this specific operation. Merged with the `params` node property. Also includes a special `ctx` key that provides access to the current operation context (node instance, deployment info, etc.). |
| `template_file` | string | No | `''` | Path to a Jinja2 template file within the blueprint directory. This template defines the REST calls to make. If empty, no calls are made for this operation. |
| `save_path` | string/boolean | No | `false` | Where to save REST call results in runtime properties. When `false` (the default), results are saved directly to the root of runtime properties. When set to a string, results are saved under that key. |
| `prerender` | boolean | No | `false` | Controls the template rendering order. When `false` (default), the template is first parsed as YAML, then Jinja2 expressions are rendered. When `true`, the template is first rendered by Jinja2, then parsed as YAML. Use `true` when the YAML structure itself needs to be generated dynamically. |
| `remove_calls` | boolean | No | `false` | When `true`, the list of REST calls is removed from the results stored in runtime properties. When `false` (default), the call details are preserved for debugging and inspection. |
| `force_rerun` | boolean | No | `false` | When `true`, the operation re-executes even if it has already been run. Useful for retry scenarios. |
| `retry_count` | integer | No | `1` | The number of times to retry the REST calls if warnings are encountered. |
| `retry_sleep` | integer | No | `15` | The number of seconds to wait between retries. |

### Lifecycle Operations

All lifecycle operations use the same underlying implementation (`rest.plugins_rest.tasks.execute`), but you configure each one with different `template_file` inputs to define what REST calls happen at each stage.

| Operation | Implementation | Description |
|---|---|---|
| `create` | `rest.plugins_rest.tasks.execute` | Executes REST calls defined in the template. Typically used to create resources on the remote API. |
| `configure` | `rest.plugins_rest.tasks.execute` | Executes REST calls. Typically used to configure resources after creation. |
| `start` | `rest.plugins_rest.tasks.execute` | Executes REST calls. Typically used to activate or start resources. |
| `stop` | `rest.plugins_rest.tasks.execute` | Executes REST calls. Typically used to deactivate or stop resources. |
| `delete` | `rest.plugins_rest.tasks.execute` | Executes REST calls. Typically used to delete resources from the remote API. |

### The Template-Based Approach

The REST node uses Jinja2 templates stored in your blueprint to define the actual HTTP requests. This is a powerful pattern that separates the connection configuration (node properties) from the request definitions (templates).

A template file typically looks like this:

```yaml
rest_calls:
  - path: /api/v1/servers
    method: POST
    headers:
      Content-Type: application/json
      Authorization: "Bearer {{ params.api_token }}"
    payload:
      name: "{{ params.server_name }}"
      ip: "{{ params.server_ip }}"
    recoverable_codes:
      - 500
    successful_codes:
      - 200
      - 201
```

Each entry in `rest_calls` defines one HTTP request with:
- `path` -- the URL path (appended to the host)
- `method` -- the HTTP method (GET, POST, PUT, DELETE, etc.)
- `headers` -- HTTP headers
- `payload` -- the request body (for POST/PUT)
- `recoverable_codes` -- HTTP status codes that trigger a retry
- `successful_codes` -- HTTP status codes that indicate success

The `{{ params.xxx }}` syntax inserts values from the `params` input, allowing you to parameterize the template.

### Example: REST Requests Node

```yaml
node_templates:
  register_server:
    type: dell.nodes.rest.Requests
    properties:
      hosts:
        - api.monitoring.example.com
      port: 443
      ssl: true
      verify: true
      params:
        api_token: { get_secret: monitoring_api_token }
    interfaces:
      dell.interfaces.lifecycle:
        create:
          inputs:
            template_file: templates/register_server.yaml
            params:
              server_name: { get_attribute: [vm, name] }
              server_ip: { get_attribute: [vm, ip] }
        delete:
          inputs:
            template_file: templates/deregister_server.yaml
            params:
              server_id: { get_attribute: [SELF, server_id] }
```

---

## Node Type: dell.nodes.rest.BunchRequests

**Derived from:** `dell.nodes.Root`

This node type is similar to `dell.nodes.rest.Requests` but designed for executing multiple templates in a batch with shared authentication. Instead of a single `template_file` input, it takes `auth` and `templates` inputs.

### Properties

The connection properties are identical to `dell.nodes.rest.Requests`:

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `hosts` | list | No | `[]` | List of host names or IP addresses of REST servers. |
| `host` | string/list | No | `[]` | Single host name or IP address. Overridden by `hosts` if both are provided. |
| `port` | integer | No | `-1` | Port number. `-1` means use default (80 for HTTP, 443 for HTTPS). |
| `ssl` | boolean | No | `false` | Whether to use HTTPS. |
| `verify` | boolean/string | No | `true` | TLS certificate verification setting. |
| `cert` | string/null | No | `null` | Client certificate for mutual TLS. |
| `timeout` | integer/null | No | `null` | Request timeout in seconds. |
| `proxies` | dict | No | `{}` | Proxy configuration. |
| `params` | dict | No | `{}` | Template parameters. |

### Operation Inputs

Every lifecycle operation accepts these inputs:

| Input | Type | Required | Default | Description |
|---|---|---|---|---|
| `auth` | dict | No | `{}` | Authentication configuration shared across all templates in the batch. |
| `templates` | dict | No | `{}` | A dictionary of template configurations. Each entry defines a REST call template with its own parameters. |

### Lifecycle Operations

| Operation | Implementation | Description |
|---|---|---|
| `create` | `rest.plugins_rest.tasks.bunch_execute` | Executes all REST call templates in the batch. |
| `configure` | `rest.plugins_rest.tasks.bunch_execute` | Executes all REST call templates in the batch. |
| `start` | `rest.plugins_rest.tasks.bunch_execute` | Executes all REST call templates in the batch. |
| `stop` | `rest.plugins_rest.tasks.bunch_execute` | Executes all REST call templates in the batch. |
| `delete` | `rest.plugins_rest.tasks.bunch_execute` | Executes all REST call templates in the batch. |

### Example: BunchRequests Node

```yaml
node_templates:
  api_setup:
    type: dell.nodes.rest.BunchRequests
    properties:
      hosts:
        - api.example.com
      ssl: true
    interfaces:
      dell.interfaces.lifecycle:
        create:
          inputs:
            auth:
              username: { get_secret: api_user }
              password: { get_secret: api_pass }
            templates:
              create_network:
                template_file: templates/create_network.yaml
                params:
                  network_name: my-network
              create_subnet:
                template_file: templates/create_subnet.yaml
                params:
                  subnet_cidr: "10.0.0.0/24"
```

## Key Concepts for New Users

### Why use templates instead of inline REST calls?

Templates provide several advantages:
1. **Separation of concerns** -- Connection settings are in the node properties, request definitions are in templates.
2. **Reusability** -- The same template can be used by multiple nodes with different parameters.
3. **Dynamic content** -- Jinja2 templating lets you use conditionals, loops, and variable substitution.
4. **Readability** -- Complex REST call sequences are easier to read in their own files.

### Requests vs. BunchRequests

Use `dell.nodes.rest.Requests` when you need to make REST calls at specific lifecycle stages (e.g., create a resource on `create`, delete it on `delete`). Use `dell.nodes.rest.BunchRequests` when you need to execute multiple different templates in a single operation with shared authentication.
