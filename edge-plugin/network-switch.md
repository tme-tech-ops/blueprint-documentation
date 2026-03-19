# Network Switch

## Prerequisites

Before working with network switch management you should be familiar with the general plugin concepts described in the [Edge Plugin Overview](overview.md).

## Overview

The `NetworkSwitch` node type allows you to create or update (upsert) a network switch record in the NativeEdge inventory. This is useful for tracking the physical network infrastructure connected to your edge endpoints -- switches, their interfaces, VLANs, hardware status, and location metadata.

The "upsert" behavior means that if a switch record with the same identifier already exists, it will be updated; otherwise a new record is created.

---

## Node Type

**Full type name:** `dell.nodes.nativeedge.template.NetworkSwitch`
**Derived from:** `dell.nodes.Root`

### Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `network_switch` | `dell.types.nativeedge.NetworkSwitch` | Yes | The network switch configuration payload. |

### Lifecycle Operations

| Operation | Implementation | Description |
|---|---|---|
| `create` | `edge.nativeedge_plugin.tasks.upsert_network_switch` | Creates or updates the network switch record in the NativeEdge inventory. |

---

## Data Types

### NetworkSwitch

**Full type name:** `dell.types.nativeedge.NetworkSwitch`

Top-level data type representing a network switch and all its associated metadata.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `service_tag` | string | No | -- | Dell service tag of the switch (if applicable). |
| `fqdn` | string | No | -- | Fully qualified domain name of the switch. |
| `ip_address` | string | No | -- | Management IP address of the switch. |
| `vendor` | string | No | -- | Switch manufacturer (e.g., `Dell`, `Cisco`). |
| `model` | string | No | -- | Switch model name or number. |
| `serial_number` | string | No | -- | Hardware serial number. |
| `mac_address` | string | No | -- | Primary MAC address of the switch. |
| `uptime_seconds` | integer | No | -- | Switch uptime in seconds. |
| `status` | string | No | -- | Current operational status (e.g., `active`, `unreachable`). |
| `extra` | `dell.types.nativeedge.SwitchExtra` | No | -- | Extended switch information including provisioning state and hardware summary. |
| `metadata` | dict | No | -- | Arbitrary key-value metadata. |
| `environment` | string | No | -- | Environment label (e.g., `production`, `staging`). |

### SwitchExtra

**Full type name:** `dell.types.nativeedge.SwitchExtra`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `provisioned_state` | string | No | -- | Current provisioning state of the switch (e.g., `provisioned`, `unprovisioned`). |
| `summary` | `dell.types.nativeedge.SwitchExtraSummary` | No | -- | Detailed hardware and network summary. |

### SwitchExtraSummary

**Full type name:** `dell.types.nativeedge.SwitchExtraSummary`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `location` | `dell.types.nativeedge.Location` | No | -- | Physical location of the switch. |
| `hardware` | `dell.types.nativeedge.Hardware` | No | -- | Hardware specification and component status. |
| `interfaces` | list of `dell.types.nativeedge.Interface` | No | -- | Network interfaces on the switch. |
| `vlans` | list of `dell.types.nativeedge.VLAN` | No | -- | VLANs configured on the switch. |

### Location

**Full type name:** `dell.types.nativeedge.Location`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `datacenter` | string | No | -- | Name of the datacenter where the switch is located. |
| `rack` | string | No | -- | Rack identifier within the datacenter. |
| `position` | string | No | -- | Position within the rack (e.g., `U12`). |

### Hardware

**Full type name:** `dell.types.nativeedge.Hardware`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `cpu_model` | string | No | -- | CPU model installed in the switch. |
| `cpu_count` | integer | No | -- | Number of CPUs. |
| `total_memory_mb` | integer | No | -- | Total memory in megabytes. |
| `fans` | list of `dell.types.nativeedge.Fan` | No | -- | Fan status and speed information. |
| `power_supplies` | list of `dell.types.nativeedge.PowerSupply` | No | -- | Power supply status and capacity. |
| `temperature` | `dell.types.nativeedge.Temperature` | No | -- | Temperature readings and thresholds. |

### Fan

**Full type name:** `dell.types.nativeedge.Fan`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | string | No | -- | Fan identifier (e.g., `Fan-1`). |
| `status` | string | No | -- | Operational status (e.g., `ok`, `failed`). |
| `speed_rpm` | integer | No | -- | Current fan speed in revolutions per minute. |

### PowerSupply

**Full type name:** `dell.types.nativeedge.PowerSupply`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `id` | string | No | -- | Power supply identifier (e.g., `PSU-1`). |
| `status` | string | No | -- | Operational status (e.g., `ok`, `failed`). |
| `capacity_watts` | integer | No | -- | Maximum power capacity in watts. |

### Temperature

**Full type name:** `dell.types.nativeedge.Temperature`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `status` | string | No | -- | Temperature status (e.g., `ok`, `warning`, `critical`). |
| `current_celsius` | float | No | -- | Current temperature reading in degrees Celsius. |
| `threshold_celsius` | float | No | -- | Temperature threshold above which warnings are generated. |

### Interface

**Full type name:** `dell.types.nativeedge.Interface`

