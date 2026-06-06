# PerfContract

**An open standard for writing runtime performance requirements down in code, in a form both humans and AI coding agents can read, enforce, and reason about.**

## The Problem

Every team has performance requirements. Almost none of them have those requirements written down anywhere a machine can check.

Think about where performance constraints actually live in a codebase today:

- Prose in a design doc someone wrote eighteen months ago that nobody re-reads
- Tribal knowledge in the head of the one engineer who remembers why the cache exists
- SLOs defined in a monitoring dashboard - discovered *after* the code ships, when the alert fires
- Implicit assumptions that surface only when something breaks under load in production

None of these travel with the code. None of them are checkable before a regression lands. We've spent decades making *correctness* machine-checkable - types, tests, linters, CI - while *performance intent* stayed locked in human prose and folklore.

AI coding agents turn that gap into an active liability. An agent will happily write functionally correct code that allocates on every pass of a hot loop, drops a synchronous network call inside a 50ms budget, or pulls in a dependency that doubles cold-start time - because nothing told it not to. The constraints exist; they just aren't anywhere the agent can see them. There's no standard way to tell it, so we don't.

PerfContract closes that gap. It's a structured, portable format for declaring performance budgets once, next to the code, so the same declaration serves a human reviewer reading a diff and an agent generating, testing, or auditing that code. It's tool-agnostic and version-controlled; the budget lives with the source, not in a wiki that drifts out of date.

## Status

RFC Draft - v0.1

## The Solution

PerfContract defines:
1. A **vocabulary** — a typed registry of performance metrics
2. A **grammar** — structured YAML for service/system scope, natural shorthand for inline
3. **Placement conventions** — where budgets live, how they cascade
4. **Verification hints** — how to test what you declared

Budgets are LLM-native (every objective can carry a `rationale` the agent reasons over) and inherit across scope hierarchies, so a system-level target cascades down to the functions that have to honor it.

## Core Principles

| # | Principle | Rationale |
|---|-----------|-----------|
| 1 | Environment is required | A budget without deployment context is a wish list |
| 2 | Inheritance by default | Child scopes inherit from parents, override tighter |
| 3 | Two representations, one grammar | Full YAML for service/system, shorthand for inline |
| 4 | Extensible taxonomy | Core registry covers 80%, `custom.*` for the rest |
| 5 | Declarative, not prescriptive | The spec defines data; agent behavior lives in tooling |
| 6 | LLM-native | `rationale` fields give agents context for tradeoff reasoning |
| 7 | Tolerance bands, not hard lines | Target / Tolerance / Stretch — systems aren't binary |

---

## Schema Reference

### Top-Level Structure

```yaml
perfcontract: v1
$schema: "https://raw.githubusercontent.com/ahartzog/perfcontract/main/v1/schema.json"

metadata:
  name: <service-or-system-name>
  owner: <team>
  last_reviewed: <date>
  description: <optional one-line summary>

environments:
  <env-name>:
    compute: { ... }
    deployment: { ... }
    network: { ... }
    constraints: [ ... ]
    rationale: <string>

budgets:
  - scope: <scope-expression>
    environment: <env-name>
    objectives:
      - metric: <metric-name>
        target: <value>
        tolerance: <value>
        stretch: <value>
        rationale: <string>
        verification: { ... }
```

### Environment Block (required)

Every performance budget must declare its target environment. A budget without environment context cannot be meaningfully interpreted by an agent or verified by a test.

Budgets reference an environment by name, and that name must be declared in the same file's `environments` block. JSON Schema validates that `environment` is present but cannot verify the name actually resolves — enforcing that reference is a linter/validator responsibility.

**Design envelope, not deployment manifest.** The environment block describes the conditions you are *designing for* — not a live snapshot of what's currently deployed. If your pods get upgraded from 4Gi to 8Gi but your budget still says 4Gi, that's correct: you're still designing for the constrained case. Drift between declared environment and actual deployment is acceptable and expected. The environment captures engineering intent ("we must work under these conditions"), not infrastructure state.

Update environment declarations when the design envelope itself changes — e.g., you're no longer targeting edge nodes, or the minimum hardware spec has been revised.

