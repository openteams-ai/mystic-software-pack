# mystic-software-pack

Nebari software pack repository for the **Mystic LOE-1 demo stack** — four
template-conformant Helm charts, one ArgoCD repo, four chart paths.

## Changelog

### charts/custody-demo v0.2.0 (2026-06-10)

- **run-artifacts volume** — custody-agent Deployment now mounts a `run-artifacts`
  volume at `/data/artifacts`.  The env var `CUSTODY_RUN_ARTIFACTS=/data/artifacts`
  is wired into the container automatically.  Default is `emptyDir` (ephemeral);
  set `artifacts.persistentVolumeClaim.claimName` in values to attach a pre-existing
  PVC for durable storage across pod restarts.

- **Live-UI routing confirmation** — the custody-agent NebariApp omits a `routes`
  list deliberately.  Per the CRD spec, omitting `routing.routes` routes all traffic
  for the hostname to the service (not just listed prefixes).  The following paths
  are therefore already covered by the agent route with no additional config needed:
  - `/v1/custody/runs/*/events` — SSE/WebSocket run-event stream
  - `/live` — live-view landing page
  - `/static-live/*` — live-view static assets
  - `/runs/*` — per-run artifact viewer
  `/healthz` remains a `publicRoute` (Keycloak bypass) for checkmaite probes.

- **WebSocket upgrade passthrough** — Envoy Gateway HTTPRoutes preserve `Connection`
  and `Upgrade` headers by default.  The Gateway API HTTPRoute specification (§7.3)
  does not require stripping hop-by-hop headers, and Envoy Gateway's xDS translation
  passes them through.  The NebariApp CRD (`reconcilers.nebari.dev/v1 v0.1.0-alpha.19`)
  has no `webSocket` or `upgrade` field — WS upgrade passthrough is implicit and
  requires no extra chart configuration.  If a future operator version adds an
  explicit `routing.websocket` toggle, set it to `true` on `nebariapp.agent`.

### charts/custody-demo v0.1.0 (initial)

- Initial four-service chart (custody-agent, custody-tools, custody-fault-proxy,
  custody-demo-ui) with stub images and NebariApp routing for agent + UI.

## Chart matrix

