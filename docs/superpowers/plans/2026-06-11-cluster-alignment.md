# Cluster Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Align `mystic-software-pack` with the already-deployed `mystic.openteams.ai` cluster: retire the langfuse AND judge charts (Langfuse is cluster-provided; the judge becomes a hosted OpenRouter endpoint), wire checkmaite to OpenRouter, add judge model/auth support to the plugin, fix storage/scheduling/hostname defaults, and sync the normative contract doc.

**Architecture:** The pack drops to two charts (custody-demo, checkmaite). The judge is `google/gemini-3.5-flash` via OpenRouter's OpenAI-compatible API; the checkmaite chart passes `JUDGE_BASE_URL`/`JUDGE_MODEL`/`JUDGE_API_KEY` env, and `checkmaite-plugin-custody`'s `JudgeClient` learns to honor the latter two (it currently hardcodes `model: "judge"` with no auth). The planner (SUT) stays `claude-sonnet-4-6` on the native Anthropic adapter.

**Tech Stack:** Helm 3, GNU Make, GitHub Actions; Python 3.11 + httpx/respx + uv (plugin repo).

**Spec:** `docs/superpowers/specs/2026-06-11-cluster-alignment-design.md`

**Working branches:** `cluster-alignment` in `mystic-software-pack` (exists). Tasks 6 and 9 touch the **sibling checkouts** `/Users/khan/openteams/mystic/checkmaite-plugin-custody` and `/Users/khan/openteams/mystic/mystic` — branch there first.

**Commit footer for every commit:**

```
Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
```

---

### Task 1: Delete the langfuse and judge charts

**Files:**
- Delete: `charts/langfuse/` and `charts/judge/` (entire directories)

- [ ] **Step 1: Remove the directories**

```bash
cd /Users/khan/openteams/mystic/mystic-software-pack
git rm -r charts/langfuse charts/judge
rm -rf charts/langfuse charts/judge   # clears untracked fetched deps (.tgz, Chart.lock)
```

- [ ] **Step 2: Verify only two charts remain**

Run: `ls charts/`
Expected: `checkmaite  custody-demo`

- [ ] **Step 3: Commit**

```bash
git commit -m "feat!: retire langfuse and judge charts — Langfuse is cluster-provided, judge moves to OpenRouter"
```

(Standard footer. `!` = breaking: deployments of these chart paths must migrate.)

---

### Task 2: Rewrite `dev/Makefile` for two charts

**Files:**
- Modify: `dev/Makefile` (full replacement)

- [ ] **Step 1: Replace the file contents with exactly this**

```make
# dev/Makefile — local lint / template targets for mystic-software-pack
#
# Prerequisites: helm 3.8+
#
# Usage:
#   make lint          helm lint both charts
#   make template      helm template render-check for both charts
#   make all           lint + template
#
# Individual chart targets:
#   make lint-custody-demo
#   make lint-checkmaite
#   make template-custody-demo
#   make template-checkmaite
#
# NOTE: Langfuse is NOT deployed from this repo — the cluster GitOps repo
# (openteams-ai/mystic.openteams.ai) deploys nebari-langfuse-pack as release
# "langfuse" in namespace "langfuse".  The judge is NOT self-hosted — the
# checkmaite chart points at OpenRouter (contract §7).

CHARTS_DIR := ../charts

.PHONY: all lint template \
        lint-custody-demo lint-checkmaite \
        template-custody-demo template-checkmaite

all: lint template

# --------------------------------------------------------------------------
# lint — validate both charts
# --------------------------------------------------------------------------
lint: lint-custody-demo lint-checkmaite

lint-custody-demo:
	helm lint $(CHARTS_DIR)/custody-demo/

lint-checkmaite:
	helm lint $(CHARTS_DIR)/checkmaite/

# --------------------------------------------------------------------------
# template — render-check both charts (both enabled/disabled NebariApp)
# --------------------------------------------------------------------------
template: template-custody-demo template-checkmaite

template-custody-demo:
	@echo "--- custody-demo (nebariapp.enabled=false) ---"
	helm template t $(CHARTS_DIR)/custody-demo/ --set nebariapp.enabled=false > /dev/null
	@echo "--- custody-demo (nebariapp.enabled=true) ---"
	helm template t $(CHARTS_DIR)/custody-demo/ \
		--set nebariapp.enabled=true \
		--set nebariapp.agent.hostname=agent.custody.local \
		--set nebariapp.ui.hostname=demo.custody.local > /dev/null
	@echo "custody-demo: OK"

template-checkmaite:
	@echo "--- checkmaite (nebariapp.enabled=false) ---"
	helm template t $(CHARTS_DIR)/checkmaite/ \
		--set image.repository=registry.example.com/checkmaite > /dev/null
	@echo "--- checkmaite (nebariapp.enabled=true + nightly) ---"
	helm template t $(CHARTS_DIR)/checkmaite/ \
		--set image.repository=registry.example.com/checkmaite \
		--set nebariapp.enabled=true \
		--set nebariapp.hostname=checkmaite.custody.local \
		--set nightlyEval.enabled=true > /dev/null
	@echo "checkmaite: OK"
```

