# DigitalOcean Rust Load-Test Implementation Plan

## Goal

Build a reproducible DigitalOcean benchmark that answers:

> How much real application traffic can a given amount of self-managed
> DigitalOcean infrastructure sustain before it violates our performance
> target?

This benchmark is **DigitalOcean-only**. Do not implement or load-test
Firebase in this project.

The output should let us map:

**DigitalOcean monthly infrastructure cost -\> sustainable workload**

We can later compare that measured workload against managed-service
pricing separately.

------------------------------------------------------------------------

## Core Rules

1.  Use a real Rust HTTP application backed by PostgreSQL.
2.  Use Redis where appropriate so we can test the effect of adding a
    cache.
3.  Generate load from a **separate machine** from the
    application/database infrastructure.
4.  Never rely on autoscaling during the benchmark.
5.  Increase traffic gradually rather than immediately generating
    maximum load.
6.  Stop a test when latency/error thresholds are clearly exceeded.
7.  Record both application-level results and infrastructure
    utilization.
8.  Keep every infrastructure configuration reproducible.
9.  Keep load-test workloads identical when comparing infrastructure
    configurations.
10. Do not optimize specifically for a benchmark in ways that would be
    unrealistic in production.

------------------------------------------------------------------------

## Proposed Application

Start from a small Rust REST API using:

-   Rust
-   Axum
-   SQLx
-   PostgreSQL
-   Redis
-   Docker / Docker Compose where useful

A candidate reference project is:

`sheroz/axum-rest-api-sample`

GitHub repository:

`https://github.com/sheroz/axum-rest-api-sample`

Do not treat the repository as immutable. Fork/copy it and simplify or
modify it as necessary to make the benchmark deterministic.

The benchmark application should be small enough that we understand
every important operation.

------------------------------------------------------------------------

## Frozen Application Model

Use a simple **e-commerce/product-catalog API** as the reference
application. A product should have fields similar to `id`, `name`,
`description`, `category_id`, `price`, `inventory`, `created_at`, and
`updated_at`.

The main operations are:

-   Get product by ID.
-   List products by `category_id` with pagination and a stable sort.
-   Create product.
-   Update price and/or inventory.

Use normal production-style indexes required by these operations. Avoid
deliberately pathological SQL, full-text search, or large multi-table
joins. The goal is infrastructure capacity, not a query-optimization
contest.

If authenticated endpoints are used, generate/reuse test JWTs rather
than performing password hashing/login on every load-test iteration.

## Latency and Error Definition

Use **k6 end-to-end HTTP request duration** for the primary latency
measurement.

Initial SLO:

``` text
p95 request duration < 200 ms
error rate < 0.1%
```

Count timeouts, connection failures, failed required k6 checks, and
unexpected non-2xx responses as errors. Record p50, p95, and p99, but
use p95 plus the error threshold to decide whether a configuration
passes.

## Database State Between Runs

Create/update traffic mutates the database. Every **confirmation run
used as a published result** must start from the same deterministic
seeded database state.

Exploratory capacity-search stages may run without a complete restore
between every short step, but their results are not the final comparison
numbers. Automate database reset/restore and record the dataset
seed/version with each result.

## Target Architecture

Initial architecture:

``` text
                     +-------------------+
                     |  k6 Load Generator|
                     |  separate machine |
                     +---------+---------+
                               |
                            Internet
                               |
                               v
                     +-------------------+
                     | DigitalOcean      |
                     | Load Balancer     |
                     +---------+---------+
                               |
                    +----------+----------+
                    |                     |
                    v                     v
             +-------------+       +-------------+
             | Rust/Axum   |       | Rust/Axum   |
             | API Server  |       | API Server  |
             +------+------+       +------+------+
                    |                     |
                    +----------+----------+
                               |
                         +-----+-----+
                         |   Redis   |
                         +-----+-----+
                               |
                         +-----+-----+
                         | PostgreSQL|
                         +-----------+
```

Not every benchmark stage will use every component.

We should start with the simplest architecture and add components only
when measurements justify them.

------------------------------------------------------------------------

## Benchmark Scope: Free-Only Round

This plan covers **only the zero-cost preparation and local validation
round**.

Do not provision any DigitalOcean resource and do not run any cloud
benchmark in this round.

