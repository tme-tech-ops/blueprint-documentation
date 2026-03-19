# Files and FTP

The Utilities Plugin provides two node types for managing files:

- **`dell.nodes.File`** -- Manages files on the local system (the NativeEdge manager). It can create files from templates stored in the blueprint, set ownership and permissions, and detect configuration drift.
- **`dell.nodes.ftp`** -- Uploads files to a remote FTP server. Supports both plain FTP and FTP over TLS (FTPS).

## Prerequisites

- Import the Utilities Plugin in your blueprint. See [Utilities Plugin Overview](overview.md) for instructions.

---

## Node Type: dell.nodes.File

**Derived from:** `dell.nodes.Root`

This node type creates a file on the system where the NativeEdge manager runs (since the plugin uses the `central_deployment_agent` executor). The file source is a path within the blueprint package, and the destination is a path on the manager's filesystem.

### Data Type: dell.datatypes.File

This data type defines the file specification. It is used as the type of the `resource_config` property.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `resource_path` | string | Yes | (none) | The path to the source file relative to the blueprint directory. This file must be packaged within the blueprint archive. External URIs are not supported. For example, `"files/my_config.conf"` refers to a file at `files/my_config.conf` inside the blueprint. |
| `file_path` | string | Yes | (none) | The absolute path on the target machine where the file should be written. For example, `"/etc/myapp/config.conf"`. |
| `owner` | string | Yes | (none) | The file ownership string in the format `"user:group"`. For example, `"centos:wheel"` sets the file owner to `centos` and the group to `wheel`. |
| `mode` | integer | Yes | (none) | The file permissions as an integer. For example, `644` gives read/write to the owner and read-only to group and others. **Important:** The value must be provided as a plain integer. Values like `"0644"` (string) or `0644` (octal literal) are not valid -- use `644` instead. |
| `template_variables` | dict | No | (none) | A dictionary of variables used to render the file as a Jinja2 template. If provided, the file at `resource_path` is treated as a Jinja2 template, and the variables in this dictionary are substituted into the template before writing the output to `file_path`. |
| `use_sudo` | boolean | No | `false` | Whether to use `sudo` when performing file operations (move, rename, delete, chown, chmod). Set to `true` if the destination path requires elevated privileges. |
| `allow_failure` | boolean | No | `false` | If the file download or creation fails, log the error and continue instead of failing the operation. Useful for optional configuration files. |

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `resource_config` | dell.datatypes.File | Yes | (none) | The file specification. See the `dell.datatypes.File` data type table above. |

### Lifecycle Operations

| Operation | Implementation | Description |
|---|---|---|
| `create` | `plugins_files.plugins_files.tasks.create` | Creates the file at the specified `file_path` with the content from `resource_path`, applying template variables, ownership, and permissions. |
| `delete` | `plugins_files.plugins_files.tasks.delete` | Removes the file from the filesystem. |
| `check_drift` | `plugins_files.plugins_files.tasks.check_drift` | Checks whether the file on disk still matches the expected state (content, permissions, ownership). Returns drift status. |
| `update` | `plugins_files.plugins_files.tasks.create` | Re-creates the file. Uses the same implementation as `create`, effectively overwriting the file with the current configuration. |

### Example: Managing a Configuration File

```yaml
node_templates:
  app_config:
    type: dell.nodes.File
    properties:
      resource_config:
        resource_path: files/app.conf.j2
        file_path: /etc/myapp/app.conf
        owner: "myapp:myapp"
        mode: 644
        template_variables:
          db_host: { get_attribute: [database, host] }
          db_port: 5432
          log_level: info
        use_sudo: true
    relationships:
      - type: dell.relationships.depends_on
        target: database
```

In this example:
1. The file `files/app.conf.j2` in the blueprint is a Jinja2 template.
2. The template is rendered with the provided variables (`db_host`, `db_port`, `log_level`).
3. The rendered file is written to `/etc/myapp/app.conf` on the manager.
4. Ownership is set to `myapp:myapp` and permissions to `644`.
5. `use_sudo: true` is needed because `/etc/myapp/` may require root access.

---

## Node Type: dell.nodes.ftp

