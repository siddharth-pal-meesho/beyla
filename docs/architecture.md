# Architecture — grafana/beyla (Meesho fork)

> Companion to [`docs/wiki/`](wiki/index.md). This file is the code-grounded map (entry points,
> module boundaries, build loop, invariants). The wiki carries the long-form narrative and the
> per-concept pages; where a topic has a wiki page, this document links rather than restates it.

---

## Section 1 — High-level design

### Service purpose

Beyla is a **zero-code eBPF auto-instrumentation agent**, not a request-serving microservice. It
attaches to already-running processes on a host or Kubernetes node, reads HTTP/gRPC/SQL/messaging
activity out of eBPF ring buffers, and exports RED metrics and OpenTelemetry trace spans to an OTLP
endpoint, a Prometheus scrape endpoint, or an in-process Grafana Alloy receiver.

Since v3 this repository is a **distribution**, not the engine. The discovery, probing, decoration
and core-export machinery lives in `go.opentelemetry.io/obi` (OpenTelemetry eBPF Instrumentation,
"OBI"), consumed through a `replace` directive pointing at the `.obi-src` Git submodule and compiled
from `vendor/go.opentelemetry.io/obi/`. What `github.com/grafana/beyla/v3` itself owns is a
Grafana-flavoured config type, a set of global naming overrides, a handful of extra pipeline stages,
and a Kubernetes SDK-injection admission webhook that OBI does not ship.

**This fork** (`siddharth-pal-meesho/beyla`, default working branch `develop`) tracks upstream
`grafana/beyla` and repoints `.obi-src` at a Meesho fork of OBI carrying not-yet-upstream engine
changes — currently the `api_dependency_total` metric and trace-graph/parenting fixes.

### System context

| Direction | What | Where |
|---|---|---|
| Inbound (data) | eBPF ring buffers from instrumented processes on the same host/node | vendored OBI probes |
| Inbound (control) | YAML config file (`-config` / `$BEYLA_CONFIG_PATH`) + `BEYLA_*` env vars | `cmd/beyla/main.go:loadConfig` |
| Inbound (control) | Kubernetes API — informers for Pod/ReplicaSet metadata | vendored OBI `kube/`, or `cmd/k8s-cache` |
| Inbound (HTTP) | Admission review requests, when the injection webhook is enabled | `pkg/webhook/server.go:NewServer` |
| Outbound | OTLP metrics + traces (incl. Grafana Cloud credential path) | `pkg/beyla/config_obi.go:overrideOBI` |
| Outbound | Prometheus scrape endpoint | `pkg/export/prom/` + vendored OBI |
| Outbound | Grafana Alloy in-process traces receiver | `pkg/export/alloy/traces.go` |
| Outbound | Sigil GenAI trace export | `pkg/export/otel/sigil_export.go` |

There are **no Meesho platform dependencies** — no go-core, mCache, mQ, mSearch, Prism, MySQL,
Scylla, or ZooKeeper. Consequently this repo has no `docs/downstreams.md` / `docs/infrastructure.md`;
there is nothing for them to describe.

### Module boundaries

Three concentric rings — the framing is developed in
[wiki 01-ARCHITECTURE](wiki/pages/01-ARCHITECTURE.md).

| Ring | Directory | Owns |
|---|---|---|
| Inner — engine | `vendor/go.opentelemetry.io/obi/` | eBPF probes, process discovery, span decoration, core OTLP/Prom exporters. **Generated state.** |
| Middle — config & naming | `pkg/beyla/`, `pkg/helpers/config/`, `pkg/config/` | Beyla `Config`, reflection conversion to OBI config, `BEYLA_*`→`OTEL_EBPF_*` and `beyla.*` metric renames |
| Outer — extra stages | `pkg/internal/pipe/`, `pkg/export/`, `pkg/webhook/`, `pkg/services/` | Process metrics, cluster-connector spans, Sigil export, survey info metrics, SDK injection, selection criteria |

Supporting trees: `cmd/` (three binaries), `charts/beyla` (Helm), `internal/test/` +
`internal/testgenerated/` (integration suites), `internal/tools/` (pinned tool module),
`docs/sources/` (published Grafana docs, Vale-linted), `ops/` (mixin, runbook), `devdocs/`
(contributor notes incl. the pipeline mermaid map).

### Architecture philosophy

<!-- assumption: Phase 5/6 interviews were not run — this session is non-interactive. The statements
     below are inferred from code and commit history, not confirmed by a maintainer. -->

