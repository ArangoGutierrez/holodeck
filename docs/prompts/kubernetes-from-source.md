# Kubernetes Installation from Custom Sources

## Overview

This document specifies how Holodeck should support installing Kubernetes from
multiple sources: official releases, git references (commit, branch, tag), and
forks. This enables testing pre-release Kubernetes versions, debugging specific
commits, and validating patches from forks before upstream merge.

## Research Summary

### KIND (Kubernetes IN Docker)

KIND provides the most flexible approach for custom Kubernetes builds:

- **Build from source**: `kind build node-image --type source /path/to/k8s`
- **Build from release**: `kind build node-image --type release v1.31.0`
- **Build from URL/file**: `kind build node-image --type url <tarball-url>`

KIND builds a complete node image containing all Kubernetes components
(kubeadm, kubelet, kubectl, kube-apiserver, etc.) that can be used to create
clusters. This is the recommended pattern for Holodeck integration.

**Reference**: [kind.sigs.k8s.io](https://kind.sigs.k8s.io/docs/user/quick-start/)

### Minikube

Minikube supports version selection via `--kubernetes-version=vX.Y.Z` but only
for official releases. It does not provide built-in tooling for building from
arbitrary branches or commits.

**Reference**: [minikube.sigs.k8s.io](https://minikube.sigs.k8s.io/docs/handbook/config/)

### Kubernetes Release Tooling

The `kubernetes/release` repository contains official release infrastructure:

- **krel**: Release toolbox for creating alpha/beta/RC/official releases
- **kpromo**: Artifact promotion tool
- **Release branches**: Named `release-X.Y` (e.g., `release-1.31`)
- **Build commands**:
  - Full release: `make quick-release`
  - Specific binaries: `make WHAT=cmd/kubeadm cmd/kubelet cmd/kubectl`

**Reference**: [github.com/kubernetes/release](https://github.com/kubernetes/release)

## Proposed Schema

### Types Definition

```go
// K8sSource defines the installation source for Kubernetes.
// +kubebuilder:validation:Enum=release;git;latest
type K8sSource string

const (
    // K8sSourceRelease installs from official releases (default)
    K8sSourceRelease K8sSource = "release"
    // K8sSourceGit installs from a specific git reference
    K8sSourceGit K8sSource = "git"
    // K8sSourceLatest tracks a moving branch at provision time
    K8sSourceLatest K8sSource = "latest"
)

// K8sReleaseSpec defines configuration for release-based installation.
type K8sReleaseSpec struct {
    // Version specifies the Kubernetes version (e.g., "v1.31.0").
    // +required
    Version string `json:"version"`
}

// K8sGitSpec defines configuration for git-based installation.
type K8sGitSpec struct {
    // Repo is the git repository URL.
    // +kubebuilder:default="https://github.com/kubernetes/kubernetes.git"
    // +optional
    Repo string `json:"repo,omitempty"`

    // Ref is the git reference (commit SHA, tag, branch, or PR ref).
    // Examples: "v1.31.0", "refs/tags/v1.31.0", "refs/heads/master",
    //           "abc123", "refs/pull/123/head"
    // +required
    Ref string `json:"ref"`
}

// K8sLatestSpec defines configuration for latest branch tracking.
type K8sLatestSpec struct {
    // Track specifies the branch to track at provision time.
    // +kubebuilder:default=master
    // +optional
    Track string `json:"track,omitempty"`

    // Repo is the git repository URL.
    // +kubebuilder:default="https://github.com/kubernetes/kubernetes.git"
    // +optional
    Repo string `json:"repo,omitempty"`
}

// Updated Kubernetes struct
type Kubernetes struct {
    Install bool `json:"install"`

    // Source determines installation method.
    // +kubebuilder:default=release
    // +optional
    Source K8sSource `json:"source,omitempty"`

    // Release source configuration (when source=release).
    // +optional
    Release *K8sReleaseSpec `json:"release,omitempty"`

    // Git source configuration (when source=git).
    // +optional
    Git *K8sGitSpec `json:"git,omitempty"`

    // Latest source configuration (when source=latest).
    // +optional
    Latest *K8sLatestSpec `json:"latest,omitempty"`

    // KubernetesInstaller specifies the installer to use.
    // +kubebuilder:validation:Enum=kubeadm;kind;microk8s
    // +kubebuilder:default=kubeadm
    KubernetesInstaller string `json:"installer,omitempty"`

    // ... existing fields for backward compatibility ...
    KubeConfig            string   `json:"kubeConfig,omitempty"`
    KubernetesFeatures    []string `json:"Features,omitempty"`
    KubernetesVersion     string   `json:"Version,omitempty"`
    // ... etc ...
}
```

## Installation Strategies by Installer

### kubeadm Installer

When using `source: git` or `source: latest` with kubeadm, Holodeck must:

1. **Clone the repository** at the specified ref
2. **Build Kubernetes binaries**:
   ```bash
   make WHAT="cmd/kubeadm cmd/kubelet cmd/kubectl"
   ```
3. **Install binaries** to `/usr/local/bin/`
4. **Continue with kubeadm init** using built binaries

**Build Requirements:**
- Go 1.22+ (matching kubernetes/kubernetes requirements)
- rsync, make, gcc
- ~4GB disk space for build artifacts

**Build Flow:**
```bash
# Clone repository
git clone --depth 1 ${GIT_REPO} /tmp/kubernetes
cd /tmp/kubernetes
git fetch --depth 1 origin ${GIT_REF}
git checkout FETCH_HEAD

# Install Go (matching version from go.mod)
GO_VERSION=$(grep '^go ' go.mod | awk '{print $2}')
curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz" | \
    sudo tar -C /usr/local -xzf -
export PATH="/usr/local/go/bin:$PATH"

# Build binaries
make WHAT="cmd/kubeadm cmd/kubelet cmd/kubectl"

# Install binaries
sudo install -m 755 _output/bin/{kubeadm,kubelet,kubectl} /usr/local/bin/
```

### KIND Installer

When using `source: git` or `source: latest` with KIND, Holodeck should:

1. **Option A - Build node image** (recommended):
   ```bash
   git clone ${GIT_REPO} /tmp/kubernetes
   cd /tmp/kubernetes && git checkout ${GIT_REF}
   kind build node-image --type source /tmp/kubernetes --image holodeck/k8s:${SHORT_SHA}
   kind create cluster --image holodeck/k8s:${SHORT_SHA}
   ```

2. **Option B - Use KIND with custom binaries**:
   Build binaries separately and mount into KIND containers.

### MicroK8s Installer

MicroK8s uses snap packages with channel-based versioning. Git source
installation is not directly supported. Holodeck should:

1. **For releases**: Map version to snap channel (e.g., `1.31/stable`)
2. **For git sources**: Not supported - recommend using kubeadm or KIND instead

## Multinode Cluster Support

This section describes how custom Kubernetes sources integrate with multinode
cluster provisioning ([Issue #562](https://github.com/NVIDIA/holodeck/issues/562)).

### Challenge: Build Once, Distribute to All

Building Kubernetes from source is resource-intensive:
- **Time**: 10-30 minutes depending on hardware
- **Disk**: ~4GB for build artifacts
- **Memory**: 4GB+ recommended

Building on every node is wasteful. Instead, Holodeck should **build once and
distribute** the binaries to all nodes.

### Strategy 1: Build on First Control-Plane, Distribute via SCP

**Flow:**
```
┌─────────────────────────────────────────────────────────────────────────┐
│  1. Provision all EC2 instances in parallel                             │
├─────────────────────────────────────────────────────────────────────────┤
│  2. On control-plane-1:                                                 │
│     a. Clone repo, checkout ref                                         │
│     b. Build: make WHAT="cmd/kubeadm cmd/kubelet cmd/kubectl"           │
│     c. Create tarball: tar -czf k8s-binaries.tar.gz _output/bin/        │
│     d. kubeadm init --upload-certs                                      │
├─────────────────────────────────────────────────────────────────────────┤
│  3. Distribute binaries (parallel):                                     │
│     - SCP k8s-binaries.tar.gz to control-plane-2, control-plane-3       │
│     - SCP k8s-binaries.tar.gz to worker-1, worker-2, ...                │
├─────────────────────────────────────────────────────────────────────────┤
│  4. On control-plane-2, control-plane-3:                                │
│     a. Extract and install binaries                                     │
│     b. kubeadm join --control-plane                                     │
├─────────────────────────────────────────────────────────────────────────┤
│  5. On worker nodes (parallel):                                         │
│     a. Extract and install binaries                                     │
│     b. kubeadm join                                                     │
└─────────────────────────────────────────────────────────────────────────┘
```

**Template additions for multinode:**
```bash
# On first control-plane (after build)
BINARIES_TARBALL="/tmp/k8s-binaries-${GIT_COMMIT}.tar.gz"
tar -czf "${BINARIES_TARBALL}" -C _output/bin kubeadm kubelet kubectl

# Generate join commands with tokens
kubeadm init --upload-certs --config=/etc/kubernetes/kubeadm-config.yaml
CONTROL_PLANE_JOIN=$(kubeadm token create --print-join-command) \
    --certificate-key $(kubeadm init phase upload-certs --upload-certs 2>/dev/null | tail -1)
WORKER_JOIN=$(kubeadm token create --print-join-command)

# Store join commands for distribution
echo "${CONTROL_PLANE_JOIN}" > /tmp/control-plane-join.sh
echo "${WORKER_JOIN}" > /tmp/worker-join.sh
```

```bash
# On joining nodes (control-plane or worker)
# Binaries already distributed via SCP
tar -xzf /tmp/k8s-binaries-*.tar.gz -C /usr/local/bin/
chmod +x /usr/local/bin/{kubeadm,kubelet,kubectl}

# Execute appropriate join command
bash /tmp/join-command.sh
```

### Strategy 2: Build Locally, Upload to S3 Artifact Storage

For larger clusters or repeated provisioning, pre-building and storing artifacts
is more efficient:

**Flow:**
```
┌─────────────────────────────────────────────────────────────────────────┐
│  Pre-provisioning (on Holodeck host or CI):                             │
│  1. Clone repo, checkout ref                                            │
│  2. Build binaries (cross-compile if needed)                            │
│  3. Upload to S3: s3://holodeck-artifacts/k8s/${COMMIT}/binaries.tar.gz │
├─────────────────────────────────────────────────────────────────────────┤
│  Provisioning (all nodes in parallel):                                  │
│  1. Download from S3                                                    │
│  2. Extract and install binaries                                        │
│  3. Run kubeadm init/join as appropriate                                │
└─────────────────────────────────────────────────────────────────────────┘
```

**Schema addition for artifact caching:**
```go
type K8sGitSpec struct {
    Repo string `json:"repo,omitempty"`
    Ref  string `json:"ref"`

    // ArtifactCache specifies pre-built binary location.
    // If set, download instead of building.
    // +optional
    ArtifactCache *ArtifactCacheSpec `json:"artifactCache,omitempty"`
}

type ArtifactCacheSpec struct {
    // S3 bucket URL for pre-built binaries
    S3URL string `json:"s3url,omitempty"`
    // HTTP URL for pre-built binaries
    URL string `json:"url,omitempty"`
}
```

### Strategy 3: KIND with Custom Node Image (Multinode)

KIND handles multinode elegantly - build one node image, use it for all nodes:

**Flow:**
```
┌─────────────────────────────────────────────────────────────────────────┐
│  1. Build custom node image (once, on Holodeck host):                   │
│     kind build node-image --type source /path/to/k8s                    │
│       --image holodeck/k8s:${COMMIT}                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  2. Create multinode cluster using the image:                           │
│     kind create cluster --image holodeck/k8s:${COMMIT}                  │
│       --config kind-multinode.yaml                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

**KIND multinode config with custom image:**
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: control-plane
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
```

Since KIND runs on a single EC2 instance with Docker, the node image is local
and immediately available to all "nodes" (containers).

### Multinode YAML Examples

#### HA Cluster with Custom K8s from Fork

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: ha-custom-k8s
spec:
  provider: aws
  auth:
    keyName: my-key
    publicKey: ~/.ssh/id_rsa.pub
    privateKey: ~/.ssh/id_rsa
  cluster:
    nodes:
      controlPlane:
        count: 3
        instanceType: m5.xlarge
        dedicated: true
      workers:
        count: 5
        instanceType: g4dn.xlarge
    highAvailability:
      enabled: true
      etcdTopology: stacked
  kubernetes:
    install: true
    source: git
    git:
      repo: https://github.com/myorg/kubernetes.git
      ref: feature/dra-scheduling-fix
    installer: kubeadm
  nvidiaDriver:
    install: true
  containerRuntime:
    install: true
    name: containerd
  nvidiaContainerToolkit:
    install: true
```

#### KIND Multinode with Custom K8s

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: kind-multinode-custom
spec:
  provider: aws
  instance:
    type: m5.4xlarge  # Larger instance for multiple KIND nodes
    region: us-west-2
    os: ubuntu-22.04
  kubernetes:
    install: true
    source: git
    git:
      ref: refs/pull/123456/head  # Test a PR
    installer: kind
    kindConfig: |
      kind: Cluster
      apiVersion: kind.x-k8s.io/v1alpha4
      nodes:
        - role: control-plane
        - role: worker
        - role: worker
```

#### Pre-built Artifacts for Large Cluster

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: large-cluster-prebuilt
spec:
  cluster:
    nodes:
      controlPlane:
        count: 3
        instanceType: m5.xlarge
      workers:
        count: 20
        instanceType: g4dn.xlarge
  kubernetes:
    install: true
    source: git
    git:
      repo: https://github.com/kubernetes/kubernetes.git
      ref: v1.32.0-alpha.2
      artifactCache:
        s3url: s3://holodeck-artifacts/k8s/v1.32.0-alpha.2/
    installer: kubeadm
```

### Multinode Implementation Considerations

#### Node Role-Specific Provisioning

| Step | Control-Plane-1 | Control-Plane-N | Worker |
|------|-----------------|-----------------|--------|
| Install Go | ✅ | ❌ | ❌ |
| Clone repo | ✅ | ❌ | ❌ |
| Build binaries | ✅ | ❌ | ❌ |
| Receive binaries | ❌ | ✅ (SCP) | ✅ (SCP) |
| Install binaries | ✅ | ✅ | ✅ |
| kubeadm init | ✅ | ❌ | ❌ |
| kubeadm join --cp | ❌ | ✅ | ❌ |
| kubeadm join | ❌ | ❌ | ✅ |

#### Parallelization Opportunities

1. **Instance provisioning**: All EC2 instances in parallel
2. **Base package installation**: All nodes in parallel
3. **Binary distribution**: All joining nodes in parallel (after build)
4. **Worker join**: All workers in parallel (after control-plane ready)

#### Error Handling for Multinode

| Error | Resolution |
|-------|------------|
| Build fails on CP-1 | Abort entire provisioning, report build logs |
| SCP distribution fails | Retry with backoff, fallback to direct download |
| Join fails on CP-N | Retry, then mark node failed, continue with others |
| Join fails on worker | Retry, mark failed, continue (cluster still functional) |

### Dependency on Issue #562

Full multinode support with custom Kubernetes sources requires the multinode
infrastructure from [Issue #562](https://github.com/NVIDIA/holodeck/issues/562):

| #562 Component | Required for Custom K8s |
|----------------|------------------------|
| Multi-instance AWS provisioning | ✅ Binary distribution targets |
| kubeadm join templates | ✅ Join commands for custom binaries |
| Token management | ✅ Secure distribution of join tokens |
| NLB for HA control-plane | ✅ API endpoint for custom builds |
| Node status tracking | ✅ Track build/distribution progress |

**Implementation Order:**
1. Single-node custom K8s (this spec, Phase 1-3)
2. Multinode infrastructure (#562, Phase 1-4)
3. Multinode + custom K8s integration (combined)

## YAML Configuration Examples

### Release (Default - Current Behavior)

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: k8s-release
spec:
  kubernetes:
    install: true
    source: release
    release:
      version: v1.31.0
    installer: kubeadm
```

### Git - Official Tag

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: k8s-tag-test
spec:
  kubernetes:
    install: true
    source: git
    git:
      ref: v1.32.0-alpha.1  # Pre-release testing
    installer: kubeadm
```

### Git - Specific Commit

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: k8s-bisect
spec:
  kubernetes:
    install: true
    source: git
    git:
      ref: abc123def456  # Commit SHA for bisecting
    installer: kubeadm
```

### Git - Branch from Fork

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: k8s-feature-test
spec:
  kubernetes:
    install: true
    source: git
    git:
      repo: https://github.com/myorg/kubernetes.git
      ref: feature/my-cool-feature
    installer: kubeadm
```

### Git - Pull Request

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: k8s-pr-review
spec:
  kubernetes:
    install: true
    source: git
    git:
      ref: refs/pull/123456/head  # Test a PR before merge
    installer: kubeadm
```

### Latest - Track Master

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: k8s-bleeding-edge
spec:
  kubernetes:
    install: true
    source: latest
    latest:
      track: master  # Track latest development
    installer: kubeadm
```

### Latest - Track Release Branch

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: k8s-release-track
spec:
  kubernetes:
    install: true
    source: latest
    latest:
      track: release-1.31  # Track release branch for patches
    installer: kubeadm
```

### Combined - GPU Testing with Custom K8s

```yaml
apiVersion: holodeck.nvidia.com/v1alpha1
kind: Environment
metadata:
  name: gpu-with-custom-k8s
spec:
  provider: aws
  auth:
    keyName: my-key
    publicKey: ~/.ssh/id_rsa.pub
    privateKey: ~/.ssh/id_rsa
  instance:
    type: g4dn.xlarge
    region: us-west-2
    os: ubuntu-22.04
  nvidiaDriver:
    install: true
  containerRuntime:
    install: true
    name: containerd
  nvidiaContainerToolkit:
    install: true
    source: latest
    latest:
      track: main
  kubernetes:
    install: true
    source: git
    git:
      repo: https://github.com/myorg/kubernetes.git
      ref: feature/dra-improvements
    installer: kubeadm
```

## Provenance Tracking

All installation sources should write provenance information to
`/etc/kubernetes/PROVENANCE.json`:

```json
{
  "source": "git",
  "repo": "https://github.com/kubernetes/kubernetes.git",
  "ref": "refs/heads/master",
  "commit": "abc12345",
  "version": "v1.32.0-alpha.0-12-gabc12345",
  "installer": "kubeadm",
  "installed_at": "2026-01-15T10:30:00-05:00"
}
```

The version string for git builds should be derived from:
```bash
git describe --tags --always --dirty
```

## Implementation Plan

### Phase 1: Schema Updates (Single-Node Foundation)

1. Add `K8sSource`, `K8sGitSpec`, `K8sLatestSpec` types to `api/holodeck/v1alpha1/types.go`
2. Update `Kubernetes` struct with new fields
3. Add validation for mutual exclusivity of source configurations
4. Generate deepcopy functions

### Phase 2: Git Ref Resolution

1. Extend `pkg/gitref/resolver.go` to support kubernetes/kubernetes repository
2. Add `DefaultK8sRepo` constant
3. Ensure ref resolution works for all supported formats

### Phase 3: Template Implementation (Single-Node)

1. Create `kubernetesGitTemplate` in `pkg/provisioner/templates/kubernetes.go`
2. Create `kubernetesLatestTemplate` for branch tracking
3. Update `NewKubernetes()` to handle source selection
4. Implement `SetResolvedCommit()` for resolved SHA tracking

**Template variables needed:**
```go
type KubernetesGit struct {
    // Existing fields
    Version             string
    Arch               string
    K8sEndpointHost    string
    // ... etc ...

    // New fields for git source
    Source              string  // "release", "git", "latest"
    GitRepo             string
    GitRef              string
    GitCommit           string  // Resolved short SHA
    TrackBranch         string  // For latest source
}
```

### Phase 4: KIND Integration

1. Update KIND template to support custom node images
2. Add `kind build node-image` step when source is git/latest
3. Handle image tagging with commit SHA

### Phase 5: Testing (Single-Node)

1. **Unit tests**: Schema validation, source detection, version parsing
2. **Integration tests**: Build from tag, branch, commit
3. **E2E tests**: Full cluster deployment with custom sources

### Phase 6: Documentation

1. Update `docs/guides/` with Kubernetes source documentation
2. Add examples to `examples/` directory
3. Update command documentation for new options

### Phase 7: Multinode Integration (Depends on #562)

**Prerequisites**: Phases 1-3 of [Issue #562](https://github.com/NVIDIA/holodeck/issues/562)
must be implemented first.

1. **Binary distribution infrastructure**
   - Add tarball creation step after build on control-plane-1
   - Implement SCP-based binary distribution to other nodes
   - Add parallel distribution with retry logic

2. **Multinode kubeadm templates**
   - Create `kubernetesGitJoinTemplate` for joining nodes
   - Template receives pre-built binaries path
   - Separate templates for control-plane join vs worker join

3. **Artifact caching (optional)**
   - Add `ArtifactCacheSpec` to schema
   - Implement S3 upload after build
   - Implement S3 download on all nodes for cached builds

4. **KIND multinode with custom images**
   - Extend KIND template to build node image first
   - Pass custom image to multinode KIND config

### Phase 8: Multinode Testing

1. **Integration tests**:
   - 3-node cluster (1 CP + 2 workers) with git source
   - 5-node HA cluster (3 CP + 2 workers) with git source
   - KIND multinode with custom image

2. **Performance tests**:
   - Measure build + distribution time vs release installation
   - Validate parallelization efficiency

3. **Failure mode tests**:
   - Build failure handling
   - Distribution failure recovery
   - Partial cluster functionality with failed workers

## Validation Rules

1. **Mutual exclusivity**: Only one of `release`, `git`, `latest` should be set
2. **Required fields**:
   - `release.version` when source is `release`
   - `git.ref` when source is `git`
3. **Default values**:
   - `source` defaults to `release` for backward compatibility
   - `git.repo` defaults to official kubernetes repo
   - `latest.track` defaults to `master`
   - `latest.repo` defaults to official kubernetes repo

## Error Handling

| Error | Resolution |
|-------|------------|
| Git ref not found | Clear error message with ref and repo |
| Build failure | Show build logs, suggest checking Go version |
| Incompatible installer | Error if git source with MicroK8s |
| Network timeout | Retry with exponential backoff |

## Backward Compatibility

The existing `KubernetesVersion` field is preserved for backward compatibility:

```yaml
spec:
  kubernetes:
    install: true
    Version: v1.31.0  # Legacy field, equivalent to release.version
```

When `source` is not specified and `Version` is set, it defaults to `release`
source behavior.

## CLI Enhancements

Future CLI improvements could include:

```bash
# List available Kubernetes versions
holodeck k8s list-versions

# List branches from a repo
holodeck k8s list-branches --repo https://github.com/kubernetes/kubernetes

# Validate a ref before provisioning
holodeck k8s validate-ref --ref abc123 --repo https://github.com/myorg/kubernetes
```

## Security Considerations

1. **Repository validation**: Only allow GitHub URLs (https:// or git@)
2. **Ref sanitization**: Validate ref format before use in git commands
3. **Build isolation**: Build in temporary directory, clean up after
4. **Checksum verification**: For release artifacts, verify SHA256

## Related Issues

- **[Issue #562](https://github.com/NVIDIA/holodeck/issues/562)**: Multinode Cluster
  Support with High Availability Control Plane - Required for multinode custom K8s
- **[Issue #567](https://github.com/NVIDIA/holodeck/issues/567)**: Provision Core
  Dependencies from Multiple Sources - Parent epic for this feature
- **[Issue #566](https://github.com/NVIDIA/holodeck/issues/566)**: NVIDIA Container
  Toolkit Installation from Multiple Sources - Template for implementation pattern

## References

- [KIND Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [KIND Configuration](https://kind.sigs.k8s.io/docs/user/configuration/)
- [KIND Building Images](https://kind.sigs.k8s.io/docs/user/quick-start/#building-images)
- [Minikube Configuration](https://minikube.sigs.k8s.io/docs/handbook/config/)
- [Kubernetes Release Process](https://kubernetes.io/releases/release/)
- [kubernetes/release Repository](https://github.com/kubernetes/release)
- [Kubernetes Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/)
- [Building Kubernetes from Source](https://github.com/kubernetes/kubernetes/blob/master/build/README.md)
- [kubeadm HA Topology](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/ha-topology/)