- [ ] **Step 2: Run the full check**

Run: `cd /Users/khan/openteams/mystic/mystic-software-pack/dev && make all`
Expected: lint passes for both charts; template prints `custody-demo: OK`, `checkmaite: OK`; exit 0.

- [ ] **Step 3: Commit**

```bash
cd /Users/khan/openteams/mystic/mystic-software-pack
git add dev/Makefile
git commit -m "chore(dev): two-chart Makefile — drop langfuse and judge targets"
```

---

### Task 3: Update CI workflow to mirror the Makefile

**Files:**
- Modify: `.github/workflows/lint.yaml`

- [ ] **Step 1: Delete the langfuse dependency step**

Remove this block (directly after the `Set up Helm` step):

```yaml
      # ----------------------------------------------------------------
      # langfuse — dependency chart must be fetched before lint/template
      # ----------------------------------------------------------------
      - name: Update langfuse dependency
        run: helm dependency update charts/langfuse/
```

- [ ] **Step 2: Delete the judge section**

Remove the `# judge` comment banner, the `Lint judge` step, and the
`Template judge (NebariApp disabled — judge is always cluster-internal)` step.

- [ ] **Step 3: Delete the langfuse section**

Remove everything from the `# langfuse` comment banner through the end of the
`Template langfuse (NebariApp enabled)` step (lint + both template steps with
their `--set ...password=ci` flags).

- [ ] **Step 4: Verify no stale references and exact Makefile mirroring**

Run: `grep -in "langfuse\|judge" .github/workflows/lint.yaml`
Expected: no output.
Remaining helm commands must be exactly: 2 lint (custody-demo, checkmaite) + 4 template (2 custody-demo, 2 checkmaite) — same as the Makefile.

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/lint.yaml
git commit -m "ci: drop langfuse and judge steps to mirror two-chart Makefile"
```

---

### Task 4: checkmaite chart — OpenRouter judge, cluster Langfuse, longhorn, hostname

**Files:**
- Modify: `charts/checkmaite/values.yaml`
- Modify: `charts/checkmaite/templates/deployment.yaml`
- Modify: `charts/checkmaite/templates/cronjob-nightly.yaml`

- [ ] **Step 1: values — langfuse host**

Replace:

```yaml
langfuse:
  # K8s Secret in this namespace with LANGFUSE_PUBLIC_KEY / LANGFUSE_SECRET_KEY
  # (contract §9: Secret name "langfuse-keys")
  secretName: langfuse-keys
  host: http://langfuse-web.nebari-langfuse-pack.svc:3000
```

with:

```yaml
langfuse:
  # K8s Secret in this namespace with LANGFUSE_PUBLIC_KEY / LANGFUSE_SECRET_KEY
  # (contract §9: Secret name "langfuse-keys")
  secretName: langfuse-keys
  # Cluster-provided Langfuse: the GitOps repo (openteams-ai/mystic.openteams.ai)
  # deploys nebari-langfuse-pack as release "langfuse" in namespace "langfuse".
  host: http://langfuse-web.langfuse.svc:3000
```

- [ ] **Step 2: values — judge block (new) directly after the langfuse block**

Insert:

```yaml
# =============================================================================
# Judge (hosted OpenRouter endpoint — contract §7)
# =============================================================================
judge:
  # K8s Secret in this namespace with key OPENROUTER_API_KEY
  secretName: judge-keys
  # Vision-capable model that supports response_format json_object
  model: google/gemini-3.5-flash
```

- [ ] **Step 3: values — judge base URL**

In the `endpoints:` block, replace:

```yaml
  judgeBaseUrl: http://judge.nebari-judge-pack.svc:8000/v1