The purpose of this round is to prove that the benchmark harness is
correct before any money can be spent.

### Allowed tests

Run only on hardware/environments already available locally at no
incremental cost:

``` text
local k6 -> local Rust/Axum -> local PostgreSQL
                               |
                               +-> local Redis when testing cache behavior
```

Allowed work:

1.  Build the Rust application in release mode.
2.  Run PostgreSQL locally.
3.  Seed the deterministic dataset.
4.  Verify all benchmark endpoints.
5.  Build and validate `workload-v1`.
6.  Run short smoke tests at low load.
7.  Verify k6 checks, latency metrics, error accounting, and hard stop
    conditions.
8.  Verify deterministic database reset/restore.
9.  Verify Redis behavior and cache hit/miss measurement locally.
10. Verify result collection and summary generation.
11. Prepare Terraform/OpenTofu configuration, but run **plan/validation
    only**. Never apply it.
12. Prepare DigitalOcean pricing/configuration files without creating
    resources.
13. Prepare teardown scripts/configuration for the later paid round, but
    do not need to execute them against cloud resources.

### Explicitly forbidden in this round

-   `terraform apply` against DigitalOcean.
-   Creating a Droplet.
-   Creating a DigitalOcean load balancer.
-   Creating a managed database.
-   Creating Spaces, volumes, snapshots, reserved IPs, or other billable
    resources.
-   Running a cloud-based k6 load generator.
-   Running high-throughput tests intended to find production capacity.
-   Any test whose purpose is to establish DigitalOcean RPS capacity.
-   Any action that may create a DigitalOcean charge.

If an action may incur a charge or its cost cannot be established as
zero, **stop and ask for a future plan update instead of performing
it**.

The paid/capacity-testing phases will be added to this document only
when explicitly requested later.

## Reference Dataset

Seed PostgreSQL before every benchmark with a deterministic dataset.

Initial target:

-   approximately 1,000,000 application records;
-   realistic indexes;
-   enough data that important queries do not trivially fit into tiny
    test tables;
-   deterministic random seed so runs can be reproduced.

Provide a script such as:

``` text
scripts/seed-db.sh
```

or a Rust seed utility.

The seed operation must be separate from the load test.

Document:

-   table sizes;
-   row counts;
-   indexes;
-   approximate database size;
-   seed used.

------------------------------------------------------------------------

## Workload

Create a fixed workload mix rather than benchmarking only one endpoint.

Initial target distribution:

  Operation                 Approx. share Purpose
  ----------------------- --------------- ------------------------
  Get item by ID                      50% Common indexed read
  List/query items                    20% Indexed database query
  Create item                         10% Database write
  Update item                         10% Database write
  Cacheable/public read               10% Redis/cache path

These percentages are initial assumptions and should live in
configuration so they can be changed easily.

Avoid expensive login/password hashing on every virtual-user iteration.

If authenticated endpoints are required, generate/reuse valid test JWTs
so we primarily benchmark the application architecture rather than
password hashing.

------------------------------------------------------------------------

## Load Generator

Use **k6**.

The k6 process must run on infrastructure separate from the target
API/database servers.

The load-generator machine must have enough CPU/network capacity that it
does not become the bottleneck.

Record its CPU/network utilization during high-load tests as a sanity
check.

Use arrival-rate-based scenarios where possible so the test requests a
known number of requests per second.

------------------------------------------------------------------------

## Local Smoke-Test Strategy

This round is not a capacity benchmark.

Use only enough local traffic to prove that the workload generator and
measurements behave correctly.

Suggested approach:

``` text
very low RPS -> short run -> verify expected operations and metrics -> stop
```

Keep test duration short and choose conservative local RPS values. Do
not search for the maximum throughput of the local machine because that
result is not part of the DigitalOcean comparison.

Verify that:

-   operation proportions approximately match `workload-v1`;
-   k6 records p50/p95/p99;
-   timeouts and failed checks count as errors;
-   the SLO logic reports pass/fail correctly;
-   maximum RPS and maximum duration caps work;
-   the test terminates cleanly;
-   database reset produces the same initial state;
-   Redis hit/miss metrics are captured when Redis is enabled.

## Performance Definition

Define capacity before running the benchmark.

Initial SLO:

