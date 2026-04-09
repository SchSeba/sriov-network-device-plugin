# AGENTS.md — AI Agent Guide for sriov-network-device-plugin

This document provides guidance for AI agents (and developers) working with the
**sriov-network-device-plugin** codebase. It covers project structure, build
commands, testing conventions, coding style, and common workflows.

---

## Project Overview

The SR-IOV Network Device Plugin is a Kubernetes device plugin that discovers
and advertises SR-IOV network virtual functions (VFs), accelerator devices, and
auxiliary network devices to the kubelet. It implements the Kubernetes Device
Plugin API (`k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1`) so that pods can
request SR-IOV VF resources via standard Kubernetes resource requests.

**Key concepts:**

- **ResourceConfig** — A user-supplied JSON configuration that defines a pool of
  devices (by selectors such as vendor, device ID, driver, PF name, etc.) to
  expose as a Kubernetes extended resource.
- **DeviceProvider** — Discovers and filters host devices of a given type (net,
  accelerator, auxiliary).
- **ResourcePool** — Manages a set of devices that map to a single Kubernetes
  extended resource.
- **ResourceServer** — A gRPC server that implements the Device Plugin API for
  one ResourcePool.
- **ResourceFactory** — Creates DeviceProviders, ResourcePools, ResourceServers,
  and device selectors.

---

## Repository Layout

```
.
├── cmd/sriovdp/            # Main binary entry point
│   ├── main.go             # CLI flags, signal handling, program entry
│   └── manager.go          # resourceManager — orchestrates config, discovery, and servers
├── pkg/                    # Library packages
│   ├── types/              # Interfaces & data structures (the "API contract")
│   │   ├── types.go        # Core interfaces: ResourceFactory, ResourcePool, DeviceProvider, etc.
│   │   └── mocks/          # mockery-generated mocks for all interfaces
│   ├── factory/            # ResourceFactory implementation (creates all components)
│   ├── resources/          # ResourcePool (pool_stub.go) and ResourceServer (server.go) impls
│   │   ├── server.go       # gRPC device-plugin server
│   │   ├── pool_stub.go    # Generic resource pool implementation
│   │   ├── deviceSelectors.go  # Vendor/device/driver selector logic
│   │   ├── ddpSelector.go  # DDP profile selector
│   │   └── pKeySelector.go # InfiniBand partition key selector
│   ├── netdevice/          # Network (PCI VF) device provider & resource pool
│   ├── accelerator/        # Accelerator device provider & resource pool
│   ├── auxnetdevice/       # Auxiliary network device provider & resource pool
│   ├── infoprovider/       # DeviceInfoProviders (VFIO, UIO, RDMA, vDPA, vhost-net, extra, generic)
│   ├── devices/            # Low-level host device abstractions (PCI, netdev, RDMA, vDPA)
│   ├── cdi/                # Container Device Interface (CDI) support
│   └── utils/              # Utility functions, provider abstractions (netlink, rdma, sriovnet, vdpa)
├── deployments/            # Kubernetes YAML manifests (DaemonSet, ConfigMap, sample pods)
├── images/                 # Dockerfile and entrypoint script
├── docs/                   # User documentation (DPDK, RDMA, vDPA, DDP, config, etc.)
├── Makefile                # Build, test, lint, image targets
├── go.mod / go.sum         # Go module definition
├── .golangci.yml           # Linter configuration (golangci-lint v2)
└── .github/workflows/      # CI workflows (build, test, lint, image push)
```

---

## Build & Development

### Prerequisites

- **Go 1.25+** (see `go.mod` and CI workflow for the exact version)
- **Docker/Podman** (for container image builds)
- **hwdata** package (required at test time for PCI ID lookups: `apt-get install hwdata`)

### Key Makefile Targets

| Command                | Description                                          |
| ---------------------- | ---------------------------------------------------- |
| `make build`           | Compile the `sriovdp` binary to `build/sriovdp`      |
| `make test`            | Run unit tests (15s timeout)                         |
| `make test-race`       | Run unit tests with the Go race detector             |
| `make test-coverage`   | Run tests with coverage; output to `test/coverage/`  |
| `make lint`            | Run `golangci-lint` (v2) using `.golangci.yml`       |
| `make lint-fix`        | Run linter with `--fix` to auto-correct issues       |
| `make image`           | Build the Docker image (`ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin`) |
| `make generate-mocks`  | Regenerate mockery mocks in `pkg/types/mocks/`, `pkg/utils/mocks/`, `pkg/cdi/mocks/` |
| `make clean`           | Remove build artifacts, caches, and test output      |
| `make deps-update`     | Run `go mod tidy`                                    |
| `make all`             | Lint → Build → Test (the default target)             |

### Build Flags

- Static builds: `make build STATIC=1` (sets `CGO_ENABLED=0` and `-extldflags "-static"`)
- The binary is built with `-tags no_openssl` by default.

### Docker Image

