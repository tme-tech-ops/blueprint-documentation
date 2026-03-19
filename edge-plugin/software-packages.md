# Software Packages

## Prerequisites

Before working with software packages you should be familiar with the general plugin concepts described in the [Edge Plugin Overview](overview.md).

## Overview

The edge plugin can download software packages (binaries, container images, manifests, and other artifacts) to the NativeEdge orchestrator's local repository. Once downloaded, these packages are available for use by other deployment workflows, scripts, or services running on the orchestrator or its connected endpoints.

Two node types handle software package downloads:

| Node Type | Description |
|---|---|
| `dell.nodes.nativeedge.template.SoftwarePackages` | Downloads a list of software packages to the orchestrator repository. |
| `dell.nodes.nativeedge.template.OxyServerSoftwarePackages` | Extends `SoftwarePackages` with Oxy Server-specific metadata. Downloads packages that are required by an Oxy Server proxy. |

---

## SoftwarePackages

**Full type name:** `dell.nodes.nativeedge.template.SoftwarePackages`
**Derived from:** `dell.nodes.Root`

### Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `software_packages` | `dell.datatypes.SoftwarePackages` | Yes | Wrapper containing the list of software package configurations to download. |

### Lifecycle Operations

| Operation | Implementation | Description |
|---|---|---|
| `create` | `edge.nativeedge_plugin.tasks.download_software` | Downloads all specified software packages to the orchestrator repository. The download URLs are resolved, credentials are applied where needed, and each package is fetched and stored locally. |

---

## OxyServerSoftwarePackages

**Full type name:** `dell.nodes.nativeedge.template.OxyServerSoftwarePackages`
**Derived from:** `dell.nodes.Root`

This node type is functionally identical to `SoftwarePackages` but accepts additional Oxy Server metadata. An **Oxy Server** is a NativeEdge proxy component that relays communication between the orchestrator and endpoints. Software packages downloaded through this node type are associated with a specific Oxy Server instance.

### Properties

| Property | Type | Required | Description |
|---|---|---|---|
| `oxy_server_packages` | `dell.datatypes.OxyServerSoftwarePackages` | Yes | Wrapper containing the list of software packages plus Oxy Server identification. |

### Lifecycle Operations

| Operation | Implementation | Description |
|---|---|---|
| `create` | `edge.nativeedge_plugin.tasks.download_software` | Same implementation as `SoftwarePackages`. Downloads all packages, associating them with the specified Oxy Server. |

---

## Data Types

### SoftwarePackages

**Full type name:** `dell.datatypes.SoftwarePackages`

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `software_packages` | list of `dell.datatypes.SoftwarePackageConfig` | Yes | `[]` | The list of software packages to download. |

### OxyServerSoftwarePackages

**Full type name:** `dell.datatypes.OxyServerSoftwarePackages`
**Derived from:** `dell.datatypes.SoftwarePackages`

Inherits the `software_packages` property and adds Oxy Server identification.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `software_packages` | list of `dell.datatypes.SoftwarePackageConfig` | Yes | `[]` | (Inherited) The list of software packages to download. |
| `oxy_server_id` | string | No | `""` | Unique identifier of the Oxy Server instance. |
| `oxy_server_url` | string | No | `""` | URL of the Oxy Server instance. |

### SoftwarePackageConfig

**Full type name:** `dell.datatypes.SoftwarePackageConfig`

Each entry in the `software_packages` list describes one package to download.

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `package_name` | string | Yes | -- | Human-readable name of the software package (e.g., `nginx`, `kustomize`). Not required to be unique on its own. |
| `unique_key` | string | Yes | -- | Unique key for this package. Convention is `<package_name>_<version>` (e.g., `nginx_0.1`). This key is used to retrieve the download result URL from node capabilities. |
| `type` | string | Yes | -- | The original package type. Common values: `tarball`, `image`, `manifest`. |
| `url` | string | Yes | -- | Full download URL for the package. Examples: `https://github.com/.../kustomize_v5.0.3_linux_amd64.tar.gz` for a binary, `ghcr.io/dexidp/dex:v2.36.0` for a Docker image. Set to `uploaded` for packages that are already pre-uploaded to the orchestrator -- the plugin will query the catalog service for the actual path. |
| `version` | string | No | `""` | Package version. For Docker images this is typically the image tag; for binaries it is the release version. |
| `secret_key` | string | No | `""` | Name of a NativeEdge secret containing credentials for the external repository. For binary files, use a secret of type "Binary Configuration" (the plugin reads the `user` and `token` fields). For Docker images, use a secret of type "Docker Configuration". One secret can be shared across multiple packages. |

---

## Blueprint Example

```yaml
tosca_definitions_version: dell_1_0

imports:
  - dell/types/types.yaml
  - plugin:edge-plugin

node_templates:

  packages:
    type: dell.nodes.nativeedge.template.SoftwarePackages
    properties:
      software_packages:
        software_packages:
          - package_name: kustomize
            unique_key: kustomize_5.0.3
            type: tarball
            url: "https://github.com/kubernetes-sigs/kustomize/releases/download/kustomize%2Fv5.0.3/kustomize_v5.0.3_linux_amd64.tar.gz"
            version: "5.0.3"
          - package_name: dex
            unique_key: dex_2.36.0
            type: image
            url: "ghcr.io/dexidp/dex:v2.36.0"
            version: "v2.36.0"
```

### Explanation

1. The `packages` node downloads two software packages:
   - **kustomize** -- a tarball binary downloaded from GitHub.
   - **dex** -- a Docker image pulled from the GitHub Container Registry.
2. Each package has a `unique_key` that can later be used to retrieve the local download path from the node's capabilities.
3. No `secret_key` is needed because both sources are publicly accessible.

---

## Blueprint Example -- Oxy Server

```yaml
node_templates:

  oxy_packages:
    type: dell.nodes.nativeedge.template.OxyServerSoftwarePackages
    properties:
      oxy_server_packages:
        oxy_server_id: "oxy-server-001"
        oxy_server_url: "https://oxy.example.com"
        software_packages:
          - package_name: agent-binary
            unique_key: agent-binary_1.2.0
            type: tarball
            url: "https://artifacts.example.com/agent-1.2.0.tar.gz"
            version: "1.2.0"
            secret_key: external_repo_creds
```

### Explanation

- `oxy_server_id` and `oxy_server_url` identify the Oxy Server this download is associated with.
- The `secret_key` field references a NativeEdge secret named `external_repo_creds` that contains credentials for the private artifact repository.
