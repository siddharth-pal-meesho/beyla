<!-- m-wiki: type=top-level slug=06-kubernetes-integration topic=null base-sha=b4da5978747b generated-at=2026-08-25T07:57:48Z sources=[] -->

> Generated 2026-08-25 at base-sha b4da5978747b. Type: top-level. 0 sources.

# Kubernetes integration

Kubernetes shows up in Beyla in three distinct roles, and conflating them causes confusion. It is a **metadata source** (decorating spans with pod, namespace and owner attributes), a **deployment target** (the Helm chart, DaemonSet and RBAC), and — optionally — a **control surface**, through an admission webhook that injects OpenTelemetry SDKs into other people's pods.

## TL;DR

- Metadata comes from a `kube.MetadataProvider` constructed once in `pkg/components/beyla.go:buildCommonContextInfo` and shared through `global.ContextInfo`.
- The informer can run in-process or delegate to a separate cache service addressed by `MetaCacheAddress`; that service is this repo's `cmd/k8s-cache` binary.
- Informer failure is non-fatal: `pkg/internal/appolly/appolly.go:setupKubernetes` logs an error and calls `ForceDisable()`, so the agent keeps running with undecorated telemetry.
- `MetaRestrictLocalNode` limits watches to the local node — the main lever for informer memory on large clusters.
- The SDK-injection webhook is a separate, experimental feature gated by `Injector.Webhook.Enabled()`.

## Mental model

The provider is built eagerly but *warmed* lazily. `buildCommonContextInfo` constructs `kube.NewMetadataProvider` with a `MetadataConfig` assembled from `Attributes.Kubernetes`, but nothing contacts the API server until someone calls `Get`. `setupKubernetes` is that someone: it forces a fetch so the cache is populated before spans start flowing, and treats failure as a downgrade rather than an abort.

Two deprecated label mechanisms are merged at this point. `MetaSourceLabels.ServiceName` and `.ServiceNamespace` are prepended into the `ResourceLabels` map, with a `sync.OnceFunc` guaranteeing the deprecation warning is logged exactly once regardless of how many labels were set.

The service-name template is compiled here too — `pkg/components/beyla.go:buildServiceNameTemplate` parses `Attributes.Kubernetes.ServiceNameTemplate` into a `text/template` and a parse error is fatal to startup, which is the right call for a template that would otherwise fail once per pod.

## Structure / data flow

```
buildCommonContextInfo
   └─ kube.NewMetadataProvider(MetadataConfig{
          Enable, KubeConfigPath, SyncTimeout, ResyncPeriod,
          DisabledInformers, MetaCacheAddr, ResourceLabels,
          RestrictLocalNode, ServiceNameTemplate })
              │
              ├─ in-process informers ──► Kubernetes API server
              └─ or MetaCacheAddr ──────► cmd/k8s-cache service ──► API server

   ctxInfo.K8sInformer ──┬─► span/process-event Kubernetes decorators
                         ├─► attributeGroups: GroupKubernetes
                         ├─► cluster-connector spans (ObjectMetaByIP)
                         └─► webhook server (own-pod lookup, node filtering)
```

| Role | Entry point | Deployment artefact |
|---|---|---|
| Metadata source | `pkg/components/beyla.go:buildCommonContextInfo` | in-process, or `charts/beyla/templates/cache-deployment.yaml` |
| Deployment target | — | `charts/beyla/templates/daemon-set.yaml`, `cluster-role.yaml` |
| Control surface | `pkg/webhook/server.go:NewServer` | `charts/beyla/templates/injector-role.yaml` |

## Key code locations

| What | Where |
|---|---|
| Metadata provider construction | `pkg/components/beyla.go:buildCommonContextInfo` |
| Service-name template compilation | `pkg/components/beyla.go:buildServiceNameTemplate` |
| Attribute group activation | `pkg/components/beyla.go:attributeGroups` |
| Informer warm-up + downgrade | `pkg/internal/appolly/appolly.go:setupKubernetes` |
| Forced cache population | `pkg/internal/appolly/appolly.go:refreshK8sInformerCache` |
| Cache service entry point | `cmd/k8s-cache/main.go:main` |
| Cache service config | `cmd/k8s-cache/cfg/config.go` |
| Webhook server | `pkg/webhook/server.go:Server` |
| Node-locality filter | `pkg/webhook/server.go:Server.isMyNodeEvent` |
| Informer event handler | `pkg/webhook/server.go:Server.On` |

## Sharp edges

- **A broken informer does not stop the agent.** `setupKubernetes` logs "you can't setup Kubernetes discovery and your traces won't be decorated" and disables the informer. Missing `k8s.*` attributes in production are far more likely to be this than a decorator bug.
- **The cache service is a separate deployable with its own config path.** `cmd/k8s-cache/main.go:main` reads `BEYLA_K8S_CACHE_CONFIG_PATH`, not `BEYLA_CONFIG_PATH`, and converts through the same reflection helper.
- **`MetaSourceLabels` is deprecated but still wins.** The merge in `buildCommonContextInfo` *prepends* it to `ResourceLabels`, so a stale environment variable silently takes priority over the YAML property meant to replace it.
- **Attribute-group selection is environment-dependent.** `attributeGroups` adds the container group only when Kubernetes is disabled *and* Docker metadata is available — the two are mutually exclusive by construction.

## Related concepts

- [Kubernetes metadata cache service](kubernetes/k8s-metadata-cache-service.md)
- [SDK injection webhook](kubernetes/sdk-injection-webhook.md)
- [Cluster connector spans](pipeline/cluster-connector-spans.md)

## Notes

<!-- Anything below is human-owned. wiki-init never reads or modifies content under this heading.
     Use this for tribal knowledge, incident references, dates, decisions that synthesis missed. -->

---

[← Previous](05-EXPORTERS.md) · [Index](../index.md) · [Next →](07-BUILD-AND-VENDORING.md)