```

with:

```yaml
  judgeBaseUrl: https://openrouter.ai/api/v1
```

- [ ] **Step 4: values — analytics storage class**

Replace:

```yaml
# =============================================================================
# Analytics PVC (EFS, RWX — contract §9: /data/analytics)
# =============================================================================
analytics:
  storageClassName: efs-sc
```

with:

```yaml
# =============================================================================
# Analytics PVC (RWX via longhorn share-manager — contract §9: /data/analytics)
# =============================================================================
analytics:
  storageClassName: longhorn
```

- [ ] **Step 5: values — hostname placeholder**

Replace `# hostname: checkmaite.custody.demo.openteams.com  # Required when enabled` with `# hostname: checkmaite.mystic.openteams.ai  # Required when enabled`.

- [ ] **Step 6: deployment.yaml — judge model + key env**

Replace:

```yaml
            - name: JUDGE_BASE_URL
              value: {{ .Values.endpoints.judgeBaseUrl }}
```

with:

```yaml
            - name: JUDGE_BASE_URL
              value: {{ .Values.endpoints.judgeBaseUrl }}
            - name: JUDGE_MODEL
              value: {{ .Values.judge.model }}
            - name: JUDGE_API_KEY
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.judge.secretName }}
                  key: OPENROUTER_API_KEY
```

- [ ] **Step 7: cronjob-nightly.yaml — same env additions**

Replace:

```yaml
                - name: JUDGE_BASE_URL
                  value: {{ .Values.endpoints.judgeBaseUrl }}
```

with:

```yaml
                - name: JUDGE_BASE_URL
                  value: {{ .Values.endpoints.judgeBaseUrl }}
                - name: JUDGE_MODEL
                  value: {{ .Values.judge.model }}
                - name: JUDGE_API_KEY
                  valueFrom:
                    secretKeyRef:
                      name: {{ .Values.judge.secretName }}
                      key: OPENROUTER_API_KEY
```

(Note the deeper indentation in the CronJob — match the surrounding entries.)

- [ ] **Step 8: Verify rendered env**

Run:

```bash
cd /Users/khan/openteams/mystic/mystic-software-pack
helm template t charts/checkmaite/ \
  --set image.repository=registry.example.com/checkmaite \
  --set nightlyEval.enabled=true \
  | grep -A6 "JUDGE_"
```

Expected: `JUDGE_BASE_URL` = `https://openrouter.ai/api/v1`, `JUDGE_MODEL` = `google/gemini-3.5-flash`, and a `JUDGE_API_KEY` secretKeyRef (`judge-keys` / `OPENROUTER_API_KEY`) appear **twice** (Deployment + CronJob).

Run: `cd dev && make lint-checkmaite template-checkmaite && cd ..`
Expected: lint passes, `checkmaite: OK`.

- [ ] **Step 9: Commit**

```bash
git add charts/checkmaite/
git commit -m "feat(checkmaite): OpenRouter judge (gemini-3.5-flash, judge-keys secret); cluster Langfuse; longhorn analytics PVC"
```

---

### Task 5: custody-demo chart — Langfuse URL, GPU selector, hostnames

**Files:**
- Modify: `charts/custody-demo/values.yaml`

- [ ] **Step 1: LANGFUSE_HOST**

Replace:

```yaml
      - name: LANGFUSE_HOST
        value: http://langfuse-web.nebari-langfuse-pack.svc:3000
```

with:

```yaml
      - name: LANGFUSE_HOST
        # Cluster-provided Langfuse (GitOps repo, namespace "langfuse")
        value: http://langfuse-web.langfuse.svc:3000
```

- [ ] **Step 2: GPU node selector (shared `gpu:` block, used by custody-tools when `gpu: true`)**

Replace:

```yaml
gpu:
  nodeSelector:
    nodegroup: gpu
```

with:

```yaml
gpu:
  # gpu-small node group (g4dn.xlarge, T4 16 GB, scale-to-zero)
  nodeSelector:
    eks.amazonaws.com/nodegroup: gpu-small
```

(The `tolerations` sub-block below it is unchanged.)

- [ ] **Step 3: Hostname placeholders**

Replace `# hostname: agent.custody.demo.openteams.com` with `# hostname: agent.mystic.openteams.ai`, and `# hostname: demo.custody.demo.openteams.com` with `# hostname: demo.mystic.openteams.ai`.

