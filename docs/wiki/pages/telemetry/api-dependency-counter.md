<!-- m-wiki: type=concept slug=api-dependency-counter topic=telemetry base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# API dependency counter (fork-local)

`api_dependency_total` is a metric this fork adds and upstream OBI does not ship. It joins a service instance's outbound client spans to the inbound entry span that caused them, producing a bounded counter of *entry API → downstream API* edges. It exists to answer "which of my endpoints drives this dependency?" — a question service-graph metrics answer only at service granularity.

## Where it applies in this repo

The implementation lives entirely in the vendored tree at `vendor/go.opentelemetry.io/obi/pkg/export/otel/api_dependency.go`, constructed by `vendor/go.opentelemetry.io/obi/pkg/export/otel/api_dependency.go:newAPIDepTracker` and driven by `vendor/go.opentelemetry.io/obi/pkg/export/otel/api_dependency.go:apiDepTracker`.

Every span for the instance passes through `Span`, which routes client spans to `addEgress` (buffer by trace ID) and everything else to `flushEntry` (drain the buffer and emit). The ordering rationale is recorded in the file header: client spans complete *before* their enclosing entry span, so egress calls must be buffered until the entry arrives.

Emitted labels are `entry_api`, `dst_api`, `server_address` and `parented`, where `parented` is `w3c` when the client span carried a valid parent span ID and `inferred` otherwise.

Gating is by the `application_api_dependency` feature — `FeatureAPIDependency` in `vendor/go.opentelemetry.io/obi/pkg/export/feature.go:43` — checked by `setupAPIDependencyMeter` in the vendored `metrics.go`.

## Why this design

Three caps make the tracker incapable of unbounded growth, each with its own failure behaviour: at most 8192 pending traces per instance (oldest evicted), at most 64 buffered egress calls per trace (excess dropped), and at most 2048 distinct label pairs (excess counted into `api_dependency_overflow_total` rather than emitted). Traces whose entry span never arrives — background jobs, evicted buffers — are dropped rather than guessed, with the header noting that service-graph metrics still cover those at coarser granularity.

The FIFO used for eviction is cleaned lazily and compacted only when it grows past twice the cap, trading a little memory for avoiding a sweep on every flush.

> ⚠ This code lives in `vendor/`. It will be erased by `make vendor-obi` unless the same change is present in the `.obi-src` submodule — see [The vendored OBI boundary](../obi-integration/vendored-obi-boundary.md).

## Related

- [Feature gating](feature-gating.md)
- [The vendored OBI boundary](../obi-integration/vendored-obi-boundary.md)
- [Build and the OBI vendoring workflow](../07-BUILD-AND-VENDORING.md)
- [Exporters and telemetry surface](../05-EXPORTERS.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD and from the commit message of `b4da5978747b`.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