The multi-stage Dockerfile lives at `images/Dockerfile`. It:
1. Builds the Go binary in an Alpine-based Go image.
2. Builds the `ddptool` utility from a vendored tarball.
3. Produces a minimal Alpine final image with `sriovdp`, `ddptool`, and `hwdata-pci`.

---

## Testing

### Framework

Tests use **Ginkgo v2** (`github.com/onsi/ginkgo/v2`) with **Gomega**
(`github.com/onsi/gomega`) as the matcher library. Every package under `pkg/`
has a corresponding `_test.go` file and a `*_suite_test.go` bootstrap file.

### Mocks

Mocks are generated with **mockery** (v2) and live alongside the interfaces
they mock:

| Interface Package | Mock Location          |
| ----------------- | ---------------------- |
| `pkg/types/`      | `pkg/types/mocks/`     |
| `pkg/utils/`      | `pkg/utils/mocks/`     |
| `pkg/cdi/`        | `pkg/cdi/mocks/`       |

**Do not hand-edit mocks.** Regenerate them with `make generate-mocks` after
changing any interface.

The mocks directory contains two variants per interface:
- `InterfaceName.go` — the mockery v1-style mock (used in existing tests via `testify/mock`)
- `mock_InterfaceName.go` — the mockery v2-style mock

### Test Naming Conventions

- Test files: `<source_file>_test.go` (e.g., `pciNetDevice.go` → `pciNetDevice_test.go`)
- Suite bootstrap files: `<package>_suite_test.go`
- Ginkgo `Describe`/`Context`/`It` blocks mirror the function or type being tested.
- Mocks are set up per-test using `testify/mock` expectations.

### Running Tests

```bash
make test            # Quick run
make test-race       # With race detector (used in CI)
make test-coverage   # Coverage report → test/coverage/cover.out
```

Tests exclude `**/mocks` packages automatically (`PKGS` variable in the
Makefile filters them out).

### E2E Tests

End-to-end tests run in CI on SR-IOV-capable hardware using the
**sriov-network-operator** test suite. These are triggered automatically on PRs
and are not expected to be run locally.

---

## Code Style & Conventions

### Go Style

