# AGENTS.md — SR-IOV Network Device Plugin

> This file provides guidance for AI coding agents (and human contributors) working on the SR-IOV Network Device Plugin codebase. It describes the project architecture, conventions, build/test workflows, and key patterns to follow.

## Project Overview

The **SR-IOV Network Device Plugin** is a Kubernetes device plugin that discovers and advertises SR-IOV networking resources (Virtual Functions, Physical Functions, and Auxiliary network devices such as Subfunctions) on Kubernetes nodes. It registers these resources with the Kubelet so that pods can request them via standard Kubernetes resource requests.

- **Language:** Go (see `go.mod` for the current version)
- **License:** Apache 2.0
- **Repository:** `github.com/k8snetworkplumbingwg/sriov-network-device-plugin`
- **Primary branch:** `master`
- **Binary:** `sriovdp` (built from `cmd/sriovdp/`)

## Repository Structure

```
.
├── cmd/sriovdp/              # Main application entry point
│   ├── main.go               # CLI flag parsing, signal handling, startup
│   ├── manager.go            # resourceManager — reads config, discovers devices, manages servers
│   └── manager_test.go
├── pkg/
│   ├── types/                # Core interfaces and type definitions (the "contract")
│   │   ├── types.go          # All interfaces: HostDevice, PciDevice, NetDevice, ResourcePool, etc.
│   │   └── mocks/            # Auto-generated mock implementations (mockery)
│   ├── factory/              # ResourceFactory — creates device providers, resource pools, servers
│   │   └── factory.go
│   ├── devices/              # Device abstraction layer
│   │   ├── api.go            # APIDevice base implementation
│   │   ├── gen_pci.go        # Generic PCI device
│   │   ├── gen_net.go        # Generic network device
│   │   ├── host.go           # Host device implementation
│   │   ├── rdma.go           # RDMA spec implementation
│   │   └── vdpa.go           # vDPA device implementation
│   ├── netdevice/            # SR-IOV network device provider & resource pool
│   │   ├── pciNetDevice.go   # PCI net device implementation
│   │   ├── netDeviceProvider.go
│   │   ├── netResourcePool.go
│   │   └── nadutils.go       # Network Attachment Definition utilities
│   ├── accelerator/          # Accelerator (FPGA, etc.) device provider & resource pool
│   │   ├── accelDevice.go
│   │   ├── accelDeviceProvider.go
│   │   └── accelResourcePool.go
│   ├── auxnetdevice/         # Auxiliary network device (Subfunctions) provider & resource pool
│   │   ├── auxNetDevice.go
│   │   ├── auxNetDeviceProvider.go
│   │   └── auxNetResourcePool.go
│   ├── resources/            # Resource server (gRPC device plugin) and device selectors
│   │   ├── server.go         # gRPC server implementing K8s device plugin API
│   │   ├── deviceSelectors.go
│   │   ├── ddpSelector.go    # DDP (Dynamic Device Personalization) selector
│   │   ├── pKeySelector.go   # InfiniBand partition key selector
│   │   └── pool_stub.go      # Shared resource pool base
│   ├── infoprovider/         # Device info providers for environment variables and device specs
│   │   ├── genericInfoProvider.go
│   │   ├── vfioInfoProvider.go
│   │   ├── uioInfoProvider.go
│   │   ├── rdmaInfoProvider.go
│   │   ├── vdpaInfoProvider.go
│   │   ├── vhostNetInfoProvider.go
│   │   └── extraInfoProvider.go
│   ├── cdi/                  # Container Device Interface (CDI) support
│   │   └── cdi.go
│   └── utils/                # Utility functions and provider abstractions
│       ├── utils.go          # General utility functions
│       ├── ddp.go            # DDP profile utilities
│       ├── netlink_provider.go
│       ├── rdma_provider.go
│       ├── sriovnet_provider.go
│       ├── vdpa_provider.go
│       └── mocks/            # Auto-generated mocks for utility interfaces
├── deployments/              # Kubernetes manifests
│   ├── configMap.yaml        # SR-IOV resource pool configuration
│   ├── sriovdp-daemonset.yaml
│   ├── sriov-crd.yaml        # NetworkAttachmentDefinition CRD
│   └── cdi/                  # CDI-related deployment manifests
├── images/                   # Container image files
│   ├── Dockerfile            # Multi-stage build (Go build + ddptool + Alpine runtime)
│   └── entrypoint.sh
├── docs/                     # Documentation
│   ├── vf-setup.md           # VF creation guide
│   ├── subfunctions/         # Subfunctions documentation
│   ├── dpdk/                 # DPDK setup guides
│   ├── rdma/                 # RDMA configuration
│   ├── vdpa/                 # vDPA documentation
│   ├── ddp/                  # DDP profile documentation
│   ├── bond/                 # Bond device documentation
│   ├── config-file/          # Config file documentation
│   └── udev/                 # udev rules
├── .github/workflows/        # CI/CD pipelines
│   ├── build-test-lint.yml   # Main CI: build, test, lint, e2e
│   ├── codeql.yml            # Security analysis
│   ├── image-push-master.yml # Push image on master merge
│   └── image-push-release.yml
├── Makefile                  # Build, test, lint, image, mock generation
├── go.mod / go.sum           # Go module dependencies
├── .golangci.yml             # Linter configuration (golangci-lint v2)
├── CONTRIBUTING.md           # Contribution guidelines
└── README.md                 # User-facing documentation
```

