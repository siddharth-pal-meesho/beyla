# beyla — Wiki

> Synthesized view of this codebase. Code is the prior; this wiki is downstream.
> Owned by `m-wiki`. Edit via skills, not directly.

## Top-level pages

| #   | Page | What you'll learn |
|-----|------|-------------------|
| 01  | [Architecture — Beyla as an OBI distribution](pages/01-ARCHITECTURE.md) | Beyla is not a standalone eBPF agent |
| 02  | [Entry points and process lifecycle](pages/02-ENTRYPOINT.md) | This repository builds three binaries |
| 03  | [Configuration model](pages/03-CONFIGURATION.md) | Beyla exposes its own configuration surface — a large Config struct with YAML tags and BEYLA_-prefi… |
| 04  | [Application observability pipeline](pages/04-APPO11Y-PIPELINE.md) | Application observability is Beyla's primary mode |
| 05  | [Exporters and telemetry surface](pages/05-EXPORTERS.md) | Beyla emits telemetry through OBI's OTLP and Prometheus exporters, but it renames almost everything… |
| 06  | [Kubernetes integration](pages/06-KUBERNETES-INTEGRATION.md) | Kubernetes shows up in Beyla in three distinct roles, and conflating them causes confusion |
| 07  | [Build and the OBI vendoring workflow](pages/07-BUILD-AND-VENDORING.md) | The build is unusual because the engine is not a normal dependency |
| 08  | [Operating and deploying Beyla](pages/08-OPERATIONS.md) | Beyla runs as a privileged agent next to the workloads it instruments — normally a Kubernetes Daemo… |

## Topics

### kubernetes (2 pages)

_Cluster metadata sourcing and the SDK-injection admission webhook_

- [Kubernetes metadata cache service](pages/kubernetes/k8s-metadata-cache-service.md) — An optional second deployable — built from cmd/k8s-cache — that runs the Kubernetes informers once …
- [SDK injection webhook](pages/kubernetes/sdk-injection-webhook.md) — An experimental Kubernetes admission controller that mutates other workloads' Pods to preload an Op…

### obi-integration (3 pages)

_How this repo consumes, converts and overrides the vendored OBI engine_

- [OBI global overrides](pages/obi-integration/obi-global-overrides.md) — A startup routine that mutates package-level variables inside the vendored OBI module so that the t…
- [Reflection-based config conversion](pages/obi-integration/config-conversion-reflection.md) — Beyla and OBI keep two separate configuration structs with deliberately similar shapes, and a small…
- [The vendored OBI boundary](pages/obi-integration/vendored-obi-boundary.md) — The line between code this repository authors and code it merely carries

### pipeline (4 pages)

_Swarm and queue wiring of the instrumentation, decoration and export stages_

- [Cluster connector spans](pages/pipeline/cluster-connector-spans.md) — An optional trace branch that emits synthetic "connector" spans for traffic identified as leaving t…
- [Process discovery and survey mode](pages/pipeline/process-discovery-and-survey.md) — Discovery is how Beyla decides which processes to instrument
- [Process metrics sub-pipeline](pages/pipeline/process-metrics-subpipeline.md) — An optional Beyla-owned branch that turns discovered application spans into per-process resource me…
- [The swarm instancer model](pages/pipeline/swarm-instancer-model.md) — Beyla composes its processing stages as a *swarm*: stages are registered as instancer functions, in…

### telemetry (3 pages)

_Feature gating and the Beyla-specific metrics layered on OBI's exporters_

- [API dependency counter (fork-local)](pages/telemetry/api-dependency-counter.md) — A fork-local counter joining a service's outbound client spans to the entry span that caused them, giving…
- [Feature gating](pages/telemetry/feature-gating.md) — Beyla has two independent gating layers, and mixing them up is a common source of "my metric isn't …
- [Survey info metrics](pages/telemetry/survey-info-metrics.md) — An info-style metric reporting which processes Beyla *discovered and matched*, independent of wheth…

## How to read this wiki

