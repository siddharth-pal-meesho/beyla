<!-- m-wiki: type=concept slug=feature-gating topic=telemetry base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# Feature gating

Beyla has two independent gating layers, and mixing them up is a common source of "my metric isn't appearing". The outer layer is a bitmask of *observability features* deciding which top-level goroutines start; the inner layer is a per-exporter *feature list* deciding which metric families each exporter emits.

## Where it applies in this repo

**Outer layer.** `pkg/beyla/config.go:Feature` is a `uint` bitmask with three values declared at `pkg/beyla/config.go:49` — `FeatureAppO11y`, `FeatureNetO11y`, `FeatureStatsO11y`. `pkg/beyla/config.go:Config.Enabled` answers whether a given feature is on, backed by helper predicates `pkg/beyla/config.go:Config.appO11yEnabled`, `pkg/beyla/config.go:Config.promNetO11yEnabled`, `pkg/beyla/config.go:Config.otelNetO11yEnabled`, `pkg/beyla/config.go:Config.promStatsO11yEnabled` and `pkg/beyla/config.go:Config.otelStatsO11yEnabled`. Note the shape of those names: a feature is on when *some exporter is configured for it*, not merely when a flag is set.

`pkg/components/beyla.go:RunBeyla` reads all three plus `cfg.Injector.Webhook.Enabled()` and starts one goroutine per enabled feature.

**Inner layer.** The vendored `Features` bitmask in `vendor/go.opentelemetry.io/obi/pkg/export/feature.go` selects metric families by string name — `application`, `application_span`, `application_service_graph`, `application_process`, `application_host`, `application_runtime`, `network`, and this fork's `application_api_dependency`. Beyla consults it in `pkg/internal/pipe/proc_pipeline.go:isProcessSubPipeEnabled` and in `pkg/internal/appolly/appolly.go:newRuntimeMetricsQueue`.

## Why this design

Deriving the outer layer from exporter configuration removes a whole class of misconfiguration: you cannot enable network observability and then forget to point it anywhere, because "enabled" *means* "an exporter is configured". The cost is that the enablement rule is spread across five predicates rather than one boolean.

The inner layer is a string-keyed bitmask because it is a user-facing list in YAML, and because feature names have to remain stable across renames — `application_jvm` survives as a deprecated alias for `application_runtime` in the vendored mapper for exactly this reason.

The practical debugging rule: a missing metric family is usually the *inner* gate, while a missing signal entirely is usually the *outer* one.

## Related

- [Entry points and process lifecycle](../02-ENTRYPOINT.md)
- [API dependency counter](api-dependency-counter.md)
- [Process metrics sub-pipeline](../pipeline/process-metrics-subpipeline.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