``` text
p95 latency < 200 ms
HTTP/application error rate < 0.1%
```

The final benchmark capacity for a configuration is:

> The highest sustained request rate that satisfies the SLO during the
> confirmation run.

Also record:

-   p50 latency;
-   p95 latency;
-   p99 latency;
-   achieved requests/sec;
-   error rate;
-   total requests;
-   bytes sent/received.

Do not call a configuration capable of X RPS merely because it
technically returned responses at X RPS.

------------------------------------------------------------------------

## Infrastructure Metrics

For every run, capture as much of the following as practical.

### API servers

-   CPU utilization;
-   RAM utilization;
-   network throughput;
-   open connections;
-   process/container CPU;
-   process/container memory.

### PostgreSQL

-   CPU;
-   RAM;
-   active connections;
-   queries/sec if available;
-   disk I/O;
-   cache/buffer behavior if practical;
-   slow queries;
-   lock contention where relevant.

### Redis

-   CPU;
-   RAM;
-   operations/sec;
-   hit rate;
-   miss rate;
-   connections.

### Load balancer

Record available request/connection metrics where practical.

### Load generator

-   CPU;
-   RAM;
-   network utilization.

This is necessary to distinguish:

> "The application broke"

from:

> "The load generator couldn't generate any more traffic."

------------------------------------------------------------------------

## Cost / Cloud Guardrail

The required experiment spend for this round is:

``` text
$0.00
```

There is no `$5` approval threshold in this version because **no paid
test is authorized at all**.

DigitalOcean infrastructure may be modeled in Terraform/OpenTofu and
pricing files, but it must not be created.

Rules:

-   Do not call any command that creates, modifies, reserves, or retains
    a billable DigitalOcean resource.
-   `terraform validate`, formatting, static checks, and a plan that is
    guaranteed not to create resources are allowed.
-   Never run `terraform apply`.
-   `doctl` may be used only for non-mutating inspection if needed; do
    not use it to create resources.
-   Do not rely on DigitalOcean free credits, promotions, or assumptions
    about billing granularity to classify a test as free.
-   If there is uncertainty about whether an action can cost money,
    treat it as paid and do not perform it.
-   Do not perform even a short cloud benchmark in this round.
-   Do not provision a cloud load generator.
-   Do not test the maximum capacity of DigitalOcean infrastructure.

A later version of this plan can explicitly authorize paid DigitalOcean
capacity testing with a spending threshold and provisioning workflow.

## Results Required From This Round

The free-only round is successful when it produces:

-   a working Rust/Axum application;
-   deterministic PostgreSQL seed/reset tooling;
-   working local Redis integration;
-   frozen `workload-v1`;
-   working k6 scripts;
-   verified SLO/error calculations;
-   verified safety caps for duration and RPS;
-   repeatable local smoke-test results;
-   infrastructure-as-code that can be reviewed for a future
    DigitalOcean round;
-   pricing/configuration structure ready for future cost data;
-   documentation explaining exactly how the later cloud benchmark will
    be launched once authorized.

Do not claim any DigitalOcean throughput, capacity, or cost-performance
result from this round.

## Reproducibility

The repository should contain everything needed to reproduce the
benchmark except credentials.

Suggested layout:

``` text
.
├── app/
│   └── Rust/Axum application
├── loadtest/
│   ├── workload.js
│   ├── scenarios/
│   └── config/
├── infra/
│   ├── terraform-or-opentofu/
│   └── README.md
├── scripts/
│   ├── seed-db.sh
│   ├── run-benchmark.sh
│   ├── collect-results.sh
│   └── teardown.sh
├── pricing/
│   └── digitalocean.yaml
├── results/
│   └── .gitkeep
└── README.md
```

Never commit:

-   DigitalOcean API tokens;
-   SSH private keys;
-   database passwords;
-   JWT secrets;
-   other credentials.

Provide `.env.example` or equivalent.

------------------------------------------------------------------------

## Automation

Prefer scripts over manual benchmark steps.

A target workflow could eventually look like:

``` bash
# Provision selected benchmark configuration
./scripts/provision.sh baseline

# Seed database
./scripts/seed-db.sh

# Run exploratory capacity search
./scripts/run-benchmark.sh baseline search

# Run confirmation test
./scripts/run-benchmark.sh baseline confirm

# Collect/summarize results
./scripts/collect-results.sh baseline

# Destroy infrastructure
./scripts/teardown.sh
```

