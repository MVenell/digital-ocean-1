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

## Benchmark Stages

### Stage 1 - Single Server Baseline

Start with:

``` text
k6 -> Rust/Axum -> PostgreSQL
```

No load balancer.

No Redis.

Prefer keeping the API and PostgreSQL separate if budget allows so
resource usage and bottlenecks are easier to interpret.

Purpose:

-   establish baseline throughput;
-   establish DB behavior;
-   verify the load-testing methodology;
-   identify whether the API or database becomes the first bottleneck.

------------------------------------------------------------------------

### Stage 2 - Horizontal API Scaling

Test:

``` text
              +-> Rust API #1 --+
k6 -> LB -----+                 +-> PostgreSQL
              +-> Rust API #2 --+
```

Purpose:

-   determine whether adding an application server meaningfully
    increases capacity;
-   determine whether PostgreSQL becomes the bottleneck;
-   measure the cost/capacity effect of horizontal scaling;
-   demonstrate the role of a load balancer.

------------------------------------------------------------------------

### Stage 3 - Add Redis

Test:

``` text
              +-> Rust API #1 --+
k6 -> LB -----+                 +-> Redis -> PostgreSQL
              +-> Rust API #2 --+
```

Cache only operations where caching would be realistic.

Purpose:

-   measure reduction in database reads;
-   measure throughput improvement;
-   measure latency improvement;
-   determine whether Redis is economically justified.

Do not make every request a cache hit.

Record cache hit/miss rates.

------------------------------------------------------------------------

### Stage 4 - Scale the Database

Only do this if PostgreSQL is demonstrably the limiting resource.

Possible experiments:

-   larger PostgreSQL Droplet;
-   PostgreSQL tuning;
-   connection pooling;
-   read replica if the workload and implementation justify it.

Do not add database complexity merely to make the architecture look
sophisticated.

The benchmark should show **why** each system-design component becomes
necessary.

------------------------------------------------------------------------

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

## Ramp Strategy

Do not immediately run long tests.

First perform a short capacity search.

Example progression:

``` text
100 RPS
250 RPS
500 RPS
1,000 RPS
2,000 RPS
4,000 RPS
...
```

Initial exploratory stages can run for approximately 30-60 seconds each.

Stop increasing traffic when the system clearly fails the defined SLO.

Once the approximate limit is known, test around that boundary more
precisely.

Example:

If:

``` text
2,000 RPS = healthy
4,000 RPS = unhealthy
```

test:

``` text
2,500
3,000
3,500
```

Then run a longer confirmation test near the highest healthy level.

Suggested confirmation duration:

``` text
5-10 minutes
```

The exact duration should be configurable.

------------------------------------------------------------------------

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

## Cost Tracking

Every infrastructure configuration needs an associated monthly cost.

Create a machine-readable configuration/result record containing at
least:

``` text
configuration name
Droplet types
number of Droplets
load balancer
Redis cost
database cost
storage cost
backup cost
other required infrastructure
estimated monthly total
```

Do not hard-code current DigitalOcean prices deep inside benchmark
logic.

Keep pricing in a separate configuration/data file so it can be updated
later.

Example:

``` text
pricing/
  digitalocean.yaml
```

The benchmark's technical result should remain valid even if
DigitalOcean changes its prices.

------------------------------------------------------------------------

## Result Format

Every completed configuration should produce a summary similar to:

``` text
Configuration: 2x API + LB + PostgreSQL + Redis

Monthly infrastructure cost: $X

Maximum healthy sustained load:
3,200 requests/sec

SLO:
p95 < 200 ms
errors < 0.1%

Measured:
p50: 42 ms
p95: 171 ms
p99: 260 ms
errors: 0.04%

API CPU: ...
PostgreSQL CPU: ...
Redis CPU: ...

Likely bottleneck:
PostgreSQL CPU / disk / API CPU / network / etc.
```

Also store the raw k6 output/results where practical.

------------------------------------------------------------------------

## Final Comparison Table