- Follow [Effective Go](https://golang.org/doc/effective_go.html) and the
  [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments).
- Maximum line length: **140 characters** (enforced by `lll` linter).
- Maximum function length: **100 lines / 50 statements** (enforced by `funlen`).
- Maximum cyclomatic complexity: **15** (enforced by `gocyclo`).

### Import Ordering

Imports must be organized into three groups (enforced by `gci` formatter):

```go
import (
    // 1. Standard library
    "fmt"
    "os"

    // 2. Third-party packages
    "github.com/golang/glog"
    "github.com/jaypipes/ghw"

    // 3. Project-local packages
    "github.com/k8snetworkplumbingwg/sriov-network-device-plugin/pkg/types"
)
```

### Logging

The project uses **glog** (`github.com/golang/glog`) for all logging. Use
`glog.Infof`, `glog.Warningf`, `glog.Errorf`, and `glog.Fatalf`.

### Error Handling

- Wrap errors with context using `fmt.Errorf("context: %v", err)` or
  `errors.Wrap` from `github.com/pkg/errors`.
- Check and return errors; do not silently discard them.
- Error type names should end with `Error` (enforced by `errname` linter).

### Interface-Driven Design

The codebase follows a strict **interface-driven** architecture. All core
abstractions are defined as interfaces in `pkg/types/types.go`:

- `ResourceFactory`, `ResourcePool`, `ResourceServer`
- `DeviceProvider`, `HostDevice`, `PciDevice`, `NetDevice`, `AccelDevice`, `AuxNetDevice`
- `DeviceInfoProvider`, `DeviceSelector`
- `RdmaSpec`, `VdpaDevice`, `NadUtils`, `LinkWatcher`

Implementations live in their respective packages. When adding a new device
type or feature, define interfaces in `pkg/types/` first, then implement them
in the appropriate package.

### Linter Configuration

The project uses **golangci-lint v2** with a comprehensive set of linters (see
`.golangci.yml`). Key enabled linters include: `errcheck`, `govet`,
`staticcheck`, `gosec`, `gocritic`, `gocyclo`, `funlen`, `lll`, `misspell`,
`mnd`, `dupl`, `ginkgolinter`, and more.

Notable lint exclusions:
- `_test.go` files are exempt from `dupl`, `goconst`, `lll`, and `gosec`.
- `pkg/utils/testing.go` and `pkg/resources/testing.go` are excluded from all
  linters.
- Ginkgo/Gomega dot-imports are whitelisted.

---

## Architecture & Key Flows

### Startup Flow (`cmd/sriovdp/`)

1. Parse CLI flags (`--config-file`, `--resource-prefix`, `--use-cdi`).
2. Create a `resourceManager` with a `ResourceFactory` and `DeviceProvider`s.
3. Read and validate the JSON config file (default: `/etc/pcidp/config.json`).
4. Discover host PCI devices via `ghw.PCI()`.
5. For each `ResourceConfig`, filter devices through selectors, create a
   `ResourcePool`, then create and start a `ResourceServer` (gRPC).
6. Block on OS signals (SIGTERM, SIGINT, etc.) and cleanly shut down.

### Device Type Hierarchy

```
HostDevice (base interface)
├── PciDevice          → adds GetPciAddr(), GetAcpiIndex()
│   ├── NetDevice      → adds GetPfNetName(), GetNetName(), RDMA, vDPA, link info
│   │   └── PciNetDevice (implementation in pkg/netdevice/)
│   └── AccelDevice    → accelerator devices
│       └── accelDevice (implementation in pkg/accelerator/)
└── AuxNetDevice       → auxiliary network devices (non-PCI path)
    └── auxNetDevice (implementation in pkg/auxnetdevice/)
```

### DeviceInfoProviders

Info providers enrich device data exposed to pods. They are composable — a
device can have multiple providers:

- `genericInfoProvider` — base device info (always present)
- `vfioInfoProvider` — VFIO device specs
- `uioInfoProvider` — UIO device specs
- `rdmaInfoProvider` — RDMA device information
- `vdpaInfoProvider` — vDPA device information
- `vhostNetInfoProvider` — vhost-net passthrough
- `extraInfoProvider` — user-defined key-value annotations

---

## CI / GitHub Actions

CI workflows are in `.github/workflows/`:

| Workflow                    | Trigger             | What it does                                        |
| --------------------------- | ------------------- | --------------------------------------------------- |
| `build-test-lint.yml`       | push, pull_request  | Build, test (race), coverage, lint, shellcheck, hadolint, go mod check, e2e |
| `image-push-master.yml`     | push to master      | Build & push container image for master             |
| `image-push-release.yml`    | release tags        | Build & push container image for releases           |
| `codeql.yml`                | schedule / PR       | GitHub CodeQL security analysis                     |

CI requires `hwdata` to be installed (`sudo apt-get install hwdata -y`) for
tests to pass.

---

## Configuration

The plugin is configured via a JSON file (default path: `/etc/pcidp/config.json`).
The schema is defined by `types.ResourceConfList`:

```json
{
  "resourceList": [
    {
      "resourceName": "sriov_net_A",
      "resourcePrefix": "intel.com",
      "deviceType": "netDevice",
      "excludeTopology": false,
      "selectors": {
        "vendors": ["8086"],
        "devices": ["154c"],
        "drivers": ["i40evf"],
        "pfNames": ["enp0s0f0"],
        "rootDevices": ["0000:86:00.0"],
        "linkTypes": ["ether"],
        "ddpProfiles": ["GTPv1-C/U IPv4/IPv6 Payload"],
        "pKeys": ["0x1"],
        "vdpaType": "virtio",
        "isRdma": false,
        "needVhostNet": false,
        "pciAddresses": ["0000:86:02.0"]
      }
    }
  ]
}
```

Supported `deviceType` values: `"netDevice"` (default), `"accelerator"`,
`"auxNetDevice"`.

---

## Common Development Tasks

### Adding a New Device Selector

1. Define the selector field in the appropriate `*Selectors` struct in
   `pkg/types/types.go`.
2. Implement the `DeviceSelector` interface in `pkg/resources/`.
3. Wire it into the factory's `GetSelector()` and `GetDeviceFilter()` in
   `pkg/factory/factory.go`.
4. Add tests using Ginkgo/Gomega with mocks.
5. Regenerate mocks: `make generate-mocks`.

### Adding a New DeviceInfoProvider

1. Create a new file in `pkg/infoprovider/` implementing the
   `DeviceInfoProvider` interface from `pkg/types/`.
2. Register it in `factory.GetDefaultInfoProvider()`.
3. Add comprehensive tests with mocks.

### Adding a New Device Type

1. Define a new `DeviceType` constant in `pkg/types/types.go`.
2. Add the PCI class code to `types.SupportedDevices`.
3. Create a new package under `pkg/` with device, provider, and resource pool
   implementations.
4. Wire it into `pkg/factory/factory.go`.
5. Regenerate mocks: `make generate-mocks`.

---

## Branch & Contribution Conventions

- Default branch: **master**
- Feature branches: use `dev/` prefix (e.g., `dev/add-new-selector`)
- Commit messages: imperative mood, reference issues with `Fixes #N`
- All PRs must pass CI (build, test-race, lint, shellcheck, hadolint, go mod tidy check)
- Run `make all` locally before submitting a PR (runs lint → build → test)

---

## Useful Commands Quick Reference

```bash
# Full local validation (what CI runs)
make all

# Build only
make build

# Run tests with race detector
make test-race

# Lint with auto-fix
make lint-fix

# Regenerate mocks after interface changes
make generate-mocks

# Build container image
make image

# Check module consistency
go mod tidy && go mod vendor
```
