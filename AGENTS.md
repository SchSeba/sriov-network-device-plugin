# AGENTS.md — Guidance for AI Agents

This document provides guidance for AI agents (and developers) working with the
**SR-IOV Network Device Plugin** codebase. It covers the project structure,
build system, testing conventions, coding style, and common workflows.

---

## Project Overview

The SR-IOV Network Device Plugin is a Kubernetes device plugin that discovers
and advertises SR-IOV network resources (Virtual Functions, auxiliary devices,
accelerators) to the kubelet. It implements the Kubernetes Device Plugin API
(`k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1`) and runs as a DaemonSet on
each node.

**Key concepts:**

- **Device Providers** discover host PCI/auxiliary devices and create typed
  `HostDevice` objects.
- **Selectors** filter discovered devices based on user-defined criteria
  (vendor, driver, PF name, PCI address, etc.).
- **Resource Pools** group filtered devices into Kubernetes extended resources.
- **Resource Servers** expose each pool via gRPC to the kubelet.
- **Info Providers** attach device-specific metadata (VFIO, UIO, RDMA, vDPA,
  vhost-net paths) to devices.
- **CDI (Container Device Interface)** support optionally exposes devices using
  CDI specs instead of raw device paths.

**Repository:** `github.com/k8snetworkplumbingwg/sriov-network-device-plugin`
**Primary branch:** `master`
**Go module path:** `github.com/k8snetworkplumbingwg/sriov-network-device-plugin`
**License:** Apache 2.0

---

## Repository Structure

```
.
├── cmd/
│   └── sriovdp/              # Main binary entry point
│       ├── main.go           # CLI flags, signal handling, startup
│       ├── manager.go        # resourceManager: config reading, device discovery, server lifecycle
│       └── manager_test.go
├── pkg/
│   ├── types/                # Core interfaces and type definitions
│   │   ├── types.go          # All interfaces: HostDevice, PciDevice, NetDevice, ResourcePool,
│   │   │                     #   ResourceServer, ResourceFactory, DeviceProvider, DeviceSelector, etc.
│   │   └── mocks/            # Auto-generated mockery mocks for all interfaces in types.go
│   ├── factory/              # ResourceFactory implementation — instantiates providers, pools, selectors, servers
│   │   ├── factory.go
│   │   └── factory_test.go
│   ├── resources/            # ResourceServer (gRPC server), ResourcePool (stub), and device selectors
│   │   ├── server.go         # gRPC device plugin server implementation
│   │   ├── pool_stub.go      # Base resource pool implementation
│   │   ├── deviceSelectors.go# Vendor, device, driver, PCI address, PF name, link type selectors
│   │   ├── ddpSelector.go    # DDP profile selector
│   │   ├── pKeySelector.go   # InfiniBand partition key selector
│   │   └── testing.go        # Test helpers (excluded from linting)
│   ├── netdevice/            # PCI network device provider, pool, and device implementations
│   │   ├── netDeviceProvider.go
│   │   ├── netResourcePool.go
│   │   ├── pciNetDevice.go
│   │   └── nadutils.go       # Network-Attachment-Definition file utilities
│   ├── accelerator/          # PCI accelerator device provider, pool, and device
│   ├── auxnetdevice/         # Auxiliary network device (e.g., subfunctions) provider, pool, and device
│   ├── devices/              # Low-level device abstractions: PCI, generic net, RDMA, vDPA, host
│   │   ├── gen_pci.go        # Generic PCI device
│   │   ├── gen_net.go        # Generic network device
│   │   ├── rdma.go           # RDMA device spec
│   │   ├── vdpa.go           # vDPA device
│   │   ├── host.go           # Host device base
│   │   └── api.go            # API device (Kubernetes device plugin spec)
│   ├── infoprovider/         # DeviceInfoProvider implementations
│   │   ├── genericInfoProvider.go
│   │   ├── vfioInfoProvider.go
│   │   ├── uioInfoProvider.go
│   │   ├── rdmaInfoProvider.go
│   │   ├── vdpaInfoProvider.go
│   │   ├── vhostNetInfoProvider.go
│   │   └── extraInfoProvider.go
│   ├── cdi/                  # CDI spec generation and cleanup
│   │   ├── cdi.go
│   │   └── mocks/
│   └── utils/                # Shared utilities: sysfs helpers, netlink, RDMA, DDP, vDPA providers
│       ├── utils.go          # File system helpers, resource name validation, plugin watch mode
│       ├── netlink_provider.go
│       ├── rdma_provider.go
│       ├── sriovnet_provider.go
│       ├── vdpa_provider.go
│       ├── ddp.go
│       ├── mocks/
│       └── testing.go        # Fake filesystem helpers for tests
├── deployments/              # Kubernetes manifests
│   ├── configMap.yaml        # Example config map with resource definitions
│   ├── sriovdp-daemonset.yaml# DaemonSet deployment manifest
│   ├── cdi/                  # CDI-specific deployment variant
│   ├── sriov-crd.yaml        # Example SriovNetworkNodePolicy CRD
│   └── pod-tc*.yaml          # Example test pod specs
├── docs/                     # Additional documentation
│   ├── vf-setup.md           # VF setup guide
│   └── (bond, ddp, dpdk, rdma, subfunctions, vdpa, config-file, udev)
├── images/
│   ├── Dockerfile            # Multi-stage Docker build
│   ├── entrypoint.sh         # Container entrypoint
│   └── ddptool-*             # DDP tool source archive
├── .github/
│   └── workflows/
│       ├── build-test-lint.yml    # Primary CI: build, test, lint, e2e
│       ├── codeql.yml             # CodeQL security analysis
│       ├── image-push-master.yml  # Image push on master merge
│       └── image-push-release.yml # Image push on release tag
├── Makefile                  # Build, test, lint, image, mock generation targets
├── .golangci.yml             # golangci-lint v2 configuration
├── go.mod / go.sum           # Go module files
├── CONTRIBUTING.md           # Contribution guidelines
└── README.md                 # User-facing documentation
```

