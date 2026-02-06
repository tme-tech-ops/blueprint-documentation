# NativeEdge Utilities Plugin Documentation

## Overview

The NativeEdge Utilities Plugin provides a comprehensive collection of utility functions and common operations that support infrastructure automation, configuration management, and application deployment workflows. It serves as a foundational plugin offering essential services used across many deployment scenarios.

## Plugin Information

- **Package Name**: utilities-plugin
- **Package Version**: 3.2.0.0
- **Executor**: central_deployment_agent

## Plugin Modules

The utilities plugin encompasses multiple specialized modules:

- **plugins_files**: File management operations
- **plugins_ftp**: FTP file transfer capabilities
- **plugins_terminal**: SSH terminal connections
- **plugins_rest**: REST API client functionality
- **plugins_secrets**: Secret management operations
- **plugins_resources**: Resource allocation and management
- **plugins_ssh_key**: SSH key generation and management
- **cloudinit**: Cloud-init configuration management
- **configuration**: Configuration loading and management
- **rest**: REST API operations
- **scalelist**: Scaling operations
- **suspend**: System suspension and resume
- **terminal**: Terminal connections
- **iso**: ISO file modification
- **resources**: Resource management

## Key Node Types

### File Management

#### dell.nodes.File

File deployment and management operations:

```yaml
node_templates:
  config_file:
    type: dell.nodes.File
    properties:
      resource_config:
        resource_path: files/app-config.yaml
        file_path: /etc/application/config.yaml
        owner: root:root
        mode: 644
        use_sudo: true
        template_variables:
          database_host: { get_input: db_host }
          api_key: { get_secret: api_key }
```

### SSH Key Management

#### dell.nodes.keys.RSAKey

SSH key generation and management:

```yaml
node_templates:
  vm_ssh_key:
    type: dell.nodes.keys.RSAKey
    properties:
      use_secret_store: true
      resource_config:
        key_name: vm-deployment-key
        algorithm: RSA
        bits: 2048
        comment: Generated for VM deployment
```

### Cloud-init Configuration

#### dell.nodes.CloudInit.CloudConfig

Cloud-init configuration management:

```yaml
node_templates:
  cloud_config:
    type: dell.nodes.CloudInit.CloudConfig
    properties:
      header: '#cloud-config'
      encode_base64: false
      resource_config:
        users:
          - name: ubuntu
            ssh_authorized_keys:
              - { get_attribute: [vm_ssh_key, public_key] }
        packages:
          - docker.io
          - nginx
        runcmd:
          - systemctl enable docker
          - systemctl start docker
```

### Terminal Connections

#### dell.nodes.terminal.Raw

SSH terminal command execution:

```yaml
node_templates:
  terminal_connection:
    type: dell.nodes.terminal.Raw
    properties:
      terminal_auth:
        user: ubuntu
        password: { get_secret: vm_password }
        ip: { get_attribute: [vm, ip] }
        port: 22
        store_logs: true
        promt_check: ['$', '#']
```

### REST API Operations

#### dell.nodes.rest.Requests

REST API client functionality:

```yaml
node_templates:
  api_client:
    type: dell.nodes.rest.Requests
    properties:
      host: api.example.com
      port: 443
      ssl: true
      verify: true
      timeout: 30
      params:
        api_version: v1
        environment: production
```

### FTP Operations

#### dell.nodes.ftp

FTP file transfer operations:

```yaml
node_templates:
  ftp_transfer:
    type: dell.nodes.ftp
    properties:
      resource_config:
        user: ftpuser
        password: { get_secret: ftp_password }
        ip: ftp.example.com
        port: 21
        tls: true
      files:
        config.yaml:
          content: |
            server:
              port: 8080
              host: localhost
```

### Secret Management

#### dell.nodes.secrets.Writer

Secret creation and management:

```yaml
node_templates:
  secret_writer:
    type: dell.nodes.secrets.Writer
    properties:
      entries:
        database_password:
          value: { get_input: db_password }
          description: Database connection password
        api_key:
          value: { get_input: api_key }
          description: External API authentication key
      do_not_delete: false
```

