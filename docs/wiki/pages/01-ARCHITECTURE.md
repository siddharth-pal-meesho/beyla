<!-- m-wiki: type=top-level slug=01-architecture topic=null base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: top-level. 0 sources.

# Architecture — Beyla as an OBI distribution

Beyla is not a standalone eBPF agent. Since v3 it is a *distribution* of OpenTelemetry eBPF Instrumentation (OBI): the discovery, eBPF probing, span decoration and exporter machinery all live in the `go.opentelemetry.io/obi` module, which this repository vendors. What remains in `github.com/grafana/beyla/v3` is a thin but load-bearing shell — a Grafana-flavoured configuration type, a set of global naming overrides, a handful of extra pipeline stages, and a Kubernetes admission webhook that OBI does not ship.

## TL;DR

- The upstream engine is vendored under `vendor/go.opentelemetry.io/obi/`, pinned by a `replace` directive to the `.obi-src` Git submodule rather than fetched from a proxy.
- Beyla owns its own `Config` struct and converts it to OBI's config by reflection at `pkg/beyla/config_obi.go:Config.AsOBI`; the converted value is cached on the struct.
- Three independent observability features — application, network and stats — plus an optional injection webhook, each run as a goroutine in one `errgroup` at `pkg/components/beyla.go:RunBeyla`.
- Beyla-specific telemetry (survey info metrics, connection spans, Sigil GenAI export, process metrics) is added as extra stages *around* OBI's own pipeline, not inside it.
- Renaming is pervasive: `BEYLA_*` environment variables are duplicated into `OTEL_EBPF_*` and metric names are rewritten to the `beyla.*` namespace at startup.

## Mental model

Think of three concentric rings.

The **inner ring** is vendored OBI. It watches the process table, matches executables against selection criteria, attaches eBPF probes, reads spans out of ring buffers, decorates them with host/Kubernetes/Docker metadata, and exports them over OTLP or Prometheus. Beyla does not modify this code in place — it is vendored verbatim from a submodule.

The **middle ring** is the Beyla configuration and naming layer. Users write Beyla-shaped YAML and `BEYLA_`-prefixed environment variables; a reflection-based converter walks that struct and populates the structurally-similar OBI config. A companion function mutates OBI package-level globals so the telemetry that comes out carries Grafana/Beyla names rather than OpenTelemetry defaults.

The **outer ring** is the extra stages Beyla bolts on: a process-metrics sub-pipeline, cluster-connector spans for inter-cluster service graphs, a Sigil GenAI trace export, an Alloy in-process receiver, survey-mode info metrics, and the SDK-injection admission webhook. These subscribe to the same span queues the inner ring publishes to.

The consequence worth internalising: **most behaviour questions are answered in `vendor/`, most configuration and naming questions are answered in `pkg/`.**

## Structure / data flow

```
cmd/beyla/main.go
      │  loadConfig → beyla.Config (YAML + BEYLA_* env)
      ▼
pkg/components/beyla.go:RunBeyla
      │  buildCommonContextInfo → global.ContextInfo
      │  (Prometheus mgr, OTEL instancer, K8s informer, Docker store, internal metrics)
      │
      ├─ FeatureAppO11y   ─► pkg/internal/appolly ─► pkg/internal/pipe:Build
      │                                                   │
      │                                                   ├─ vendored OBI appolly.Build   ← the real engine
      │                                                   ├─ Alloy traces receiver
      │                                                   ├─ cluster-connector subpipeline
      │                                                   ├─ Sigil GenAI export subpipeline
      │                                                   └─ process-metrics swarm
      │
      ├─ FeatureNetO11y   ─► vendored OBI netolly agent.FlowsAgent
      ├─ FeatureStatsO11y ─► vendored OBI statsolly agent.StatsAgent
      └─ Injector.Webhook ─► pkg/webhook:NewServer  (admission controller)
```

| Ring | Where | Owns |
|---|---|---|
| Inner (engine) | `vendor/go.opentelemetry.io/obi/` | eBPF probes, discovery, decoration, core exporters |
| Middle (config/naming) | `pkg/beyla/`, `pkg/helpers/config/` | Beyla `Config`, OBI conversion, global renames |
| Outer (extra stages) | `pkg/internal/pipe/`, `pkg/export/`, `pkg/webhook/` | Process metrics, connector spans, Sigil, survey metrics, injection |

## Key code locations

| What | Where |
|---|---|
| Feature fan-out and lifecycle | `pkg/components/beyla.go:RunBeyla` |
| Shared context construction | `pkg/components/beyla.go:buildCommonContextInfo` |
| Beyla configuration root | `pkg/beyla/config.go:Config` |
| Beyla → OBI config bridge | `pkg/beyla/config_obi.go:Config.AsOBI` |
| OBI → Beyla config bridge | `pkg/beyla/config_obi.go:FromOBI` |
| Global name/prefix overrides | `pkg/beyla/config_obi.go:OverrideOBIGlobalConfig` |
| App pipeline assembly | `pkg/internal/pipe/instrumenter.go:Build` |
| Vendored OBI module pin | `go.mod` — `replace go.opentelemetry.io/obi => ./.obi-src` |

## Sharp edges

- **`vendor/` is not incidental.** Editing a file under `vendor/go.opentelemetry.io/obi/` is a real code change in this fork, but `make vendor-obi` regenerates the tree from the `.obi-src` submodule and will discard it. Fork-local engine changes belong in the submodule repository, then get vendored in. See [OBI submodule and vendoring workflow](07-BUILD-AND-VENDORING.md).
- **`.obi-src` is empty on a fresh clone.** The `replace go.opentelemetry.io/obi => ./.obi-src` directive in `go.mod` points at a submodule that must be initialised (`make obi-submodule`) before any code generation target will work. Builds still succeed from `vendor/` alone.
- **Conversion is by reflection and panics on mismatch.** `pkg/helpers/config/convert.go:Convert` matches fields by name; anything present in one struct but absent in the other must be explicitly listed as `SkipConversion` or the run aborts. Adding a Beyla-only config field is therefore a two-place edit.
- **`AsOBI()` caches.** The converted OBI config is memoised on the `Config` value, so mutating the Beyla config after the first `AsOBI()` call has no effect on what the engine sees.

## Related concepts

- [The vendored OBI boundary](obi-integration/vendored-obi-boundary.md)
- [Reflection-based config conversion](obi-integration/config-conversion-reflection.md)
- [OBI global overrides](obi-integration/obi-global-overrides.md)
- [Feature gating](telemetry/feature-gating.md)

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, incident references, dates, decisions that synthesis missed. -->

---

[← Previous](02-ENTRYPOINT.md) · [Index](../index.md) · [Next →](02-ENTRYPOINT.md)
