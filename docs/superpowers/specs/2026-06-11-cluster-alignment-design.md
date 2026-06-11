# Design: Align mystic-software-pack with the mystic.openteams.ai cluster

**Date:** 2026-06-11
**Status:** Approved
**Repos touched:** `mystic-software-pack` (primary), `mystic` program repo (contract sync only)

## Context

The target cluster is deployed from the GitOps repo
[`openteams-ai/mystic.openteams.ai`](https://github.com/openteams-ai/mystic.openteams.ai)
(ArgoCD app-of-apps on EKS, us-west-2, domain `mystic.openteams.ai`). It already
provides the full Nebari foundational stack — Envoy Gateway, cert-manager,
Keycloak, nebari-operator, Postgres, LGTM, MLflow, RayServe — **and Langfuse**,
deployed from `nebari-dev/nebari-langfuse-pack` (release `langfuse`, namespace
`langfuse`, hostname `langfuse.mystic.openteams.ai`, Keycloak SSO).

That makes our `charts/langfuse` wrapper redundant and exposes several
defaults in the remaining charts that don't match the cluster.

## Decisions (made 2026-06-11)

| Decision | Choice |
|---|---|
| Langfuse chart | Retire; consume the cluster-provided instance |
| Judge GPU placement | Configurable; default = `gpu-small` (g4dn.xlarge T4) serving the 3B model; commented H100 profile |
| Namespaces | Keep contract names (`nebari-custody-demo-pack`, `nebari-judge-pack`, etc.) |
| RWX storage class | `longhorn` (cluster standard; `efs-sc` is unverified there) |
| Contract docs | Update `mystic/docs/shared-contracts.md` in the same pass |

## Changes

### 1. Retire `charts/langfuse/`

- Delete the chart directory (including the fetched `charts/` dependency).
- `dev/Makefile`: remove the `dep` target (langfuse was the only chart with
  dependencies), `lint-langfuse`, `template-langfuse`; update `.PHONY`, `all`,
  `lint`, `template` aggregates and header comments.
- `.github/workflows/lint.yaml`: remove the langfuse dependency-update step and
  the langfuse lint/template steps (CI must keep mirroring the Makefile).
- `README.md`: drop the langfuse chart row and its ArgoCD example; add a note
  that Langfuse is provided by the cluster GitOps repo (`nebari-langfuse-pack`,
  namespace `langfuse`) and consumed via the `langfuse-keys` secret.

### 2. Langfuse URL

The deployed instance lives in namespace `langfuse`, so the in-cluster URL is
`http://langfuse-web.langfuse.svc:3000` (was
`http://langfuse-web.nebari-langfuse-pack.svc:3000`). Update:

- `charts/checkmaite/values.yaml` — `langfuse.host`
- `charts/custody-demo/values.yaml` — the `LANGFUSE_HOST` env default

### 3. Judge: configurable GPU profile, default T4 + 3B

`charts/judge/values.yaml`:

- `model.useFallback: true` → `Qwen/Qwen2.5-VL-3B-Instruct` serves by default
  (7B stays as `model.primary` for larger GPUs).
- `nodeSelector`: `eks.amazonaws.com/nodegroup: gpu-small` (replaces
  `nodegroup: gpu`, which matches nothing on this cluster). Keep the existing
  `nvidia.com/gpu` toleration.
- Add a commented-out H100 profile: `nodeSelector` `gpu.type: h100`, toleration
  for the `gpu.type=h100:NoSchedule` taint, `useFallback: false`.
- Right-size resources for g4dn.xlarge (4 vCPU / 16 GiB): requests
  `cpu: "2"` / `memory: 8Gi`, limits `memory: 12Gi` (GPU request/limit stay
  `nvidia.com/gpu: 1`). Note that the 4 Gi `/dev/shm` emptyDir
  (`medium: Memory`) counts against the container memory limit.

### 4. Storage classes

`efs-sc` → `longhorn`:

- `charts/judge/values.yaml` — `cache.storageClassName`
- `charts/checkmaite/values.yaml` — `analytics.storageClassName`

### 5. Hostname placeholders

Commented examples change from `*.custody.demo.openteams.com` to
`agent.mystic.openteams.ai`, `demo.mystic.openteams.ai`,
`checkmaite.mystic.openteams.ai` in values files and README.

The OTel endpoint default stays `http://otel-collector.opentelemetry.svc:4318`:
contract §6 says Team 1 provides the ExternalName alias. (The deployed
collector currently exports to `debug` only and has no alias — that fix
belongs to the GitOps repo, out of scope here.)

### 6. Contract sync (`mystic` program repo)

`docs/shared-contracts.md`:

- §6: `LANGFUSE_HOST=http://langfuse-web.langfuse.svc:3000`
- §9: langfuse service entry → namespace `langfuse`; note that Langfuse is
  cluster-provided (GitOps repo), not deployed by the software pack.

## Out of scope (tracked for other repos / infra)

- OTel collector exporter config (Tempo + Langfuse OTLP) and the
  `otel-collector.opentelemetry.svc` ExternalName alias — GitOps repo.
- `custody-demo-data` S3 bucket and IRSA roles — infra/Terraform.
- `langfuse-keys` and `model-api-keys` secrets — manual bootstrap, keys
  generated in the deployed Langfuse.
- Three ArgoCD Application manifests in `mystic.openteams.ai/apps/apps/`
  (custody-demo, checkmaite, judge) following house conventions.

## Verification

- `cd dev && make all` passes (three charts).
- Judge chart templates cleanly with the H100 profile values overridden.
- CI workflow steps match the Makefile targets one-for-one.