```yaml
environments:
  edge-node:
    compute:
      cpu: "2 vCPU"
      memory: "4Gi"
      architecture: arm64         # optional
    deployment:
      model: kubernetes           # kubernetes | bare-metal | lambda | ecs | edge | embedded
      replicas: 3                 # optional
      region: us-west-2           # optional
    network:                      # optional
      bandwidth: "1Gbps"
      latency_to_db: "2ms"
    constraints:                  # optional, free-form
      - "Shared tenancy — noisy neighbor risk"
      - "No GPU available"
    rationale: >                  # optional but strongly recommended
      Regional edge nodes with limited compute.
      Memory is the primary constraint; latency budget
      assumes co-located database.
```

Named environments allow a single service to declare different budgets per deployment target:

```yaml
environments:
  edge-node:
    compute: { cpu: "2 vCPU", memory: "4Gi" }
    rationale: "Constrained edge deployment"
  cloud-primary:
    compute: { cpu: "8 vCPU", memory: "32Gi" }
    rationale: "Production cloud — optimize for throughput"
```

### Environment Field Dictionaries

Each sub-block of environment has a typed schema. All fields within sub-blocks are optional unless marked required.

#### `compute`

| Field | Type | Units/Values | Description |
|-------|------|--------------|-------------|
| `cpu` | string | vCPU notation ("2 vCPU", "500m") | Available CPU budget |
| `memory` | string | IEC binary (Ki, Mi, Gi) | Available memory budget |
| `architecture` | enum | amd64, arm64, riscv64 | Target CPU architecture |
| `gpu` | string | model or "none" | GPU availability and type |
| `storage` | string | IEC binary (Gi, Ti) | Available local storage |
| `storage_type` | enum | ssd, hdd, nvme, ephemeral | Storage medium characteristics |

#### `deployment`

| Field | Type | Units/Values | Description |
|-------|------|--------------|-------------|
| `model` | enum | kubernetes, bare-metal, lambda, ecs, edge, embedded | Deployment substrate |
| `replicas` | integer | count | Expected replica count |
| `region` | string | free-form (us-west-2, eu-central-1) | Deployment region |
| `scaling` | enum | fixed, hpa, keda, manual | Autoscaling strategy |
| `tenancy` | enum | dedicated, shared, burstable | Resource isolation model |
| `orchestrator` | string | free-form (k8s 1.28, nomad) | Orchestrator version |

#### `network`

| Field | Type | Units/Values | Description |
|-------|------|--------------|-------------|
| `bandwidth` | string | bps notation (100Mbps, 1Gbps) | Available link capacity |
| `latency` | string | ms, s | RTT to primary dependencies |
| `latency_to_db` | string | ms, s | RTT to database |
| `latency_to_cache` | string | ms, s | RTT to cache layer |
| `jitter` | string | ms | Latency variance (± value) |
| `packet_loss` | string | % | Expected packet loss rate |
| `mtu` | integer | bytes | Maximum transmission unit |
| `topology` | enum | co-located, cross-az, cross-region, satellite-link, mesh | Network topology class |
| `dns_resolution` | string | ms | DNS lookup latency |

### Budgets Block

Each budget entry declares objectives for a given scope and environment:

```yaml
budgets:
  - scope: service
    environment: edge-node
    objectives:
      - metric: latency.p99
        target: 50ms
        tolerance: 100ms
        stretch: 25ms
      - metric: memory.resident
        target: 256Mi
        tolerance: 384Mi
      - metric: throughput.rps
        target: 500
        tolerance: 200
        stretch: 1000
```

### Scope Expressions

Scopes form a hierarchy. Each level can declare its own budgets and inherits from parents.

| Scope Level | Expression Syntax | Example |
|-------------|-------------------|---------|
| System | `scope: system` | Multi-service aggregate |
| Service | `scope: service` | Single service process |
| Endpoint | `scope: endpoint:<path>` | `endpoint:/api/v1/plan` |
| Stream | `scope: stream:<name>` | `stream:VehiclePositions` |
| Component | `scope: component:<name>` | `component:route-cache` |
| Class | `scope: class:<name>` | `class:PlanGenerator` |
| Function | `scope: function:<name>` | `function:generatePlan` |