## Architecture & Key Concepts

### Device Type Hierarchy

The plugin supports three device types (defined in `pkg/types/types.go`):

| DeviceType       | PCI Class | Description |
|------------------|-----------|-------------|
| `netDevice`      | `0x02`    | SR-IOV network VFs and PFs |
| `accelerator`    | `0x12`    | FPGA and other processing accelerators |
| `auxNetDevice`   | `0x02`    | Auxiliary network devices (Subfunctions) |

### Core Interface Chain

The type system follows a layered interface hierarchy defined in `pkg/types/types.go`:

```
HostDevice (base — vendor, driver, deviceID)
  ├── PciDevice (adds PCI address, ACPI index)
  │     ├── PciNetDevice (adds net name, link type, RDMA, VDP, etc.)
  │     └── AccelDevice (FPGA/accelerator specific)
  └── AuxNetDevice (auxiliary/subfunction specific)
```

### Plugin Lifecycle

1. **Configuration** (`cmd/sriovdp/manager.go`): `resourceManager.readConfig()` reads the JSON config from `/etc/pcidp/config.json` (or CLI-specified path).
2. **Discovery** (`resourceManager.discoverHostDevices()`): Uses `ghw` (Go HardWare) library to discover PCI devices, then each `DeviceProvider` filters devices matching the configured selectors.
3. **Server Init** (`resourceManager.initServers()`): Creates `ResourcePool` and `ResourceServer` instances via the `ResourceFactory`.
4. **gRPC Registration** (`pkg/resources/server.go`): Each `ResourceServer` registers with the Kubelet as a device plugin over a Unix socket, implementing `ListAndWatch` and `Allocate` RPCs.
5. **Shutdown**: Handles `SIGHUP`, `SIGINT`, `SIGTERM`, `SIGQUIT` for graceful shutdown.

### Factory Pattern

`pkg/factory/factory.go` implements `ResourceFactory` — a central factory that creates all major components:

- `GetDeviceProvider()` → returns device-type-specific provider
- `GetResourcePool()` → returns the appropriate resource pool
- `GetResourceServer()` → creates the gRPC device plugin server
- `GetSelector()` → returns device selectors for filtering
- `GetDefaultInfoProvider()` → returns device info providers (VFIO, UIO, RDMA, vDPA, etc.)

### Selector System

Device filtering uses a composable selector pattern (`pkg/resources/deviceSelectors.go`):

- **Common selectors**: vendor, device ID, driver, PCI address
- **Net-specific selectors**: pfName, rootDevice, linkType, RDMA, vDPA, DDP profile, pKey, ACPI index, vhostNet
- **Accelerator selectors**: vendor, device, driver, PCI address
- **Auxiliary selectors**: vendor, driver, auxType, pfName, rootDevice, linkType, RDMA, ACPI index, vhostNet

### Info Providers

`pkg/infoprovider/` contains implementations that populate `DeviceSpec`, environment variables, and mount points for allocated devices:

- `genericInfoProvider` — basic PCI device info
- `vfioInfoProvider` — VFIO device passthrough
- `uioInfoProvider` — UIO device passthrough
- `rdmaInfoProvider` — RDMA device info
- `vdpaInfoProvider` — vDPA device info
- `vhostNetInfoProvider` — vhost-net sharing
- `extraInfoProvider` — user-defined additional info

### Container Device Interface (CDI)

CDI support (`pkg/cdi/cdi.go`) provides an alternative mechanism to expose devices to containers using the CDI specification. Enabled via the `--use-cdi` CLI flag.

## Build & Development

### Prerequisites

- **Go**: See `go.mod` for the required version (currently Go 1.25.x)
- **Docker/Podman**: For building container images
- **hwdata**: Required for running tests (`sudo apt-get install hwdata`)

