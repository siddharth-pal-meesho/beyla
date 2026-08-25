<!-- m-wiki: type=concept slug=cluster-connector-spans topic=pipeline base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# Cluster connector spans

An optional trace branch that emits synthetic "connector" spans for traffic identified as leaving the local Kubernetes cluster. Tempo uses them to stitch together service-graph edges that span clusters — connections Beyla cannot resolve on its own because it has no metadata about the remote side.

## Where it applies in this repo

`pkg/internal/pipe/instrumenter.go:clusterConnectorsSubpipeline` assembles the branch and guards it with three preconditions: the `TopologyInterCluster` value must appear in `config.Topology.Spans`, the Kubernetes informer must be enabled, and the informer store must be retrievable.

Given those, it creates an `externalTraces` queue and registers three stages:

- `traces.SelectExternal` — filters the main span stream, classifying a peer as external when `store.ObjectMetaByIP(ip)` returns nil (the cluster's metadata store has never seen that address).
- `alloy.ConnectionSpansReceiver` — makes the same spans available to an in-process Alloy consumer.
- `pkg/export/otel/connect_spans.go:ConnectionSpansExport` — exports them over OTLP.

The span synthesis itself lives in `pkg/export/otel/connect_spans.go:GenerateConnectSpans`, with grouping in `pkg/export/otel/connect_spans.go:GroupConnectionSpans` and attribute selection in `pkg/export/otel/connect_spans.go:ConnectionSpanAttributes`.

## Why this design

A service graph built from single-cluster observation has holes exactly where traffic crosses a cluster boundary: the local agent sees a call to an IP address it cannot name, and the remote agent sees an inbound call from an IP *it* cannot name. Neither side can close the edge alone.

Emitting an explicit connector span moves the join to the backend, which can see both clusters' data. The "is this external?" test is deliberately cheap and local — absence from the Kubernetes metadata store — rather than an attempt at cross-cluster discovery.

The failure mode is chosen carefully. If the informer store cannot be fetched, `clusterConnectorsSubpipeline` logs an error and returns, disabling the feature while leaving the rest of the agent running. A partial service graph beats an agent that will not start.

## Related

- [Kubernetes integration](../06-KUBERNETES-INTEGRATION.md)
- [Exporters and telemetry surface](../05-EXPORTERS.md)
- [The swarm instancer model](swarm-instancer-model.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