**Stream scope:** For long-lived server-streaming RPCs and pub/sub patterns that don't fit request/response semantics. Stream budgets express subscriber fan-out, delivery latency, and backpressure characteristics rather than per-request latency.

**Inheritance rules:**
- Child scopes inherit environment from their parent unless overridden
- Child scopes inherit budget objectives from their parent
- Children can only **tighten** budgets (lower targets), not relax them
- Explicit `override: true` on an objective bypasses this constraint (with required rationale)

### Objectives

Each objective declares a metric with a three-value tolerance band:

```yaml
- metric: latency.p99
  target: 50ms        # Design goal — what we're optimizing for
  tolerance: 100ms    # Acceptable under degraded conditions
  stretch: 25ms       # Exceptional — what we'd celebrate
  rationale: "User-facing endpoint; 50ms keeps UI responsive"
```

**Defaults when omitted** (the multiplier direction depends on metric polarity):
- `tolerance` — always the *degraded* direction: `2× target` for lower-is-better metrics (latency, memory); `0.5× target` for higher-is-better metrics (throughput).
- `stretch` — always the *favorable* direction: `0.5× target` for lower-is-better; `2× target` for higher-is-better.
- `rationale`: inherited from parent scope

### Parameterized Budgets (`at_scale`)

Performance often degrades as a function of input size. The `at_scale` block declares what conditions the budget assumes:

```yaml
- metric: latency.p99
  target: 100ms
  tolerance: 150ms
  at_scale:
    stop_count: 1500
  rationale: "Batch re-optimization must finish within the 200ms dispatch cycle at expected fleet load"

- metric: latency.p99
  target: 200ms
  at_scale:
    plan_stops: 1
    branch_depth: 0
  rationale: "Single-stop, no-branch plan — the common case"
```

`at_scale` keys are free-form (domain-specific). The schema validates the structure but not the key names — this is intentionally extensible. Common patterns:
- `entity_count`, `subscriber_count`, `queue_depth` — cardinality dimensions
- `payload_size`, `message_count` — volume dimensions
- `branch_depth`, `hop_count` — complexity dimensions

Multiple budgets for the same metric at different scales are valid:

```yaml
- metric: latency.p99
  target: 50ms
  at_scale: { entity_count: 1000 }
- metric: latency.p99
  target: 200ms
  at_scale: { entity_count: 15000 }
- metric: latency.p99
  target: 500ms
  at_scale: { entity_count: 50000 }
```

### Dependency Budgets

When a service's latency budget depends on downstream services responding within a certain time, declare `dependencies`:

```yaml
budgets:
  - scope: endpoint:/api/v1/plans/dispatch
    environment: production
    dependencies:
      - service: dispatch-service
        call: AssignDriver
        latency.p99: 300ms
        rationale: "DispatchPlan calls dispatch synchronously; our 500ms budget assumes it responds in <300ms"
      - service: maps-service
        call: ComputeRouteMatrix
        latency.p99: 300ms
        rationale: "Route optimization budget assumes the maps service responds within 300ms for the batch matrix path"
    objectives:
      - metric: latency.p99
        target: 500ms
        tolerance: 2000ms
```

Dependency budgets are informational — they document the assumptions your budget relies on. They enable agents to:
- Understand where latency is consumed vs. where it's controllable
- Flag when a dependency's actual performance would violate your budget
- Reason about whether a proposed change affects a dependency call count

### Verification Hints

Optional metadata suggesting how to test an objective:

```yaml
verification:
  method: load_test           # load_test | benchmark | profile | trace | static_analysis
  tool_hint: k6               # free-form; e.g. k6, go_bench, pprof, criterion, jmh, pytest-benchmark, hyperfine
  conditions: "50 concurrent users, sustained 60s, warm cache"
```

**Default inference when omitted:**
- Function scope → `benchmark`
- Endpoint scope → `load_test`
- Service scope → `load_test` (integration)
- Memory/allocation → `profile`

---

## Metric Registry (v1)

The registry defines canonical metric names, their valid scopes, and units.