Represents a single network interface (port) on the switch.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | string | No | -- | Interface name (e.g., `Ethernet1/1`, `GigabitEthernet0/1`). |
| `mac_address` | string | No | -- | MAC address of the interface. |
| `admin_status` | string | No | -- | Administrative status: `up` or `down`. |
| `oper_status` | string | No | -- | Operational status: `up` or `down`. |
| `speed_mbps` | integer | No | -- | Link speed in megabits per second. |
| `duplex` | string | No | -- | Duplex mode (e.g., `full`, `half`). |
| `description` | string | No | -- | User-defined description for the interface. |
| `vlans` | list of `dell.types.nativeedge.VLANAssignment` | No | -- | VLANs assigned to this interface. |

### VLANAssignment

**Full type name:** `dell.types.nativeedge.VLANAssignment`

Represents a VLAN assignment on a specific interface.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `vlan_id` | integer | No | -- | Numeric VLAN identifier (1-4094). |
| `name` | string | No | -- | Human-readable VLAN name. |
| `mode` | string | No | -- | Port VLAN mode (e.g., `access`, `trunk`, `native`). |

### VLAN

**Full type name:** `dell.types.nativeedge.VLAN`

Represents a VLAN configured on the switch.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `vlan_id` | integer | No | -- | Numeric VLAN identifier (1-4094). |
| `name` | string | No | -- | Human-readable VLAN name. |
| `status` | string | No | -- | VLAN status (e.g., `active`, `suspended`). |
| `interfaces` | list of string | No | -- | List of interface names that are members of this VLAN. |
| `ip_interface` | `dell.types.nativeedge.IPInterface` | No | -- | Layer 3 IP interface associated with this VLAN (SVI). |

### IPInterface

**Full type name:** `dell.types.nativeedge.IPInterface`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `ip_address` | string | No | -- | IP address assigned to the VLAN interface. |
| `subnet_mask` | string | No | -- | Subnet mask for the IP address. |

---

## Blueprint Example

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin

node_templates:

  top_of_rack_switch:
    type: dell.nodes.nativeedge.template.NetworkSwitch
    properties:
      network_switch:
        service_tag: "SWITCH001"
        fqdn: "tor-switch-01.dc1.example.com"
        ip_address: "10.0.1.1"
        vendor: "Dell"
        model: "PowerSwitch S5248F-ON"
        serial_number: "SN123456789"
        mac_address: "00:11:22:33:44:55"
        uptime_seconds: 8640000
        status: "active"
        environment: "production"
        metadata:
          managed_by: "nativeedge"
          firmware_version: "10.5.4.2"
        extra:
          provisioned_state: "provisioned"
          summary:
            location:
              datacenter: "DC-East-1"
              rack: "Rack-A12"
              position: "U42"
            hardware:
              cpu_model: "ARM Cortex-A72"
              cpu_count: 4
              total_memory_mb: 8192
              fans:
                - id: "Fan-1"
                  status: "ok"
                  speed_rpm: 5400
                - id: "Fan-2"
                  status: "ok"
                  speed_rpm: 5350
              power_supplies:
                - id: "PSU-1"
                  status: "ok"
                  capacity_watts: 460
                - id: "PSU-2"
                  status: "ok"
                  capacity_watts: 460
              temperature:
                status: "ok"
                current_celsius: 38.5
                threshold_celsius: 75.0
            interfaces:
              - name: "Ethernet1/1"
                mac_address: "00:11:22:33:44:56"
                admin_status: "up"
                oper_status: "up"
                speed_mbps: 25000
                duplex: "full"
                description: "Uplink to spine"
                vlans:
                  - vlan_id: 100
                    name: "management"
                    mode: "trunk"
              - name: "Ethernet1/2"
                mac_address: "00:11:22:33:44:57"
                admin_status: "up"
                oper_status: "up"
                speed_mbps: 10000
                duplex: "full"
                description: "Server port 1"
                vlans:
                  - vlan_id: 200
                    name: "workload"
                    mode: "access"
            vlans:
              - vlan_id: 100
                name: "management"
                status: "active"
                interfaces:
                  - "Ethernet1/1"
                ip_interface:
                  ip_address: "10.0.1.1"
                  subnet_mask: "255.255.255.0"
              - vlan_id: 200
                name: "workload"
                status: "active"
                interfaces:
                  - "Ethernet1/2"
```

### Explanation

1. **top_of_rack_switch** -- Registers (or updates) a network switch record in NativeEdge:
   - Basic identity: `service_tag`, `fqdn`, `ip_address`, `vendor`, `model`, `serial_number`, `mac_address`.
   - Status and uptime are reported for monitoring.
   - `metadata` holds arbitrary key-value data (here, firmware version and management tool).
   - `extra.summary.location` describes the physical position in the datacenter.
   - `extra.summary.hardware` includes CPU, memory, fan status, power supply status, and temperature.
   - `extra.summary.interfaces` lists physical ports with their link status, speed, and VLAN assignments.
   - `extra.summary.vlans` lists all VLANs on the switch, their member interfaces, and any associated IP interfaces (SVIs).
2. The `create` operation performs an **upsert**: if a switch with `service_tag: "SWITCH001"` already exists, its record is updated; otherwise a new record is created.
