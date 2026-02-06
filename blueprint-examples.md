# Blueprint Examples Documentation

## Overview

The blueprint examples demonstrate practical implementations of NativeEdge plugins in real-world scenarios. These examples showcase various deployment patterns, integration techniques, and best practices for infrastructure automation and application deployment.

## Example Deployments

### 1. Simple VM Deployment (`simple_vm_example_for_NED.yaml`)

**Purpose**: Basic Ubuntu VM creation with minimal user input

**Key Features**:
- Ubuntu 22.04 cloud-init VM deployment
- SSH key generation and management
- Basic network configuration
- Cloud-init user setup

**Plugins Used**:
- HZP Edge Plugin (VM creation)
- Utilities Plugin (SSH key generation, Cloud-init configuration)

**Node Types Highlighted**:
- `dell.nodes.nativeedge.template.NativeEdgeVM`
- `dell.nodes.keys.RSAKey`
- `dell.nodes.CloudInit.CloudConfig`

**Example Configuration**:
```yaml
node_templates:
  ssh_key:
    type: dell.nodes.keys.RSAKey
    properties:
      use_secret_store: true
      resource_config:
        key_name: simple-vm-key

  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: { get_input: service_tag }
        name: { get_input: vm_name }
        image: { get_input: os_image }
        os_type: linux
        resource_constraints:
          cpu: 2
          memory: 4GB
          storage: 40GB
        network_settings:
          - name: VNIC1
            segment_name: { get_input: network_segment }
        ssh_keys: { get_attribute: [ssh_key, public_key] }
```

### 2. MQTT Node-Red Deployment (`mqtt_nodered_example_for_NED.yaml`)

**Purpose**: Containerized IoT platform deployment

**Key Features**:
- Docker installation and configuration
- MQTT broker deployment
- Node-Red flow automation platform
- Container orchestration via Docker Compose
- SSL/TLS configuration

**Plugins Used**:
- HZP Edge Plugin (VM creation)
- Ansible Plugin (Docker installation, container deployment)
- Utilities Plugin (Configuration management, secrets)

**Node Types Highlighted**:
- `dell.nodes.nativeedge.template.NativeEdgeVM`
- `dell.nodes.ansible.Executor`
- `dell.nodes.File`
- `dell.nodes.secrets.Writer`

**Example Configuration**:
```yaml
node_templates:
  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: { get_input: service_tag }
        name: mqtt-nodered-vm
        # ... VM configuration

  install_docker:
    type: dell.nodes.ansible.Executor
    properties:
      playbook_path: playbooks/docker-install.yaml
      sources:
        vms:
          hosts:
            docker_host:
              ansible_host: { get_attribute: [vm, ip] }
              ansible_user: ubuntu
              ansible_ssh_private_key_file: { get_secret: vm_ssh_key }
      run_data:
        docker_users: [ubuntu]
        docker_compose_version: "2.20.2"
      galaxy_collections:
        - community.docker
    relationships:
      - target: vm
        type: dell.relationships.depends_on

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

### 3. NGINX Application Deployment (`nginx_app_example_for_NED.yaml`)

**Purpose**: Web server deployment with SSL and authentication

**Key Features**:
- NGINX web server installation
- SSL/TLS certificate configuration
- Basic authentication setup
- Static file serving
- Git repository cloning

**Plugins Used**:
- HZP Edge Plugin (VM creation)
- Fabric Plugin (Command execution)
- Utilities Plugin (File management, secrets)

**Node Types Highlighted**:
- `dell.nodes.nativeedge.template.NativeEdgeVM`
- `dell.nodes.fabric.RemoteExecutor`
- `dell.nodes.File`
- `dell.nodes.secrets.Writer`

**Example Configuration**:
```yaml
node_templates:
  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: { get_input: service_tag }
        name: nginx-app-vm
        # ... VM configuration

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
        - name: setup_ssl
          command: |
            sudo mkdir -p /etc/nginx/ssl
            echo "{ get_secret: ssl_cert_content }" | sudo tee /etc/nginx/ssl/cert.pem
            echo "{ get_secret: ssl_key_content }" | sudo tee /etc/nginx/ssl/key.pem
        - name: configure_nginx
          command: |
            sudo systemctl enable nginx
            sudo systemctl start nginx
    relationships:
      - target: vm
        type: dell.relationships.depends_on
