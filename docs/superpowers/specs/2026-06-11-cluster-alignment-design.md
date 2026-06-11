# Design: Align mystic-software-pack with the mystic.openteams.ai cluster

**Date:** 2026-06-11
**Status:** Approved
**Repos touched:** `mystic-software-pack` (primary), `checkmaite-plugin-custody` (judge model/auth support), `mystic` program repo (contract sync)
**Revised:** 2026-06-11 — judge moved from self-hosted vLLM to OpenRouter (`google/gemini-3.5-flash`); `charts/judge` retired

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
| Judge | **Hosted on OpenRouter** (no self-hosted vLLM): retire `charts/judge`; default model `google/gemini-3.5-flash` (vision + json_object, verified available); new `judge-keys` secret with key `OPENROUTER_API_KEY` |
| Judge plugin support | Included in this pass: `JUDGE_MODEL` + `JUDGE_API_KEY` support in `checkmaite-plugin-custody` (client hardcodes `model: "judge"`, sends no auth header today) |
| Planner (SUT) model | Unchanged: `claude-sonnet-4-6` via native Anthropic adapter. The agent already supports `openrouter/<vendor>/<model>` per-run model IDs; adding `OPENROUTER_API_KEY` to the `model-api-keys` secret enables them with zero chart changes |
| Namespaces | Keep contract names (`nebari-custody-demo-pack`, etc.) |
| RWX storage class | `longhorn` (cluster standard; `efs-sc` is unverified there) |
| Contract docs | Update `mystic/docs/shared-contracts.md` in the same pass |

## Changes

### 1. Retire `charts/langfuse/` AND `charts/judge/`

The pack drops from four charts to **two** (custody-demo, checkmaite):
Langfuse is cluster-provided, and the judge becomes a hosted OpenRouter
endpoint (no vLLM, no GPU scheduling, no weight-cache PVC).

- Delete both chart directories (including langfuse's fetched `charts/`
  dependency).
- `dev/Makefile`: remove the `dep` target (langfuse was the only chart with
  dependencies), `lint-langfuse`, `template-langfuse`, `lint-judge`,
  `template-judge`; update `.PHONY`, `all`, `lint`, `template` aggregates and
  header comments.
- `.github/workflows/lint.yaml`: remove the langfuse dependency-update step
  and the langfuse + judge lint/template steps (CI must keep mirroring the
  Makefile).
- `README.md`: drop the langfuse and judge chart rows and ArgoCD examples; add
  notes that Langfuse is provided by the cluster GitOps repo
  (`nebari-langfuse-pack`, namespace `langfuse`) and the judge is OpenRouter.

### 2. Langfuse URL

The deployed instance lives in namespace `langfuse`, so the in-cluster URL is
`http://langfuse-web.langfuse.svc:3000` (was
`http://langfuse-web.nebari-langfuse-pack.svc:3000`). Update:

- `charts/checkmaite/values.yaml` — `langfuse.host`
- `charts/custody-demo/values.yaml` — the `LANGFUSE_HOST` env default

### 3. Judge: OpenRouter endpoint

`charts/checkmaite/values.yaml`:

- `endpoints.judgeBaseUrl: https://openrouter.ai/api/v1`
- New `judge:` block: `secretName: judge-keys` (K8s Secret with key
  `OPENROUTER_API_KEY`), `model: google/gemini-3.5-flash` (vision-capable,
  supports `response_format: json_object`; verified on OpenRouter's model
  catalog 2026-06-11).

`charts/checkmaite/templates/deployment.yaml` and
`templates/cronjob-nightly.yaml` gain two env vars next to `JUDGE_BASE_URL`:
`JUDGE_MODEL` (from `.Values.judge.model`) and `JUDGE_API_KEY`
(secretKeyRef → `.Values.judge.secretName` / `OPENROUTER_API_KEY`).

### 3b. Plugin: judge model + auth support (`checkmaite-plugin-custody` repo)

`judge_client.py` hardcodes `"model": "judge"` and sends no Authorization
header — OpenRouter rejects both. Changes (TDD with the existing respx
patterns in `tests/test_judge_client.py`):

- `JudgeClient.__init__` gains `model: str | None = None` and
  `api_key: str | None = None`; fallbacks `$JUDGE_MODEL` (default `"judge"`)
  and `$JUDGE_API_KEY` (default none), mirroring the existing `base_url` /
  `$JUDGE_BASE_URL` pattern.
- `_chat()` sends `"model": self._model` and, when an API key is present, a
  per-request `Authorization: Bearer <key>` header (per-request so injected
  `httpx.Client`s work unchanged).
- Backward compatible: with no env/args set, behavior is byte-identical to
  today (model `"judge"`, no auth header).

### 3c. Custody-demo GPU selector (custody-tools still runs on GPU)

`charts/custody-demo/values.yaml` shared `gpu:` block (used by custody-tools
when `gpu: true`): `gpu.nodeSelector` changes from `nodegroup: gpu` (matches
nothing on this cluster) to `eks.amazonaws.com/nodegroup: gpu-small`. The
H100 node group is no longer needed by anything in this pack.

### 3d. Planner (SUT) stays native Anthropic

`nightlyEval.modelId: claude-sonnet-4-6` is unchanged. For per-run OpenRouter
SUT experiments (`model_id: openrouter/<vendor>/<model>`, already supported by
the agent's adapter resolver), add `OPENROUTER_API_KEY` to the existing
`model-api-keys` secret — documentation only, no chart change
(`envFromSecrets` injects every key).

### 4. Storage classes

`efs-sc` → `longhorn`:

- `charts/checkmaite/values.yaml` — `analytics.storageClassName`

(The judge weight-cache PVC disappears with the judge chart.)

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
- §7: judge is no longer self-hosted vLLM. New contract: OpenAI-compatible
  endpoint `JUDGE_BASE_URL` (default `https://openrouter.ai/api/v1`), model
  `JUDGE_MODEL` (default `google/gemini-3.5-flash`), bearer auth
  `JUDGE_API_KEY` from secret `judge-keys` (key `OPENROUTER_API_KEY`).
- §9: drop the langfuse + judge namespace entries (cluster-provided / hosted);
  judge row → OpenRouter incl. the `judge-keys` secret reference; analytics
  row → longhorn. (Contracts list no secrets table — `judge-keys` rides in
  the Judge row; the optional `OPENROUTER_API_KEY` in `model-api-keys` for
  per-run OpenRouter SUT models is documented in the pack README only.)

## Out of scope (tracked for other repos / infra)

- OTel collector exporter config (Tempo + Langfuse OTLP) and the
  `otel-collector.opentelemetry.svc` ExternalName alias — GitOps repo.
- `custody-demo-data` S3 bucket and IRSA roles — infra/Terraform.
- `langfuse-keys`, `model-api-keys`, and `judge-keys` secrets — manual
  bootstrap (Langfuse keys from the deployed instance; OpenRouter key from
  openrouter.ai).
- Two ArgoCD Application manifests in `mystic.openteams.ai/apps/apps/`
  (custody-demo, checkmaite) following house conventions.
- Egress: the checkmaite pod must be able to reach `https://openrouter.ai`
  (default EKS networking allows this; flag if a NetworkPolicy lands later).

## Verification

- `cd dev && make all` passes (two charts).
- Rendered checkmaite manifests carry `JUDGE_BASE_URL`, `JUDGE_MODEL`, and a
  `JUDGE_API_KEY` secretKeyRef in both the Deployment and the CronJob.
- `uv run pytest` passes in `checkmaite-plugin-custody` (new judge
  model/auth tests included).
- CI workflow steps match the Makefile targets one-for-one.
