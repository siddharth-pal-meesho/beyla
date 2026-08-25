<!-- m-wiki: type=top-level slug=04-appo11y-pipeline topic=null base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: top-level. 0 sources.

# Application observability pipeline

Application observability is Beyla's primary mode. It has two halves that are deliberately kept separate: a *finder* that watches the process table and attaches eBPF probes, and a *reader/decorator* that pulls spans out of those probes and forwards them to exporters. The split exists so the two can eventually run as separate executables with different privilege levels.

## TL;DR

- `pkg/internal/appolly/appolly.go:Instrumenter` owns the whole feature: discovery, tracer lifecycle, and the export pipeline runner.
- Discovery and export communicate through typed message queues, not direct calls — `tracesInput` carries `[]request.Span`, `processEventInput` carries process lifecycle events.
- The export pipeline is assembled by `pkg/internal/pipe/instrumenter.go:Build`, which nests the **vendored OBI pipeline** inside a Beyla-owned swarm alongside extra stages.
- Every discovered executable gets its own `ebpf.ProcessTracer` goroutine, tracked by a `WaitGroup` so shutdown can wait for probe teardown.
- Process events flow through a second, independent graph that decorates them with host, Kubernetes and Docker metadata.

## Mental model

Read `pkg/internal/appolly/appolly.go:New` as a constructor that builds *two* graphs and one queue set.

The **process-event graph** (`peGraphBuilder`) is a three-stage decorator chain: host → Kubernetes → Docker. Its input is fed by the instrumented event loop whenever a process appears or dies; its output drives survey-style info metrics.

The **span pipeline** (`bp`) is built by `pkg/internal/pipe/instrumenter.go:Build`. That function's own comment describes it as "a swarm containing two swarms": the first swarm is OBI's real `appolly.Build` pipeline, the second is Beyla's process-metrics sub-pipeline connected to the first's output. Between them, three optional stages are conditionally added — the Alloy traces receiver, the cluster-connector subpipeline, and the Sigil GenAI export.

`FindAndInstrument` and `ReadAndForward` are called in sequence by `pkg/components/beyla.go:setupAppO11y`, but only the second blocks. The first starts background loops and returns immediately; the second starts the pipeline and waits on its `Done()` channel, then runs a bounded graceful stop.

## Structure / data flow

```
ProcessFinder (pkg/internal/discover)
      │  Event[*ebpf.Instrumentable]
      ▼
Instrumenter.instrumentedEventLoop
      ├─ EventCreated  → go tracer.Run(ctx, ebpfEventContext, tracesInput)   [WaitGroup +1]
      ├─ EventDeleted  → tracer.UnlinkExecutable(fileInfo)
      └─ dispatches exec.ProcessEvent ──► processEventInput
                                              │
                        ┌─────────────────────┘
                        ▼
        HostProcessEventDecorator → KubeProcessEventDecorator → DockerProcessEventDecorator

tracesInput ([]request.Span)
      ▼
pipe.Build  ─┬─ vendored OBI appolly.Build          ← decoration + core exporters
             ├─ alloy.TracesReceiver
             ├─ clusterConnectorsSubpipeline        (optional: topology inter-cluster)
             ├─ sigilExportSubpipeline              (optional: SigilExport enabled)
             └─ ProcessMetricsSwarmInstancer        (optional: process feature + exporter)
```

For the stage-by-stage view *inside* the vendored OBI pipeline — read decorator, routes, Kubernetes decorator, name resolver, attribute filter — see `devdocs/pipeline-map.md`, which documents the engine's own graph.

## Key code locations

| What | Where |
|---|---|
| Feature owner type | `pkg/internal/appolly/appolly.go:Instrumenter` |
| Graph + queue construction | `pkg/internal/appolly/appolly.go:New` |
| Start discovery, non-blocking | `pkg/internal/appolly/appolly.go:Instrumenter.FindAndInstrument` |
| Tracer lifecycle event loop | `pkg/internal/appolly/appolly.go:Instrumenter.instrumentedEventLoop` |
| Run pipeline, blocking | `pkg/internal/appolly/appolly.go:Instrumenter.ReadAndForward` |
| Bounded graceful stop | `pkg/internal/appolly/appolly.go:Instrumenter.stop` |
| Kubernetes informer warm-up | `pkg/internal/appolly/appolly.go:setupKubernetes` |
| Pipeline assembly | `pkg/internal/pipe/instrumenter.go:Build` |
| Inter-cluster connector spans | `pkg/internal/pipe/instrumenter.go:clusterConnectorsSubpipeline` |
| Sigil GenAI export branch | `pkg/internal/pipe/instrumenter.go:sigilExportSubpipeline` |
| Process metrics sub-pipeline | `pkg/internal/pipe/proc_pipeline.go:ProcessMetricsSwarmInstancer` |
| Process finder | `pkg/internal/discover/finder.go:ProcessFinder` |

## Sharp edges

- **Shutdown is bounded and can fail.** `pkg/internal/appolly/appolly.go:Instrumenter.stop` races the tracer `WaitGroup` against `config.ShutdownTimeout` and returns `errShutdownTimeout` if probes do not unload in time. That error propagates up and makes the process exit non-zero.
- **The runtime-metrics queue can legitimately be nil.** `pkg/internal/appolly/appolly.go:newRuntimeMetricsQueue` returns `nil` when no runtime metric feature is on or no exporter endpoint is configured; downstream code must tolerate it.
- **Cluster-connector spans silently disable themselves.** `pkg/internal/pipe/instrumenter.go:clusterConnectorsSubpipeline` returns early when the topology option is absent, when Kubernetes is not enabled, *or* when the informer store cannot be fetched — the last case logs an error and continues rather than failing startup.
- **The process-metrics sub-pipeline subscribes eagerly.** `pkg/internal/pipe/proc_pipeline.go:ProcessMetricsSwarmInstancer` calls `Subscribe` outside the instancer closure, with a comment explaining that this must happen early enough to catch every message from the vendored OBI swarm.
- **An unknown discovery event type is a logged bug, not a crash** — the default branch of the event loop emits `BUG ALERT!` and keeps going.

## Related concepts

- [The swarm instancer model](pipeline/swarm-instancer-model.md)
- [Process discovery and survey mode](pipeline/process-discovery-and-survey.md)
- [Process metrics sub-pipeline](pipeline/process-metrics-subpipeline.md)
- [Cluster connector spans](pipeline/cluster-connector-spans.md)

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, incident references, dates, decisions that synthesis missed. -->

---

[← Previous](03-CONFIGURATION.md) · [Index](../index.md) · [Next →](05-EXPORTERS.md)
