# NativeEdge Ansible Plugin Documentation

## Overview

The NativeEdge Ansible Plugin enables the execution of Ansible playbooks and automation tasks within NativeEdge deployments. It provides a robust framework for configuration management, application deployment, and infrastructure automation using Ansible's powerful capabilities.

## Plugin Information

- **Package Name**: ansible-plugin
- **Package Version**: 4.1.7.0
- **Executor**: central_deployment_agent

## Key Node Types

### dell.nodes.ansible.Executor

The primary node type for executing Ansible playbooks and managing automation tasks.

#### Properties

- **playbook_config**: Comprehensive Ansible execution configuration
- **installation_source**: Custom Ansible installation source
- **connectivity_test_url**: URL for connectivity verification
- **inventory_proxy_settings**: Proxy configuration for inventory hosts
- **proxy_settings**: Global proxy configuration

## Configuration Properties

### Playbook Configuration

#### Core Execution Settings

- **playbook_path**: Path to main playbook file (site.yaml/main.yaml)
- **playbook_source_path**: URL or path containing playbook files
- **additional_playbook_files**: Additional files to download to playbook directory

#### Environment Configuration

- **ansible_external_venv**: External Python virtual environment path
- **extra_packages**: Python packages to install in controller venv
- **galaxy_collections**: Ansible Galaxy collections to install
- **roles**: Ansible roles to install
- **module_path**: Location for Ansible modules

#### Execution Control

- **run_data**: Variable values for playbook execution
- **tags**: Specific tags to execute
- **auto_tags**: Auto-generate tags from playbook
- **start_at_task**: Start execution from specific task
- **number_of_attempts**: Retry count for failed executions

#### Connection Settings

- **sources**: Inventory sources (YAML or file path)
- **ansible_env_vars**: Environment variables
- **timeout**: Connection timeout in seconds
- **ssh_common_args**: Common SSH arguments
- **ssh_extra_args**: Additional SSH arguments

#### Debugging and Logging

- **debug_level**: Debug verbosity level
- **log_stdout**: Output execution logs
- **log_runner_stdout**: Log all stdout output

## Data Types

### dell.datatypes.ansible.ProxySettings

Proxy configuration for Ansible execution:

```yaml
proxy_settings:
  disable: false
  interface_type: OOB  # OOB or INB
  enable_socks5: false
  auto_resolve: true
  target_id: endpoint-id  # For managed assets
  oxy_id: proxy-id       # For unmanaged endpoints
```

## Blueprint Example Usage

### Basic Ansible Execution

Based on the MQTT Node-Red deployment example:

```yaml
tosca_definitions_version: tosca_simple_yaml_1_3

imports:
  - mqtt_nodered/definitions.yaml

node_templates:
  ansible_docker_install:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_source_path: https://github.com/example/ansible-playbooks.git
      playbook_path: docker-install/site.yaml
      sources:
        vms:
          hosts:
            docker_host:
              ansible_host: { get_attribute: [vm, ip] }
              ansible_user: ubuntu
              ansible_ssh_private_key_file: { get_secret: vm_ssh_key }
      run_data:
        docker_users:
          - ubuntu
        docker_compose_version: "2.20.2"
      galaxy_collections:
        - community.docker
      extra_packages:
        - docker
        - docker-compose
```

### Advanced Configuration with Proxy

```yaml
node_templates:
  ansible_with_proxy:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: configuration/site.yaml
      sources:
        servers:
          hosts:
            web_server:
              ansible_host: { get_attribute: [vm, ip] }
              ansible_port: 22
      inventory_proxy_settings:
        web_server:
          port: 22
          ip: 192.168.1.100
      proxy_settings:
        disable: false
        interface_type: OOB
        target_id: proxy-endpoint-id
      ansible_env_vars:
        ANSIBLE_HOST_KEY_CHECKING: "False"
        ANSIBLE_RETRY_FILES_ENABLED: "False"
      tags:
        - install
        - configure
      number_of_attempts: 3
```

### Multi-Playbook Execution

```yaml
node_templates:
  ansible_stage1:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: stage1/site.yaml
      sources:
        hosts:
          target_host:
            ansible_host: { get_attribute: [vm, ip] }
      run_data:
        stage: 1
        environment: production

  ansible_stage2:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: stage2/site.yaml
      sources:
        hosts:
          target_host:
            ansible_host: { get_attribute: [vm, ip] }
      run_data:
        stage: 2
        depends_on_stage1: true
    relationships:
      - target: ansible_stage1
        type: dell.relationships.depends_on
```