### Common Commands

```bash
# Build the binary
make build

# Run all tests
make test

# Run tests with race detector
make test-race

# Run tests with coverage
make test-coverage

# Run linter (golangci-lint v2)
make lint

# Fix lint issues automatically
make lint-fix

# Build Docker image
make image

# Generate mocks (after interface changes)
make generate-mocks

# Update dependencies
make deps-update

# Clean build artifacts
make clean
```

### Build Output

- Binary: `build/sriovdp`
- Docker image: `ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin:latest`

### Static Build

```bash
make build STATIC=1
```

## Testing

### Test Framework

- **Ginkgo v2** + **Gomega** for BDD-style tests
- **Mockery** for generating mock implementations
- **testify** for assertions in some test files

### Test Organization

Every package has corresponding `*_test.go` files alongside the source. Test suites use Ginkgo's `RunSpecs`:

```
pkg/<package>/<package>_suite_test.go  # Suite bootstrap
pkg/<package>/*_test.go                # Test specs
```

### Mock Generation

Mocks are generated from interfaces in `pkg/types/types.go`, `pkg/utils/`, and `pkg/cdi/` using `mockery`:

```bash
make generate-mocks
```

Generated mocks live in:
- `pkg/types/mocks/`
- `pkg/utils/mocks/`
- `pkg/cdi/mocks/`

**Important:** After modifying any interface in these packages, regenerate mocks before running tests.

### Running Tests

```bash
# All tests
make test

# With race detection
make test-race

# With coverage report
make test-coverage
# Coverage output: test/coverage/cover.out
```

## CI/CD Pipeline

The main CI workflow (`.github/workflows/build-test-lint.yml`) runs on every push and pull request:

| Job | Description |
|-----|-------------|
| `build` | Compiles the binary with `make build` |
| `test` | Runs `make test-race` (requires `hwdata` package) |
| `test-coverage` | Runs coverage tests and reports to Coveralls |
| `golangci` | Runs `make lint` |
| `shellcheck` | Lints shell scripts |
| `hadolint` | Lints the Dockerfile |
| `go-check` | Verifies `go mod tidy` and `go mod vendor` are clean |
| `sriov-operator-e2e-test` | End-to-end tests using the SR-IOV Network Operator (runs on `sriov`-labeled runners) |

### Additional Workflows

- **CodeQL** (`codeql.yml`): Security and code quality analysis
- **Image Push (master)** (`image-push-master.yml`): Pushes container image on master merges
- **Image Push (release)** (`image-push-release.yml`): Pushes container image on release tags

## Code Style & Conventions

### Linting

The project uses **golangci-lint v2** with an extensive configuration in `.golangci.yml`. Key rules:

- **Line length limit**: 140 characters
- **Function length limit**: 50 statements / 100 lines
- **Cyclomatic complexity limit**: 15
- **Import ordering** (enforced by `gci`):
  1. Standard library
  2. Third-party packages
  3. Project packages (`github.com/k8snetworkplumbingwg/sriov-network-device-plugin`)
- **No magic numbers** in arguments, cases, conditions, and returns
- **Ginkgo linter** enabled — `FIt`/`FDescribe` (focused tests) are forbidden

### Error Handling

- Use `github.com/golang/glog` for logging (not logrus — enforced by `depguard`)
- Use `github.com/pkg/errors` for error wrapping where applicable

### Naming Conventions

- Device types use CamelCase constants: `NetDeviceType`, `AcceleratorType`, `AuxNetDeviceType`
- Interface names describe capability: `HostDevice`, `PciDevice`, `NetDevice`, `DeviceProvider`
- Factory methods follow `Get<Thing>()` pattern: `GetResourceServer()`, `GetDeviceProvider()`
- Test files mirror source files: `foo.go` → `foo_test.go`

### License Header

All `.go` files must include the Apache 2.0 license header:

```go
// Copyright 2018 Intel Corp. All Rights Reserved.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// ...
```

## Configuration

The plugin reads its configuration from a JSON file (default: `/etc/pcidp/config.json`). The schema is:

```json
{
  "resourceList": [
    {
      "resourceName": "resource_name",
      "resourcePrefix": "optional.prefix",
      "deviceType": "netDevice|accelerator|auxNetDevice",
      "excludeTopology": false,
      "selectors": [
        {
          "vendors": ["8086"],
          "devices": ["154c"],
          "drivers": ["iavf"],
          "pfNames": ["enp0s0f0"],
          "pciAddresses": ["0000:01:10.0"],
          "rootDevices": ["0000:01:00.0"],
          "linkTypes": ["ether"],
          "ddpProfiles": ["GTPv1-C/U IPv4/IPv6 Payload"],
          "pKeys": ["0x1"],
          "acpiIndexes": ["196609"],
          "isRdma": false,
          "needVhostNet": false,
          "vdpaType": "vhost|virtio"
        }
      ],
      "additionalInfo": {
        "key": { "infokey": "infovalue" }
      }
    }
  ]
}
```