---

## Build System

The project uses a **Makefile** as the primary build orchestrator.

### Key Make Targets

| Target              | Description                                        |
| ------------------- | -------------------------------------------------- |
| `make build`        | Build the `sriovdp` binary into `build/`           |
| `make test`         | Run unit tests (15s timeout)                       |
| `make test-race`    | Run tests with the Go race detector                |
| `make test-coverage`| Run tests with coverage output to `test/coverage/` |
| `make lint`         | Run golangci-lint (v2)                             |
| `make lint-fix`     | Run golangci-lint with `--fix`                     |
| `make image`        | Build Docker image                                 |
| `make generate-mocks` | Regenerate mockery mocks                        |
| `make clean`        | Remove build artifacts, caches                     |
| `make deps-update`  | Run `go mod tidy`                                  |
| `make all`          | lint → build → test                                |

### Build Details

- **Binary:** `sriovdp`, output to `build/sriovdp`
- **Build command:** `go build` from `cmd/sriovdp/`
- **Default build tags:** `-tags no_openssl`
- **Static builds:** Set `STATIC=1` to produce a statically linked binary
  (`CGO_ENABLED=0 -extldflags "-static"`)
- **Docker image tag:** `ghcr.io/k8snetworkplumbingwg/sriov-network-device-plugin`
- **Dockerfile:** Multi-stage Alpine-based build at `images/Dockerfile`

---

## Testing

### Framework

- **Ginkgo v2** (`github.com/onsi/ginkgo/v2`) + **Gomega** (`github.com/onsi/gomega`)
  for BDD-style tests.
- **testify** (`github.com/stretchr/testify`) is also used in some test files.
- **Mockery** (`github.com/vektra/mockery/v2`) generates mocks for all
  interfaces.

### Test File Conventions

- Test files are co-located with the code they test (e.g., `factory.go` →
  `factory_test.go`).
- Each package with Ginkgo tests has a `*_suite_test.go` file that bootstraps
  the Ginkgo test runner:
  ```go
  func TestResources(t *testing.T) {
      RegisterFailHandler(Fail)
      RunSpecs(t, "Resources Suite")
  }
  ```
- Test helper files named `testing.go` exist in `pkg/resources/` and
  `pkg/utils/` — these are excluded from linting via `.golangci.yml`.

### Running Tests