| Chart | Namespace | Services | NebariApp | Notes |
|-------|-----------|----------|-----------|-------|
| `charts/custody-demo` | `nebari-custody-demo-pack` | custody-agent (8080), custody-tools (8090), custody-fault-proxy (8085), custody-demo-ui (8060) | agent + ui only | Multi-service; see §Multi-service adaptation |
| `charts/checkmaite` | `nebari-checkmaite-pack` | checkmaite-serve (5006) | yes | Includes nightly eval CronJob |
| `charts/judge` | `nebari-judge-pack` | judge/vLLM (8000) | none (cluster-internal) | GPU; weight-cache PVC |
| `charts/langfuse` | `nebari-langfuse-pack` | langfuse-web + worker + deps | yes | Wraps official langfuse chart; /api/public/* bypasses Keycloak |

All charts are **NebariApp-conformant**:
`apiVersion: reconcilers.nebari.dev/v1 / kind: NebariApp` (per the
[nebari-software-pack-template](https://github.com/nebari-dev/nebari-software-pack-template)
— NOT the older `nebari.dev/v1alpha1 / NebariApplication` form).

## How ArgoCD consumes this repo

Four ArgoCD `Application` resources point at the **same `repoURL`** with
different `path:` values:

```yaml
# custody-demo
source:
  repoURL: https://github.com/openteams/mystic-software-pack
  path: charts/custody-demo
  helm:
    releaseName: custody-demo
    valuesObject:
      nebariapp:
        enabled: true
        agent:
          hostname: agent.custody.demo.openteams.com
        ui:
          hostname: demo.custody.demo.openteams.com

# checkmaite
source:
  repoURL: https://github.com/openteams/mystic-software-pack
  path: charts/checkmaite
  helm:
    releaseName: checkmaite
    valuesObject:
      nebariapp:
        enabled: true
        hostname: checkmaite.custody.demo.openteams.com
      image:
        repository: <ECR>/checkmaite

# judge
source:
  repoURL: https://github.com/openteams/mystic-software-pack
  path: charts/judge
  helm:
    releaseName: judge

# langfuse
source:
  repoURL: https://github.com/openteams/mystic-software-pack
  path: charts/langfuse
  helm:
    releaseName: langfuse
    valuesObject:
      nebariapp:
        enabled: true
        hostname: langfuse.custody.demo.openteams.com
      langfuse:
        langfuse:
          nextauth:
            url: https://langfuse.custody.demo.openteams.com
        postgresql:
          auth:
            password: "<from vault>"
        ...
```

## Local helm-template instructions

```bash
# Install/update dependencies (langfuse subchart)
cd dev && make dep

# Lint all four charts
make lint

# Render-check all four charts
make template

# Or individually
make lint-langfuse
make template-custody-demo
```

For langfuse, `helm dependency update charts/langfuse/` must be run before
any lint or template operation (the langfuse subchart `.tgz` is not committed).

## Multi-service NebariApp adaptation (custody-demo)

The standard template assumes one NebariApp per chart.  The `custody-demo`
chart contains four services but only two are externally routed (`custody-agent`
and `custody-demo-ui`).  To stay as close to the template as possible:

- Two NebariApp sub-blocks live under `values.nebariapp`: `nebariapp.agent.*`
  and `nebariapp.ui.*`.
- `templates/nebariapp.yaml` renders two `NebariApp` resources (one per routed
  service) when `nebariapp.enabled=true`.
- The un-routed services (`custody-tools`, `custody-fault-proxy`) have no
  NebariApp at all — they are cluster-internal only (contract §4).

## Required K8s Secrets

Secrets must exist in the release namespace **before** `helm install`.  Never
inline credential values in values files.

### `charts/custody-demo` (namespace `nebari-custody-demo-pack`)

| Secret name | Keys |
|-------------|------|
| `model-api-keys` | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY` |
| `langfuse-keys` | `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` |

### `charts/checkmaite` (namespace `nebari-checkmaite-pack`)

| Secret name | Keys |
|-------------|------|
| `langfuse-keys` | `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` |

### `charts/langfuse` (namespace `nebari-langfuse-pack`)

| Secret name | Keys |
|-------------|------|
| `langfuse-core` | `salt`, `encryption-key`, `nextauth-secret` |
| `langfuse-init` | `LANGFUSE_INIT_ORG_ID`, `LANGFUSE_INIT_ORG_NAME`, `LANGFUSE_INIT_PROJECT_ID`, `LANGFUSE_INIT_PROJECT_NAME`, `LANGFUSE_INIT_PROJECT_PUBLIC_KEY`, `LANGFUSE_INIT_PROJECT_SECRET_KEY`, `LANGFUSE_INIT_USER_EMAIL`, `LANGFUSE_INIT_USER_NAME`, `LANGFUSE_INIT_USER_PASSWORD` |

## Swapping stub images (Team 2 handoff)

`custody-demo` ships `hashicorp/http-echo` stubs.  Swapping to a real image is
a values-only change in the ArgoCD Application — no chart edits needed:

```yaml
services:
  custody-agent:
    image:
      repository: <ECR>/custody-agent
      tag: "<tag>"
    args: []   # real image entrypoint takes over
```

For `custody-tools`, additionally set `gpu: true` (adds `nvidia.com/gpu`
resource, nodeSelector, and NoSchedule toleration).

## Enabling the nightly eval CronJob (Team 3 handoff)

Once `checkmaite-plugin-custody` is baked into the checkmaite image, flip:

```yaml
nightlyEval:
  enabled: true
```

in the ArgoCD Application values and bump `image.tag` to the build that
includes the plugin.  The CronJob clones `custody-benchmarks` at runtime
(no static data baked in), syncs GT files from S3 (IRSA), and invokes
`checkmaite_plugin_custody.cli run` with the exact flags from contract §10.

## Contract reference

Service names, images, ports, namespaces, and env vars are normative in:
`/docs/superpowers/plans/2026-06-09-shared-contracts.md` §9