### CLI Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--config-file` | `/etc/pcidp/config.json` | Path to the JSON config file |
| `--resource-prefix` | `intel.com` | Resource name prefix for extended resources |
| `--use-cdi` | `false` | Enable Container Device Interface mode |

## Key Dependencies

| Dependency | Purpose |
|------------|---------|
| `k8s.io/kubelet` | Kubernetes device plugin and plugin registration APIs |
| `google.golang.org/grpc` | gRPC framework for device plugin server |
| `github.com/jaypipes/ghw` | Hardware discovery (PCI devices) |
| `github.com/jaypipes/pcidb` | PCI device database |
| `github.com/vishvananda/netlink` | Linux netlink interface for network device info |
| `github.com/k8snetworkplumbingwg/sriovnet` | SR-IOV network utility functions |
| `github.com/k8snetworkplumbingwg/govdpa` | vDPA device management |
| `github.com/Mellanox/rdmamap` | RDMA device mapping |
| `github.com/k8snetworkplumbingwg/network-attachment-definition-client` | Multus NetworkAttachmentDefinition types |
| `github.com/container-orchestrated-devices/container-device-interface` | CDI specification |
| `github.com/golang/glog` | Structured logging |
| `github.com/onsi/ginkgo/v2` + `github.com/onsi/gomega` | BDD test framework |
| `github.com/stretchr/testify` | Test assertions and mocking |

## Adding a New Device Type

To add support for a new device type:

1. **Define the type** in `pkg/types/types.go`:
   - Add a new `DeviceType` constant
   - Add the PCI class code to `SupportedDevices`
   - Define a new device interface extending `HostDevice`
   - Define selector struct and any new selector fields

2. **Implement the device** in a new package under `pkg/`:
   - Create `<type>Device.go` implementing the device interface
   - Create `<type>DeviceProvider.go` implementing `DeviceProvider`
   - Create `<type>ResourcePool.go` implementing `ResourcePool`

3. **Register in the factory** (`pkg/factory/factory.go`):
   - Add case handling in `GetDeviceProvider()`
   - Add case handling in `GetResourcePool()`
   - Add case handling in `GetDeviceFilter()`

4. **Add selectors** in `pkg/resources/deviceSelectors.go` if needed

5. **Add tests** for all new code — follow existing patterns with Ginkgo/Gomega

6. **Regenerate mocks**: `make generate-mocks`

## Common Pitfalls

- **Always regenerate mocks** after changing interfaces in `pkg/types/`, `pkg/utils/`, or `pkg/cdi/`.
- **The `hwdata` package** must be installed on the system to run tests (provides PCI device database).
- **Import ordering** is strictly enforced — run `make lint-fix` to auto-fix.
- **Do not use `logrus`** — the `depguard` linter rule will reject it. Use `glog` instead.
- **Focused Ginkgo tests** (`FIt`, `FDescribe`, `FContext`) will fail CI — the `ginkgolinter` forbids them.
- **Vendor directory** must be consistent — run `go mod vendor` if dependencies change and commit the result.
- **Build tags**: The project uses `-tags no_openssl` for builds.
- **Test timeout**: Default test timeout is 15 seconds. Coverage tests use 30 seconds.

## Deployment

The plugin is deployed as a **DaemonSet** on Kubernetes nodes with SR-IOV capable NICs:

1. Create the ConfigMap with resource pool definitions: `kubectl create -f deployments/configMap.yaml`
2. Deploy the DaemonSet: `kubectl create -f deployments/sriovdp-daemonset.yaml`
3. (Optional) Create NetworkAttachmentDefinition CRD for Multus: `kubectl create -f deployments/sriov-crd.yaml`

The plugin works with CNI meta-plugins (**Multus** or **DANM**) and requires a compatible CNI plugin (**SR-IOV CNI** for VFs, **Host Device CNI** for PFs).

## Supported Hardware

The plugin has been tested with (but is not limited to):

- Intel® Ethernet 800/700/500 Series
- Mellanox ConnectX-4/4 Lx/5/5 Ex/6/6 Dx
- Mellanox BlueField-2®
- Broadcom NetXtreme-E Series

Any SR-IOV capable NIC should work if it exposes standard PCI SR-IOV capabilities.
