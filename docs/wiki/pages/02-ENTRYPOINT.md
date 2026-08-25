<!-- m-wiki: type=top-level slug=02-entrypoint topic=null base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: top-level. 0 sources.

# Entry points and process lifecycle

This repository builds three binaries. `cmd/beyla` is the agent itself; `cmd/k8s-cache` is an optional out-of-process Kubernetes metadata cache; `cmd/beyla-schema` is a build-time tool that emits the JSON schema for the configuration struct. Only the first is long-lived in production, and its startup sequence is strictly ordered: capability checks, configuration load, validation, log setup, then feature fan-out.

## TL;DR

- `cmd/beyla/main.go:main` runs OS-support and capability checks *before* reading configuration, so an unsupported kernel fails fast.
- Configuration comes from a file named by `-config` or `BEYLA_CONFIG_PATH`, merged with environment variables; a nil reader is legal and yields an all-defaults config.
- Capability failures are fatal only when `EnforceSysCaps` is set; otherwise they downgrade to a warning and the agent continues, likely degraded.
- Shutdown is signal-driven: a `signal.NotifyContext` is installed *before* the pipeline is built, so an interrupt during startup still unwinds cleanly.
- `pkg/components/beyla.go:RunBeyla` blocks until every enabled feature goroutine returns; the `errgroup` means one feature's failure cancels the others.

## Mental model

`main` is deliberately dumb. It does five things — check the OS, load config, set the log level, optionally open a pprof port, and hand off — and then blocks. All the interesting composition happens one layer down in `RunBeyla`, which builds a single shared `global.ContextInfo` and then starts one goroutine per enabled feature inside an `errgroup`.

The `errgroup` choice encodes a policy: **features are not independent failure domains.** The comment at `pkg/components/beyla.go:RunBeyla` states it directly — if one node fails, the other should stop. An agent that half-works is treated as worse than an agent that exits and gets restarted by its supervisor.

The ordering of the two capability checks is worth noting. `obi.CheckOSSupport()` runs before any configuration exists, because it asks a question configuration cannot influence (is this kernel capable at all). `obi.CheckOSCapabilities(config.AsOBI())` runs after, because which capabilities are required depends on which features the user enabled.

## Structure / data flow

```
main()
  ├─ slog default handler, level=INFO
  ├─ obi.CheckOSSupport()                    → fatal on unsupported kernel
  ├─ flag -config / $BEYLA_CONFIG_PATH
  ├─ loadConfig()  ──► beyla.LoadConfig(reader)
  ├─ config.Validate()                       → fatal on bad config
  ├─ lvl.UnmarshalText(config.LogLevel)      → fatal on unknown level
  ├─ obi.CheckOSCapabilities(config.AsOBI())
  │      └─ fatal iff config.EnforceSysCaps, else warn
  ├─ optional pprof listener on config.ProfilePort
  ├─ logConfig(config)                       → optional YAML/JSON config dump
  ├─ signal.NotifyContext(SIGINT, SIGTERM)   ← installed BEFORE pipeline build
  └─ components.RunBeyla(ctx, config)        ← blocks here
```

| Binary | Entry | Purpose |
|---|---|---|
| `beyla` | `cmd/beyla/main.go:main` | The agent |
| `k8s-cache` | `cmd/k8s-cache/main.go:main` | Out-of-process Kubernetes metadata cache |
| `beyla-schema` | `cmd/beyla-schema/main.go:SchemaGenerator` | Generates `docs/config-schema.json` from the config struct |

## Key code locations

| What | Where |
|---|---|
| Agent entry point | `cmd/beyla/main.go:main` |
| Config file/env loading | `cmd/beyla/main.go:loadConfig` |
| Optional startup config dump | `cmd/beyla/main.go:logConfig` |
| Feature fan-out under errgroup | `pkg/components/beyla.go:RunBeyla` |
| Health check listener selection | `pkg/components/beyla.go:startHealthCheck` |
| Global input normalisation | `pkg/components/beyla.go:normalizeConfig` |
| App feature bootstrap | `pkg/components/beyla.go:setupAppO11y` |
| Network feature bootstrap | `pkg/components/beyla.go:setupNetO11y` |
| Stats feature bootstrap | `pkg/components/beyla.go:setupStatsO11y` |
| Webhook bootstrap | `pkg/components/beyla.go:setupWebhook` |
| Graceful stop with timeout | `pkg/internal/appolly/appolly.go:Instrumenter.stop` |

## Sharp edges

- **The shutdown hook is registered before `RunBeyla`, on purpose.** The comment in `cmd/beyla/main.go:main` explains why: registering after the pipe build would leak a child process if the target is never found.
- **Health check is either-or, not both.** `pkg/components/beyla.go:startHealthCheck` uses a `switch`: a configured Unix socket path wins outright and the TCP port is never opened.
- **A missing config file is fatal, but no config file is fine.** `cmd/beyla/main.go:loadConfig` exits if an explicitly-named path cannot be opened, yet passes a nil reader through when no path was given at all.
- **`GOCOVERDIR` changes exit timing.** When that variable is set, `main` sleeps one second before returning so coverage data can flush — coverage-instrumented builds are therefore not byte-identical in shutdown latency.

## Related concepts

- [Feature gating](telemetry/feature-gating.md)
- [The swarm instancer model](pipeline/swarm-instancer-model.md)
- [Kubernetes metadata cache service](kubernetes/k8s-metadata-cache-service.md)

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, incident references, dates, decisions that synthesis missed. -->

---

[← Previous](01-ARCHITECTURE.md) · [Index](../index.md) · [Next →](03-CONFIGURATION.md)
