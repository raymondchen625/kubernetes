# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

### Core Build
- `make` or `make all` - Build all code (produces Linux binaries by default)
- `make all WHAT=cmd/kubelet` - Build specific component (e.g., kubelet)
- `make all DBG=1` - Build unstripped binaries for debugging (enables use of delve and other debuggers)
- `make cross` - Build for all platforms
- `make quick-release` - Quick release build using Docker

### Build in Docker (Hermetic)
- `build/run.sh make` - Build inside Docker container for reproducible builds
- `build/shell.sh` - Drop into build container shell

## Testing Commands

### Unit Tests
- `make test` - Run all unit tests
- `make test WHAT=./pkg/kubelet` - Test specific package
- `make test WHAT=./pkg/kubelet GOFLAGS=-v` - Run with verbose output
- `make test KUBE_COVER=y` - Run tests with coverage

### Integration Tests
- `make test-integration` - Run all integration tests
- `make test-integration WHAT=./test/integration/pods` - Test specific integration suite
- `make test-integration WHAT=./test/integration/kubelet GOFLAGS="-v -coverpkg=./pkg/kubelet/..." KUBE_COVER=y` - Integration tests with coverage
- `make test-integration WHAT=./test/integration/pods KUBE_TEST_ARGS='-run ^TestPodUpdateActiveDeadlineSeconds$$'` - Run specific test by name

### Node E2E Tests
- `make test-e2e-node` - Run node end-to-end tests
- `make test-e2e-node FOCUS=Kubelet` - Run tests matching "Kubelet"
- `make test-e2e-node SKIP="\[Flaky\]"` - Skip flaky tests
- `make test-e2e-node LABEL_FILTER="node && !slow"` - Filter using Ginkgo labels

## Code Quality & Verification

### Verification (Pre-submission)
- `make verify` - Run all presubmission verifications (comprehensive, includes all checks below)
- `make quick-verify` - Run fast verifications only (recommended during development)
- `make verify WHAT="gofmt typecheck"` - Run specific verifications

### Update Generated Code
- `make update` - Update all generated code and documentation
- `hack/update-codegen.sh` - Update generated code
- `hack/update-vendor.sh` - Update vendor dependencies

### Individual Verification Scripts (in hack/)
- `hack/verify-gofmt.sh` - Check Go formatting
- `hack/verify-golangci-lint.sh` - Run golangci-lint
- `hack/verify-imports.sh` - Check import statements
- `hack/verify-vendor.sh` - Verify vendor directory
- `hack/verify-codegen.sh` - Verify generated code is up to date

## Development Workflow

### Before Submitting PRs
1. Run `make quick-verify` during development to catch issues early
2. Run `make verify` before submitting to ensure all checks pass
3. If verification fails, run `make update` to regenerate code
4. Run relevant tests for modified components
5. Ensure no linter warnings remain

### Working with Tests
- Tests use Ginkgo testing framework for integration and e2e tests
- Cache mutation detector is enabled by default - catches improper object mutation
- Watch decode errors cause panics to surface bugs during testing
- Use `KUBE_TEST_ARGS` to pass flags directly to `go test` (e.g., `-run` for specific tests)

## Repository Structure

### Key Directories
- `/cmd` - Entry points for all Kubernetes binaries (kubectl, kubelet, kube-apiserver, kube-scheduler, kube-controller-manager, kube-proxy)
- `/pkg` - Core library code organized by component
- `/staging/src/k8s.io/*` - Code for separately published k8s.io/* repositories (authoritative source)
- `/api` - API definitions and rules
- `/test` - All test suites (unit, integration, e2e, conformance)
- `/hack` - Development scripts and build tools
- `/build` - Build scripts and Dockerfiles
- `/_output` - Build artifacts (gitignored)

### Staging Repositories
The `/staging` directory contains authoritative code for 30+ separately published k8s.io/* repositories (e.g., client-go, api, apimachinery, apiserver). These are managed via Go workspaces:
- Direct modification is allowed and encouraged in staging directories
- Imports of k8s.io/* packages resolve to staging via `go.work`
- Published periodically to separate GitHub repositories
- Never directly import `k8s.io/kubernetes` - use the published staging repos instead

### Major Components in /pkg
- `api/` - API machinery
- `controller/` - 40+ controllers
- `kubelet/` - Kubelet implementation
- `kubectl/` - Kubectl implementation
- `scheduler/` - Scheduler implementation
- `proxy/` - kube-proxy implementation

## Architecture Overview

### Kubernetes Components
Kubernetes is a distributed system with multiple components:

1. **Control Plane Components**:
   - `kube-apiserver` - REST API server, central communication hub
   - `kube-controller-manager` - Runs controllers that regulate cluster state
   - `kube-scheduler` - Assigns pods to nodes
   - `etcd` - Consistent key-value store for cluster state

2. **Node Components**:
   - `kubelet` - Ensures containers are running on nodes
   - `kube-proxy` - Network proxy maintaining network rules
   - Container runtime (containerd, CRI-O) - Runs containers

3. **User-facing Tools**:
   - `kubectl` - CLI for interacting with clusters

### Code Generation
Kubernetes heavily uses code generation for:
- Client libraries (client-go)
- Deep copy functions
- Informers and listers
- API conversions
- OpenAPI specs
- Protobuf bindings

Generated code must be kept in sync - run `make update` after API changes.

### API Machinery
Kubernetes uses a sophisticated API machinery built on:
- **Scheme** - Registry of API types
- **Codecs** - Serialization/deserialization
- **Storage** - etcd backend abstraction
- **Admission** - Validation and mutation webhooks
- **APIServer** - Generic server implementation

Core API machinery lives in `staging/src/k8s.io/apimachinery` and `staging/src/k8s.io/apiserver`.

## Build Environment

### Requirements
- Go 1.25.3 (see `.go-version`)
- Docker with buildx plugin (for containerized builds)
- bash shell
- jq (for JSON processing)

### Environment Variables
- `WHAT` - Specify which component to build/test
- `GOFLAGS` - Extra flags for go commands
- `GOLDFLAGS` - Extra linker flags
- `KUBE_COVER` - Set to 'y' to enable coverage
- `KUBE_TEST_ARGS` - Arguments passed to go test
- `KUBE_VERBOSE` - Verbosity level (default 1)
- `DBG=1` - Build unstripped binaries for debugging

## Common Patterns

### Running Single Tests
```bash
# Unit test
make test WHAT=./pkg/kubelet KUBE_TEST_ARGS='-run ^TestSpecificFunction$'

# Integration test
make test-integration WHAT=./test/integration/pods KUBE_TEST_ARGS='-run ^TestPodCreation$'
```

### Debugging
- Build with `DBG=1` to enable debugger support
- Use `build/shell.sh` to debug build environment issues
- Check `_output/` for build artifacts and logs

### Linting Configuration
- Primary config: `golangci.yaml`
- Additional hints: `golangci-hints.yaml`
- Linter runs as part of `make verify`

## Go Workspace
This repo uses Go workspaces (go.work) to manage the main module and 30+ staging modules. The workspace:
- Enables seamless development across staging repos
- Resolves k8s.io/* imports to local staging directories
- Auto-generated - do not edit manually