- [ ] **Step 4: Verify**

Run: `grep -rn "nebari-langfuse-pack\|custody.demo.openteams.com\|nodegroup: gpu$" charts/custody-demo/`
Expected: no output.
Run: `cd dev && make lint-custody-demo template-custody-demo && cd ..`
Expected: lint passes, `custody-demo: OK`.

- [ ] **Step 5: Commit**

```bash
git add charts/custody-demo/values.yaml
git commit -m "feat(custody-demo): cluster Langfuse URL; gpu-small selector; mystic.openteams.ai hostnames"
```

---

### Task 6: Plugin — JUDGE_MODEL + JUDGE_API_KEY support (sibling repo, TDD)

**Files:**
- Modify: `/Users/khan/openteams/mystic/checkmaite-plugin-custody/src/checkmaite_plugin_custody/judge_client.py`
- Test: `/Users/khan/openteams/mystic/checkmaite-plugin-custody/tests/test_judge_client.py`

**Note:** Separate git checkout. Branch first. Uses uv, not Poetry.

- [ ] **Step 1: Branch**

```bash
cd /Users/khan/openteams/mystic/checkmaite-plugin-custody
git status --short          # must be clean before branching
git checkout -b judge-openrouter
```

- [ ] **Step 2: Write the failing tests**

Append to `tests/test_judge_client.py` (it already imports `json`, `httpx`, `pytest`, `respx`, `JudgeClient`, and defines `JUDGE_URL`, `rubrics_dir`, `_completion`):

```python
@respx.mock
def test_judge_model_and_auth_from_constructor(rubrics_dir: Path) -> None:
    route = respx.post(f"{JUDGE_URL}/chat/completions").mock(
        return_value=_completion('{"score": 1.0, "reasons": "ok"}')
    )
    client = JudgeClient(
        base_url=JUDGE_URL,
        rubrics_dir=rubrics_dir,
        model="google/gemini-3.5-flash",
        api_key="sk-or-test",
    )
    client.judge_citation("aGVsbG8=", "white SUV")

    request = route.calls.last.request
    assert json.loads(request.content)["model"] == "google/gemini-3.5-flash"
    assert request.headers["Authorization"] == "Bearer sk-or-test"


@respx.mock
def test_judge_model_and_auth_from_env(
    rubrics_dir: Path, monkeypatch: pytest.MonkeyPatch
) -> None:
    monkeypatch.setenv("JUDGE_MODEL", "google/gemini-3.5-flash")
    monkeypatch.setenv("JUDGE_API_KEY", "sk-or-env")
    route = respx.post(f"{JUDGE_URL}/chat/completions").mock(
        return_value=_completion('{"score": 1.0, "reasons": "ok"}')
    )
    client = JudgeClient(base_url=JUDGE_URL, rubrics_dir=rubrics_dir)
    client.judge_citation("aGVsbG8=", "white SUV")

    request = route.calls.last.request
    assert json.loads(request.content)["model"] == "google/gemini-3.5-flash"
    assert request.headers["Authorization"] == "Bearer sk-or-env"


@respx.mock
def test_judge_defaults_unchanged_without_model_or_key(
    rubrics_dir: Path, monkeypatch: pytest.MonkeyPatch
) -> None:
    monkeypatch.delenv("JUDGE_MODEL", raising=False)
    monkeypatch.delenv("JUDGE_API_KEY", raising=False)
    route = respx.post(f"{JUDGE_URL}/chat/completions").mock(
        return_value=_completion('{"score": 1.0, "reasons": "ok"}')
    )
    client = JudgeClient(base_url=JUDGE_URL, rubrics_dir=rubrics_dir)
    client.judge_citation("aGVsbG8=", "white SUV")

    request = route.calls.last.request
    assert json.loads(request.content)["model"] == "judge"
    assert "Authorization" not in request.headers
```

- [ ] **Step 3: Run the new tests — they must fail**

Run: `uv run pytest tests/test_judge_client.py -v -k "model_and_auth or defaults_unchanged"`
Expected: `test_judge_model_and_auth_from_constructor` and `test_judge_model_and_auth_from_env` FAIL (`TypeError: unexpected keyword argument 'model'` / model assertion); `test_judge_defaults_unchanged_without_model_or_key` PASSES (documents current behavior).

