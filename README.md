# mystic-software-pack

Nebari software pack repository for the **Mystic LOE-1 demo stack** — two
template-conformant Helm charts, one ArgoCD repo, two chart paths.

Not deployed from this repo:

- **Langfuse** — the cluster GitOps repo
  ([openteams-ai/mystic.openteams.ai](https://github.com/openteams-ai/mystic.openteams.ai))
  deploys `nebari-langfuse-pack` as release `langfuse` in namespace `langfuse`.
- **Judge** — hosted on OpenRouter (`google/gemini-3.5-flash` by default);
  the checkmaite chart wires `JUDGE_BASE_URL`/`JUDGE_MODEL`/`JUDGE_API_KEY`.

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

All charts are **NebariApp-conformant**:
`apiVersion: reconcilers.nebari.dev/v1 / kind: NebariApp` (per the
[nebari-software-pack-template](https://github.com/nebari-dev/nebari-software-pack-template)
— NOT the older `nebari.dev/v1alpha1 / NebariApplication` form).

## How ArgoCD consumes this repo

Two ArgoCD `Application` resources (added to the cluster GitOps repo's
`apps/apps/` directory) point at the **same `repoURL`** with different
`path:` values:

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
          hostname: agent.mystic.openteams.ai
        ui:
          hostname: demo.mystic.openteams.ai

# checkmaite
source:
  repoURL: https://github.com/openteams/mystic-software-pack
  path: charts/checkmaite
  helm:
    releaseName: checkmaite
    valuesObject:
      nebariapp:
        enabled: true
        hostname: checkmaite.mystic.openteams.ai
      image:
        repository: <ECR>/checkmaite
```

## Local helm-template instructions

```bash
# Lint both charts
cd dev && make lint

# Render-check both charts
make template

# Or individually
make lint-checkmaite
make template-custody-demo
```

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
| `model-api-keys` | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`; optional `OPENROUTER_API_KEY` (enables per-run `openrouter/<vendor>/<model>` SUT model IDs) |
| `langfuse-keys` | `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` |

### `charts/checkmaite` (namespace `nebari-checkmaite-pack`)

| Secret name | Keys |
|-------------|------|
| `langfuse-keys` | `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY` |
| `judge-keys` | `OPENROUTER_API_KEY` |

The `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` values are generated inside
the cluster-provided Langfuse instance (`https://langfuse.mystic.openteams.ai`,
project settings → API keys).  The `OPENROUTER_API_KEY` comes from
[openrouter.ai](https://openrouter.ai) (used by the judge; the default judge
model is `google/gemini-3.5-flash`, overridable via `judge.model`).

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

Service names, images, ports, namespaces, and env vars are normative in the
`mystic` program repo: `docs/shared-contracts.md` §9.
