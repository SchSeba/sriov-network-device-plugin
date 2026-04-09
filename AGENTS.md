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
  and device selectors. Singleton pattern via `NewResourceFactory()`.

---

## Repository Layout

```
.
├── cmd/sriovdp/            # Main binary entry point
│   ├── main.go             # CLI flags, signal handling, program entry
│   ├── manager.go          # resourceManager — orchestrates config, discovery, and servers
│   └── manager_test.go     # Unit tests for the resource manager
├── pkg/                    # Library packages
│   ├── types/              # Interfaces & data structures (the "API contract")
│   │   ├── types.go        # Core interfaces: ResourceFactory, ResourcePool, DeviceProvider, etc.
│   │   └── mocks/          # mockery-generated mocks for all interfaces
│   ├── factory/            # ResourceFactory implementation (creates all components)
│   ├── resources/          # ResourcePool and ResourceServer implementations
│   │   ├── server.go       # gRPC device-plugin server (implements pluginapi + registrationapi)
│   │   ├── pool_stub.go    # Generic resource pool implementation
│   │   ├── deviceSelectors.go  # Vendor/device/driver selector logic
│   │   ├── ddpSelector.go  # DDP profile selector
│   │   ├── pKeySelector.go # InfiniBand partition key selector
│   │   └── testing.go      # Test helper utilities (excluded from linting)
│   ├── devices/            # Low-level host device abstractions
│   │   ├── api.go          # APIDeviceImpl — wraps pluginapi.Device with info providers
│   │   ├── host.go         # HostDeviceImpl — base device with vendor/driver/deviceCode
│   │   ├── gen_pci.go      # GenPciDevice — embeddable PCI device (address, ACPI index)
│   │   ├── gen_net.go      # GenNetDevice — embeddable network device (PF, link, RDMA)
│   │   ├── rdma.go         # RDMA spec implementation
│   │   └── vdpa.go         # vDPA device implementation
│   ├── netdevice/          # Network (PCI VF) device provider & resource pool
│   │   ├── pciNetDevice.go     # PciNetDevice: composes HostDevice + GenPciDevice + GenNetDevice
│   │   ├── netDeviceProvider.go # Discovers and filters PCI net devices
│   │   ├── netResourcePool.go  # Net-specific resource pool (NAD support, vhost-net)
│   │   └── nadutils.go     # Network Attachment Definition file utilities
│   ├── accelerator/        # Accelerator device provider & resource pool
│   │   ├── accelDevice.go      # AccelDevice implementation
│   │   ├── accelDeviceProvider.go
│   │   └── accelResourcePool.go
│   ├── auxnetdevice/       # Auxiliary network device provider & resource pool (Subfunctions)
│   │   ├── auxNetDevice.go
│   │   ├── auxNetDeviceProvider.go
│   │   └── auxNetResourcePool.go
│   ├── infoprovider/       # DeviceInfoProviders (composable device data enrichment)
│   │   ├── genericInfoProvider.go  # Base device info (always present)
│   │   ├── vfioInfoProvider.go     # VFIO device specs
│   │   ├── uioInfoProvider.go      # UIO device specs
│   │   ├── rdmaInfoProvider.go     # RDMA device information
│   │   ├── vdpaInfoProvider.go     # vDPA device information
│   │   ├── vhostNetInfoProvider.go # vhost-net passthrough
│   │   └── extraInfoProvider.go    # User-defined key-value annotations
│   ├── cdi/                # Container Device Interface (CDI) support
│   │   ├── cdi.go          # CDI spec creation and cleanup
│   │   └── mocks/          # Mocks for CDI interface
│   └── utils/              # Utility functions and provider abstractions
│       ├── utils.go        # Core utils (sysfs parsing, driver info, VF detection, etc.)
│       ├── netlink_provider.go  # NetlinkProvider — abstraction over vishvananda/netlink
│       ├── rdma_provider.go     # RdmaProvider — abstraction over Mellanox/rdmamap
│       ├── sriovnet_provider.go # SriovnetProvider — abstraction over sriovnet
│       ├── vdpa_provider.go     # VdpaProvider — abstraction over govdpa
│       ├── ddp.go          # DDP (Dynamic Device Personalization) profile utilities
│       ├── testing.go      # Test helper utilities (excluded from linting)
│       ├── mocks/          # Mocks for all provider interfaces
│       └── testdata/       # Test fixture data
├── deployments/            # Kubernetes YAML manifests
│   ├── sriovdp-daemonset.yaml  # DaemonSet for the device plugin
│   ├── configMap.yaml          # Sample ConfigMap with resource configurations
│   ├── sriov-crd.yaml          # Sample NetworkAttachmentDefinition
│   ├── flannel-network.yaml    # Sample flannel NetworkAttachmentDefinition
│   ├── pod-tc1.yaml, pod-tc2.yaml  # Sample test pods
│   └── cdi/                    # CDI-specific deployment manifests
├── images/                 # Container image files
│   ├── Dockerfile          # Multi-stage build (Go binary + ddptool + Alpine)
│   └── entrypoint.sh       # Container entrypoint (CLI flag parsing)
├── docs/                   # User documentation
│   ├── dpdk/               # DPDK application guide
│   ├── rdma/               # RDMA application guide
│   ├── vdpa/               # vDPA usage guide
│   ├── ddp/                # DDP profile usage (e800, x700)
│   ├── bond/               # Bond interface failover guide
│   ├── subfunctions/       # Scalable Functions (SFs) guide
│   ├── udev/               # UDEV rules for NIC naming
│   └── config-file/        # Node-specific config file guide
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
2. Builds the `ddptool` utility from a vendored tarball (`images/ddptool-1.0.1.12.tar.gz`).
3. Produces a minimal Alpine final image with `sriovdp`, `ddptool`, and `hwdata-pci`.

The container entrypoint (`images/entrypoint.sh`) accepts runtime flags:
`--log-dir`, `--log-level` (default 10), `--resource-prefix`, `--config-file`,
and `--use-cdi`.

---

## Testing

### Framework

Tests use **Ginkgo v2** (`github.com/onsi/ginkgo/v2`) with **Gomega**
(`github.com/onsi/gomega`) as the matcher library. Every package under `pkg/`
has a corresponding `_test.go` file and a `*_suite_test.go` bootstrap file.
The `cmd/sriovdp/` package also has its own `manager_test.go`.

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

### Test Helpers

Two files provide shared test helpers and are excluded from all linting:
- `pkg/utils/testing.go` — utility test helpers (e.g., fake sysfs setup)
- `pkg/resources/testing.go` — resource pool test helpers

Test fixture data is stored in `pkg/utils/testdata/`.

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
and are not expected to be run locally. The CI job builds a local container image
and feeds it to the operator's virtual cluster e2e suite.

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

### License Headers

Every `.go` file must include a license header. The project has two styles:

**Intel-originated files** (most of the codebase):
```go
// Copyright 20XX Intel Corp. All Rights Reserved.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// ...
```

**NVIDIA-contributed files** (primarily `pkg/devices/`):
```go
/*
 * SPDX-FileCopyrightText: Copyright (c) 2022 NVIDIA CORPORATION & AFFILIATES.
 * SPDX-License-Identifier: Apache-2.0
 * ...
 */
