# HZP Edge Plugin Documentation

## Overview

The HZP Edge Plugin provides comprehensive virtual machine management capabilities for NativeEdge Endpoints. It enables the creation, configuration, and lifecycle management of VMs with advanced networking, storage, and hardware passthrough options.

## Plugin Information

- **Package Name**: edge-plugin
- **Package Version**: 3.3.22.0
- **Executor**: central_deployment_agent

## Key Node Types

### dell.nodes.nativeedge.template.NativeEdgeVM

The primary node type for VM deployment and management.

#### Properties

- **vm_config** (VMDeployDefinition): Core VM deployment configuration
- **config** (Config): Legacy API configuration (optional)

#### Key Capabilities

- VM creation from cloud images or external sources
- Network configuration with multiple NICs
- Hardware passthrough (GPU, USB, serial, PCIe)
- Cloud-init support for automated configuration
- Port forwarding rules for NAT networks
- Storage management with multiple disks

### dell.nodes.nativeedge.template.ApplicationVM

Extended VM node type for application-specific deployments.

#### Additional Properties

- **app_vm_config** (AppVMDeploy): Application VM specific settings

### dell.nodes.nativeedge.template.NativeEdgeCompose

Container orchestration node type for deploying Docker Compose applications directly on NativeEdge Endpoints.

#### Properties

- **compose_config** (NativeEdgeComposeConfig): Compose deployment configuration

## Data Types

### VMDeployDefinition

Comprehensive VM configuration including:

- **location**: ECE service tag for deployment target
- **name**: VM identifier
- **image**: OS image reference
- **resource_constraints**: CPU, memory, storage specifications
- **network_settings**: Network interface configurations
- **hardware_options**: vTPM, secure boot, firmware settings
- **ports**: Passthrough device configurations
- **cloudinit**: Cloud-init configuration or script

### VMDeployNetworkSettings

Network interface configuration:

- **name**: VNIC identifier (VNIC1-10)
- **segment_name**: Network segment assignment
- **vm_mac**: MAC address specification
- **port_fwd_rules**: Port forwarding configurations

### VMDeployPorts

Hardware passthrough configuration:

- **usb**: USB device passthrough list
- **serial_port**: Serial port configuration
- **gpu**: GPU passthrough devices
- **video**: Video passthrough options
- **pcie**: PCIe device passthrough

## Blueprint Example Usage

### Simple VM Deployment

Based on `simple_vm_example_for_NED.yaml`:

```yaml
tosca_definitions_version: tosca_simple_yaml_1_3

imports:
  - vm/definitions.yaml

node_templates:
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
        ssh_keys: { get_secret: vm_ssh_key }
```

### VM with Hardware Passthrough

Advanced configuration with GPU and USB passthrough:

```yaml
node_templates:
  gpu_vm:
    type: dell.nodes.nativeedge.template.NativeEdgeVM
    properties:
      vm_config:
        location: { get_input: service_tag }
        name: gpu-enabled-vm
        image: { get_input: os_image }
        resource_constraints:
          cpu: 4
          memory: 8GB
          storage: 100GB
        ports:
          gpu: ['NVIDIA Corporation L4']
          usb: ['USB-1']
          serial_port:
            - port: 'COM-1'
              mode: 'RS-232'
```

### Container Deployment

NativeEdge Compose deployment:

```yaml
node_templates:
  compose_deployment:
    type: dell.nodes.nativeedge.template.NativeEdgeCompose
    properties:
      compose_config:
        service_tag: { get_input: service_tag }
        definition:
          name: iot-platform
          compose: |
            version: '3.8'
            services:
              nodered:
                image: nodered/node-red:latest
                ports:
                  - "1880:1880"
              influxdb:
                image: influxdb:2.0
                ports:
                  - "8086:8086"
```

## Lifecycle Operations

The plugin supports comprehensive VM lifecycle management:

- **precreate**: Preparation and validation
- **create**: VM provisioning and configuration
- **start**: VM startup and initialization
- **poststart**: Post-startup configuration
- **stop**: VM shutdown
- **restart**: VM restart operations
- **suspend**: VM suspension
- **resume**: VM resume from suspension
- **check_drift**: Configuration drift detection
- **update_config**: Configuration updates
- **update_apply**: Apply configuration changes
- **delete**: VM removal and cleanup

## Network Configuration

### Bridge vs NAT Networks

- **Bridge**: Direct network access with MAC address
- **NAT**: Port-forwarded access through host IP

### Port Forwarding Example

```yaml
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

## Storage Management

### Main Disk Configuration

```yaml
main_disk:
  name: main_disk
  disconnect_flag: false
```

### Additional Disks

```yaml
additional_disks:
  - image:
      filename: data-disk.img
      name: data_disk
      controller: SCSI
    storage: 100GB
    disk: datastore1
```

## Best Practices

1. **Resource Planning**: Allocate appropriate CPU, memory, and storage based on workload requirements
2. **Network Design**: Use bridge networks for direct access, NAT for security
3. **Hardware Passthrough**: Validate device compatibility before configuration
4. **Cloud-init**: Use cloud-init for automated VM configuration
5. **Security**: Implement SSH key authentication and proper network segmentation

## Troubleshooting

Common issues and solutions:

- **VM Creation Failures**: Verify image accessibility and endpoint resources
- **Network Connectivity**: Check segment configuration and port forwarding rules
- **Hardware Passthrough**: Ensure devices are available and properly configured
- **Performance**: Monitor resource utilization and adjust constraints accordingly

## Integration Examples

The HZP Edge Plugin integrates seamlessly with other plugins:

- **Ansible Plugin**: Use Ansible playbooks for post-deployment configuration
- **Utilities Plugin**: Leverage file operations and secret management
- **Fabric Plugin**: Execute commands on provisioned VMs

See the [blueprint examples](../blueprint_example/infra-dev-apps/) for complete implementation examples.