- **Vendor the engine, don't fork in place.** Generating eBPF objects needs clang and a container
  image; vendoring the generated output keeps `go build` working with nothing but a Go toolchain.
  The cost is that `vendor/` is *generated state the toolchain will overwrite*.
- **Features are not independent failure domains.** `pkg/components/beyla.go:RunBeyla` runs every
  enabled feature in one `errgroup`, so one failure cancels the rest — a half-working agent is
  treated as worse than one that exits and gets restarted.
- **Enablement is derived, not declared.** A feature is "on" when *some exporter is configured for
  it*, not when a boolean is set (`pkg/beyla/config.go:Config.appO11yEnabled` and siblings). This
  removes the "enabled but pointed nowhere" class of misconfiguration.
- **Bounded by construction.** Fork-local telemetry uses hard caps with explicit overflow counters
  rather than unbounded maps — see the API dependency tracker below.

### Data flow

```
cmd/beyla/main.go:main
  ├─ obi.CheckOSSupport()                  → fatal on unsupported kernel (before config exists)
  ├─ loadConfig()  → beyla.LoadConfig      (YAML + BEYLA_* env, nil reader ⇒ all defaults)
  ├─ config.Validate() / log level / obi.CheckOSCapabilities(config.AsOBI())
  ├─ signal.NotifyContext(SIGINT, SIGTERM) ← installed BEFORE the pipeline is built, on purpose
  └─ pkg/components/beyla.go:RunBeyla      ← blocks
        │  buildCommonContextInfo → global.ContextInfo (Prom mgr, OTEL instancer, K8s informer,
        │                                               Docker store, internal metrics)
        ├─ FeatureAppO11y   → pkg/internal/appolly → pkg/internal/pipe/instrumenter.go:Build
        │                        ├─ vendored OBI appolly.Build      ← the real engine, as a node
        │                        ├─ alloy.TracesReceiver
        │                        ├─ clusterConnectorsSubpipeline
        │                        ├─ sigilExportSubpipeline
        │                        └─ ProcessMetricsSwarmInstancer
        ├─ FeatureNetO11y   → vendored OBI netolly  agent.FlowsAgent
        ├─ FeatureStatsO11y → vendored OBI statsolly agent.StatsAgent
        └─ Injector.Webhook → pkg/webhook:NewServer
```

The stage-level view of the app pipeline (discovery finder → decoration → exporters) is drawn as
mermaid in [`devdocs/pipeline-map.md`](../devdocs/pipeline-map.md); the composition mechanism is
[wiki: the swarm instancer model](wiki/pages/pipeline/swarm-instancer-model.md).

### Cross-cutting concerns

| Concern | Implementation | Notes |
|---|---|---|
| Config conversion | `pkg/helpers/config/convert.go:Convert` | Reflection, matches by field name, **panics on mismatch** unless the field is listed `SkipConversion` |
| Global naming | `pkg/beyla/config_obi.go:OverrideOBIGlobalConfig` | Duplicates `BEYLA_*` env into `OTEL_EBPF_*`; rewrites vendor prefix to `beyla` |
| Feature gating | Two independent layers | [wiki: feature gating](wiki/pages/telemetry/feature-gating.md) |
| Logging | stdlib `log/slog` only | No zap/logrus/zerolog anywhere in `pkg/` or `cmd/` |
| Env interpolation in YAML | `pkg/config/env_replacer.go:ReplaceEnv` | Supports `${VAR}`, `${env:VAR}`, `${VAR:-default}`, `$$`-escape |
| Error handling | stdlib `errors` + `fmt.Errorf("%w")` | No custom error framework; `errorlint`/`errcheck` enforced by golangci |

---

## Section 2 — Low-level implementation

### Entry points

| Binary | Entry | Purpose |
|---|---|---|
| `beyla` | `cmd/beyla/main.go:main` | The agent. Long-lived, privileged. |
| `k8s-cache` | `cmd/k8s-cache/main.go:main` | Optional out-of-process Kubernetes metadata cache — [wiki page](wiki/pages/kubernetes/k8s-metadata-cache-service.md) |
| `beyla-schema` | `cmd/beyla-schema/main.go:SchemaGenerator` | Build-time tool; emits `docs/config-schema.json`. **Requires an initialised `.obi-src`** — it reads `.obi-src/pkg/obi` and exits non-zero without it. |

### Key code locations

