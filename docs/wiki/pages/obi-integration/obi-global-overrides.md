<!-- m-wiki: type=concept slug=obi-global-overrides topic=obi-integration base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# OBI global overrides

A startup routine that mutates package-level variables inside the vendored OBI module so that the telemetry it emits carries Beyla and Grafana naming rather than OpenTelemetry defaults. It is configuration by global mutation — unusual, deliberate, and order-sensitive.

## Where it applies in this repo

Everything happens in `pkg/beyla/config_obi.go:OverrideOBIGlobalConfig`, which does two separable jobs.

**Environment duplication.** It compiles the pattern `^BEYLA_(OTEL_)?` and rewrites every matching variable in `os.Environ()` to an `OTEL_EBPF_` prefix, setting the result only when that target variable is currently empty. Vendored code that reads its own native variable names therefore observes the user's Beyla-prefixed settings.

**Name reassignment.** It then overwrites exported globals in the vendored packages:

- Build metadata — `obibuildinfo.Version` and `obibuildinfo.Revision` take Beyla's values.
- Cloud host key — `otel2.CloudHostIDKey` and `prom.CloudHostIDKey` become `grafana_host_id`.
- Attribute vocabulary — `attr.VendorPrefix`, `attr.VendorSDKName` become `beyla`; `attr.OBIIP` becomes `beyla.ip`.
- Metric names — `attributes.NetworkFlow`, `NetworkFlowPackets`, `NetworkInterZone`, `StatTCPRtt`, `StatTCPFailedConnections`, `StatTCPRetransmits` and `StatTCPIo` are each replaced with a `beyla_*` / `beyla.*` triple.

Endpoint-level Grafana overrides are handled separately, per-config, in `pkg/beyla/config_obi.go:overrideOBI`.

## Why this design

The names OBI emits are baked into exported package variables rather than threaded through its config struct. Reassigning them is the only way to rename the output without patching the vendored source — and patching vendored source would be erased by the next `make vendor-obi`.

The environment duplication solves a subtler problem. Configuration reaches the engine through the converted struct, but vendored code also reads environment variables directly in places. Duplicating rather than renaming preserves both spellings, and the "only if unset" guard means an explicitly-set native variable keeps priority.

The consequence to respect: this is global mutation with a happens-before requirement. It must run before any exporter or attribute selector captures those values.

## Related

- [The vendored OBI boundary](vendored-obi-boundary.md)
- [Reflection-based config conversion](config-conversion-reflection.md)
- [Exporters and telemetry surface](../05-EXPORTERS.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