```

### 4. Container-based Deployment (`container_based_example_for_NED.yaml`)

**Purpose**: Bare-metal container deployment on NativeEdge Endpoint

**Key Features**:
- Direct container deployment on endpoint
- No VM overhead
- NativeEdge Compose orchestration
- Multi-container application stack

**Plugins Used**:
- HZP Edge Plugin (NativeEdge Compose)
- Utilities Plugin (Configuration management)

**Node Types Highlighted**:
- `dell.nodes.nativeedge.template.NativeEdgeCompose`
- `dell.nodes.File`

**Example Configuration**:
```yaml
node_templates:
  compose_deployment:
    type: dell.nodes.nativeedge.template.NativeEdgeCompose
    properties:
      compose_config:
        service_tag: { get_input: service_tag }
        definition:
          name: mfg-tools
          compose: |
            version: '3.8'
            services:
              nodered:
                image: nodered/node-red:latest
                ports:
                  - "1880:1880"
                environment:
                  - NODE_RED_ENABLE_PROJECTS=true
              influxdb:
                image: influxdb:2.0
                ports:
                  - "8086:8086"
                environment:
                  - DOCKER_INFLUXDB_INIT_MODE=setup
                  - DOCKER_INFLUXDB_INIT_USERNAME=admin
                  - DOCKER_INFLUXDB_INIT_PASSWORD={ get_secret: influxdb_password }
              grafana:
                image: grafana/grafana:latest
                ports:
                  - "3000:3000"
                environment:
                  - GF_SECURITY_ADMIN_PASSWORD={ get_secret: grafana_password }
```

### 5. Remote Host Deployment (`remote_host_example.yaml`)

**Purpose**: Application deployment on external/third-party hosts

**Key Features**:
- Remote host management
- External system integration
- SSH key authentication
- No NativeEdge Endpoint requirement

**Plugins Used**:
- Fabric Plugin (Remote command execution)
- Utilities Plugin (SSH key management, secrets)

**Node Types Highlighted**:
- `dell.nodes.fabric.RemoteExecutor`
- `dell.nodes.keys.RSAKey`
- `dell.nodes.secrets.Writer`

**Example Configuration**:
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
        - name: setup_nginx
          command: |
            sudo systemctl enable nginx
            sudo systemctl start nginx
```

## Common Patterns

### 1. VM Creation Pattern

All VM-based deployments follow this pattern:

1. **SSH Key Generation**: Create SSH keys for authentication
2. **VM Provisioning**: Deploy VM with specified configuration
3. **Cloud-init Configuration**: Apply initial system configuration
4. **Application Deployment**: Install and configure applications

### 2. Authentication Pattern

Consistent authentication approach across examples:

```yaml
# SSH Key Management
ssh_key:
  type: dell.nodes.keys.RSAKey
  properties:
    use_secret_store: true
    resource_config:
      key_name: deployment-key

# SSH Key Usage in VMs
vm:
  type: dell.nodes.nativeedge.template.NativeEdgeVM
  properties:
    ssh_keys: { get_attribute: [ssh_key, public_key] }

# SSH Key Usage in Remote Connections
remote_executor:
  type: dell.nodes.fabric.RemoteExecutor
  properties:
    host_config:
      private_key: { get_secret: vm_ssh_key }
```

### 3. Secret Management Pattern

Consistent secret usage for sensitive data:

```yaml
# Secret Creation
secrets:
  type: dell.nodes.secrets.Writer
  properties:
    entries:
      password:
        value: { get_input: plain_password }
        description: Application password

# Secret Usage
application_config:
  type: dell.nodes.File
  properties:
    template_variables:
      app_password: { get_secret: app_password }
```

### 4. Network Configuration Pattern

Flexible network configuration options:

```yaml
# Bridge Network
network_settings:
  - name: VNIC1
    segment_name: bridge-network

# NAT Network with Port Forwarding
network_settings:
  - name: VNIC1
    segment_name: nat-network
    port_fwd_rules:
      - service_type: HTTP
        protocol: TCP
        host_ip: 192.168.1.100
        host_port: 8080
        vm_port: 80
```

## Integration Examples

### Multi-Plugin Integration

The MQTT Node-Red example demonstrates complex plugin integration:

1. **HZP Edge Plugin**: Creates the base VM
2. **Utilities Plugin**: Manages SSH keys and secrets
3. **Ansible Plugin**: Installs Docker and deploys containers

### Dependency Management

Proper relationship management ensures correct execution order:

```yaml
node_templates:
  vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM

  ssh_key:
    type: dell.nodes.keys.RSAKey

  cloud_config:
    type: dell.nodes.CloudInit.CloudConfig
    relationships:
      - target: ssh_key
        type: dell.relationships.depends_on

  application:
    type: dell.nodes.ansible.Executor
    relationships:
      - target: vm
        type: dell.relationships.depends_on
      - target: cloud_config
        type: dell.relationships.depends_on
```

## Best Practices Demonstrated

### 1. Resource Management

- **IP Allocation**: Use resource pools for IP address management
- **SSH Key Reuse**: Generate keys once and reuse across deployments
- **Secret Storage**: Store sensitive data in NativeEdge secret store

### 2. Configuration Management

- **Template Usage**: Use Jinja templates for dynamic configuration
- **Environment Variables**: Pass configuration through environment variables
- **Validation**: Validate inputs and configurations

### 3. Error Handling

- **Retry Logic**: Implement retry mechanisms for network operations
- **Health Checks**: Verify service availability after deployment
- **Rollback**: Include rollback procedures for critical operations

### 4. Security

- **Key-based Authentication**: Use SSH keys instead of passwords
- **Least Privilege**: Configure minimal required permissions
- **Network Segmentation**: Use appropriate network configurations

## Deployment Prerequisites

### Common Requirements

1. **NativeEdge Orchestrator**: Version 1.0.0.0 or higher
2. **Network Connectivity**: Internet access for package installation
3. **Secret Configuration**: Required secrets created in Orchestrator

### VM-based Deployments

1. **NativeEdge Endpoint**: Onboarded endpoint with virtual network
2. **OS Images**: Ubuntu 22.04 cloud-init image
3. **Storage**: Sufficient storage for VM disks

### Container Deployments

1. **Container Registry**: Access to Docker registry or local registry
2. **Resource Allocation**: Sufficient CPU and memory on endpoint

### Remote Host Deployments

1. **SSH Access**: SSH keys configured on remote hosts
2. **Sudo Access**: Appropriate sudo permissions
3. **Network Connectivity**: Network access from Orchestrator

## Capabilities and Outputs

### Standard VM Outputs

- Service Tag
- VM hostname
- VM IP address
- VM name
- SSH private key secret name
- Username

### Application-specific Outputs

- **MQTT Node-Red**: MQTT endpoint, Node-Red URL
- **NGINX**: NGINX URL with SSL configuration
- **Container Deployment**: Service URLs for each container
- **Remote Host**: Application URLs and access information

## Customization Guidelines

### VM Customization

Modify `vm/inputs.yaml` to enable hardware passthrough:

```yaml
gpu_passthrough:
  type: list
  default: []
  hidden: true  # Change to false to enable

usb_passthrough:
  type: list
  default: []
  hidden: true  # Change to false to enable
```

### Application Customization

Update playbooks and configurations in respective directories:

- `mqtt_nodered/playbooks/`: Ansible playbooks for container deployment
- `nginx_app/files/`: Configuration files for NGINX
- `container_based/compose/`: Docker Compose configurations

### Network Customization

Configure network segments and port forwarding rules in VM configurations:

```yaml
network_settings:
  - name: VNIC1
    segment_name: custom-network
    port_fwd_rules:
      - service_type: CUSTOM
        protocol: TCP
        host_port: 9090
        vm_port: 8080
```

These blueprint examples provide comprehensive templates for various deployment scenarios and can be customized to meet specific requirements while following established best practices and patterns.