| What | Where |
|---|---|
| Feature fan-out and lifecycle | `pkg/components/beyla.go:RunBeyla` |
| Shared context construction | `pkg/components/beyla.go:buildCommonContextInfo` |
| Health-check listener (UDS **or** TCP, never both) | `pkg/components/beyla.go:startHealthCheck` |
| Beyla configuration root | `pkg/beyla/config.go:Config` |
| Beyla → OBI config bridge (memoised) | `pkg/beyla/config_obi.go:Config.AsOBI` |
| OBI → Beyla config bridge | `pkg/beyla/config_obi.go:FromOBI` |
| Grafana Cloud endpoint/header override | `pkg/beyla/config_obi.go:overrideOBI` |
| Global name/prefix overrides | `pkg/beyla/config_obi.go:OverrideOBIGlobalConfig` |
| App pipeline assembly | `pkg/internal/pipe/instrumenter.go:Build` |
| Cluster-connector subpipeline | `pkg/internal/pipe/instrumenter.go:clusterConnectorsSubpipeline` |
| Sigil GenAI subpipeline | `pkg/internal/pipe/instrumenter.go:sigilExportSubpipeline` |
| Process-metrics subpipeline gate | `pkg/internal/pipe/proc_pipeline.go:isProcessSubPipeEnabled` |
| Admission webhook server | `pkg/webhook/server.go:NewServer`, mutation in `pkg/webhook/mutator.go` |
| Selection criteria / default excludes | `pkg/services/criteria.go`, `pkg/services/instrumentable_type.go` |
| Fork-local API dependency metric | `vendor/go.opentelemetry.io/obi/pkg/export/otel/api_dependency.go:newAPIDepTracker` |
| Vendored OBI module pin | `go.mod` — `replace go.opentelemetry.io/obi => ./.obi-src` |

### API surface

Beyla exposes no business API. Its listeners are all operational and all optional:

| Listener | Config key | Code |
|---|---|---|
| Prometheus scrape | `prometheus_export.port` / `.path` | vendored OBI + `pkg/export/prom/` |
| Internal metrics (Prometheus) | `internal_metrics.prometheus.port` / `.path` | `pkg/components/beyla.go:internalMetrics` |
| Health check | `health_check.port` **or** `health_check.unix_socket_path` | `pkg/components/beyla.go:startHealthCheck` |
| pprof | `profile_port` (`BEYLA_PROFILE_PORT`) | `cmd/beyla/main.go` |
| Admission webhook | `injector.webhook.*` | `pkg/webhook/server.go` |

### Telemetry surface (what this repo adds on top of OBI)

| Family | Code | Notes |
|---|---|---|
| Survey info metrics | `pkg/export/otel/metrics_survey.go`, `pkg/export/prom/prom_survey.go` | Which processes were *discovered and matched*, independent of export — [wiki page](wiki/pages/telemetry/survey-info-metrics.md) |
| Process metrics | `pkg/export/otel/metrics_proc.go`, `pkg/export/prom/prom_proc.go` | Gated by the `application_process` feature |
| Cluster-connector spans | `pkg/export/otel/connect_spans.go` | Tags external traffic `beyla.topology=external` |
| Sigil GenAI export | `pkg/export/otel/sigil_export.go` | Separate TracesConfig, env prefix `BEYLA_GRAFANA_AI_` |
| `api_dependency_total` **(fork-local)** | `vendor/.../export/otel/api_dependency.go` | Joins outbound client spans to their entry span. Labels: `entry_api`, `dst_api`, `server_address`, `parented` (`w3c`\|`inferred`). Caps: 8192 pending traces / 64 egress per trace / 2048 label pairs, overflow into `api_dependency_overflow_total`. [wiki page](wiki/pages/telemetry/api-dependency-counter.md) |

### Configuration touch points

`pkg/beyla/config.go:Config` is the root. Fields are YAML-tagged and mostly `BEYLA_*`-env-tagged;
defaults come from OBI's defaults via `FromOBI(&obi.DefaultConfig)` in
`pkg/beyla/config.go:DefaultConfig`, with Beyla-specific overrides applied after.

Runtime-semantic keys worth knowing:

| Key | Effect |
|---|---|
| `enforce_sys_caps` | Whether missing kernel capabilities are fatal or only warned about |
| `health_check.unix_socket_path` | Wins outright over `health_check.port` — the TCP listener is then never opened |
| `grafana.otlp.submit` | Defaults to `["traces"]` only; metrics go through span2metrics unless added |
| `discovery.*` | Selection criteria; `DefaultExcludeServices` / `DefaultExcludeInstrument` come from `pkg/services` |
| `topology` | Enables inter-cluster connection spans |
| `injector.webhook` | Enables the experimental admission controller |
| `channel_send_timeout_panic` | Turns a blocked internal queue into a panic — debugging aid, not for production |