The repository should make it easy to produce something like:

  ----------------------------------------------------------------------------
  Architecture       Monthly cost Max healthy RPS Cost per 1k RPS Bottleneck
  --------------- --------------- --------------- --------------- ------------
  1 API +                     \$X               X             \$X ...
  PostgreSQL                                                      

  2 API + LB +                \$X               X             \$X ...
  PostgreSQL                                                      

  2 API + LB +                \$X               X             \$X ...
  Redis +                                                         
  PostgreSQL                                                      

  Larger DB                   \$X               X             \$X ...
  configuration                                                   
  ----------------------------------------------------------------------------

The most important outputs are:

``` text
monthly infrastructure cost
vs.
measured sustainable workload
```

------------------------------------------------------------------------

## Safety / Cost Controls

This benchmark must be designed to avoid accidental cloud bills.

### Hard rules

-   Do not enable autoscaling.
-   Do not provision large infrastructure automatically without explicit
    confirmation.
-   Keep benchmark infrastructure clearly tagged/labeled.
-   Provide a teardown command/script.
-   Avoid unnecessary managed resources.
-   Use short exploratory load tests.
-   Put an explicit maximum RPS in the load-test configuration.
-   Put an explicit maximum duration in the load-test configuration.
-   Never create an unbounded k6 test.
-   Do not create resources repeatedly if existing benchmark resources
    can be reused safely.
-   Document which resources continue billing when tests stop.

Prefer an infrastructure workflow that makes it easy to see exactly what
will be created before creation.

If Terraform/OpenTofu is used, inspect the plan before applying it.

------------------------------------------------------------------------

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

## Expected Implementation Sequence

### Phase 1 - Local benchmark

Before spending money:

1.  Get the Rust project running locally.
2.  Get PostgreSQL running.
3.  Seed the deterministic dataset.
4.  Implement/verify benchmark endpoints.
5.  Create the k6 mixed workload.
6.  Verify metrics and thresholds.
7.  Verify that the load test stops correctly.
8.  Verify result collection.
9.  Test Redis locally.
10. Document the exact workload.

No DigitalOcean resources should be necessary to validate the basic
benchmark harness.

------------------------------------------------------------------------

### Phase 2 - DigitalOcean baseline

Provision the smallest useful baseline.

Run:

1.  smoke test;
2.  low-load validation;
3.  exploratory ramp;
4.  boundary search;
5.  confirmation test.

Record results and cost.

------------------------------------------------------------------------

### Phase 3 - Scale based on evidence

Inspect the baseline bottleneck.

Only then choose the next architecture.

Examples:

``` text
API CPU saturated
-> add/resize API capacity

PostgreSQL saturated
-> improve/resize database

read-heavy DB workload
-> test Redis

single API capacity reached
-> add second API + load balancer
```

The architecture should evolve because of measured bottlenecks, not
because the final diagram was decided in advance.

------------------------------------------------------------------------

### Phase 4 - Produce comparison data

For each tested configuration:

1.  save raw results;
2.  save infrastructure specification;
3.  save monthly cost;
4.  save sustainable RPS;
5.  save latency/error statistics;
6.  identify likely bottleneck;
7.  document any tuning performed.

Generate a final CSV/JSON/Markdown summary that can later be used for
charts and cost comparisons.

------------------------------------------------------------------------

## Definition of Done

The DigitalOcean experiment is complete when we can make a statement of
this form with measured evidence:

> On configuration X, costing approximately \$Y/month at the recorded
> DigitalOcean prices, the Rust/PostgreSQL application sustained Z
> requests/second while maintaining p95 latency below 200 ms and errors
> below 0.1%.

Ideally we will have several configurations so we can plot:

``` text
DigitalOcean monthly cost
        vs.
sustainable application throughput
```

The purpose is **not** to prove DigitalOcean is cheap or expensive.

The purpose is to measure the self-managed infrastructure cost curve
honestly enough that it can later be compared against managed-service
pricing.