```bash
# All tests
make test

# With race detection (used in CI)
make test-race

# With coverage
make test-coverage

# Run tests for a specific package
go test ./pkg/netdevice/...

# Run a specific test by name (Ginkgo)
go test ./pkg/factory/ -run "TestFactory" -v -ginkgo.focus "GetSelector"
```

### Mocks

Mocks are generated with **mockery** and stored in `mocks/` subdirectories:

- `pkg/types/mocks/` — Mocks for all core interfaces (HostDevice, PciDevice,
  ResourcePool, ResourceFactory, DeviceProvider, etc.)
- `pkg/utils/mocks/` — Mocks for utility providers
- `pkg/cdi/mocks/` — Mocks for CDI interface

**To regenerate mocks:**

```bash
make generate-mocks
```

> **Important:** After modifying any interface in `pkg/types/types.go`,
> `pkg/utils/`, or `pkg/cdi/`, you must regenerate mocks. The generated mock
> files follow the pattern `mock_<InterfaceName>.go` and `<InterfaceName>.go`
> (two different naming conventions coexist).

### Test Patterns

Tests typically follow this pattern:

```go
var _ = Describe("ComponentName", func() {
    Context("when condition X", func() {
        It("should do Y", func() {
            // Setup mock
            mockDevice := mocks.PciNetDevice{}
            mockDevice.On("GetVendor").Return("8086")

            // Execute
            result := functionUnderTest(...)

            // Assert
            Expect(result).To(Equal(expected))
            mockDevice.AssertExpectations(GinkgoT())
        })
    })
})
```

### Test Dependencies

- Tests in `pkg/utils/` use a fake filesystem (see `pkg/utils/testing.go`) to
  mock sysfs paths.
- Tests in `pkg/resources/` use helper functions from `pkg/resources/testing.go`
  to create test fixtures.
- The `hwdata` system package is required for tests to pass (installed via
  `apt-get install hwdata` in CI).

---

## CI Pipeline

CI is defined in `.github/workflows/build-test-lint.yml` and runs on every push
and pull request.

### CI Jobs

1. **build** — Compiles the binary (`make build`)
2. **test** — Runs tests with race detector (`make test-race`); requires
   `hwdata`
3. **test-coverage** — Runs coverage tests, reports to Coveralls
4. **golangci** — Runs `make lint`
5. **shellcheck** — Lints shell scripts
6. **hadolint** — Lints the Dockerfile
7. **go-check** — Verifies `go mod tidy` and `go mod vendor` produce no diff
8. **sriov-operator-e2e-test** — End-to-end tests using the SR-IOV Network
   Operator on hardware-equipped runners (`runs-on: [sriov]`)

### Before Submitting a PR

Ensure all of the following pass locally:

```bash
make lint        # Linting
make build       # Compilation
make test-race   # Tests with race detector
go mod tidy      # Module consistency
```

---

## Coding Conventions

### Go Style