### `latency.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `latency.p50` | Median latency | ms, s | function, component, endpoint, service, system |
| `latency.p95` | 95th percentile | ms, s | function, component, endpoint, service, system |
| `latency.p99` | 99th percentile | ms, s | function, component, endpoint, service, system |
| `latency.max` | Maximum observed | ms, s | endpoint, service, system |
| `latency.cold_start` | First-invocation latency | ms, s | function, service |

### `throughput.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `throughput.rps` | Requests per second | count/s | endpoint, service, system |
| `throughput.records_per_sec` | Records processed per second | count/s | component, service, system |
| `throughput.bytes_per_sec` | Data throughput | B/s, Ki/s, Mi/s | component, service, system |

### `memory.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `memory.resident` | RSS / resident set size | Ki, Mi, Gi | service, system |
| `memory.heap_peak` | Peak heap allocation | Ki, Mi, Gi | component, class, service |
| `memory.allocation` | Per-operation allocation | B, Ki | function, component |
| `memory.allocation_rate` | Allocation velocity | Ki/s, Mi/s | component, service |

### `cpu.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `cpu.utilization` | CPU usage percentage | % | service, system |
| `cpu.cores` | Core count budget | count | service, system |
| `cpu.time_per_op` | CPU time per operation | ms, µs | function, component |

### `io.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `io.disk_read` | Disk read throughput | B/s, Ki/s, Mi/s | component, service |
| `io.disk_write` | Disk write throughput | B/s, Ki/s, Mi/s | component, service |
| `io.connections` | Active connections | count | service, system |
| `io.file_descriptors` | Open file descriptors | count | service, system |

### `concurrency.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `concurrency.max_parallel` | Max concurrent operations | count | component, service |
| `concurrency.queue_depth` | Backlog depth | count | component, service |
| `concurrency.contention` | Lock contention rate | % | component, class |

### `startup.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `startup.time` | Time to first ready | ms, s | service |
| `startup.ready_time` | Time to pass health check | ms, s | service |

### `queue.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `queue.depth` | Current queue/channel depth | count | component, service |
| `queue.utilization` | Queue fill level relative to capacity | % | component, service |
| `queue.drain_rate` | Consumer processing rate | count/s | component |
| `queue.drop_rate` | Messages dropped due to full queue | count/s | component, service |
| `queue.backpressure_duration` | Time producers are blocked by full queue | ms | component |

### `freshness.*`

| Metric | Description | Units | Valid Scopes |
|--------|-------------|-------|--------------|
| `freshness.max_staleness` | Maximum age of data delivered to consumers | ms, s | component, service |
| `freshness.delivery_lag` | Time from event occurrence to consumer receipt | ms, s | stream, service |
| `freshness.sync_age` | Age of the oldest unsynced state after restart | s | service |

### `custom.*`

User-defined metrics in the `custom` namespace:

```yaml
- metric: custom.plan_quality_score
  target: 0.95
  tolerance: 0.90
  rationale: "Domain-specific quality metric from the routing team"
```

Custom metrics must declare their own units in the objective.

---

## Shorthand Syntax

For inline use in source code comments and CLAUDE.md sections.

### Grammar

```
@perfcontract <metric> <operator> <value> [, <metric> <operator> <value>]* [(key: value, ...)]
```

Supported operators: `<`, `>`, `<=`, `>=`

### Inline Environment Declaration

In CLAUDE.md and source comments, an environment can be declared inline with a compact form:

```
environment: <name> { <field>: <value>, ... }
```

Two compact conventions are permitted in shorthand only (the full YAML always uses the canonical forms):
- No space in compute values: `2vCPU` is equivalent to the canonical `2 vCPU`
- `arch` is an accepted alias for `architecture`

Example: `environment: edge-node { cpu: 2vCPU, memory: 4Gi, arch: arm64, model: kubernetes }`

### In Source Code

```go
// @perfcontract latency.p99 < 15ms, memory.allocation < 4Ki (env: edge-node)
func generatePlan(ctx context.Context, req *PlanRequest) (*Plan, error) {
```

```rust
/// @perfcontract latency.p99 < 1ms, cpu.time_per_op < 500µs
fn lookup_entity(id: EntityId) -> Option<&Entity> {
```