**If you're brand new:**
1. [Architecture — Beyla as an OBI distribution](pages/01-ARCHITECTURE.md) — start here; the vendored-engine framing explains everything else.
2. [Entry points and process lifecycle](pages/02-ENTRYPOINT.md) — how a run actually starts and stops.
3. [Application observability pipeline](pages/04-APPO11Y-PIPELINE.md) — the main data path.
4. [The vendored OBI boundary](pages/obi-integration/vendored-obi-boundary.md) — which half of the code you are looking at.

**If you're debugging:**
- **A service is missing entirely** → [Process discovery and survey mode](pages/pipeline/process-discovery-and-survey.md), then [Survey info metrics](pages/telemetry/survey-info-metrics.md).
- **A metric family is missing** → [Feature gating](pages/telemetry/feature-gating.md) — check the inner (per-exporter) gate before the outer one.
- **Kubernetes attributes are absent** → [Kubernetes integration](pages/06-KUBERNETES-INTEGRATION.md); informer failure is a logged downgrade, not a crash.
- **A config value seems ignored** → [Reflection-based config conversion](pages/obi-integration/config-conversion-reflection.md) and the `AsOBI()` caching note in [Configuration model](pages/03-CONFIGURATION.md).
- **Names look wrong in Grafana** → [OBI global overrides](pages/obi-integration/obi-global-overrides.md).

**If you're adding a feature:**
1. Decide which side of the boundary the change belongs on — [The vendored OBI boundary](pages/obi-integration/vendored-obi-boundary.md). Engine changes go in the `.obi-src` submodule, not `vendor/`.
2. Adding config? Read [Configuration model](pages/03-CONFIGURATION.md) — struct field, `SkipConversion` hint, regenerated schema.
3. Adding a pipeline stage? Read [The swarm instancer model](pages/pipeline/swarm-instancer-model.md) and register it in `pkg/internal/pipe/instrumenter.go:Build`.
4. Adding a metric? Read [Feature gating](pages/telemetry/feature-gating.md); [API dependency counter](pages/telemetry/api-dependency-counter.md) is a worked example of a fork-local one.
5. Re-vendor and verify — [Build and the OBI vendoring workflow](pages/07-BUILD-AND-VENDORING.md).

## Unexplored topics

Candidates the bootstrap identified but did not generate pages for. Good ingest targets:

- **Network observability (NetO11y) internals** — flow capture, dedup, CIDR/GeoIP decoration. Currently only sketched in `devdocs/pipeline-map.md`; the implementation is vendored under `vendor/go.opentelemetry.io/obi/pkg/netolly/`.
- **Stats observability (StatsO11y)** — the TCP RTT / retransmit / failed-connection metric family. Beyla renames these but the collection path is vendored.
- **eBPF probe internals** — the `bpf/` C sources and the bpf2go generation step live in the `.obi-src` submodule, absent on a fresh clone.
- **Java agent and SDK injection images** — `internal/java/agent/`, `pkg/webhook/image/` and `pkg/webhook/lang/`.
- **Integration test topology** — `internal/test/integration/` and the many `pull_request_*_integration_tests` workflows.
- **Release engineering** — the release-train workflows and Helm chart publication under `.github/workflows/`.

Drop notes on any of these into `raw/<topic>/<file>.md` and re-run `/m-wiki:wiki-init` to fold them in.

## Operating this wiki

| Action | How |
|---|---|
| Add a source | Drop the file into `raw/<topic>/<file>.md`, then run `/m-wiki:wiki-init` |
| Pull an external source | `/m-wiki:wiki-pull-external` (Confluence / Drive), then `/m-wiki:wiki-synthesize-external` |
| Ask a question | `qmd query "<question>" --collection beyla-wiki`, or `/m-wiki:wiki-search` across services |
| Health-check | `/m-wiki:wiki-verify` |
| Re-sync after code changes | `/m-wiki:wiki-init` |

---

<!-- m-wiki: index-version=1 generated-at=2026-08-25T07:57:48Z base-sha=b4da5978747b9823660933b101baebbd8df45c87 -->