- Follow [Effective Go](https://golang.org/doc/effective_go.html) and
  [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments).
- **Max line length:** 140 characters (enforced by `lll` linter).
- **Max function length:** 100 lines / 50 statements (enforced by `funlen`).
- **Max cyclomatic complexity:** 15 (enforced by `gocyclo`).

### Import Ordering

Imports are organized into three groups (enforced by `gci` formatter):

1. Standard library
2. Third-party packages
3. Project packages (`github.com/k8snetworkplumbingwg/sriov-network-device-plugin`)

```go
import (
    "fmt"
    "os"

    "github.com/golang/glog"
    "github.com/jaypipes/ghw"

    "github.com/k8snetworkplumbingwg/sriov-network-device-plugin/pkg/types"
    "github.com/k8snetworkplumbingwg/sriov-network-device-plugin/pkg/utils"
)
```

### Logging

- Uses **glog** (`github.com/golang/glog`) — not logrus (logrus is explicitly
  banned by `depguard` in `.golangci.yml`).
- Use `glog.Infof()`, `glog.Warningf()`, `glog.Errorf()`, `glog.Fatalf()`.
- Use `glog.V(level).Infof()` for verbose/debug output.

### Error Handling

- Return errors rather than panicking.
- Use `fmt.Errorf()` for error wrapping (the project does not currently use
  `%w` wrapping extensively).
- `pkg/errors` (`github.com/pkg/errors`) is a dependency but used sparingly.

### Interface-Driven Design

The codebase is heavily interface-driven. All major components are defined as
interfaces in `pkg/types/types.go`:

| Interface            | Purpose                                              |
| -------------------- | ---------------------------------------------------- |
| `HostDevice`         | Base device interface (vendor, driver, device ID)    |
| `PciDevice`          | PCI device (adds PCI address, ACPI index)            |
| `NetDevice`          | Network device (adds PF name, link type, RDMA)       |
| `PciNetDevice`       | PCI network device (adds DDP, vDPA, PKey)            |
| `AccelDevice`        | PCI accelerator device                               |
| `AuxNetDevice`       | Auxiliary network device (adds aux type)              |
| `DeviceProvider`     | Discovers and filters devices of a specific type     |
| `DeviceSelector`     | Filters devices by attribute (vendor, driver, etc.)  |
| `DeviceInfoProvider` | Provides device specs, env vars, and mounts          |
| `ResourcePool`       | Groups devices into a Kubernetes extended resource   |
| `ResourceServer`     | gRPC server implementing the Device Plugin API       |
| `ResourceFactory`    | Factory for creating all the above                   |
| `RdmaSpec`           | RDMA device specifications                           |
| `VdpaDevice`         | vDPA device information                              |
| `NadUtils`           | Network-Attachment-Definition file I/O               |

When adding a new feature, prefer implementing an existing interface or
extending one rather than adding standalone functions.

### Device Type System

Three device types are supported, defined in `pkg/types/types.go`:

| `DeviceType`       | PCI Class | Provider Package   | Device Interface |
| ------------------ | --------- | ------------------ | ---------------- |
| `netDevice`        | `0x02`    | `pkg/netdevice`    | `PciNetDevice`   |
| `accelerator`      | `0x12`    | `pkg/accelerator`  | `AccelDevice`    |
| `auxNetDevice`     | `0x02`    | `pkg/auxnetdevice` | `AuxNetDevice`   |

Each device type has a matching set of:
- `*DeviceProvider` — discovers devices and applies selectors
- `*ResourcePool` — manages the device pool for kubelet
- `*Device` — represents a single device instance
- `*DeviceSelectors` — defines selector fields for that type

### Adding a New Selector

1. Define the selector field in the appropriate `*DeviceSelectors` struct in
   `pkg/types/types.go`.
2. Implement the `DeviceSelector` interface in `pkg/resources/` (see existing
   selectors like `VendorSelector`, `DriverSelector`).
3. Register it in `factory.go`'s `GetSelector()` switch statement.
4. Apply it in the appropriate `DeviceProvider.GetFilteredDevices()` method.
5. Add tests.
6. Regenerate mocks: `make generate-mocks`.

### Adding a New Device Type

1. Define a new `DeviceType` constant and PCI class in `pkg/types/types.go`.
2. Create a new package under `pkg/` with provider, pool, and device
   implementations.
3. Register the device type in `pkg/factory/factory.go` (GetDeviceProvider,
   GetResourcePool, GetDeviceFilter).
4. Add tests and regenerate mocks.

---

## Configuration

The plugin reads its configuration from a JSON file (default:
`/etc/pcidp/config.json`), typically provided via a Kubernetes ConfigMap.

**Example configuration:**

```json
{
  "resourceList": [
    {
      "resourceName": "intel_sriov_netdevice",
      "resourcePrefix": "intel.com",
      "deviceType": "netDevice",
      "selectors": {
        "vendors": ["8086"],
        "devices": ["154c", "10ed"],
        "drivers": ["i40evf", "iavf"]
      }
    }
  ]
}
```

**CLI flags:**

| Flag                | Default                    | Description                                 |
| ------------------- | -------------------------- | ------------------------------------------- |
| `-config-file`      | `/etc/pcidp/config.json`   | Path to the JSON config file                |
| `-resource-prefix`  | `intel.com`                | Kubernetes resource name prefix             |
| `-use-cdi`          | `false`                    | Use CDI to expose devices in containers     |

---

## Docker Image

The Docker image uses a multi-stage Alpine-based build:

1. **Builder stage:** Compiles `sriovdp` using Go 1.25
2. **DDP builder stage:** Compiles the `ddptool` utility
3. **Runtime stage:** Alpine with `hwdata-pci`, the compiled binary, ddptool,
   and the entrypoint script

```bash
# Build the image
make image

# Build with custom tag
make image TAG=my-registry/sriov-dp:dev

# Build with proxy
make image HTTP_PROXY=http://proxy:8080
```

---

## Key Dependencies

| Dependency                                | Purpose                                    |
| ----------------------------------------- | ------------------------------------------ |
| `k8s.io/kubelet`                          | Kubernetes Device Plugin API (v1beta1)     |
| `google.golang.org/grpc`                  | gRPC server/client                         |
| `github.com/jaypipes/ghw`                 | Hardware discovery (PCI enumeration)       |
| `github.com/vishvananda/netlink`          | Netlink interface for network info         |
| `github.com/k8snetworkplumbingwg/sriovnet`| SR-IOV network utilities                   |
| `github.com/k8snetworkplumbingwg/govdpa`  | vDPA device management                    |
| `github.com/Mellanox/rdmamap`             | RDMA device mapping                        |
| `github.com/k8snetworkplumbingwg/network-attachment-definition-client` | Multus NAD support |
| `github.com/container-orchestrated-devices/container-device-interface` | CDI spec support |
| `github.com/golang/glog`                  | Logging                                    |
| `github.com/onsi/ginkgo/v2` + `gomega`   | Testing framework                          |
| `github.com/stretchr/testify`             | Test assertions and mocking                |

---

## Common Workflows

### Full Development Cycle

```bash
# 1. Create a feature branch
git checkout -b dev/my-feature

# 2. Make changes

# 3. Regenerate mocks if interfaces changed
make generate-mocks

# 4. Format and lint
make lint-fix

# 5. Build
make build

# 6. Run tests
make test-race

# 7. Ensure module consistency
go mod tidy

# 8. Commit and push
git add -A
git commit -m "Add feature description"
git push -u origin dev/my-feature
```

### Commit Message Format

```
Change summary

More detailed explanation of your changes: Why and how.
Wrap it to 72 characters.

[Fixes #NUMBER]
```

Use imperative mood: "Add feature" not "Added feature".

---

## Linting Configuration

The project uses **golangci-lint v2** with extensive configuration in
`.golangci.yml`. Notable settings:

- **Enabled linters:** ~30 linters including `govet`, `staticcheck`, `gosec`,
  `errcheck`, `gocyclo`, `funlen`, `lll`, `misspell`, `dupl`, and
  `ginkgolinter`.
- **Focus containers forbidden:** `ginkgolinter` enforces no `FDescribe`,
  `FIt`, etc. in tests.
- **Import formatting:** `gci` enforces standard → third-party → project import
  order. `goimports` uses local prefix for the project module path.
- **Excluded paths:** `.github/`, `deployments/`, `docs/`, and specific
  `testing.go` files.
- **Test relaxations:** `dupl`, `goconst`, `lll`, and `gosec` are relaxed in
  `_test.go` files.

---

## Tips for AI Agents

1. **Always run `make lint` before committing.** The CI will reject PRs that
   fail linting.
2. **Regenerate mocks after interface changes.** Forgetting this is a common
   source of CI failures.
3. **Test files use Ginkgo/Gomega.** Write new tests using `Describe`,
   `Context`, `It`, `Expect` — not standard `t.Run` + `if` patterns (unless
   adding to a file that already uses testify).
4. **Interfaces live in `pkg/types/types.go`.** Don't scatter interface
   definitions across packages.
5. **The `factory` package is the central wiring point.** When adding new device
   types, selectors, or providers, the factory is where they get registered.
6. **Use glog for logging, never logrus.** The linter will reject logrus
   imports.
7. **Check sysfs paths in tests.** Many tests mock the filesystem. Look at
   `pkg/utils/testing.go` for helpers.
8. **`go mod tidy` must produce no diff.** CI checks this explicitly.
9. **Line length limit is 140 characters.** Keep lines within this bound.
10. **The `master` branch is the primary branch**, not `main`.