Fields that exist on one side of the Beyla↔OBI boundary only must be declared in the
`SkipConversion` map in `pkg/beyla/config_obi.go:FromOBI` — currently `.obi`, `.TracesReceiver`,
`.SigilExport`, `.Processes`, `.Grafana`, `.Topology`, `.Discovery.Survey`, `.Injector`.

### Build & vendoring loop

Full detail: [wiki 07-BUILD-AND-VENDORING](wiki/pages/07-BUILD-AND-VENDORING.md).

```
make vendor-obi = obi-submodule → docker-generate (clang/bpf2go, inside .obi-src)
                → generate-obi-tests → copy-obi-vendor (go get / tidy / vendor)
make verify     = prereqs + lint-dashboard + vendor-obi + lint + test
make build      = vendor-obi + verify + compile
```

A fork-local engine change is a **three-part commit**: the change lands in the `.obi-src` submodule
repo, the submodule pointer moves here, and the regenerated `vendor/` diff is committed alongside.
`git show b4da5978` is exactly that shape and is the reference example.

### Critical invariants & gotchas

1. **`vendor/` is generated state.** Editing `vendor/go.opentelemetry.io/obi/**` compiles and works,
   and `make vendor-obi` (and therefore `make build` and `make verify`) silently erases it. This is
   the single most common way to lose work here. Use `make compile` for a plain build.
2. **`.obi-src` is empty on a fresh clone.** Builds still succeed from `vendor/` alone, but every
   generation target — `make generate`, `docker-generate`, `generate-config-schema`,
   `check-config-schema` — fails until `make obi-submodule` runs. Verified: `go run ./cmd/beyla-schema`
   fails with `open .obi-src/pkg/obi: no such file or directory`.
3. **Config conversion panics on mismatch.** Adding a Beyla-only field to `Config` without adding it
   to the `SkipConversion` map aborts the process at startup, not at compile time.
4. **`AsOBI()` memoises.** The converted OBI config is cached on the `Config` value; mutating the
   Beyla config after the first `AsOBI()` call has no effect on what the engine sees.
5. **The config schema is CI-enforced** (`make check-config-schema`) — but only on PRs into `main`
   or `release-*` (see below).
6. **No CI runs on `develop`.** Every `pull_request_*` workflow is scoped
   `branches: ["main", "release-*"]`. A PR into `develop` — this fork's working branch — runs no
   build, lint, unit tests, schema check, license check, or integration tests. Only `helm-test.yml`
   (unrestricted `pull_request:`) and `vale.yml` (`docs/sources/**` paths) can fire. **Run
   `make lint` and `make test` locally; nothing else will.**
7. **`make test` needs envtest assets.** It shells `setup-envtest` for `KUBEBUILDER_ASSETS`; run
   `make prereqs` first. Plain `go test -mod vendor ./...` works for packages with no envtest
   dependency.
8. **`-mod vendor` everywhere.** Every build/test target passes it explicitly; a bare `go test ./...`
   may resolve differently.
9. **`GOOS`/`GOARCH` default to `linux`/`amd64`** in the Makefile regardless of host — `make compile`
   on a Mac cross-compiles rather than building a native binary.
10. **`EXCLUDE_COVERAGE_FILES` still matches `/beyla/v2/` paths** while the module is `v3`, so those
    exclusions no longer fire and coverage numbers include files that were meant to be excluded.
11. **`make clang-tidy` is broken in this repo** — it does `cd bpf`, and `bpf/` lives in `.obi-src`,
    not here.
12. **`make install-hooks` is conditional.** It only copies `hooks/pre-commit` when
    `.git/hooks/pre-commit` does not already exist, so a hook from another tool silently wins.

### Testing

50 `_test.go` files outside `vendor/`, plain `testing.T` with `stretchr/testify`
(`assert`/`require`), table-driven where it fits. Kubernetes-touching packages use
`controller-runtime` envtest. Integration suites live in `internal/test/integration/` and
`internal/testgenerated/` (the latter is generated by `scripts/generate-obi-tests.sh` and is not in
git); they are driven by docker-compose and the `make integration-test*` / `make oats-test*` targets.

---

<!-- meesho-init: plugin-version=2.1.3 generated-at=2026-08-25T07:40:42Z base-sha=b4da5978747b9823660933b101baebbd8df45c87+dirty -->
