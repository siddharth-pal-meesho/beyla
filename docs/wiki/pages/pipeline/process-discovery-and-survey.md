<!-- m-wiki: type=concept slug=process-discovery-and-survey topic=pipeline base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# Process discovery and survey mode

Discovery is how Beyla decides which processes to instrument. It runs continuously, not once at startup, and has two modes: the normal *instrumentation* path that attaches eBPF probes to matching executables, and *survey* mode, which observes and reports matching processes without instrumenting them.

## Where it applies in this repo

`pkg/internal/discover/finder.go:ProcessFinder` is the Beyla-side wrapper; `pkg/internal/discover/finder.go:NewProcessFinder` constructs it with the config, shared context, the span queue and the shared eBPF event context.

`pkg/internal/discover/finder.go:ProcessFinder.Start` dispatches between two pipeline shapes — `startSuveyPipeline` when only survey criteria exist, and `startMixedPipeline` when instrumentation is also active. `pkg/internal/discover/finder.go:ProcessFinder.IsInstrumentationEnabled` is the predicate behind that choice, and `pkg/internal/discover/finder.go:ProcessFinder.connectSurveySubPipeline` grafts the survey branch onto a Kubernetes-enriched event stream.

Survey selection criteria are assembled in `pkg/internal/discover/survey.go:SurveyCriteriaMatcherProvider`, with `pkg/internal/discover/survey.go:surveyCriteria` and `pkg/internal/discover/survey.go:surveyExcludingCriteria` reading the include and exclude lists from Beyla's discovery configuration.

Consumption of discovery output happens in `pkg/internal/appolly/appolly.go:Instrumenter.instrumentedEventLoop`, which starts a tracer goroutine per created executable and unlinks it on deletion.

## Why this design

Continuous discovery is a requirement, not a refinement: in a container platform the interesting processes appear after the agent does. Modelling it as an event stream of created/deleted/instance-deleted events lets the tracer lifecycle follow process lifecycle exactly.

Survey mode exists because the most common Beyla support question is "why is my service missing?", and the two candidate answers — never discovered, or discovered but not exporting — are otherwise hard to tell apart. Emitting an info metric for matched-but-uninstrumented processes turns that guess into a query. It also offers a low-risk rollout: survey a fleet first, confirm the selection criteria match what you expect, then enable instrumentation.

## Related

- [Survey info metrics](../telemetry/survey-info-metrics.md)
- [The swarm instancer model](swarm-instancer-model.md)
- [Application observability pipeline](../04-APPO11Y-PIPELINE.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