```python
# @perfcontract memory.allocation < 1Ki, latency.p99 < 5ms (tolerance: 10ms)
def process_event(event: Event) -> Result:
```

### In CLAUDE.md

```markdown
## Performance Budgets

environment: edge-node { cpu: 2vCPU, memory: 4Gi, model: kubernetes }

- service: latency.p99 < 50ms, memory.resident < 256Mi, throughput.rps > 500
- endpoint:/api/v1/plan: latency.p99 < 30ms (tolerance: 60ms)
- function:generatePlan: latency.p99 < 15ms, memory.allocation < 4Ki
- component:stop-ingest: throughput.records_per_sec > 10000, memory.heap_peak < 512Mi
```

### Inheritance in Shorthand

Inline annotations inherit environment from:
1. Nearest `@perfcontract-env` comment in the same file
2. Nearest `.perfcontract.yml` in directory tree
3. CLAUDE.md declarations

```go
// @perfcontract-env edge-node
package planner

// @perfcontract latency.p99 < 15ms
// (inherits edge-node environment from package-level declaration)
func generatePlan(...) { ... }
```

---

## File Placement

### Resolution Order

Agents discover applicable budgets by searching (in order):

1. Inline `@perfcontract` annotation on the current function/class
2. Nearest `.perfcontract.yml` in the directory tree (walking up)
3. Root `.perfcontract.yml`
4. CLAUDE.md `## Performance Budgets` section (at any directory level)

All matching budgets apply — the most specific scope wins for any given metric.

### Conventions

| Project Shape | Recommended Placement |
|---------------|----------------------|
| Single service | `.perfcontract.yml` at repo root |
| Monorepo | `perfcontract/` directory at root + per-service `.perfcontract.yml` |
| Small project | `## Performance Budgets` section in CLAUDE.md |
| Library | Inline `@perfcontract` on public API functions |

---

## Agent Behavior Guide

This section is informational — it describes how agents should consume PerfContract declarations. It is not normative (agents may implement different strategies).

### Design-Time Guidance

When writing code within a scope that has declared budgets, agents should:
- Factor budgets into architectural decisions (caching strategy, data structure choice, I/O patterns)
- Consider the environment context when making tradeoffs
- Respect the inheritance hierarchy — a function inside a 50ms service shouldn't introduce a 40ms blocking call
- Use `rationale` fields to understand *why* a budget exists, not just its value

### Test Generation

When budgets include verification hints, agents should generate tests that:
- Assert against the `target` value (not tolerance — tolerance is for degraded conditions)
- Use the suggested `tool_hint` if available
- Reproduce the declared `conditions`
- Fail clearly with a message referencing the budget declaration

When no verification hint exists, agents should infer:
- Function → benchmark test
- Endpoint → load test
- Service → integration load test
- Memory metrics → profiling assertion

### Code Review / Audit

When reviewing code changes, agents should:
- Check whether new code paths affect budgeted scopes
- Flag changes that could push metrics past `target` toward `tolerance`
- Warn explicitly when changes risk breaching `tolerance`

---

## Examples

### Full Example: Delivery Route Planner

