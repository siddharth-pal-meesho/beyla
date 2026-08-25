<!-- m-wiki: type=concept slug=process-metrics-subpipeline topic=pipeline base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# Process metrics sub-pipeline

An optional Beyla-owned branch that turns discovered application spans into per-process resource metrics — CPU time and utilisation, memory usage and virtual size, disk and network I/O — exported through either OTEL or Prometheus. It is a sub-swarm nested inside the main application pipeline.

## Where it applies in this repo

`pkg/internal/pipe/proc_pipeline.go:ProcessMetricsSwarmInstancer` is the whole feature. It is registered last in `pkg/internal/pipe/instrumenter.go:Build`, and it reads the same span queue the exporters use (`ctxInfo.OverrideAppExportQueue`).

Enablement is a conjunction, computed by `pkg/internal/pipe/proc_pipeline.go:isProcessSubPipeEnabled`: at least one metrics endpoint must be enabled (Prometheus *or* OTEL), **and** the `FeatureProcess` bit must be set in `cfg.Metrics.Features`. When either is false the instancer returns `swarm.DirectInstance` with a no-op run function.

Inside, it builds a nested `swarm.Instancer` with a collector (`process.NewCollectorProvider`) feeding a `processCollectStatus` queue, which in turn feeds the OTEL and Prometheus process exporters.

The metric names themselves are declared as `attributes.Name` values at `pkg/export/extraattributes/metric.go:8` — `process.cpu.time`, `process.cpu.utilization`, `process.memory.usage`, `process.memory.virtual`, `process.disk.io` and `process.network.io`, each with its Prometheus spelling.

## Why this design

Process metrics are derived from discovery, not from a separate scrape. Beyla already knows which PIDs belong to which instrumented service, so attributing resource usage to a *service* rather than to a bare PID comes for free — that correlation is the feature's real value, and it is why the collector subscribes to the span queue rather than reading `/proc` independently.

Making it opt-in on two axes is defensive. Sampling per-process resource usage is not free, and neither is the extra metric cardinality, so it activates only when the operator has asked for the feature *and* configured somewhere to send it.

The eager `Subscribe` outside the closure is load-bearing — see [The swarm instancer model](swarm-instancer-model.md).

## Related

- [The swarm instancer model](swarm-instancer-model.md)
- [Exporters and telemetry surface](../05-EXPORTERS.md)
- [Feature gating](../telemetry/feature-gating.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
