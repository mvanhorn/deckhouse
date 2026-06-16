# E2E Testing with Kyverno Chainsaw

## What is Chainsaw

[Chainsaw](https://kyverno.github.io/chainsaw/) is a declarative e2e testing tool for Kubernetes. Tests are defined in YAML as a sequence of steps with `try`/`catch`/`finally` blocks. Chainsaw creates a temporary namespace per test, applies resources, runs assertions and scripts, and cleans up automatically.

## Installation

**Homebrew (macOS/Linux):**

```bash
brew tap kyverno/chainsaw https://github.com/kyverno/chainsaw
brew install kyverno/chainsaw/chainsaw
```

**Go install:**

```bash
go install github.com/kyverno/chainsaw@latest
```

**Verify:**

```bash
chainsaw version
```

## Prerequisites

- `kubectl` configured with access to a target Kubernetes cluster
- RBAC to create/delete namespaces and workloads, **plus Deckhouse ClusterAdmin-level rights to manage `Descheduler` custom resources** (the `d8:user-authz:descheduler:cluster-admin` role grants `create/patch/delete`; a plain `kubernetes-admin` identity is denied `create deschedulers` and the tests will fail at the CR-apply step)
- Descheduler module enabled with `deschedulingInterval: Frequent` (5m) in ModuleConfig for faster test cycles
- At least 2 schedulable worker nodes for the StatefulSet and utilization tests

## Running Tests

The recommended way to run tests is via [go-task](https://taskfile.dev/) using the provided `Taskfile.yml`:

```bash
# Run all tests
task run

# Run a specific test
task run:low-node-utilization
task run:high-node-utilization
task run:exclude-namespaces
task run:statefulset
task run:statefulset-remove-duplicates
task run:statefulset-pdb-blocks-eviction
task run:statefulset-pdb-allows-one-disruption
task run:statefulset-single-replica-eviction
task run:minreplicas-not-supported

# Run with verbose output
task run:verbose

# Dry run — validate YAML without executing (no cluster required)
task dry-run

# Pause on failure for debugging
task run:debug
```

Alternatively, you can use `chainsaw` directly:

```bash
# Run all tests
chainsaw test --test-dir ./tests/

# Run a specific test
chainsaw test --test-dir ./tests/low-node-utilization/

# Skip cleanup — keep created resources for debugging
chainsaw test --test-dir ./tests/low-node-utilization/ --skip-delete

# Run tests in parallel (default: unlimited)
chainsaw test --test-dir ./tests/ --parallel 4

# Override timeouts
chainsaw test --test-dir ./tests/low-node-utilization/ \
  --apply-timeout 60s \
  --assert-timeout 300s \
  --exec-timeout 300s
```

**Key concepts:**
- `try` — main operations; step fails if any operation fails
- `catch` — runs only on failure (diagnostics collection)
- `cleanup` — runs after step completes (resource deletion)
- `$NAMESPACE` — auto-generated test namespace, available in scripts
- `bindings` — named values (including `x_k8s_list`/`x_k8s_get` lookups) usable in manifests with `template: true` and in assertions
- `concurrent: false` — tests that reshape pod placement on shared nodes are marked non-concurrent so they cannot interfere with each other

## Test Structure

```text
e2e/
  Taskfile.yml                  — Task runner for convenient test execution
  e2e.yaml                      — E2E configuration (constraints, etc.; empty for now)
  tests/
    common/                     — Shared files used by multiple tests
      assert-descheduler-ready.yaml             — Shared assertion: descheduler deployment is ready
      assert-descheduler-rollout-complete.yaml  — Shared assertion: rollout finished, pod runs current policy
      sts-pinned.yaml                           — Shared StatefulSet template placed on a target node
      sts-unpin-patch.yaml                      — Shared patch clearing the template nodeName
    low-node-utilization/
      chainsaw-test.yaml             — Test definition
      manifests/                         — K8s manifests applied by the test
      low_node_utilization.md        — Test documentation
    high-node-utilization/
      chainsaw-test.yaml
      manifests/
      high_node_utilization.md
    exclude-namespaces-from-processing/
      chainsaw-test.yaml
      manifests/
      exclude_namespaces_from_processing.md
    statefulset-remove-duplicates/
      chainsaw-test.yaml
      manifests/
      statefulset_remove_duplicates.md
    statefulset-pdb-blocks-eviction/
      chainsaw-test.yaml
      manifests/
      statefulset_pdb_blocks_eviction.md
    statefulset-pdb-allows-one-disruption/
      chainsaw-test.yaml
      manifests/
      statefulset_pdb_allows_one_disruption.md
    statefulset-single-replica-eviction/
      chainsaw-test.yaml
      manifests/
      statefulset_single_replica_eviction.md
    descheduler-minreplicas-not-supported/
      chainsaw-test.yaml
      manifests/
      descheduler_minreplicas_not_supported.md
```

## Available Tests

| Task command | Test directory | Description |
|--------------|----------------|-------------|
| `task run:low-node-utilization` | `tests/low-node-utilization/` | Validates LowNodeUtilization plugin rebalances pods from overloaded nodes |
| `task run:high-node-utilization` | `tests/high-node-utilization/` | Validates HighNodeUtilization plugin consolidates pods to fewer nodes |
| `task run:exclude-namespaces` | `tests/exclude-namespaces-from-processing/` | Validates Deckhouse patch preventing eviction of pods in `d8-*` and `kube-system` namespaces |
| `task run:statefulset-remove-duplicates` | `tests/statefulset-remove-duplicates/` | StatefulSet without PDB: RemoveDuplicates evicts duplicate pods and they spread across nodes |
| `task run:statefulset-pdb-blocks-eviction` | `tests/statefulset-pdb-blocks-eviction/` | StatefulSet + PDB `maxUnavailable: 0`: every eviction is blocked, pods stay in place |
| `task run:statefulset-pdb-allows-one-disruption` | `tests/statefulset-pdb-allows-one-disruption/` | StatefulSet + PDB `maxUnavailable: 1`: evictions are serialized, StatefulSet stays available |
| `task run:statefulset-single-replica-eviction` | `tests/statefulset-single-replica-eviction/` | Single-replica StatefulSet is evicted — no `minReplicas` protection exists in Deckhouse |
| `task run:minreplicas-not-supported` | `tests/descheduler-minreplicas-not-supported/` | `spec.minReplicas` cannot be persisted in the CR; manual ConfigMap edits are overwritten by Deckhouse |