```yaml
perfcontract: v1

metadata:
  name: route-planner
  owner: logistics-platform
  last_reviewed: 2026-06-01

environments:
  edge-on-prem:
    compute: { cpu: "2 vCPU", memory: "4Gi", architecture: arm64 }
    deployment: { model: kubernetes, replicas: 3 }
    network: { bandwidth: "1Gbps", latency_to_db: "2ms" }
    constraints:
      - "Shared tenancy — noisy neighbor risk"
      - "Intermittent connectivity to upstream services"
    rationale: >
      Regional depot edge deployment. Memory is the binding constraint.
      Latency budgets assume co-located database with 2ms RTT.

  cloud-dev:
    compute: { cpu: "8 vCPU", memory: "32Gi" }
    deployment: { model: kubernetes, replicas: 1 }
    rationale: "Development environment — relaxed budgets for iteration speed"

budgets:
  - scope: service
    environment: edge-on-prem
    objectives:
      - metric: memory.resident
        target: 256Mi
        tolerance: 384Mi
        rationale: "Pod memory limit is 512Mi — need headroom for spikes"
      - metric: startup.ready_time
        target: 5s
        tolerance: 10s
        verification:
          method: benchmark
          conditions: "Cold start, no cached state"

  - scope: endpoint:/api/v1/plans
    environment: edge-on-prem
    objectives:
      - metric: latency.p99
        target: 50ms
        tolerance: 100ms
        stretch: 25ms
        verification:
          method: load_test
          tool_hint: k6
          conditions: "50 concurrent users, 60s sustained, warm cache"
      - metric: throughput.rps
        target: 200
        tolerance: 100

  - scope: component:plan-generator
    environment: edge-on-prem
    objectives:
      - metric: latency.p99
        target: 200ms
        tolerance: 500ms
        rationale: "Computationally intensive path — keep the user-visible wait brief"
      - metric: memory.allocation
        target: 16Ki
        tolerance: 32Ki
        rationale: "Invoked frequently — allocation pressure causes GC pauses"
        verification:
          method: benchmark
          tool_hint: go_bench

  - scope: function:scoreCandidates
    environment: edge-on-prem
    objectives:
      - metric: cpu.time_per_op
        target: 2ms
        stretch: 500µs
        rationale: "Called per-candidate in a loop — dominates plan generation time"
      - metric: memory.allocation
        target: 512B
        verification:
          method: benchmark
          tool_hint: go_bench
          conditions: "10 candidates, realistic payload"
```

### CLAUDE.md Example

```markdown
## Performance Budgets

environment: edge-on-prem { cpu: 2vCPU, memory: 4Gi, arch: arm64, model: kubernetes }
rationale: Regional depot nodes, memory-constrained, co-located DB at 2ms RTT.

### Service Level
- memory.resident < 256Mi (tolerance: 384Mi)
- startup.ready_time < 5s

### Endpoints
- /api/v1/plans: latency.p99 < 50ms (tolerance: 100ms), throughput.rps > 200
- /api/v1/plans/generate: latency.p99 < 200ms (tolerance: 500ms)

### Hot Functions
- scoreCandidates: cpu.time_per_op < 2ms, memory.allocation < 512B
- lookupStop: latency.p99 < 1ms
```

### Inline Example

```go
// @perfcontract-env edge-on-prem
package planner

// @perfcontract latency.p99 < 200ms, memory.allocation < 16Ki
func GeneratePlan(ctx context.Context, req *PlanRequest) (*Plan, error) {
    candidates := fetchCandidates(ctx, req)
    return scoreCandidates(candidates)
}

// @perfcontract cpu.time_per_op < 2ms, memory.allocation < 512B
func scoreCandidates(candidates []Candidate) *Plan {
    // Agent knows: this is the hot loop, must be allocation-free
}
```

---

## Versioning

The spec uses `perfcontract: v1` as the version identifier. Breaking changes increment the version. Non-breaking additions (new metrics, new verification methods) do not.

The canonical spec and JSON Schema will be published at:
- Spec: `https://github.com/ahartzog/perfcontract`
- Schema: `https://raw.githubusercontent.com/ahartzog/perfcontract/main/v1/schema.json`

---

## Future Directions (out of scope for v1)

- **Frontend domain**: bundle size, TTI, LCP, animation frame budgets
- **ML/inference domain**: model latency, VRAM, batch throughput
- **Embedded domain**: interrupt latency, stack depth, flash size
- **Relative targets**: "30% faster than baseline" (requires baseline declaration)
- **Composition rules**: formal model for how function budgets sum to endpoint budgets
- **CI integration**: GitHub Action / pre-commit hook that validates budgets against test results
- **VS Code extension**: inline display of budget status from last test run
- **Per-tenant / per-producer budgets**: resource isolation in multi-tenant deployments
- **Non-linear tolerance curves**: expressing drop probability functions (e.g., quadratic backoff)
- **Tick/loop budget ratio**: expressing "must complete within X% of loop period" as a first-class constraint
- **Reliability namespace**: best-effort side-effect delivery rates, subscription durability guarantees

---

## License

Licensed under the [Apache License 2.0](LICENSE). Copyright 2026 Alek Hartzog.
