# Summary

A one-page map of the 3CP documentation set (69 documents across 8 sections).

## By Section

- **01-introduction** — `vision`, `architecture-overview`, `terminology`, `threat-model`
- **02-3cp-specification** — `protocol-overview`, `cryptography`, `consensus`, `sparse-merkle-tree`, `vrf`, `anchoring`, `networking`, `protobuf`, `protocol-state-machine`
- **03-gleipnir** — `architecture`, `engine`, `state-machine`, `scheduler`, `storage`, `grpc-api`, `rest-api`, `metrics`, `deployment`, `production`, `troubleshooting`
- **04-divergence** — `overview`, `mandates`, `consensus`, `vrf`, `observability`, `grpc-api`, `state-machine`, `errors`, `carcosa`
- **04-carcosa** — `architecture`, `air`, `cli`, `proving`, `verification`, `witness`, `trace-generation`, `audit`, `anchor-integration`
- **05-operations** — `security`, `observability`, `backup`, `restore`, `disaster-recovery`, `cluster-bootstrap`, `rolling-upgrades`, `incident-response`, `performance`, `capacity-planning`
- **06-developer-guide** — `repository-layout`, `implementing-a-client`, `extending-the-engine`
- **07-reference** — `glossary`, `error-reference`, `metrics-reference`, `api-reference`, `configuration-reference`, `protobuf-reference`

## By Audience

| Reader | Start here |
|--------|-----------|
| Executive / newcomer | `01-introduction/vision`, `01-introduction/architecture-overview` |
| Protocol implementer | `02-3cp-specification/*`, `07-reference/protobuf-reference` |
| Gleipnir operator | `05-operations/*`, `03-gleipnir/deployment`, `03-gleipnir/troubleshooting` |
| Carcosa integrator | `04-carcosa/architecture`, `04-carcosa/air`, `04-carcosa/anchor-integration` |
| Auditor | `04-divergence/overview`, `02-3cp-specification/anchoring`, `04-carcosa/audit` |
| Client developer | `06-developer-guide/implementing-a-client`, `07-reference/api-reference` |

## Divergence Count

12 tracked divergences (D1–D12) across consensus, networking, VRF, observability,
Carcosa, and config. See `04-divergence/overview`.

## See Also

- `README` — entry point and protocol thesis
- `DOCUMENTATION_SPEC` — mandatory documentation rules
