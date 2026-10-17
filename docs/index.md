# Documentation index

> Map of every documentation surface in this repository. Start here to find the right
> document, then follow the link.
>
> This index is navigational only — it holds no content of its own. When you add,
> move, or delete a doc, update the matching row here in the same PR.

This repo is **Grafana Beyla** (Meesho fork), an eBPF zero-code instrumentation agent.
Most of the instrumentation engine itself lives upstream in
[OpenTelemetry eBPF Instrumentation (OBI)](https://github.com/open-telemetry/opentelemetry-ebpf-instrumentation),
embedded here as the `.obi-src` git submodule and vendored into `vendor/`. See
[`.github/instructions/obi-integration.instructions.md`](../.github/instructions/obi-integration.instructions.md)
for what that means when changing code.

## Where documentation lives

| Location | Audience | Published? |
|---|---|---|
| [`docs/sources/`](sources/_index.md) | Users of Beyla | Yes — rendered to grafana.com by Hugo |
| [`devdocs/`](../devdocs/README.md) | Contributors to this repo | No |
| [`ops/`](../ops/runbook.md) | Operators running Beyla | No |
| [`pkg/webhook/docs/`](../pkg/webhook/docs/design.md) | Contributors to the webhook | No |
| [`.github/instructions/`](../.github/instructions/) | Coding agents and reviewers | No |
| `docs/` root (this file, [acronyms](acronyms.md)) | Everyone | No |

Only `docs/sources/**` is built by Hugo and linted by Vale in CI
(`.github/workflows/vale.yml`). Files at `docs/` root are repo-local notes and are not
part of the published site.

## User documentation — `docs/sources/`

Ordered as the published site orders it (by front-matter `weight`).

**[Grafana Beyla](sources/_index.md)** — landing page: what Beyla is, requirements, and stability guarantees.

### [Set up](sources/setup/_index.md)

| Page | What it covers |
|---|---|
| [Standalone](sources/setup/standalone.md) | Run Beyla as a standalone Linux process |
| [Docker](sources/setup/docker.md) | Run Beyla as a container instrumenting another container |
| [Helm chart for Grafana Cloud](sources/setup/kubernetes-helm-appolly.md) | Deploy for Knowledge Graph and Application Observability in Grafana Cloud |
| [Kubernetes Monitoring Helm chart](sources/setup/kubernetes-helm-k8s-monitoring.md) | Deploy via the Kubernetes Monitoring Helm chart |
| [Helm chart for other use cases](sources/setup/kubernetes-helm.md) | Deploy the standalone Beyla Helm chart |
| [Kubernetes](sources/setup/kubernetes.md) | Deploy in Kubernetes without Helm |
| [Grafana Alloy Helm chart](sources/setup/helm-alloy.md) | Run Beyla inside Grafana Alloy |

### [Quickstart](sources/quickstart/_index.md)

One guide per language — [C/C++](sources/quickstart/cpp.md), [Go](sources/quickstart/golang.md),
[Java](sources/quickstart/java.md), [Node.js](sources/quickstart/nodejs.md),
[Python](sources/quickstart/python.md), [Ruby](sources/quickstart/ruby.md),
[Rust](sources/quickstart/rust.md) — plus a
[Kubernetes quickstart](sources/quickstart/kubernetes.md) that exports to Grafana Cloud.
The runnable sample services behind these guides live in
[`examples/quickstart/`](../examples/quickstart/README.md).

### [Network metrics](sources/network/_index.md)

| Page | What it covers |
|---|---|
| [Quickstart](sources/network/quickstart.md) | Produce network metrics from Beyla |
| [Set up Asserts network](sources/network/asserts.md) | Install network metrics in Kubernetes with Helm for Asserts |
| [Measure traffic between Cloud availability zones](sources/network/inter-az.md) | Inter-AZ traffic measurement |
| [Configuration](sources/network/config.md) | Network metrics configuration options |

### [Configure](sources/configure/_index.md)

| Page | What it covers |
|---|---|
| [Export modes](sources/configure/export-modes.md) | Export directly to OTLP or through Alloy |
| [Global properties](sources/configure/options.md) | Global configuration properties for Beyla core |
| [Export data](sources/configure/export-data.md) | Prometheus and OpenTelemetry metric and trace export |
| [Service discovery](sources/configure/service-discovery.md) | How Beyla searches for processes to instrument |
| [Metrics attributes](sources/configure/metrics-traces-attributes.md) | Which attributes are reported, instance ID and Kubernetes metadata decoration |
| [Name resolver](sources/configure/name-resolver.md) | Service host and peer name resolution |
| [Controlling instrumentation](sources/configure/controlling-instrumentation.md) | Per-protocol and per-language instrumentation behavior |
| [Language specific agents](sources/configure/language-agents.md) | Options for Beyla's language specific agents |
| [Filter data](sources/configure/filter-metrics-traces.md) | Filter metrics and traces by attribute values |
| [Routes decorator](sources/configure/routes-decorator.md) | Route grouping before data moves down the pipeline |
| [Metrics histograms](sources/configure/metrics-histograms.md) | Native and exponential histogram configuration |
| [Sample traces](sources/configure/sample-traces.md) | OpenTelemetry trace sampling |
| [Internal metrics reporter](sources/configure/internal-metrics-reporter.md) | Metrics about Beyla's own behavior |
| [Tune performance](sources/configure/tune-performance.md) | Tune the eBPF tracer's overhead |
| [YAML example](sources/configure/example.md) | A complete example configuration file |

### Reference and background

| Page | What it covers |
|---|---|
| [Exported metrics](sources/metrics.md) | The HTTP/gRPC metrics Beyla exports |
| [Security](sources/security.md) | Privileges and Linux capabilities Beyla requires |
| [Distributed traces](sources/distributed-traces.md) | Beyla's distributed tracing support |
| [Cilium compatibility](sources/cilium-compatibility.md) | Running Beyla alongside Cilium |
| [RED metrics dashboard](sources/beyla-dashboard.md) | Using the Beyla RED metrics dashboard |
| [Metrics cardinality](sources/cardinality.md) | Estimating cardinality of a default install |
| [Performance overhead](sources/performance.md) | Measured overhead and the methodology behind it |
| [Measuring request times](sources/requesttime.md) | Request time versus service time |

Supporting assets: [`sources/assets/`](sources/assets/) (images),
[`sources/flowchart/`](sources/flowchart/) (Mermaid `.mmd` sources — regenerate the PNGs
with `make -C docs mermaid`), and
[`sources/configure/resources/`](sources/configure/resources/) (sample Alloy and
instrumenter configs referenced by the configure pages).

## Contributor documentation — `devdocs/`

| Page | What it covers |
|---|---|
| [README](../devdocs/README.md) | What belongs in `devdocs/` |
| [Pipeline map](../devdocs/pipeline-map.md) | The two Beyla pipelines and every stage in them |
| [Add a new TCP based BPF tracer](../devdocs/new-tracer.md) | Steps to add a protocol tracer |
| [Profiling](../devdocs/profiling.md) | Profiling Beyla via `BEYLA_PROFILE_PORT` and pprof |
| [Release](../devdocs/new-release.md) | The release train, and how Beyla and OBI versions stay in lockstep |

## Operations — `ops/`

| Page | What it covers |
|---|---|
| [Runbook](../ops/runbook.md) | Per-alert response procedures |
| [Troubleshooting](../ops/troubleshooting.md) | Diagnosing a Beyla that crashes or reports nothing |
| [Monitoring mixin](../ops/beyla-mixin/README.md) | Grafana dashboards and alerts for Beyla |

## Component documentation

| Page | What it covers |
|---|---|
| [Webhook package](../pkg/webhook/README.md) | The mutating admission webhook that injects OpenTelemetry SDKs |
| [Webhook design](../pkg/webhook/docs/design.md) | Instrumentation modes and how injection works |
| [Injector image testing](../pkg/webhook/docs/ImageTesting.md) | Building and inspecting the injector image locally |

## Helm charts and examples

| Page | What it covers |
|---|---|
| [Chart contributing guide](../charts/README.md) | Run `make helm-docs` after changing the chart |
| [`beyla` chart reference](../charts/beyla/README.md) | Chart values reference — **generated**, do not hand-edit |
| [Alloy example](../examples/alloy/README.md) | Running Beyla inside Alloy on Kubernetes |
| [Quickstart sample services](../examples/quickstart/README.md) | Sample apps used by the quickstart guides |

## Repo-local references

| Page | What it covers |
|---|---|
| [Acronyms](acronyms.md) | Domain acronyms used across this repo's code and docs |
| [`config-schema.json`](config-schema.json) | JSON Schema for Beyla configuration — **generated** by `make generate-config-schema`, verified in CI by `make check-config-schema` |
| [Go instructions](../.github/instructions/go.instructions.md) | Go conventions enforced on this repo |
| [OBI integration instructions](../.github/instructions/obi-integration.instructions.md) | Rules for touching vendored and submodule OBI code |

## Project governance

[README](../README.md) ·
[Maintainers](../MAINTAINERS.md) ·
[Governance](../GOVERNANCE.md) ·
[Code of conduct](../CODE_OF_CONDUCT.md) ·
[License](../LICENSE)

## Building the docs locally

```sh
make -C docs docs      # serve docs/sources locally (prints the URL; default port 3002)
make -C docs vale      # lint docs/sources with the Grafana Vale style
make -C docs mermaid   # regenerate diagram PNGs from docs/sources/flowchart/*.mmd
```

Both `docs` and `vale` run containers via `docs/make-docs` against `docs/sources` only;
nothing else under `docs/` is included in either the build or the lint.