Do not blindly implement these exact interfaces if a cleaner solution
emerges.

The important requirement is that benchmark runs are repeatable and
difficult to perform incorrectly.

------------------------------------------------------------------------

## Benchmark Integrity

Avoid common benchmark mistakes.

### Do not:

-   run k6 on the target application server;
-   benchmark an empty database;
-   change the workload between infrastructure configurations;
-   compare cached results against uncached results without labeling
    them;
-   silently change application code between infrastructure tests;
-   use unrealistic 100% cache-hit workloads;
-   report maximum RPS without latency/error constraints;
-   let the load generator become the bottleneck;
-   extrapolate a short spike as sustainable capacity;
-   tune only one configuration unfairly;
-   include initialization/seed traffic in benchmark results;
-   use development/debug builds of the Rust application.

Compile the Rust service in release mode.

------------------------------------------------------------------------

## Comparison Freeze

Once `workload-v1`, the seed dataset, application behavior, indexes,
SLOs, and benchmark methodology are frozen, do not silently modify them
between configurations.

> Optimize the experiment for reproducibility and comparability, not
> maximum benchmark numbers. The workload must remain fixed while
> infrastructure changes. Every published configuration must begin from
> the same database state. Components such as Redis, additional API
> servers, and larger database servers should only be introduced in
> response to a measured bottleneck. Never silently change application
> behavior, workload distribution, dataset, indexes, SLOs, or benchmark
> methodology between configurations.

If any frozen element must change, create a new explicitly versioned
experiment and rerun the configurations that will be compared.

## Expected Implementation Sequence

### Phase 1 - Build locally

1.  Get the Rust project running locally in release mode.
2.  Run PostgreSQL locally.
3.  Implement/verify the product-catalog endpoints.
4.  Seed the deterministic dataset.
5.  Implement deterministic reset/restore.
6.  Add Redis integration without changing the external workload.
7.  Verify application tests.

### Phase 2 - Build the benchmark harness

1.  Implement frozen `workload-v1`.
2.  Add required k6 checks.
3.  Add p50/p95/p99 and error reporting.
4.  Add explicit maximum RPS and maximum duration caps.
5.  Run only short, low-load local smoke tests.
6.  Verify workload proportions and result output.
7.  Verify failure/error accounting.
8.  Verify database reset between official local validation runs.
9.  Verify Redis hit/miss reporting.

### Phase 3 - Prepare, but do not provision, DigitalOcean infrastructure

1.  Create Terraform/OpenTofu definitions for the future baseline
    architecture.
2.  Run formatting and validation.
3.  Review the resource graph/configuration.
4.  Prepare cloud-init/configuration scripts.
5.  Prepare DigitalOcean pricing data structure.
6.  Prepare future benchmark and teardown scripts.
7.  Document the future provisioning sequence.

**Do not run `terraform apply`.**

### Phase 4 - Stop

At this point, stop the project.

Do not proceed into DigitalOcean capacity testing.

Wait for explicit authorization to revise this plan with a paid-testing
phase.

## Definition of Done

This free-only round is complete when we can demonstrate locally that:

> The Rust/PostgreSQL/Redis benchmark application, deterministic
> dataset, fixed workload, k6 measurements, error/SLO logic, result
> collection, and infrastructure definitions are ready for a future
> DigitalOcean capacity benchmark without having created any billable
> cloud resources.

Required conditions:

-   Actual authorized cloud experiment spend: **\$0.00**.
-   No DigitalOcean benchmark infrastructure has been provisioned.
-   No DigitalOcean capacity/RPS claim has been made.
-   `workload-v1` is frozen.
-   Database reset is deterministic.
-   Local smoke tests pass.
-   k6 safety limits have been verified.
-   Future DigitalOcean infrastructure can be reviewed as code.
-   The next phase requires an explicit update to this plan before any
    paid resource can be created.

The purpose of this round is **benchmark preparation and validation
only**.

Do not attempt to answer yet whether DigitalOcean is cheaper, how much
traffic it can handle, or where the managed-vs-self-managed crossover
occurs. Those questions belong to the next explicitly authorized testing
round.
