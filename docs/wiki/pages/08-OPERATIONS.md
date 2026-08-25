<!-- m-wiki: type=top-level slug=08-operations topic=null base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: top-level. 0 sources.

# Operating and deploying Beyla

Beyla runs as a privileged agent next to the workloads it instruments — normally a Kubernetes DaemonSet, sometimes a sidecar or a plain process on a VM. This page collects the operational surface: the chart, the self-observability signals, the health endpoint, and where the runbooks live.

## TL;DR

- The Helm chart in `charts/beyla/` ships the DaemonSet, RBAC, optional cache Deployment, ServiceMonitor and injector role.
- A health endpoint is available over TCP or a Unix socket, selected exclusively by `pkg/components/beyla.go:startHealthCheck`.
- Self-observability ("internal metrics") is separate from workload telemetry and is chosen in `pkg/components/beyla.go:internalMetrics`.
- Dashboards and alerts are maintained as a Grafana mixin under `ops/beyla-mixin/`, lint-gated by `make lint-dashboard`.
- Operational playbooks live in `ops/runbook.md` and `ops/troubleshooting.md`.

## Mental model

Three signals answer three different questions.

*Is the agent alive?* — the health listener. `startHealthCheck` opens a Unix-domain socket when `UnixSocketPath` is set, otherwise a TCP port when `Port` is non-zero, otherwise nothing at all. Silence is a valid configuration, so an absent health endpoint is not evidence of a fault.

*Is the agent healthy?* — internal metrics. These describe Beyla itself (queue depths, export errors, probe counts) and are emitted through whichever reporter `internalMetrics` selected.

*Is the workload instrumented?* — survey and application metrics. Survey info metrics in particular exist to report *which processes were discovered*, which is the fastest way to distinguish "Beyla is broken" from "Beyla never matched your process". See [Survey info metrics](telemetry/survey-info-metrics.md).

Capacity is dominated by two things: the number of instrumented executables (one eBPF tracer goroutine each) and the Kubernetes informer cache. `MetaRestrictLocalNode` and the out-of-process cache service are the two levers for the latter.

## Structure / data flow

| Concern | Artefact |
|---|---|
| Agent workload | `charts/beyla/templates/daemon-set.yaml` |
| RBAC | `charts/beyla/templates/cluster-role.yaml`, `cluster-role-binding.yaml`, `serviceaccount.yaml` |
| Config delivery | `charts/beyla/templates/configmap.yaml` |
| Metadata cache service | `charts/beyla/templates/cache-deployment.yaml`, `cache-service.yaml` |
| Prometheus scrape | `charts/beyla/templates/servicemonitor.yaml` |
| SDK injection RBAC | `charts/beyla/templates/injector-role.yaml` |
| Dashboards | `ops/beyla-mixin/dashboards/` |
| Alerts | `ops/beyla-mixin/alerts/alerts.libsonnet` |
| Runbooks | `ops/runbook.md`, `ops/troubleshooting.md` |
| Local demo stack | `deployments/` |

## Key code locations

| What | Where |
|---|---|
| Health listener selection | `pkg/components/beyla.go:startHealthCheck` |
| Internal metrics reporter selection | `pkg/components/beyla.go:internalMetrics` |
| pprof listener | `cmd/beyla/main.go:main` |
| Startup config dump | `cmd/beyla/main.go:logConfig` |
| Enforced capability check behaviour | `cmd/beyla/main.go:main` |
| Shutdown timeout enforcement | `pkg/internal/appolly/appolly.go:Instrumenter.stop` |
| Survey metrics reporter | `pkg/export/otel/metrics_survey.go:SurveyMetricsReporter` |

## Sharp edges

- **Capability warnings are easy to miss.** With `EnforceSysCaps` false — see `cmd/beyla/main.go:main` — a missing capability produces one `WARN` at startup and then silent under-instrumentation.
- **`log_config` prints the whole configuration.** `cmd/beyla/main.go:logConfig` marshals the config to YAML or JSON and writes it to stdout; treat that output as sensitive when Grafana credentials are configured.
- **Health endpoint config is exclusive, not additive.** Setting both a socket path and a port yields only the socket.
- **Dashboard linting only runs on changed files.** `make lint-dashboard` short-circuits when git reports no modified dashboard JSON, so a pre-existing violation can persist unnoticed.
- **Shutdown can exceed expectations.** eBPF probe teardown is bounded by `ShutdownTimeout`; a too-small value turns every restart into a non-zero exit.

## Related concepts

- [Survey info metrics](telemetry/survey-info-metrics.md)
- [Feature gating](telemetry/feature-gating.md)
- [Kubernetes metadata cache service](kubernetes/k8s-metadata-cache-service.md)

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, incident references, dates, decisions that synthesis missed. -->

---

[← Previous](07-BUILD-AND-VENDORING.md) · [Index](../index.md) · [Next →](01-ARCHITECTURE.md)