#### dell.nodes.secrets.Reader

Secret retrieval operations:

```yaml
node_templates:
  secret_reader:
    type: dell.nodes.secrets.Reader
    properties:
      keys:
        - database_password
        - api_key
        - ssh_private_key
```

### Resource Management

#### dell.nodes.resources.List

Resource allocation and reservation:

```yaml
node_templates:
  ip_pool:
    type: dell.nodes.resources.List
    properties:
      resource_config:
        - 192.168.1.100
        - 192.168.1.101
        - 192.168.1.102
        - 192.168.1.103

  ip_reservation:
    type: dell.nodes.resources.ListItem
    relationships:
      - target: ip_pool
        type: dell.relationships.resources.reserve_list_item
```

### Configuration Management

#### dell.nodes.ConfigurationLoader

Dynamic configuration loading:

```yaml
node_templates:
  config_loader:
    type: dell.nodes.ConfigurationLoader
    properties:
      parameters_json: |
        {
          "database": {
            "host": "db.example.com",
            "port": 5432
          },
          "cache": {
            "redis_url": "redis://cache.example.com:6379"
          }
        }
```

## Blueprint Example Integration

### Simple VM Example

The utilities plugin supports VM deployment configurations:

```yaml
# From vm/definitions.yaml
node_templates:
  ssh_key:
    type: dell.nodes.keys.RSAKey
    properties:
      use_secret_store: true
      resource_config:
        key_name: simple-vm-key

  cloud_config:
    type: dell.nodes.CloudInit.CloudConfig
    properties:
      resource_config:
        users:
          - name: ubuntu
            ssh_authorized_keys:
              - { get_attribute: [ssh_key, public_key] }
        packages:
          - curl
          - wget
    relationships:
      - target: ssh_key
        type: dell.relationships.depends_on
```

### MQTT Node-Red Example

Configuration management for containerized deployments:

```yaml
# From mqtt_nodered/definitions.yaml
node_templates:
  nodered_config:
    type: dell.nodes.File
    properties:
      resource_config:
        resource_path: mqtt_nodered/settings.js
        file_path: /opt/nodered/settings.js
        owner: root:root
        mode: 644
        template_variables:
          mqtt_host: { get_input: mqtt_host }
          mqtt_port: 1883

  mqtt_config:
    type: dell.nodes.File
    properties:
      resource_config:
        resource_path: mqtt_nodered/mosquitto.conf
        file_path: /etc/mosquitto/mosquitto.conf
        owner: root:root
        mode: 644
```

### NGINX Application Example

SSL certificate and configuration management:

```yaml
# From nginx_app/definitions.yaml
node_templates:
  ssl_cert:
    type: dell.nodes.secrets.Writer
    properties:
      entries:
        ssl_certificate:
          value: { get_input: ssl_cert_content }
          description: SSL certificate for NGINX
        ssl_private_key:
          value: { get_input: ssl_key_content }
          description: SSL private key for NGINX

  nginx_config:
    type: dell.nodes.File
    properties:
      resource_config:
        resource_path: nginx_app/nginx.conf
        file_path: /etc/nginx/sites-available/default
        owner: root:root
        mode: 644
        template_variables:
          server_name: { get_input: server_name }
          ssl_cert_path: /etc/ssl/certs/nginx.crt
          ssl_key_path: /etc/ssl/private/nginx.key
```

## Advanced Features

### ISO Modification

#### dell.nodes.resources.ModifiedIso

ISO file customization:

```yaml
node_templates:
  custom_iso:
    type: dell.nodes.resources.ModifiedIso
    properties:
      iso_path: /path/to/original.iso
      output_iso_path: /path/to/custom.iso
      new_files:
        - iso_path: /cloud-init/
          file_context: |
            #cloud-config
            hostname: custom-vm
          rr_name: user-data
        - iso_path: /scripts/
          file_context: |
            #!/bin/bash
            echo "Custom initialization script"
          rr_name: setup.sh
          file_mode: 755
```

### REST API Batching