- [ ] **Step 4: Implement**

In `src/checkmaite_plugin_custody/judge_client.py`:

(a) Below `DEFAULT_JUDGE_BASE_URL = "http://judge.nebari-judge-pack.svc:8000/v1"` add:

```python
DEFAULT_JUDGE_MODEL = "judge"
```

(b) Extend `__init__` — replace:

```python
    def __init__(
        self,
        base_url: str | None = None,
        rubrics_dir: str | Path | None = None,
        timeout_s: float = 120.0,
        http: httpx.Client | None = None,
    ) -> None:
        self._base_url = (base_url or os.environ.get("JUDGE_BASE_URL", DEFAULT_JUDGE_BASE_URL)).rstrip("/")
```

with:

```python
    def __init__(
        self,
        base_url: str | None = None,
        rubrics_dir: str | Path | None = None,
        timeout_s: float = 120.0,
        http: httpx.Client | None = None,
        model: str | None = None,
        api_key: str | None = None,
    ) -> None:
        self._base_url = (base_url or os.environ.get("JUDGE_BASE_URL", DEFAULT_JUDGE_BASE_URL)).rstrip("/")
        self._model = model or os.environ.get("JUDGE_MODEL", DEFAULT_JUDGE_MODEL)
        self._api_key = api_key or os.environ.get("JUDGE_API_KEY")
```

Also extend the class docstring's parameter list with:

```
    model:
        Model name sent in the chat payload.  Falls back to ``$JUDGE_MODEL``
        then ``"judge"`` (the self-hosted contract name).  For OpenRouter use
        the catalog slug, e.g. ``google/gemini-3.5-flash``.
    api_key:
        Bearer token for hosted endpoints (e.g. OpenRouter).  Falls back to
        ``$JUDGE_API_KEY``; when unset no Authorization header is sent.
```

(c) In `_chat()`, replace:

```python
        payload = {
            "model": "judge",
```

with:

```python
        payload = {
            "model": self._model,
```

and replace the POST call:

```python
                response = self._http.post(f"{self._base_url}/chat/completions", json=payload)
```

with:

```python
                headers = {"Authorization": f"Bearer {self._api_key}"} if self._api_key else {}
                response = self._http.post(
                    f"{self._base_url}/chat/completions", json=payload, headers=headers
                )
```

(Per-request headers, not client-level, so caller-injected `httpx.Client`s keep working.)

- [ ] **Step 5: Run the new tests — all pass**

Run: `uv run pytest tests/test_judge_client.py -v`
Expected: all tests in the file PASS (the pre-existing ones prove backward compat — they assert `payload["model"] == "judge"`).

- [ ] **Step 6: Run the full suite**

Run: `uv run pytest`
Expected: full suite passes.

- [ ] **Step 7: Commit**

```bash
git add src/checkmaite_plugin_custody/judge_client.py tests/test_judge_client.py
git commit -m "feat(judge): JUDGE_MODEL and JUDGE_API_KEY support for hosted OpenAI-compatible endpoints (OpenRouter)"
```

(Standard footer.)

---

### Task 7: README overhaul

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Intro paragraph**

Replace:

```markdown
Nebari software pack repository for the **Mystic LOE-1 demo stack** — four
template-conformant Helm charts, one ArgoCD repo, four chart paths.
```

with:

```markdown
Nebari software pack repository for the **Mystic LOE-1 demo stack** — two
template-conformant Helm charts, one ArgoCD repo, two chart paths.

Not deployed from this repo:

- **Langfuse** — the cluster GitOps repo
  ([openteams-ai/mystic.openteams.ai](https://github.com/openteams-ai/mystic.openteams.ai))
  deploys `nebari-langfuse-pack` as release `langfuse` in namespace `langfuse`.
- **Judge** — hosted on OpenRouter (`google/gemini-3.5-flash` by default);
  the checkmaite chart wires `JUDGE_BASE_URL`/`JUDGE_MODEL`/`JUDGE_API_KEY`.
```

- [ ] **Step 2: Chart matrix — drop the langfuse and judge rows**

Delete the lines:

```markdown
| `charts/judge` | `nebari-judge-pack` | judge/vLLM (8000) | none (cluster-internal) | GPU; weight-cache PVC |
| `charts/langfuse` | `nebari-langfuse-pack` | langfuse-web + worker + deps | yes | Wraps official langfuse chart; /api/public/* bypasses Keycloak |
```

