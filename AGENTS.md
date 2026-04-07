# AGENTS.md — SR-IOV Network Device Plugin

This file provides guidance for AI coding agents (and human contributors) working
on the **sriov-network-device-plugin** codebase. It describes the project layout,
architecture, build system, testing strategy, coding conventions, and CI
expectations so that any agent can orient quickly and make correct, consistent
changes.

---

## Table of Contents

- [Project Overview](#project-overview)
- [Repository Layout](#repository-layout)
- [Architecture](#architecture)
- [Key Interfaces & Types](#key-interfaces--types)
- [Build & Development](#build--development)
- [Testing](#testing)
- [Linting & Code Style](#linting--code-style)
- [Mock Generation](#mock-generation)
- [Docker Image](#docker-image)
- [CI / GitHub Actions](#ci--github-actions)
- [Configuration](#configuration)
- [Deployment](#deployment)
- [Contributing Conventions](#contributing-conventions)
- [Common Pitfalls](#common-pitfalls)

---

## Project Overview

The SR-IOV Network Device Plugin is a **Kubernetes device plugin** that discovers
and advertises networking resources on a host, including:

- **SR-IOV Virtual Functions (VFs)**
- **PCI Physical Functions (PFs)**
- **Auxiliary network devices** (Subfunctions / SFs)

It implements the Kubernetes Device Plugin API (`k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1`)
and registers discovered devices with the kubelet so that pods can request them
via extended resources (e.g., `intel.com/intel_sriov_netdevice`).

The plugin supports devices with both **kernel** and **userspace** (UIO / VFIO)
drivers, RDMA devices, vDPA devices, DDP profiles, and Container Device
Interface (CDI) integration.

**Language:** Go (see `go.mod` for the exact version — currently Go 1.25.x)
**License:** Apache 2.0

---

## Repository Layout

```
.
├── cmd/
│   └── sriovdp/              # Main application entry point
│       ├── main.go           # CLI flag parsing, signal handling, startup
│       ├── manager.go        # resourceManager — orchestrates config, discovery, servers
│       └── manager_test.go
│
├── pkg/                      # Core library packages
│   ├── types/                # Shared interfaces and type definitions
│   │   ├── types.go          # ALL core interfaces (HostDevice, ResourcePool, etc.)
│   │   └── mocks/            # Auto-generated mockery mocks for types
│   │
│   ├── factory/              # ResourceFactory — creates providers, pools, servers
│   │   ├── factory.go
│   │   └── factory_test.go
│   │
│   ├── resources/            # ResourcePool and ResourceServer (gRPC device plugin)
│   │   ├── server.go         # gRPC server implementing DevicePlugin API
│   │   ├── deviceSelectors.go
│   │   ├── ddpSelector.go
│   │   ├── pKeySelector.go
│   │   └── pool_stub.go
│   │
│   ├── netdevice/            # Network device provider (SR-IOV VFs, PFs)
│   │   ├── pciNetDevice.go   # PciNetDevice implementation
│   │   ├── netDeviceProvider.go
│   │   ├── netResourcePool.go
│   │   └── nadutils.go       # Network-Attachment-Definition utilities
│   │
│   ├── accelerator/          # Accelerator device provider (FPGAs, etc.)
│   │   ├── accelDevice.go
│   │   ├── accelDeviceProvider.go
│   │   └── accelResourcePool.go
│   │
│   ├── auxnetdevice/         # Auxiliary network device provider (Subfunctions)
│   │   ├── auxNetDevice.go
│   │   ├── auxNetDeviceProvider.go
│   │   └── auxNetResourcePool.go
│   │
│   ├── devices/              # Low-level device abstractions
│   │   ├── gen_pci.go        # Generic PCI device
│   │   ├── gen_net.go        # Generic network device
│   │   ├── host.go           # Host device implementation
│   │   ├── rdma.go           # RDMA spec implementation
│   │   ├── vdpa.go           # vDPA device implementation
│   │   └── api.go            # API device (k8s device plugin API adapter)
│   │
│   ├── infoprovider/         # DeviceInfoProvider implementations
│   │   ├── genericInfoProvider.go
│   │   ├── vfioInfoProvider.go
│   │   ├── uioInfoProvider.go
│   │   ├── rdmaInfoProvider.go
│   │   ├── vdpaInfoProvider.go
│   │   ├── vhostNetInfoProvider.go
│   │   └── extraInfoProvider.go
│   │
│   ├── cdi/                  # Container Device Interface support
│   │   ├── cdi.go
│   │   ├── cdi_test.go
│   │   └── mocks/
│   │
│   └── utils/                # Shared utility functions & provider abstractions
│       ├── utils.go          # General utilities (DetectPluginWatchMode, etc.)
│       ├── ddp.go            # DDP (Dynamic Device Personalization) helpers
│       ├── netlink_provider.go
│       ├── rdma_provider.go
│       ├── sriovnet_provider.go
│       ├── vdpa_provider.go
│       └── mocks/            # Auto-generated mocks for utils interfaces
│
├── deployments/              # Kubernetes deployment manifests
│   ├── sriovdp-daemonset.yaml
│   ├── configMap.yaml        # Example ConfigMap with device selectors
│   ├── cdi/                  # CDI-specific deployment manifests
│   └── ...
│
├── docs/                     # Extended documentation & example configs
│   ├── dpdk/                 # DPDK deployment examples
│   ├── rdma/                 # RDMA deployment examples
│   ├── vdpa/                 # vDPA deployment examples
│   ├── ddp/                  # DDP deployment examples
│   ├── bond/                 # Bond device examples
│   └── subfunctions/         # Subfunctions documentation
│
├── images/                   # Container image build files
│   ├── Dockerfile            # Multi-stage build (builder → ddp-builder → runtime)
│   └── entrypoint.sh
│
├── .github/
│   └── workflows/
│       ├── build-test-lint.yml      # PR CI: build, test, lint, e2e
│       ├── image-push-master.yml    # Push multi-arch image on merge to master
│       ├── image-push-release.yml   # Push image on release tags
│       └── codeql.yml               # CodeQL security analysis
│
├── Makefile                  # Build, test, lint, image, mock generation
├── go.mod / go.sum           # Go module definition
├── .golangci.yml             # Linter configuration (golangci-lint v2)
├── CONTRIBUTING.md           # Contribution guidelines
├── README.md                 # User-facing documentation
└── LICENSE                   # Apache 2.0
```

---

## Architecture

The plugin follows a **provider / factory / pool / server** architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                      resourceManager                         │
│  (cmd/sriovdp/manager.go)                                    │
│                                                              │
│  1. Reads config.json (ResourceConfList)                     │
│  2. Uses DeviceProviders to discover host devices            │
│  3. Creates ResourcePools via ResourceFactory                │
│  4. Starts ResourceServers (gRPC) for each pool              │
└──────────┬──────────────────────────────────┬────────────────┘
           │                                  │
    ┌──────▼──────┐                   ┌───────▼───────┐
    │DeviceProvider│                  │ResourceFactory │
    │ (per type)   │                  │(pkg/factory)   │
    │              │                  │                │
    │- netDevice   │                  │Creates:        │
    │- accelerator │                  │- ResourcePool  │
    │- auxNetDevice│                  │- ResourceServer│
    └──────┬───────┘                  │- InfoProviders │
           │                          │- Selectors     │
    ┌──────▼──────┐                   └───────┬────────┘
    │  HostDevice  │                          │
    │ (discovered) │                  ┌───────▼────────┐
    │              │                  │ ResourceServer  │
    │ PciNetDevice │                  │ (gRPC)          │
    │ AccelDevice  │                  │                 │
    │ AuxNetDevice │                  │ Implements:     │
    └──────────────┘                  │ - ListAndWatch  │
                                      │ - Allocate      │
                                      │ - GetDeviceInfo │
                                      └─────────────────┘
```

### Key Flow

1. **Config** → `resourceManager.readConfig()` reads `/etc/pcidp/config.json`
   containing a `ResourceConfList` (list of `ResourceConfig` entries).
2. **Discovery** → Each `DeviceProvider` scans the host's PCI topology (via
   `ghw`) and collects matching devices.
3. **Filtering** → Devices are filtered by selectors (vendor, device ID, driver,
   PF name, PCI address, link type, DDP profile, vDPA type, etc.).
4. **Pool Creation** → A `ResourcePool` is created for each resource config,
   holding the filtered set of devices.
5. **Server Start** → A `ResourceServer` (gRPC) registers with kubelet and
   serves `ListAndWatch` / `Allocate` RPCs for its pool.

### Device Type Hierarchy

| Device Type      | Package            | Key Struct          | Description                      |
|------------------|--------------------|---------------------|----------------------------------|
| `netDevice`      | `pkg/netdevice`    | `pciNetDevice`      | SR-IOV VFs / PFs (network)       |
| `accelerator`    | `pkg/accelerator`  | `accelDevice`       | FPGA / AI accelerators           |
| `auxNetDevice`   | `pkg/auxnetdevice` | `auxNetDevice`      | Auxiliary devices (Subfunctions) |

---

## Key Interfaces & Types

All core interfaces are defined in **`pkg/types/types.go`**. This is the single
source of truth for the plugin's abstraction layer.

| Interface           | Purpose                                                       |
|---------------------|---------------------------------------------------------------|
| `HostDevice`        | Base device interface — vendor, driver, device ID             |
| `PciDevice`         | Extends HostDevice with PCI address, ACPI index               |
| `NetDevice`         | Extends HostDevice with PF name, link type, RDMA, vDPA        |
| `PciNetDevice`      | Combines PciDevice + NetDevice                                |
| `AccelDevice`       | PCI accelerator device                                        |
| `AuxNetDevice`      | Auxiliary (subfunction) network device                        |
| `APIDevice`         | Exposes device specs, env vars, mounts to k8s API             |
| `ResourcePool`      | Manages a set of devices as a k8s extended resource            |
| `ResourceServer`    | gRPC server implementing the Device Plugin API                |
| `ResourceFactory`   | Factory for creating pools, servers, providers, selectors      |
| `DeviceProvider`    | Discovers and filters devices of a specific type               |
| `DeviceSelector`    | Filters devices based on selector criteria                    |
| `DeviceInfoProvider`| Provides device-specific info (VFIO, UIO, RDMA, vDPA, etc.)  |
| `RdmaSpec`          | RDMA device specification                                     |
| `VdpaDevice`        | vDPA device abstraction                                       |
| `NadUtils`          | Network-Attachment-Definition utilities                       |

**Important:** When adding a new device type or capability, start by reviewing
and potentially extending the interfaces in `pkg/types/types.go`.

---

## Build & Development

### Prerequisites

- **Go** — version specified in `go.mod` (currently 1.25.x)
- **Make** — GNU Make
- **Docker / Podman** — for building container images
- **hwdata** — `hwdata-pci` package (needed for tests; provides PCI ID database)

### Common Make Targets

| Command                | Description                                           |
|------------------------|-------------------------------------------------------|
| `make`                 | Run lint, build, and test (default target: `all`)     |
| `make build`           | Build the `sriovdp` binary to `build/sriovdp`         |
| `make test`            | Run unit tests                                        |
| `make test-race`       | Run tests with Go race detector                       |
| `make test-coverage`   | Run tests with coverage → `test/coverage/cover.out`   |
| `make lint`            | Run golangci-lint (installs if not present)            |
| `make lint-fix`        | Run golangci-lint with `--fix`                        |
| `make image`           | Build Docker image (`ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin`) |
| `make generate-mocks`  | Regenerate all mockery mocks                          |
| `make deps-update`     | Run `go mod tidy`                                     |
| `make clean`           | Remove build artifacts, caches, and test output       |

### Build Flags

- `STATIC=1` — Build a statically linked binary (`CGO_ENABLED=0`)
- `HTTP_PROXY` / `HTTPS_PROXY` — Passed to Docker build as build args
- `TAG` — Override the Docker image tag (default: `ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin`)
- `DOCKERFILE` — Override the Dockerfile path (default: `images/Dockerfile`)

### Building the Binary

```bash
# Standard build
make build

# Static build (no CGO)
STATIC=1 make build

# Binary output location
ls -la build/sriovdp
```

---

## Testing

### Test Framework

- **Ginkgo v2** (`github.com/onsi/ginkgo/v2`) — BDD-style test framework
- **Gomega** (`github.com/onsi/gomega`) — Matcher library
- **testify** (`github.com/stretchr/testify`) — Assertions (used in some tests)
- **mockery** — Mock generation for interfaces

### Running Tests

```bash
# Run all tests
make test

# Run with race detector
make test-race

# Run with coverage
make test-coverage
# Coverage output: test/coverage/cover.out

# Run a specific package's tests
go test -v ./pkg/netdevice/...

# Run specific tests by name
go test -v -run TestNetDeviceProvider ./pkg/netdevice/...
```

### Test Conventions

- Each package has a `*_suite_test.go` file that registers the Ginkgo suite.
- Test files are named `*_test.go` alongside the source files.
- Tests use **mocks** from `pkg/types/mocks/`, `pkg/utils/mocks/`, and
  `pkg/cdi/mocks/` — these are auto-generated by mockery.
- The `testing.go` files in `pkg/utils/` and `pkg/resources/` contain shared
  test helpers — these files are **excluded from linting** (see `.golangci.yml`).
- Tests requiring PCI hardware data depend on the `hwdata` system package.

### Test Timeout

Default test timeout is **15 seconds** (configurable via `TIMEOUT` in the
Makefile). Coverage tests have a **30-second** timeout.

---

## Linting & Code Style

### Linter

The project uses **golangci-lint v2** configured via `.golangci.yml`.

```bash
# Run linting
make lint

# Auto-fix lint issues
make lint-fix
```

### Key Lint Rules

- **Line length limit:** 140 characters (`lll`)
- **Function length:** max 100 lines / 50 statements (`funlen`)
- **Cyclomatic complexity:** max 15 (`gocyclo`)
- **Magic numbers:** checked in arguments, cases, conditions, returns (`mnd`)
- **Duplicate code:** threshold of 100 tokens (`dupl`)
- **Import ordering** (via `gci` formatter):
  1. Standard library
  2. Third-party packages
  3. Project packages (`github.com/k8snetworkplumbingwg/sriov-network-device-plugin`)
- **Ginkgo linter:** `forbid-focus-container: true` — no `FDescribe`/`FIt`
  allowed in committed code.
- **Spelling:** US English locale (`misspell`)

### Excluded from Linting

- `_test.go` files are exempt from: `dupl`, `goconst`, `lll`, `gosec`
- Paths excluded entirely: `.github/*`, `deployments/*`, `docs/*`,
  `pkg/utils/testing.go`, `pkg/resources/testing.go`

### Code Formatting

The project enforces formatting with:
- `gofmt` — Standard Go formatting
- `goimports` — Import organization
- `gci` — Import grouping and ordering

---

## Mock Generation

Mocks are generated using **mockery** and live alongside the interfaces they
mock.

```bash
# Regenerate all mocks
make generate-mocks
```

This generates mocks for:
- `pkg/types/` → `pkg/types/mocks/`
- `pkg/utils/` → `pkg/utils/mocks/`
- `pkg/cdi/` → `pkg/cdi/mocks/`

**Important:** If you modify any interface in `pkg/types/types.go`,
`pkg/utils/`, or `pkg/cdi/`, you **must** regenerate mocks before committing.

Both legacy mockery files (e.g., `PciDevice.go`) and newer mock files
(e.g., `mock_PciDevice.go`) may exist — the project is in transition. Use the
`mock_*.go` files for new test code.

---

## Docker Image

The container image uses a **multi-stage build** defined in `images/Dockerfile`:

1. **Builder stage** (`golang:1.25-alpine`) — Compiles the `sriovdp` binary
2. **DDP builder stage** (`golang:1.20-alpine3.16`) — Builds the `ddptool`
   utility for DDP (Dynamic Device Personalization) support
3. **Runtime stage** (`alpine:3`) — Minimal image with `hwdata-pci`, the binary,
   and the entrypoint script

### Multi-Architecture Support

Images are built for: `linux/amd64`, `linux/arm64`, `linux/ppc64le`, `linux/s390x`

```bash
# Build locally (single arch)
make image

# The CI builds multi-arch on merge to master
```

---

## CI / GitHub Actions

All workflows are in `.github/workflows/`:

### On Push & Pull Request (`build-test-lint.yml`)

| Job                          | Description                                           |
|------------------------------|-------------------------------------------------------|
| `build`                      | Compiles the binary (`make build`)                    |
| `test`                       | Runs tests with race detector (`make test-race`)      |
| `test-coverage`              | Runs coverage tests, reports to Coveralls             |
| `golangci`                   | Runs golangci-lint (`make lint`)                      |
| `shellcheck`                 | Lints shell scripts                                   |
| `hadolint`                   | Lints the Dockerfile                                  |
| `go-check`                   | Verifies `go mod tidy` and `go mod vendor` are clean  |
| `sriov-operator-e2e-test`    | E2E tests via the sriov-network-operator (on `sriov` runner) |

### On Merge to Master (`image-push-master.yml`)

Builds and pushes a multi-architecture Docker image to
`ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin` tagged as `latest`
and with the commit SHA.

### On Release Tags (`image-push-release.yml`)

Builds and pushes release-tagged images.

### CodeQL (`codeql.yml`)

Runs GitHub CodeQL security analysis.

### What Must Pass Before Merge

- All `build-test-lint.yml` jobs must pass
- `go mod tidy` and `go mod vendor` must produce no diff
- No focused Ginkgo tests (`FDescribe`, `FIt`, etc.)

---

## Configuration

The plugin is configured via a JSON config file (default: `/etc/pcidp/config.json`),
typically provided as a Kubernetes ConfigMap.

### CLI Flags

| Flag                | Default                  | Description                                |
|---------------------|--------------------------|--------------------------------------------|
| `--config-file`     | `/etc/pcidp/config.json` | Path to the JSON config file               |
| `--resource-prefix` | `intel.com`              | Resource name prefix for k8s resources     |
| `--use-cdi`         | `false`                  | Use CDI to expose devices in containers    |

### Config File Structure

```json
{
  "resourceList": [
    {
      "resourceName": "intel_sriov_netdevice",
      "resourcePrefix": "intel.com",
      "deviceType": "netDevice",
      "excludeTopology": false,
      "selectors": {
        "vendors": ["8086"],
        "devices": ["154c", "10ed"],
        "drivers": ["iavf", "vfio-pci"],
        "pfNames": ["enp0s0f0"],
        "rootDevices": ["0000:86:00.0"],
        "linkTypes": ["ether"],
        "ddpProfiles": ["GTPv1-C/U IPv4/IPv6 Payload"],
        "pciAddresses": ["0000:03:02.0"],
        "acpiIndexes": ["101"],
        "isRdma": false,
        "needVhostNet": false,
        "vdpaType": "virtio",
        "pKeys": ["0x1"]
      }
    }
  ]
}
```

### Device Types

| `deviceType`    | Selector Struct            | Description                  |
|-----------------|----------------------------|------------------------------|
| `netDevice`     | `NetDeviceSelectors`       | SR-IOV VFs / PFs (default)   |
| `accelerator`   | `AccelDeviceSelectors`     | FPGA / AI accelerators       |
| `auxNetDevice`  | `AuxNetDeviceSelectors`    | Subfunctions (SFs)           |

If `deviceType` is omitted, it defaults to `netDevice`.

---

## Deployment

The plugin runs as a **DaemonSet** on Kubernetes nodes with SR-IOV capable NICs.

### Typical Deployment

1. Create a ConfigMap with device selectors (see `deployments/configMap.yaml`)
2. Deploy the DaemonSet (see `deployments/sriovdp-daemonset.yaml`)

The DaemonSet mounts:
- `/sys` — for device discovery
- `/var/lib/kubelet/` — for device plugin socket registration
- ConfigMap — for the config file

### Related Components

The plugin works with:
- **Multus CNI** or **DANM** — Meta-CNI plugin for multi-network support
- **SR-IOV CNI** — Plumbs VFs into pod network namespaces
- **Host-device CNI** — Plumbs PFs into pod network namespaces
- **SR-IOV Network Operator** — Automates the full SR-IOV stack deployment

---

## Contributing Conventions

### Commit Messages

- Use **imperative mood**: `"Add feature"` not `"Added feature"`
- Reference issues: `"Fix device discovery bug (#42)"`
- Keep subject ≤ 50 characters; wrap body at 72

### Branch Naming

- Feature branches: `dev/<topic>` (e.g., `dev/add-vdpa-support`)
- Follow the convention from CONTRIBUTING.md

### Pull Request Checklist

1. Code compiles: `make build`
2. Tests pass: `make test`
3. Lint passes: `make lint`
4. `go mod tidy` produces no diff
5. Mocks regenerated if interfaces changed: `make generate-mocks`
6. No focused Ginkgo tests (`FDescribe`, `FIt`)
7. Documentation updated if behavior changed

---

## Common Pitfalls

1. **Forgetting to regenerate mocks** — If you change any interface in
   `pkg/types/types.go`, run `make generate-mocks` or CI will fail.

2. **Import ordering** — The linter enforces strict import grouping
   (stdlib → external → project). Use `make lint-fix` to auto-fix.

3. **Magic numbers** — The `mnd` linter flags unexplained numeric literals.
   Use named constants.

4. **Function length** — Keep functions under 100 lines / 50 statements. If a
   function grows beyond this, refactor it.

5. **go mod tidy / vendor** — CI checks that `go.mod`, `go.sum`, and the
   `vendor/` directory are consistent. Always run `go mod tidy && go mod vendor`
   after dependency changes.

6. **Focused tests** — `FDescribe`, `FIt`, `FContext` etc. will cause CI to fail
   via the `ginkgolinter` rule. Use them locally only and never commit.

7. **Test dependencies** — Tests require the `hwdata-pci` system package
   installed. The CI installs it via `apt-get install hwdata`.

8. **VFIO No-IOMMU** — The plugin supports virtual deployments without
   virtualized IOMMU. Be aware of this when modifying VFIO-related code paths.

9. **CDI mode** — CDI (`--use-cdi`) changes how devices are exposed to
   containers. Test both CDI and non-CDI paths when modifying device allocation
   logic.

10. **Multi-selector support** — `ResourceConfig.Selectors` is a
    `*json.RawMessage` that can contain multiple selector objects. The
    `SelectorObjs` field holds the parsed selectors. Handle the multi-selector
    case when working with filtering logic.
