<!-- m-wiki: type=concept slug=vendored-obi-boundary topic=obi-integration base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# The vendored OBI boundary

The line between code this repository authors and code it merely carries. Beyla v3 delegates its entire instrumentation engine to OpenTelemetry eBPF Instrumentation (OBI), consumed not from a module proxy but from a Git submodule at `.obi-src`, vendored into `vendor/go.opentelemetry.io/obi/`. Knowing which side of the boundary a symbol sits on determines where a fix belongs and whether it will survive the next dependency refresh.

## Where it applies in this repo

The boundary is crossed at three well-defined points.

**Configuration** — `pkg/beyla/config_obi.go:Config.AsOBI` converts a Beyla config into the engine's config and caches the result. Nothing in `vendor/` reads Beyla's config type directly.

**Pipeline composition** — `pkg/internal/pipe/instrumenter.go:Build` embeds the engine's own `appolly.Build` swarm inside a Beyla-owned swarm, then adds Beyla stages beside it. The engine is a *node* in Beyla's graph, not its host.

**Feature entry** — the network and stats features hand straight through to vendored agents in `pkg/components/beyla.go:setupNetO11y` and `pkg/components/beyla.go:setupStatsO11y`, which do little more than construct the OBI agent and run it.

The `replace` directive in `go.mod` (`go.opentelemetry.io/obi => ./.obi-src`) means the module version recorded in `vendor/modules.txt` is a label, not a fetchable coordinate.

## Why this design

Vendoring an engine that generates eBPF objects is not a stylistic choice. The generation step needs clang and a container image; making every consumer of this repository run that toolchain to compile a Go binary would be hostile. Vendoring the generated output means `go build` works with nothing but a Go toolchain, while `make vendor-obi` remains available for anyone who needs to regenerate.

The submodule (rather than a published version) exists because this fork carries engine changes that are not upstream yet. It gives those changes a real Git home with history, instead of leaving them as unattributed edits inside `vendor/`.

The cost is a discipline requirement: `vendor/` is generated state, and the toolchain will happily overwrite it.

## Related

- [Reflection-based config conversion](config-conversion-reflection.md)
- [OBI global overrides](obi-global-overrides.md)
- [Build and the OBI vendoring workflow](../07-BUILD-AND-VENDORING.md)
- [Architecture](../01-ARCHITECTURE.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
