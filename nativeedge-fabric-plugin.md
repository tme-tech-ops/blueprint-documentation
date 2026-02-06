# NativeEdge Fabric Plugin Documentation

## Overview

The NativeEdge Fabric Plugin provides remote command execution capabilities, enabling blueprints to run commands on remote hosts, VMs, and containers. It serves as a bridge between NativeEdge orchestration and target systems, supporting both on-premises and cloud-based deployments.

## Plugin Information

- **Package Name**: fabric-plugin
- **Package Version**: 3.3.3.0
- **Executor**: central_deployment_agent

## Key Capabilities

The Fabric Plugin specializes in:

- Remote command execution on target systems
- Script deployment and execution
- File transfer operations
- SSH-based connectivity with key and password authentication
- Proxy support for network-restricted environments

## Blueprint Example Usage

### NGINX Application Deployment

Based on `nginx_app_example_for_NED.yaml`, the plugin demonstrates web server deployment:

```yaml
tosca_definitions_version: tosca_simple_yaml_1_3

imports:
  - nginx_app/definitions.yaml

node_templates:
  nginx_deployment:
    type: dell.nodes.fabric.RemoteExecutor
    properties:
      host_config:
        host: { get_attribute: [vm, ip] }
        user: ubuntu
        private_key: { get_secret: vm_ssh_key }
      commands:
        - name: install_nginx
          command: |
            sudo apt-get update
            sudo apt-get install -y nginx apache2-utils
        - name: configure_nginx
          command: |
            sudo systemctl enable nginx
            sudo systemctl start nginx
        - name: setup_ssl
          command: |
            sudo mkdir -p /etc/nginx/ssl
            # SSL configuration commands
```

### Remote Host Deployment

Based on `remote_host_example.yaml`, demonstrates deployment to external hosts:

```yaml
node_templates:
  remote_nginx:
    type: dell.nodes.fabric.RemoteExecutor
    properties:
      host_config:
        host: { get_input: remote_host_ip }
        user: ubuntu
        private_key: { get_secret: remote_host_private_key }
        sudo_password: { get_secret: vm_password }
      commands:
        - name: update_system
          command: sudo apt-get update && sudo apt-get upgrade -y
        - name: install_packages
          command: sudo apt-get install -y nginx certbot python3-certbot-nginx
        - name: configure_firewall
          command: sudo ufw allow 'Nginx Full'
```

## Configuration Patterns

### Basic Command Execution

```yaml
node_templates:
  simple_command:
    type: dell.nodes.fabric.RemoteExecutor
    properties:
      host_config:
        host: target-host-ip
        user: ubuntu
        private_key: { get_secret: ssh_key }
      commands:
        - name: check_status
          command: systemctl status nginx
        - name: restart_service
          command: sudo systemctl restart nginx
```

### Script Execution

```yaml
node_templates:
  script_runner:
    type: dell.nodes.fabric.RemoteExecutor
    properties:
      host_config:
        host: { get_attribute: [vm, ip] }
        user: ubuntu
        private_key: { get_secret: vm_ssh_key }
      script_path: scripts/deploy.sh
      script_args:
        - --environment=production
        - --port=8080
      working_directory: /opt/application
```

### File Operations

```yaml
node_templates:
  file_manager:
    type: dell.nodes.fabric.RemoteExecutor
    properties:
      host_config:
        host: target-host
        user: ubuntu
        private_key: { get_secret: ssh_key }
      file_operations:
        - action: upload
          local_path: files/config.yaml
          remote_path: /etc/application/config.yaml
        - action: download
          remote_path: /var/log/application.log
          local_path: logs/application.log
```

## Integration Examples

### with HZP Edge Plugin

Deploy and configure VMs:

```yaml
node_templates:
  web_vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: { get_input: service_tag }
        name: web-server
        image: ubuntu-22.04
        resource_constraints:
          cpu: 2
          memory: 4GB

  web_server_config:
    type: dell.nodes.fabric.RemoteExecutor
    properties:
      host_config:
        host: { get_attribute: [web_vm, ip] }
        user: ubuntu
        private_key: { get_secret: vm_ssh_key }
      commands:
        - name: install_web_server
          command: sudo apt-get install -y apache2
        - name: deploy_website
          command: sudo cp /tmp/website/* /var/www/html/
    relationships:
      - target: web_vm
        type: dell.relationships.depends_on
```

### with Ansible Plugin

Hybrid automation approach:

```yaml
node_templates:
  ansible_setup:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: setup.yaml
      sources:
        hosts:
          target:
            ansible_host: { get_attribute: [vm, ip] }

  fabric_config:
    type: dell.nodes.fabric.RemoteExecutor
    properties:
      host_config:
        host: { get_attribute: [vm, ip] }
        user: ubuntu
        private_key: { get_secret: vm_ssh_key }
      commands:
        - name: final_config
          command: sudo systemctl restart application
    relationships:
      - target: ansible_setup
        type: dell.relationships.depends_on
```

## Authentication Methods

### SSH Key Authentication

```yaml
host_config:
  host: target-host
  user: ubuntu
  private_key: { get_secret: ssh_private_key }
  port: 22
```

### Password Authentication

```yaml
host_config:
  host: target-host
  user: ubuntu
  password: { get_secret: user_password }
  sudo_password: { get_secret: sudo_password }
```

### Certificate-based Authentication

```yaml
host_config:
  host: target-host
  user: ubuntu
  certificate: { get_secret: ssh_certificate }
  key: { get_secret: ssh_key }
```

## Advanced Configuration

### Connection Settings

```yaml
host_config:
  host: target-host
  user: ubuntu
  private_key: { get_secret: ssh_key }
  connection_timeout: 300
  command_timeout: 600
  retry_count: 3
  retry_delay: 10
```

### Environment Variables

```yaml
properties:
  host_config:
    host: target-host
    user: ubuntu
    private_key: { get_secret: ssh_key }
  environment:
    DEPLOYMENT_ENV: production
    LOG_LEVEL: INFO
    DATABASE_URL: { get_secret: database_url }
```

### Proxy Configuration

```yaml
host_config:
  host: target-host
  user: ubuntu
  private_key: { get_secret: ssh_key }
  proxy:
    host: proxy-server
    port: 8080
    username: proxy_user
    password: { get_secret: proxy_password }
```

## Error Handling and Validation

### Command Validation

```yaml
commands:
  - name: check_service
    command: systemctl status nginx
    validate_output: true
    expected_output: "active (running)"
  - name: restart_if_needed
    command: sudo systemctl restart nginx
    condition: "'inactive' in previous_output"
```

### Error Recovery

```yaml
commands:
  - name: attempt_install
    command: sudo apt-get install -y package
    on_failure: retry
    max_retries: 3
    retry_delay: 30
  - name: fallback_install
    command: sudo dpkg -i package.deb
    condition: "previous_command_failed"
```

## Best Practices

### Security

1. **Key Management**: Use NativeEdge secrets for SSH keys and passwords
2. **Least Privilege**: Configure users with minimal required permissions
3. **Network Security**: Use VPN or dedicated management networks
4. **Audit Logging**: Enable comprehensive logging for compliance

### Performance

1. **Batch Operations**: Group related commands to reduce connection overhead
2. **Parallel Execution**: Use multiple nodes for concurrent operations
3. **Connection Reuse**: Maintain persistent connections where possible
4. **Resource Monitoring**: Monitor target system resource usage

### Reliability

1. **Idempotent Commands**: Design commands that can be safely re-executed
2. **Rollback Procedures**: Include rollback commands for critical operations
3. **Health Checks**: Verify system state after operations
4. **Error Notifications**: Configure alerts for failed operations

## Troubleshooting

### Common Issues

1. **Connection Failures**: Verify network connectivity and authentication
2. **Permission Errors**: Check user permissions and sudo configuration
3. **Command Timeouts**: Adjust timeout values for long-running operations
4. **Resource Exhaustion**: Monitor target system resources

### Debug Configuration

```yaml
properties:
  debug_mode: true
  log_level: DEBUG
  capture_output: true
  preserve_temp_files: true
```

## Use Cases

### Application Deployment

- Install and configure web servers
- Deploy application code
- Configure load balancers
- Set up monitoring

### System Administration

- System updates and patching
- User management
- Service configuration
- Log management

### DevOps Automation

- CI/CD pipeline integration
- Infrastructure provisioning
- Configuration management
- Backup and restore operations

## Integration with Blueprint Examples

The Fabric Plugin is extensively used in the provided examples:

1. **NGINX Application**: Deploys and configures NGINX with SSL
2. **Remote Host**: Manages external host configuration
3. **MQTT Node-Red**: Supports container deployment operations

See the [blueprint examples](../blueprint_example/infra-dev-apps/) for complete implementation details and working configurations.
