# SpecProbe

**API test automation generated from your OpenAPI spec.** Point SpecProbe at an OpenAPI 3.0/3.1 document and a running service. It generates valid requests plus schema-mutated invalid ones, then judges every response against the contract. You don't hand-write test cases, and every failure comes with a one-line `curl` reproduction.

> **Status: in active development.** This README describes the planned design. Detection numbers are published only once they come from real eval runs.

## How it works
```
OpenAPI spec ──▶ plan ──▶ positive cases (schema-valid values, seeded & reproducible)
                   └────▶ negative cases (break exactly one constraint: missing required,
                                           wrong type, out of range, bad enum/format, …)
                                   │
                    concurrent runner (worker pool, rate limit, phased GET → POST/PUT/PATCH → DELETE)
                                   │
                    5 oracles ──▶ console · JUnit XML · JSON · curl repro per failure
```

| Oracle | Fails when |
|---|---|
| `server_error` | the service returns 5xx |
| `undocumented_status` | the status code isn't declared in the spec |
| `response_schema` | the response body doesn't match its documented schema |
| `validation_gap` | an invalid request was accepted with 2xx |
| `latency` | the response exceeded its latency budget |

## Planned usage
```bash
specprobe inspect --spec openapi.json
specprobe run --spec openapi.json --base-url http://localhost:8080 --junit report.xml
specprobe run --config probe.yaml                 # setup/teardown chains, captured variables, per-op budgets
specprobe run --config probe.yaml --only 'createItem/neg/missing_required/body.price'   # reproduce one case
```
```yaml
# GitHub Actions
- uses: NikethNath/specprobe@v0
  with:
    spec: openapi.json
    base-url: http://localhost:8080
```

## Highlights
- **Go**, a single static binary, and a distroless Docker image.
- **Deterministic:** per-operation seeded RNG and stable case IDs, so any failure can be replayed exactly.
- **Honest negatives:** a property test proves every mutated payload really violates the schema, so a `validation_gap` is always a true positive.
- **Eval harness:** a testbed API with 8 documented planted bugs is scored across 10 seeds. The fixed build must produce zero findings.
- **Real target:** CI deploys [FlowLens](https://github.com/NikethNath/flowlens) to a **kind** Kubernetes cluster and gates on SpecProbe's run.
- **Ansible:** a playbook provisions a Linux host, starts a target and runs the probe, with the JUnit report fetched back.

## Prior art
Schemathesis · RESTler (Microsoft Research) · Dredd

## License
MIT
