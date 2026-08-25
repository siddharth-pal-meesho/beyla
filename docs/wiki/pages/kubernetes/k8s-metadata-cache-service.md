<!-- m-wiki: type=concept slug=k8s-metadata-cache-service topic=kubernetes base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: concept. 0 sources.

# Kubernetes metadata cache service

An optional second deployable — built from `cmd/k8s-cache` — that runs the Kubernetes informers once for a whole cluster and serves the resulting metadata to Beyla agents over the network. It exists because Beyla normally runs as a DaemonSet, and a per-node informer means one full API-server watch per node.

## Where it applies in this repo

`cmd/k8s-cache/main.go:main` is the entry point. It mirrors the agent's startup shape — slog setup, config path from `-config` or `BEYLA_K8S_CACHE_CONFIG_PATH`, then run — but converts its file-loaded configuration into the vendored `kubecache.Config` through the same reflection helper the agent uses, `pkg/helpers/config/convert.go:Convert`. It also calls its own `overrideOBIConfiguration()` before anything else, keeping the naming conventions consistent with the agent. Its typed configuration root is `cmd/k8s-cache/cfg/config.go`.

On the agent side, the choice between in-process informers and this service is a single field. `pkg/components/beyla.go:buildCommonContextInfo` passes `config.Attributes.Kubernetes.MetaCacheAddress` into `kube.MetadataConfig` as `MetaCacheAddr`; a non-empty value routes metadata lookups to the service. Everything downstream — decorators, attribute groups, connector spans — consumes `ctxInfo.K8sInformer` and is unaware of which mode is in use.

Deployment artefacts are `charts/beyla/templates/cache-deployment.yaml` and `charts/beyla/templates/cache-service.yaml`.

## Why this design

Informer memory is the dominant scaling cost of running Beyla on a large cluster. Each agent caching every Pod and ReplicaSet multiplies that cost by node count, and the API server pays for the watches. Centralising the watch trades one shared dependency for a large, cluster-wide reduction in both.

Keeping the switch behind `MetadataProvider` rather than exposing two code paths is what makes the tradeoff cheap to reverse: no decorator knows which mode is active, so moving between them is a configuration change.

`MetaRestrictLocalNode` is the alternative lever for the in-process mode — restricting each agent's watch to its own node's objects. The two approaches solve the same problem at different points on the complexity curve; the cache service scales further, node restriction adds no moving parts.

## Related

- [Kubernetes integration](../06-KUBERNETES-INTEGRATION.md)
- [SDK injection webhook](sdk-injection-webhook.md)
- [Reflection-based config conversion](../obi-integration/config-conversion-reflection.md)

## Sources

No `raw/` sources yet — this page is synthesized from code at HEAD.

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, decisions, dates, incident references the synthesis missed. -->

---

[← Wiki index](../../index.md)

<!-- atomic: keep this page ≤600 words. New scope → new concept page that builds on this one. Do not append paragraphs here. -->
