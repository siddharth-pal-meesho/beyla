<!-- m-wiki: type=concept slug=swarm-instancer-model topic=pipeline base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# The swarm instancer model

Beyla composes its processing stages as a *swarm*: stages are registered as instancer functions, instantiated together into a runner, and connected by typed message queues rather than direct calls. The model comes from the vendored OBI module (`pipe/swarm`, `pipe/msg`) and Beyla builds on it in its own pipeline code.

## Where it applies in this repo

There are two phases. **Registration** — a `swarm.Instancer` collects `swarm.InstanceFunc` values via `Add`. **Instantiation** — `Instance(ctx)` turns the collection into a `swarm.Runner` that is later `Start`ed and exposes a `Done()` channel.

`pkg/internal/pipe/instrumenter.go:Build` shows the pattern at its most expressive: it registers a closure that instantiates the entire vendored OBI swarm as *one node*, then adds the Alloy receiver, the conditional connector and Sigil subpipelines, and the process-metrics instancer beside it.

`pkg/internal/appolly/appolly.go:New` builds a second, independent swarm for process lifecycle events — three decorator stages chained host → Kubernetes → Docker, each connected by a queue created through `msg2.QueueFromConfig`.

Conditional stages are expressed as early returns during registration, not as runtime branches: `pkg/internal/pipe/instrumenter.go:clusterConnectorsSubpipeline` and `pkg/internal/pipe/instrumenter.go:sigilExportSubpipeline` simply add nothing when disabled. `pkg/internal/pipe/proc_pipeline.go:ProcessMetricsSwarmInstancer` goes further and returns a `swarm.DirectInstance` no-op, with a comment noting that nothing then subscribes and no extra load is incurred.

## Why this design

Queue-connected stages give three properties that matter for an agent under load. Stages run concurrently without hand-rolled goroutine management. A disabled stage costs nothing, because absence of a subscriber means the producer's messages are never fanned out. And the graph can be split across process boundaries later — the explicit motivation recorded in `devdocs/pipeline-map.md` for keeping discovery and decoration separate.

The subtlety is subscription timing. `ProcessMetricsSwarmInstancer` calls `Subscribe` *outside* the returned closure, deliberately, so the subscription exists before the vendored swarm starts producing. Subscribing lazily inside the instancer would drop early messages.

## Related

- [Application observability pipeline](../04-APPO11Y-PIPELINE.md)
- [Process metrics sub-pipeline](process-metrics-subpipeline.md)
- [Cluster connector spans](cluster-connector-spans.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