- [ ] **Step 3: ArgoCD section — two apps, mystic.openteams.ai hostnames**

Replace the sentence:

```markdown
Four ArgoCD `Application` resources point at the **same `repoURL`** with
different `path:` values:
```

with:

```markdown
Two ArgoCD `Application` resources (added to the cluster GitOps repo's
`apps/apps/` directory) point at the **same `repoURL`** with different
`path:` values:
```

In the YAML example: replace `agent.custody.demo.openteams.com` → `agent.mystic.openteams.ai`, `demo.custody.demo.openteams.com` → `demo.mystic.openteams.ai`, `checkmaite.custody.demo.openteams.com` → `checkmaite.mystic.openteams.ai`; delete the entire `# judge` block (3 lines + blank line) and the entire `# langfuse` block (from the `# langfuse` comment through the `...` line, inclusive).

- [ ] **Step 4: Local helm-template section — no more dep step**

Replace:

```markdown
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
```

with:

```markdown
```bash
# Lint both charts
cd dev && make lint

# Render-check both charts
make template

# Or individually
make lint-checkmaite
make template-custody-demo
```
```

Then delete the trailing paragraph:

```markdown
For langfuse, `helm dependency update charts/langfuse/` must be run before
any lint or template operation (the langfuse subchart `.tgz` is not committed).
```

- [ ] **Step 5: Required secrets — update tables**

Delete the `### charts/langfuse (namespace nebari-langfuse-pack)` heading and its table (the `langfuse-core` and `langfuse-init` rows).

In the `### charts/checkmaite (namespace nebari-checkmaite-pack)` table, add a row:

```markdown
| `judge-keys` | `OPENROUTER_API_KEY` |
```

In the `### charts/custody-demo (namespace nebari-custody-demo-pack)` section, change the `model-api-keys` row to:

```markdown
| `model-api-keys` | `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`; optional `OPENROUTER_API_KEY` (enables per-run `openrouter/<vendor>/<model>` SUT model IDs) |
```

After the checkmaite secrets table, add:

```markdown
The `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` values are generated inside
the cluster-provided Langfuse instance (`https://langfuse.mystic.openteams.ai`,
project settings → API keys).  The `OPENROUTER_API_KEY` comes from
[openrouter.ai](https://openrouter.ai) (used by the judge; the default judge
model is `google/gemini-3.5-flash`, overridable via `judge.model`).
```

- [ ] **Step 6: Fix the contract reference path**

Replace:

```markdown
Service names, images, ports, namespaces, and env vars are normative in:
`/docs/superpowers/plans/2026-06-09-shared-contracts.md` §9
```

with:

```markdown
Service names, images, ports, namespaces, and env vars are normative in the
`mystic` program repo: `docs/shared-contracts.md` §9.
```

- [ ] **Step 7: Verify**

Run: `grep -in "four charts\|four chart paths\|custody.demo.openteams.com\|charts/langfuse\|charts/judge\|nebari-judge-pack" README.md`
Expected: no output. ("four-service chart" in the changelog/matrix is correct — custody-demo really has four services — and must stay.)

- [ ] **Step 8: Commit**

```bash
git add README.md
git commit -m "docs: README for two-chart pack; OpenRouter judge; cluster Langfuse; mystic.openteams.ai examples"
```

---

### Task 8: Full-repo verification sweep

**Files:** none (verification only)

- [ ] **Step 1: Stale-reference sweep**

Run:

```bash
cd /Users/khan/openteams/mystic/mystic-software-pack
grep -rn "nebari-langfuse-pack\|nebari-judge-pack\|custody\.demo\.openteams\.com\|efs-sc\|nodegroup: gpu$" \
  --include="*.yaml" --include="*.md" --include="Makefile" \
  . | grep -v "docs/superpowers/"
```

Expected: no output (the `docs/superpowers/` exclusion covers the historical spec/plan).

- [ ] **Step 2: Full make check**

Run: `cd dev && make all`
Expected: both charts lint and template clean, exit 0.

- [ ] **Step 3: Commit (only if the sweep forced fixes)**

If Steps 1–2 required edits, commit them as `fix: <what>` with the standard footer; otherwise no commit.

---

### Task 9: Contract sync in the mystic program repo

**Files:**
- Modify: `/Users/khan/openteams/mystic/mystic/docs/shared-contracts.md` (§6, §7, §9)

**Note:** Separate git checkout. Branch first.

- [ ] **Step 1: Branch**

```bash
cd /Users/khan/openteams/mystic/mystic
git checkout -b cluster-alignment
```

- [ ] **Step 2: §6 — Langfuse host**

Replace:

```markdown
  `LANGFUSE_HOST=http://langfuse-web.nebari-langfuse-pack.svc:3000`,
