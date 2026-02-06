# NativeEdge Plugin Documentation

This documentation provides comprehensive information about the NativeEdge plugins available in this repository, including their capabilities, node types, and practical examples derived from working blueprints.

## Overview

NativeEdge plugins extend the platform's capabilities by providing specialized functionality for infrastructure deployment, configuration management, and application orchestration. Each plugin offers unique node types and operations that can be combined to create powerful automation workflows.

## Plugin Documentation

- [HZP Edge Plugin](./hzp-edge-plugin.md) - Virtual machine management and deployment
- [NativeEdge Ansible Plugin](./nativeedge-ansible-plugin.md) - Ansible playbook execution and configuration management
- [NativeEdge Fabric Plugin](./nativeedge-fabric-plugin.md) - Remote command execution and deployment
- [NativeEdge Utilities Plugin](./nativeedge-utilities-plugin.md) - General utility functions and common operations

## Blueprint Examples

The [blueprint examples](../blueprint_example/infra-dev-apps/) demonstrate practical implementations of these plugins in real-world scenarios:

- **Simple VM Deployment** - Basic Ubuntu VM creation
- **MQTT Node-Red Deployment** - Containerized IoT platform
- **NGINX Application** - Web server with SSL and authentication
- **Container-based Deployment** - Bare-metal container orchestration
- **Remote Host Deployment** - Third-party host management

## Getting Started

1. Review the plugin documentation to understand available node types and capabilities
2. Examine the blueprint examples for implementation patterns
3. Combine plugins as needed to create comprehensive automation solutions
4. Refer to the [blueprint README](../blueprint_example/infra-dev-apps/README.md) for detailed setup instructions

## Prerequisites

- NativeEdge Orchestrator version 1.0.0.0 or higher
- Onboarded NativeEdge Endpoints (for VM/container deployments)
- Appropriate secrets and credentials configured in Orchestrator
- Network connectivity for package installation and container downloads