## Lifecycle Operations

### Standard Operations

- **precreate**: Environment preparation and package installation
- **start**: Execute Ansible playbook
- **poststart**: Store Ansible facts and results
- **delete**: Cleanup and environment removal

### Custom Operations

- **reload**: Re-execute playbook with current configuration

## Runtime Properties

The Executor node maintains extensive runtime state:

- **playbook_venv**: Virtual environment location
- **sources**: Inventory configuration
- **connected**: Connectivity test result
- **installed_galaxy_collections**: List of installed collections
- **installed_roles**: List of installed roles
- **VENV_PACKAGE_LIST**: Python packages in virtual environment
- **___COMPLETED_TAGS**: Executed tags tracking

## Workflows

### reload_ansible_playbook

Re-execute Ansible playbook with updated parameters:

```yaml
reload_ansible_playbook:
  node_instance_ids: [ansible-executor-id]
  run_data:
    updated_parameter: new_value
  tags:
    - updated-tag
```

### update_playbook_venv

Update virtual environment with new packages:

```yaml
update_playbook_venv:
  extra_packages:
    - new-package
  galaxy_collections:
    - community.general
  roles:
    - geerlingguy.apache
```

## Integration with Blueprint Examples

### MQTT Node-Red Deployment

The plugin installs Docker and deploys containerized services:

```yaml
# From mqtt_nodered/definitions.yaml
node_templates:
  install_docker:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: playbooks/docker-install.yaml
      sources:
        vms:
          hosts:
            docker_host:
              ansible_host: { get_attribute: [vm, ip] }
      run_data:
        docker_users: [ubuntu]
        docker_compose_version: "2.20.2"

  deploy_containers:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: playbooks/deploy-containers.yaml
      sources:
        vms:
          hosts:
            docker_host:
              ansible_host: { get_attribute: [vm, ip] }
      run_data:
        mqtt_password: { get_secret: mqtt_password }
        nodered_admin_password: { get_secret: nodered_password }
    relationships:
      - target: install_docker
        type: dell.relationships.depends_on
```

## Best Practices

### Playbook Structure

1. **Modular Design**: Separate playbooks by function (install, configure, deploy)
2. **Idempotency**: Ensure playbooks can be safely re-executed
3. **Error Handling**: Use proper error handling and rollback mechanisms
4. **Variable Management**: Use run_data for parameterization

### Environment Management

1. **Virtual Environments**: Use isolated Python environments
2. **Collection Management**: Install only required Galaxy collections
3. **Version Control**: Pin specific versions of collections and roles

### Security Considerations

1. **SSH Keys**: Use key-based authentication
2. **Secret Management**: Store sensitive data in NativeEdge secrets
3. **Network Security**: Configure proper network access controls

## Troubleshooting

### Common Issues

1. **Connection Failures**: Check network connectivity and SSH configuration
2. **Module Not Found**: Verify Galaxy collections and Python packages
3. **Permission Errors**: Ensure proper user permissions on target hosts
4. **Timeout Issues**: Adjust timeout values for long-running operations

### Debug Configuration

```yaml
debug_level: 4  # Verbose debugging
log_stdout: true
ansible_env_vars:
  ANSIBLE_VERBOSITY: 3
  ANSIBLE_DEBUG: "true"
```

## Advanced Features

### Conditional Execution

```yaml
tags:
  - install
  - configure
auto_tags: true  # Auto-generate from playbook
start_at_task: "configure_database"  # Start from specific task
```

### Retry Logic

```yaml
number_of_attempts: 5
timeout: '300'  # 5 minutes
additional_args: '--forks 10'  # Parallel execution
```

### Custom Environment

```yaml
ansible_external_venv: /opt/custom-ansible-venv
extra_packages:
  - boto3
  - requests
galaxy_collections:
  - amazon.aws
  - community.aws
```

## Integration Examples

The Ansible Plugin integrates with other NativeEdge plugins:

- **HZP Edge Plugin**: Configure provisioned VMs
- **Utilities Plugin**: Use file operations and secret management
- **Fabric Plugin**: Execute commands alongside Ansible tasks

See the [MQTT Node-Red example](../blueprint_example/infra-dev-apps/mqtt_nodered/) for complete implementation details.