**Derived from:** `dell.nodes.Root`

This node type uploads files to a remote FTP server. It supports both raw files (binary files from the blueprint) and content-based files (files whose content is defined inline in the blueprint).

### Data Type: dell.datatypes.ftp_auth

This data type defines the FTP server connection credentials. It is used as the type of the `resource_config` property.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `user` | string | No | `''` | The FTP login username. Leave empty for anonymous access. |
| `password` | string | No | `''` | The FTP login password. |
| `ip` | string | No | `''` | The IP address or hostname of the FTP server. |
| `port` | integer | No | `21` | The FTP server port. The standard FTP port is 21. |
| `ignore_host` | boolean | No | `false` | When `true`, ignores the host address returned in FTP server responses. This can be necessary when the FTP server returns an internal IP address that is not reachable from the manager. |
| `tls` | boolean | No | `false` | When `true`, uses FTP over TLS (FTPS) for encrypted file transfers. The connection is upgraded to TLS after the initial connection. |

### Properties

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `resource_config` | dell.datatypes.ftp_auth | Yes | (none) | The FTP server connection configuration. See the `dell.datatypes.ftp_auth` data type table above. |
| `raw_files` | dict | No | `{}` | A dictionary of files from the blueprint to upload as-is (binary). The keys are the remote file paths on the FTP server, and the values are the local file paths within the blueprint. |
| `files` | dict | No | `{}` | A dictionary of files with inline content to upload. The keys are the remote file paths on the FTP server, and the values are the file contents defined in the blueprint. |

### Runtime Properties

| Runtime Property | Type | Description |
|---|---|---|
| `files` | list | A list of file names that have been successfully uploaded to the FTP server. |

### Lifecycle Operations

| Operation | Implementation | Inputs | Description |
|---|---|---|---|
| `create` | `plugins_ftp.plugins_ftp.tasks.create` | `resource_config`, `raw_files`, `files` | Connects to the FTP server and uploads all specified files. |
| `check_drift` | `plugins_ftp.plugins_ftp.tasks.check_drift` | `resource_config`, `raw_files`, `files` | Checks whether the files on the FTP server still match the expected state. |
| `update` | `plugins_ftp.plugins_ftp.tasks.update` | `resource_config`, `raw_files`, `files` | Re-uploads files, updating any that have changed. |
| `delete` | `plugins_ftp.plugins_ftp.tasks.delete` | `resource_config` | Removes the uploaded files from the FTP server. |

All operation inputs default to the corresponding node property values using `{ get_property: [SELF, <property_name>] }`.

### Example: Uploading Files via FTP

```yaml
node_templates:
  firmware_upload:
    type: dell.nodes.ftp
    properties:
      resource_config:
        user: { get_secret: ftp_user }
        password: { get_secret: ftp_password }
        ip: 192.168.1.100
        port: 21
        tls: true
      raw_files:
        /uploads/firmware.bin: firmware/device_firmware.bin
      files:
        /uploads/config.txt: |
          hostname=edge-device-01
          mode=production
          version=2.0
```

In this example:
1. The node connects to the FTP server at `192.168.1.100` using TLS.
2. The binary file `firmware/device_firmware.bin` from the blueprint is uploaded to `/uploads/firmware.bin` on the FTP server.
3. A text file with inline content is created and uploaded to `/uploads/config.txt` on the FTP server.

## Key Concepts for New Users

### File Node vs. FTP Node

- Use `dell.nodes.File` when you need to create files on the NativeEdge manager's local filesystem.
- Use `dell.nodes.ftp` when you need to upload files to a remote FTP server.

### Template Variables in File Nodes

The `template_variables` property in the File data type enables dynamic file content. Your source file uses Jinja2 syntax (e.g., `{{ db_host }}`), and the plugin substitutes the values at deployment time. This is a common pattern for generating configuration files that depend on other deployment resources.

### Drift Detection

Both node types support `check_drift` operations. Drift detection compares the current state of the managed resource (file on disk or files on FTP) with the expected state defined in the blueprint. This is used by NativeEdge's update and reconciliation workflows to detect and remediate configuration changes made outside of the deployment.