#### dell.nodes.rest.BunchRequests

Multiple REST API operations:

```yaml
node_templates:
  api_operations:
    type: dell.nodes.rest.BunchRequests
    properties:
      host: api.example.com
      port: 443
      ssl: true
    interfaces:
      dell.interfaces.lifecycle:
        create:
          inputs:
            templates:
              create_user:
                method: POST
                path: /api/v1/users
                body:
                  username: newuser
                  email: user@example.com
              get_token:
                method: POST
                path: /api/v1/auth
                body:
                  username: admin
                  password: { get_secret: admin_password }
```

## Workflows

### Configuration Update

Dynamic configuration updates:

```yaml
configuration_update:
  mapping: configuration.plugins_configuration.tasks.update
  description: Updates and merges configuration values
  parameters:
    params: '{"database": {"host": "new-db.example.com"}}'
    configuration_node_id: config_loader
    merge_dict: true
```

### System Operations

System management workflows:

```yaml
suspend:
  mapping: suspend.plugins_suspend.workflows.suspend
  description: Pauses deployment execution

resume:
  mapping: suspend.plugins_suspend.workflows.resume
  description: Resumes execution of suspended workflow

backup:
  mapping: suspend.plugins_suspend.workflows.backup
  description: Creates system backup
  parameters:
    snapshot_name: deployment-backup
    snapshot_incremental: true
```

## Best Practices

### Secret Management

1. **Centralized Storage**: Use NativeEdge secret store for sensitive data
2. **Minimal Exposure**: Limit secret scope to required components
3. **Regular Rotation**: Implement secret rotation policies
4. **Access Control**: Configure proper secret access permissions

### File Operations

1. **Template Usage**: Use Jinja templates for dynamic configuration
2. **Permission Management**: Set appropriate file permissions and ownership
3. **Validation**: Validate file content before deployment
4. **Backup Strategy**: Maintain backups of critical configuration files

### Resource Management

1. **Pool Management**: Use resource pools for IP addresses and other scarce resources
2. **Reservation Cleanup**: Implement proper resource cleanup on deletion
3. **Capacity Planning**: Monitor resource utilization and plan for scaling
4. **Conflict Resolution**: Handle resource conflicts gracefully

## Integration Patterns

### with HZP Edge Plugin

```yaml
node_templates:
  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: { get_input: service_tag }
        name: application-vm

  vm_config:
    type: dell.nodes.File
    properties:
      resource_config:
        resource_path: files/app-config.yaml
        file_path: /etc/application/config.yaml
    relationships:
      - target: vm
        type: dell.relationships.depends_on
```

### with Ansible Plugin

```yaml
node_templates:
  ansible_vars:
    type: dell.nodes.secrets.Writer
    properties:
      entries:
        ansible_vars:
          value: |
            database_host: db.example.com
            api_key: secret-key
          description: Ansible variables

  ansible_executor:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: deploy.yaml
      run_data: { get_attribute: [ansible_vars, data, ansible_vars] }
    relationships:
      - target: ansible_vars
        type: dell.relationships.depends_on
```

## Troubleshooting

### Common Issues

1. **File Permission Errors**: Check user permissions and sudo configuration
2. **Secret Access**: Verify secret permissions and existence
3. **Resource Exhaustion**: Monitor resource pool availability
4. **Template Rendering**: Validate template syntax and variables

### Debug Configuration

```yaml
properties:
  debug_mode: true
  log_level: DEBUG
  preserve_temp_files: true
  validate_operations: true
```

## Use Cases

### Infrastructure Provisioning

- SSH key generation for VM access
- Cloud-init configuration for automated setup
- Configuration file deployment
- Resource allocation and management

### Application Deployment

- Dynamic configuration generation
- Secret management for applications
- API integration and configuration
- File transfer and setup

### System Administration

- Configuration management
- Backup and restore operations
- Resource monitoring and allocation
- System maintenance tasks

The Utilities Plugin provides the foundational building blocks for complex deployment scenarios and is extensively used throughout the blueprint examples to support various automation requirements.