```

with:

```markdown
  `LANGFUSE_HOST=http://langfuse-web.langfuse.svc:3000` (cluster-provided
  Langfuse: the GitOps repo deploys `nebari-langfuse-pack` as release
  `langfuse` in namespace `langfuse`),
```

- [ ] **Step 3: §7 — judge contract rewrite (full-section replacement)**

Replace the entire §7 body:

```markdown
OpenAI-compatible vLLM endpoint:
`http://judge.nebari-judge-pack.svc:8000/v1`, model name `judge`
(serving `Qwen/Qwen2.5-VL-7B-Instruct`). Judge rubric prompts live in
`custody-benchmarks/rubrics/*.md`. Judge calls request
`response_format={"type": "json_object"}` and must return
`{"score": float, "reasons": str}`.
```

with:

```markdown
OpenAI-compatible **hosted** endpoint (OpenRouter):
`JUDGE_BASE_URL` (default `https://openrouter.ai/api/v1`), model
`JUDGE_MODEL` (default `google/gemini-3.5-flash` — must be vision-capable and
support JSON-object response format), bearer auth `JUDGE_API_KEY` sourced
from K8s Secret `judge-keys` (key `OPENROUTER_API_KEY`) in the checkmaite
namespace. The plugin's `JudgeClient` falls back to model name `judge` and no
Authorization header when these env vars are unset (self-hosted
compatibility). Judge rubric prompts live in
`custody-benchmarks/rubrics/*.md`. Judge calls request
`response_format={"type": "json_object"}` and must return
`{"score": float, "reasons": str}`.
```

- [ ] **Step 4: §9 — judge row**

Replace:

```markdown
| Judge | vLLM, port 8000, served model name `judge` |
```

with:

```markdown
| Judge | hosted OpenRouter endpoint (`https://openrouter.ai/api/v1`), default model `google/gemini-3.5-flash`, auth via Secret `judge-keys` key `OPENROUTER_API_KEY` |
```

- [ ] **Step 5: §9 — Namespaces row**

Replace:

```markdown
| Namespaces | one per pack: `nebari-checkmaite-pack`, `nebari-langfuse-pack`, `nebari-judge-pack`, `nebari-custody-demo-pack` |
```

with:

```markdown
| Namespaces | one per pack: `nebari-checkmaite-pack`, `nebari-custody-demo-pack`. Langfuse is cluster-provided (the GitOps repo `openteams-ai/mystic.openteams.ai` deploys `nebari-langfuse-pack` as release `langfuse` in namespace `langfuse`); the judge is hosted (OpenRouter), no namespace |
```

- [ ] **Step 6: §9 — Analytics store row**

Replace:

```markdown
| Analytics store path | EFS mount `/data/analytics` in checkmaite pack |
```

with:

```markdown
| Analytics store path | RWX PVC (longhorn) mounted at `/data/analytics` in checkmaite pack |
```

- [ ] **Step 7: §9 — Pack repo row**

In the `| Pack repo |` row, replace the phrase `ONE repo `mystic-software-pack` holding all four charts` with `ONE repo `mystic-software-pack` holding both charts (custody-demo, checkmaite; Langfuse is cluster-provided, the judge is hosted)` — leave the rest of the row unchanged.

- [ ] **Step 8: Verify**

Run:

```bash
grep -n "nebari-langfuse-pack.svc\|nebari-judge-pack\|all four charts\|EFS mount\|vLLM, port 8000" \
  /Users/khan/openteams/mystic/mystic/docs/shared-contracts.md
```

Expected: no output. Then skim §7 once more for any remaining self-hosted-vLLM phrasing.

- [ ] **Step 9: Commit**

```bash
cd /Users/khan/openteams/mystic/mystic
git add docs/shared-contracts.md
git commit -m "docs(contracts): OpenRouter judge (gemini-3.5-flash, judge-keys); cluster-provided Langfuse; longhorn analytics; two-chart pack"
```

(Standard footer.)
