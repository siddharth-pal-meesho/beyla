<!-- m-wiki: type=top-level slug=05-exporters topic=null base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: top-level. 0 sources.

# Exporters and telemetry surface

Beyla emits telemetry through OBI's OTLP and Prometheus exporters, but it renames almost everything on the way out and adds several exporters of its own. This page maps the export surface: what comes from the vendored engine, what Beyla adds, and where the naming is rewritten.

## TL;DR

- Core metric and trace export is vendored OBI; Beyla changes the *names* rather than the machinery, via `pkg/beyla/config_obi.go:OverrideOBIGlobalConfig`.
- Beyla-specific exporters live under `pkg/export/`: survey info metrics, connection spans, Sigil GenAI traces, process metrics for both OTEL and Prometheus, and an Alloy in-process receiver.
- Internal (self-observability) metrics are selected by a four-way switch in `pkg/components/beyla.go:internalMetrics` — OTEL, Prometheus, an injected registry, or a no-op.
- Which attribute groups are enabled is decided at startup from the runtime environment, not purely from config — see `pkg/components/beyla.go:attributeGroups`.
- This fork carries an extra vendored metric, `api_dependency_total`, that upstream OBI does not ship.

## Mental model

Three different mechanisms produce the exported names you see in Grafana.

**Struct-level configuration** decides endpoints, protocols and intervals. It reaches the engine through `AsOBI()`, with Grafana Cloud credentials injected by `overrideOBI`.

**Package-level globals** decide names. `OverrideOBIGlobalConfig` reassigns OBI's exported `attributes.Name` values — network flow, inter-zone, and the four TCP stat metrics — to `beyla_*` / `beyla.*` spellings, and sets the vendor prefix and SDK name to `beyla`. This is mutation of vendored package state at startup, which is why it must run before any exporter is constructed.

**Runtime environment** decides attribute groups. `attributeGroups` adds the Kubernetes group when an informer is enabled, otherwise the container group when Docker metadata is available, plus HTTP-route, network-direction, CIDR and GeoIP groups depending on configuration.

## Structure / data flow

| Exporter | Owner | Where |
|---|---|---|
| OTLP metrics / traces | vendored OBI | `vendor/go.opentelemetry.io/obi/pkg/export/otel/` |
| Prometheus endpoint | vendored OBI | `vendor/go.opentelemetry.io/obi/pkg/export/prom/` |
| Survey info metrics (OTEL) | Beyla | `pkg/export/otel/metrics_survey.go:SurveyInfoMetrics` |
| Survey info metrics (Prom) | Beyla | `pkg/export/prom/prom_survey.go` |
| Process metrics (OTEL) | Beyla | `pkg/export/otel/metrics_proc.go` |
| Process metrics (Prom) | Beyla | `pkg/export/prom/prom_proc.go` |
| Connection (inter-cluster) spans | Beyla | `pkg/export/otel/connect_spans.go:ConnectionSpansExport` |
| Sigil GenAI traces | Beyla | `pkg/export/otel/sigil_export.go` |
| Alloy in-process receiver | Beyla | `pkg/export/alloy/traces.go` |
| Grafana Cloud OTLP resolution | Beyla | `pkg/export/otel/grafana.go` |
| `api_dependency_total` | Meesho fork (vendored) | `vendor/go.opentelemetry.io/obi/pkg/export/otel/api_dependency.go:apiDepTracker` |

## Key code locations

| What | Where |
|---|---|
| Metric/attribute global renames | `pkg/beyla/config_obi.go:OverrideOBIGlobalConfig` |
| Internal metrics selection | `pkg/components/beyla.go:internalMetrics` |
| Attribute group activation | `pkg/components/beyla.go:attributeGroups` |
| Survey metrics reporter | `pkg/export/otel/metrics_survey.go:SurveyMetricsReporter` |
| Survey reporter constructor | `pkg/export/otel/metrics_survey.go:SurveyInfoMetrics` |
| Connection span export stage | `pkg/export/otel/connect_spans.go:ConnectionSpansExport` |
| Connection span generation | `pkg/export/otel/connect_spans.go:GenerateConnectSpans` |
| Beyla attribute selector | `pkg/export/extraattributes/attr_defs.go:NewBeylaAttrSelector` |
| Process metric name definitions | `pkg/export/extraattributes/metric.go:8` |
| API dependency tracker | `vendor/go.opentelemetry.io/obi/pkg/export/otel/api_dependency.go:apiDepTracker` |

## Sharp edges

- **Global renames are order-sensitive.** `OverrideOBIGlobalConfig` mutates package-level variables in the vendored module. Anything that captured those values earlier keeps the old name.
- **Internal metrics have a legacy escape hatch.** The second branch of `pkg/components/beyla.go:internalMetrics` fires when the exporter is Prometheus *or* when `InternalMetrics.Prometheus.Port` is merely non-zero — setting the port alone silently changes the exporter.
- **There is a known dependency cycle around the Prometheus manager.** The same function carries a `TODO` noting that the manager has to be instrumented after construction because it also emits its own internal metrics.
- **Process metrics need two conditions, not one.** `pkg/internal/pipe/proc_pipeline.go:isProcessSubPipeEnabled` requires *both* an enabled metrics endpoint and the process feature bit; enabling only one produces no output and no error.
- **`api_dependency_total` is a fork-local metric.** It lives in the vendored tree and will be lost on the next `make vendor-obi` unless the change is present in the `.obi-src` submodule. See [API dependency counter](telemetry/api-dependency-counter.md).

## Related concepts

- [Feature gating](telemetry/feature-gating.md)
- [API dependency counter](telemetry/api-dependency-counter.md)
- [Survey info metrics](telemetry/survey-info-metrics.md)
- [OBI global overrides](obi-integration/obi-global-overrides.md)

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, incident references, dates, decisions that synthesis missed. -->

---

[← Previous](04-APPO11Y-PIPELINE.md) · [Index](../index.md) · [Next →](06-KUBERNETES-INTEGRATION.md)
