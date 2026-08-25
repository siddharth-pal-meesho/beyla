<!-- m-wiki: type=concept slug=sdk-injection-webhook topic=kubernetes base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# SDK injection webhook

An experimental Kubernetes admission controller that mutates other workloads' Pods to preload an OpenTelemetry SDK, turning eBPF-observed services into SDK-instrumented ones. The configuration comment in `pkg/beyla/config.go:Config` marks it explicitly as undocumented and removable without warning — treat it accordingly.

## Where it applies in this repo

`pkg/webhook/server.go:Server` is the coordinator. `pkg/webhook/server.go:NewServer` builds it — beginning with `pkg/webhook/server.go:loadOwnPod`, because the controller needs its own Pod identity before it can reason about others — and `pkg/webhook/server.go:Server.Start` runs it until context cancellation, treating `context.Canceled` as a clean exit rather than an error.

Admission decisions live in `pkg/webhook/mutator.go:PodMutator`, built by `pkg/webhook/mutator.go:NewPodMutator`. Eligibility is a conjunction of checks: `pkg/webhook/mutator.go:PodMutator.CanInstrument`, `pkg/webhook/mutator.go:PodMutator.CanInstrumentLanguage`, and the conflict guard `pkg/webhook/mutator.go:PodMutator.PreloadsSomethingElse`, which refuses to fight an existing `LD_PRELOAD`.

Language detection is layered and heuristic: `pkg/webhook/mutator.go:detectLanguageFromPodSpec`, then `pkg/webhook/mutator.go:detectLanguageFromContainer`, falling back to `pkg/webhook/mutator.go:languageFromImageName`.

Cluster state is filtered to the local node by `pkg/webhook/server.go:Server.isMyNodeEvent` and driven by informer callbacks in `pkg/webhook/server.go:Server.On`. Discovered eligible deployments are cached in an LRU and persisted to a ConfigMap by `pkg/webhook/server.go:Server.writeStateConfigMap`.

## Why this design

Two design decisions dominate.

**Debouncing.** ConfigMap writes and eligible-deployment rebuilds are both debounced — `stateConfigMapDebounceDelay` and `rebuildDeploymentsDebounceDelay` are each 10 seconds, polled on a one-second tick. Pod churn is bursty, and a controller that wrote to the API server on every event would amplify that burst back at the control plane.

**Bounded state.** The eligible-deployment cache is an LRU capped at `maxEligibleDeployments` (10,000). A controller watching a whole cluster must not have unbounded memory growth as its normal operating mode.

Node-locality filtering follows from the DaemonSet deployment model: each instance handles only what runs beside it, so the work partitions naturally across the fleet.

## Related

- [Kubernetes integration](../06-KUBERNETES-INTEGRATION.md)
- [Kubernetes metadata cache service](k8s-metadata-cache-service.md)
- [Configuration model](../03-CONFIGURATION.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