```

Both are Apache 2.0. When adding new files, use the Intel-style header unless
contributing to an existing NVIDIA-originated package.

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

### Composition Pattern in `pkg/devices/`

Device implementations use **struct embedding** (composition) rather than
inheritance. The building blocks are:

- `APIDeviceImpl` — wraps `pluginapi.Device` with a list of `DeviceInfoProvider`s
- `HostDeviceImpl` — embeds `APIDeviceImpl`, adds vendor/device/driver info
- `GenPciDevice` — embeddable PCI device (PCI address, ACPI index)
- `GenNetDevice` — embeddable network device (PF name/addr, link type/speed, RDMA)

Top-level device types compose these:
```
pciNetDevice = HostDeviceImpl + GenPciDevice + GenNetDevice + DDP/vDPA
accelDevice  = HostDeviceImpl + GenPciDevice
auxNetDevice = HostDeviceImpl + GenNetDevice
```

### Provider Abstractions in `pkg/utils/`

System-level operations are abstracted behind provider interfaces to enable
mocking in tests:

- `NetlinkProvider` — wraps `vishvananda/netlink` (link status, speed queries)
- `RdmaProvider` — wraps `Mellanox/rdmamap` (RDMA device discovery)
- `SriovnetProvider` — wraps `k8snetworkplumbingwg/sriovnet` (VF/PF relationships)
- `VdpaProvider` — wraps `k8snetworkplumbingwg/govdpa` (vDPA device management)

These are set as package-level variables and swapped out in tests via the mock
implementations in `pkg/utils/mocks/`.

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
7. If CDI is enabled, clean up CDI specs on shutdown.

### Device Discovery & Filtering Pipeline

```
ghw.PCI().Devices
    → DeviceProvider.AddTargetDevices()    # filter by PCI class code
    → DeviceProvider.GetDevices()          # get all devices for a ResourceConfig
    → DeviceProvider.GetFilteredDevices()  # apply selectors (vendor, device, driver, etc.)
    → resourceManager.excludeAllocatedDevices()  # prevent double-allocation
    → ResourceFactory.GetResourcePool()    # create pool with filtered devices
    → ResourceFactory.GetResourceServer()  # create gRPC server for the pool
