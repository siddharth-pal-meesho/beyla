<!-- m-wiki: type=concept slug=config-conversion-reflection topic=obi-integration base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# Reflection-based config conversion

Beyla and OBI keep two separate configuration structs with deliberately similar shapes, and a small reflection helper copies values between them by matching field names. It is the mechanism that lets Beyla present a `BEYLA_`-flavoured configuration surface without forking the engine's own config type.

## Where it applies in this repo

`pkg/helpers/config/convert.go:Convert` is the entry point: it takes a source value, a destination pointer and a map of *field hints*, then walks both structures recursively. `pkg/helpers/config/convert.go:handleFieldConversion` and `pkg/helpers/config/convert.go:convertStruct` do the per-field and per-struct work.

Two call sites matter for the agent:

- `pkg/beyla/config_obi.go:FromOBI` — OBI → Beyla, used once to derive defaults in `pkg/beyla/config.go:DefaultConfig`. It passes a hint map marking every Beyla-only field as skipped: `.obi`, `.TracesReceiver`, `.SigilExport`, `.Processes`, `.Grafana`, `.Topology`, `.Discovery.Survey` and `.Injector`.
- `pkg/beyla/config_obi.go:Config.AsOBI` — Beyla → OBI, used at runtime. It passes an *empty* hint map, then applies targeted fixes in `pkg/beyla/config_obi.go:overrideOBI`.

A third call site is the cache service: `cmd/k8s-cache/main.go:main` converts its own file-loaded config into the vendored `kubecache.Config`.

The sentinel that marks a field as skippable is the `SkipConversion` constant at `pkg/helpers/config/convert.go:8`.

## Why this design

The alternative — hand-written mapping code — would need a line per field across a configuration struct with dozens of nested sections, and would silently rot every time upstream added a field. Reflection makes the common case (identical field names) free.

The design choice worth noticing is that unmatched fields **panic** rather than being ignored. `pkg/helpers/config/convert.go:convert` panics on a nil or non-pointer destination, and the hint map exists precisely so that intentional asymmetry has to be declared. A silent skip would let a renamed upstream field quietly stop being configured; a panic at startup surfaces it on the first run.

The tradeoff is that adding a Beyla-only configuration field requires remembering to declare the skip — an easy step to miss, and a crash rather than a compile error when you do.

## Related

- [The vendored OBI boundary](vendored-obi-boundary.md)
- [OBI global overrides](obi-global-overrides.md)
- [Configuration model](../03-CONFIGURATION.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