```

### Kubelet Registration

The plugin supports two registration modes, auto-detected at startup:
- **Plugin Watch Mode** (modern): socket at `/var/lib/kubelet/plugins_registry/`
- **Deprecated Mode**: socket at `/var/lib/kubelet/device-plugins/`

### Device Type Hierarchy

```
HostDevice (base interface)
├── PciDevice          → adds GetPciAddr(), GetAcpiIndex()
│   ├── PciNetDevice   → adds GetPfNetName(), GetNetName(), RDMA, vDPA, DDP, link info
│   │   └── pciNetDevice (implementation in pkg/netdevice/)
│   └── AccelDevice    → accelerator devices
│       └── accelDevice (implementation in pkg/accelerator/)
└── AuxNetDevice       → auxiliary network devices (Subfunctions, non-PCI path)
    └── auxNetDevice (implementation in pkg/auxnetdevice/)
```

### DeviceInfoProviders

Info providers enrich device data exposed to pods. They are composable — a
device can have multiple providers:

- `genericInfoProvider` — base device info (always present)
- `vfioInfoProvider` — VFIO device specs (`/dev/vfio/`)
- `uioInfoProvider` — UIO device specs (`/dev/uioN`)
- `rdmaInfoProvider` — RDMA device information
- `vdpaInfoProvider` — vDPA device information
- `vhostNetInfoProvider` — vhost-net passthrough (`/dev/vhost-net`)
- `extraInfoProvider` — user-defined key-value annotations from `additionalInfo` config

Provider selection is driver-based: `vfio-pci` → vfioInfoProvider, `uio`/`igb_uio`
→ uioInfoProvider, kernel drivers → genericInfoProvider.

---

## CI / GitHub Actions

CI workflows are in `.github/workflows/`:

| Workflow                    | Trigger             | What it does                                        |
| --------------------------- | ------------------- | --------------------------------------------------- |
| `build-test-lint.yml`       | push, pull_request  | Build, test (race), coverage, lint, shellcheck, hadolint, go mod check, e2e |
| `image-push-master.yml`     | push to master      | Build & push container image for master             |
| `image-push-release.yml`    | release tags        | Build & push container image for releases           |
| `codeql.yml`                | schedule / PR       | GitHub CodeQL security analysis                     |

**CI requirements:**
- `hwdata` must be installed (`sudo apt-get install hwdata -y`) for tests.
- E2E tests run on dedicated `[sriov]` labeled runners with SR-IOV hardware.
- CI also validates `go mod tidy` and `go mod vendor` produce no diffs.

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
        "pciAddresses": ["0000:86:02.0"],
        "acpiIndexes": ["196"]
      },
      "additionalInfo": {
        "myKey": {"*": "myValue"}
      }
    }
  ]
}
```

### Supported `deviceType` Values

| Value            | Selector Struct            | Package              |
| ---------------- | -------------------------- | -------------------- |
| `"netDevice"`    | `NetDeviceSelectors`       | `pkg/netdevice/`     |
| `"accelerator"`  | `AccelDeviceSelectors`     | `pkg/accelerator/`   |
| `"auxNetDevice"` | `AuxNetDeviceSelectors`    | `pkg/auxnetdevice/`  |

Default is `"netDevice"` if omitted.

### CLI Arguments

| Flag               | Default                    | Description                              |
| ------------------ | -------------------------- | ---------------------------------------- |
| `--config-file`    | `/etc/pcidp/config.json`   | Path to JSON config file                 |
| `--resource-prefix`| `intel.com`                | Kubernetes resource name prefix          |
| `--use-cdi`        | `false`                    | Enable Container Device Interface        |

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
   implementations (follow the pattern in `pkg/netdevice/` or `pkg/accelerator/`).
4. Wire it into `pkg/factory/factory.go`.
5. Regenerate mocks: `make generate-mocks`.

### Adding a New System Provider

1. Define the provider interface in `pkg/utils/` (e.g., `MyProvider`).
2. Implement it with a real system backend and expose a package-level variable.
3. Generate mocks: add the package to `make generate-mocks` if not already covered.
4. In tests, swap the package-level provider with the mock.

---

## Key Dependencies

| Dependency | Purpose |
| --- | --- |
| `k8s.io/kubelet` | Device Plugin API (`v1beta1`) and Plugin Registration API |
| `google.golang.org/grpc` | gRPC server for Device Plugin API |
| `github.com/jaypipes/ghw` | Hardware discovery (PCI device enumeration) |
| `github.com/jaypipes/pcidb` | PCI device ID database lookups |
| `github.com/vishvananda/netlink` | Linux netlink interface (link status, speed) |
| `github.com/Mellanox/rdmamap` | RDMA device mapping |
| `github.com/k8snetworkplumbingwg/sriovnet` | SR-IOV network utilities (VF/PF relationships) |
| `github.com/k8snetworkplumbingwg/govdpa` | vDPA device management |
| `github.com/k8snetworkplumbingwg/network-attachment-definition-client` | Network Attachment Definition (NAD) utilities |
| `github.com/container-orchestrated-devices/container-device-interface` | CDI spec management |
| `github.com/golang/glog` | Logging |
| `github.com/onsi/ginkgo/v2` + `gomega` | Testing framework |
| `github.com/stretchr/testify` | Mock assertions in tests |

---

## Branch & Contribution Conventions

- Default branch: **master**
- Feature branches: use `dev/` prefix (e.g., `dev/add-new-selector`)
- Commit messages: imperative mood, reference issues with `Fixes #N`
- Keep the subject line under 50 characters; wrap body at 72
- All PRs must pass CI (build, test-race, lint, shellcheck, hadolint, go mod tidy/vendor check)
- Run `make all` locally before submitting a PR (runs lint → build → test)
- PRs should be small and focused; each PR must compile and pass tests

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

# Run a specific test package
go test -v ./pkg/netdevice/...

# Run a specific Ginkgo test by name
go test -v ./pkg/resources/... -ginkgo.focus="ResourceServer"
```
