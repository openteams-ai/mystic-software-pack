# Cluster Alignment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Align `mystic-software-pack` with the already-deployed `mystic.openteams.ai` cluster: retire the redundant langfuse chart, point everything at the cluster-provided Langfuse, make the judge chart fit the cluster's GPU nodes, switch storage to longhorn, and sync the normative contract doc.

**Architecture:** Pure configuration surgery — no new templates or services. The langfuse chart is deleted (the cluster GitOps repo already deploys `nebari-langfuse-pack`); the three remaining charts get values-default fixes; `mystic/docs/shared-contracts.md` (a sibling repo checkout) is updated in the same pass because it is normative.

**Tech Stack:** Helm 3, GNU Make, GitHub Actions. Verification is `helm lint` / `helm template` via `dev/Makefile`; there is no test suite in this repo.

**Spec:** `docs/superpowers/specs/2026-06-11-cluster-alignment-design.md`

**Working branch:** `cluster-alignment` in `/Users/khan/openteams/mystic/mystic-software-pack` (already created; spec committed). Task 8 commits to the **sibling** repo `/Users/khan/openteams/mystic/mystic` — branch there first.

**Commit footer for every commit:**

```
Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>
```

---

### Task 1: Delete the langfuse chart

**Files:**
- Delete: `charts/langfuse/` (entire directory, including any fetched `charts/*.tgz` dependency)

- [ ] **Step 1: Remove the directory**

```bash
cd /Users/khan/openteams/mystic/mystic-software-pack
git rm -r charts/langfuse
rm -rf charts/langfuse   # clears untracked fetched dependencies (.tgz, Chart.lock)
```

- [ ] **Step 2: Verify only three charts remain**

Run: `ls charts/`
Expected: `checkmaite  custody-demo  judge`

- [ ] **Step 3: Commit**

```bash
git commit -m "feat!: retire langfuse chart — cluster provides Langfuse via nebari-langfuse-pack"
```

(Use the standard footer. The `!` marks a breaking change: anyone deploying `charts/langfuse` from this repo must switch to the cluster-provided instance.)

---

### Task 2: Rewrite `dev/Makefile` for three charts

**Files:**
- Modify: `dev/Makefile` (full replacement)

- [ ] **Step 1: Replace the file contents with exactly this**

```make
# dev/Makefile — local lint / template targets for mystic-software-pack
#
# Prerequisites: helm 3.8+
#
# Usage:
#   make lint          helm lint all three charts
#   make template      helm template render-check for all three charts
#   make all           lint + template
#
# Individual chart targets:
#   make lint-custody-demo
#   make lint-checkmaite
#   make lint-judge
#   make template-custody-demo
#   make template-checkmaite
#   make template-judge
#
# NOTE: Langfuse is NOT deployed from this repo.  The cluster GitOps repo
# (openteams-ai/mystic.openteams.ai) deploys nebari-langfuse-pack as release
# "langfuse" in namespace "langfuse".

CHARTS_DIR := ../charts

.PHONY: all lint template \
        lint-custody-demo lint-checkmaite lint-judge \
        template-custody-demo template-checkmaite template-judge

all: lint template

# --------------------------------------------------------------------------
# lint — validate all three charts
# --------------------------------------------------------------------------
lint: lint-custody-demo lint-checkmaite lint-judge

lint-custody-demo:
	helm lint $(CHARTS_DIR)/custody-demo/

lint-checkmaite:
	helm lint $(CHARTS_DIR)/checkmaite/

lint-judge:
	helm lint $(CHARTS_DIR)/judge/

# --------------------------------------------------------------------------
# template — render-check all three charts (both enabled/disabled NebariApp)
# --------------------------------------------------------------------------
template: template-custody-demo template-checkmaite template-judge

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

template-judge:
	@echo "--- judge (default profile: gpu-small + 3B) ---"
	helm template t $(CHARTS_DIR)/judge/ --set nebariapp.enabled=false > /dev/null
	@echo "--- judge (7B model path) ---"
	helm template t $(CHARTS_DIR)/judge/ \
		--set nebariapp.enabled=false \
		--set model.useFallback=false > /dev/null
	@echo "judge: OK"
```

(Removed: the `dep` target — langfuse was the only chart with dependencies — plus `lint-langfuse` / `template-langfuse` and their `.PHONY`/aggregate/header mentions. Added: a second judge render exercising the 7B path.)

- [ ] **Step 2: Run the full check**

Run: `cd /Users/khan/openteams/mystic/mystic-software-pack/dev && make all`
Expected: lint passes for all three charts; template prints `custody-demo: OK`, `checkmaite: OK`, `judge: OK`; exit 0. (Note: `make template-judge` exercises judge values *before* Task 4 edits them — it must pass both before and after.)

- [ ] **Step 3: Commit**

```bash
cd /Users/khan/openteams/mystic/mystic-software-pack
git add dev/Makefile
git commit -m "chore(dev): drop langfuse targets from Makefile; add 7B render check for judge"
```

---

### Task 3: Update CI workflow to mirror the Makefile

**Files:**
- Modify: `.github/workflows/lint.yaml`

- [ ] **Step 1: Delete the langfuse dependency step**

Remove this block (lines ~20–24, directly after the `Set up Helm` step):

```yaml
      # ----------------------------------------------------------------
      # langfuse — dependency chart must be fetched before lint/template
      # ----------------------------------------------------------------
      - name: Update langfuse dependency
        run: helm dependency update charts/langfuse/
```

- [ ] **Step 2: Delete the entire langfuse lint/template section at the end of the file**

Remove everything from the comment banner `# langfuse` through the end of the `Template langfuse (NebariApp enabled)` step (the `Lint langfuse` step, both `Template langfuse` steps, and their `--set ...password=ci` flags).

- [ ] **Step 3: Mirror the new judge render in CI**

Replace:

```yaml
      - name: Template judge (NebariApp disabled — judge is always cluster-internal)
        run: helm template t charts/judge/ --set nebariapp.enabled=false
```

with:

```yaml
      - name: Template judge (NebariApp disabled — judge is always cluster-internal)
        run: helm template t charts/judge/ --set nebariapp.enabled=false

      - name: Template judge (7B model path)
        run: |
          helm template t charts/judge/ \
            --set nebariapp.enabled=false \
            --set model.useFallback=false
```

- [ ] **Step 4: Verify CI mirrors the Makefile one-for-one**

Run: `grep -c "helm " .github/workflows/lint.yaml` and compare against the Makefile target list — there must be exactly: 3 lint commands + 7 template commands (2 custody-demo, 2 checkmaite, 2 judge) = no langfuse mention anywhere.
Run: `grep -i langfuse .github/workflows/lint.yaml`
Expected: no output.

- [ ] **Step 5: Commit**

```bash
git add .github/workflows/lint.yaml
git commit -m "ci: drop langfuse steps; mirror Makefile judge 7B render check"
```

---

### Task 4: Point checkmaite + custody-demo at the cluster Langfuse; longhorn analytics PVC

**Files:**
- Modify: `charts/checkmaite/values.yaml` (langfuse host, analytics storage class, hostname placeholder)
- Modify: `charts/custody-demo/values.yaml` (`LANGFUSE_HOST` env)

- [ ] **Step 1: checkmaite — langfuse host**

In `charts/checkmaite/values.yaml`, replace:

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

- [ ] **Step 2: checkmaite — analytics storage class**

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

- [ ] **Step 3: checkmaite — hostname placeholder**

Replace:

```yaml
  # hostname: checkmaite.custody.demo.openteams.com  # Required when enabled
```

with:

```yaml
  # hostname: checkmaite.mystic.openteams.ai  # Required when enabled
```

- [ ] **Step 4: custody-demo — LANGFUSE_HOST**

In `charts/custody-demo/values.yaml`, replace:

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

- [ ] **Step 5: custody-demo — hostname placeholders**

Replace `# hostname: agent.custody.demo.openteams.com` with `# hostname: agent.mystic.openteams.ai`, and `# hostname: demo.custody.demo.openteams.com` with `# hostname: demo.mystic.openteams.ai`.

- [ ] **Step 5b: custody-demo — GPU node selector (shared `gpu:` block, used by custody-tools when `gpu: true`)**

In `charts/custody-demo/values.yaml`, replace:

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

- [ ] **Step 6: Verify**

Run: `grep -rn "nebari-langfuse-pack\|custody.demo.openteams.com\|efs-sc" charts/checkmaite charts/custody-demo`
Expected: no output.
Run: `cd dev && make template-checkmaite template-custody-demo && cd ..`
Expected: `checkmaite: OK`, `custody-demo: OK`.

- [ ] **Step 7: Commit**

```bash
git add charts/checkmaite/values.yaml charts/custody-demo/values.yaml
git commit -m "feat: consume cluster-provided Langfuse; longhorn analytics PVC; gpu-small selector; mystic.openteams.ai hostnames"
```

---

### Task 5: Judge chart — GPU profile defaults + right-sized resources + longhorn cache

**Files:**
- Modify: `charts/judge/values.yaml`

- [ ] **Step 1: Default to the 3B model (fits gpu-small / T4)**

Replace:

```yaml
model:
  # primary model — Qwen2.5-VL-7B-Instruct
  primary: Qwen/Qwen2.5-VL-7B-Instruct
  # fallback — flip useFallback: true if 7B OOMs on the GPU node
  fallback: Qwen/Qwen2.5-VL-3B-Instruct
  useFallback: false
```

with:

```yaml
model:
  # primary model — Qwen2.5-VL-7B-Instruct (~15 GB fp16 weights; needs the
  # H100 profile below, does NOT fit the default T4 node)
  primary: Qwen/Qwen2.5-VL-7B-Instruct
  # fallback — Qwen2.5-VL-3B-Instruct fits gpu-small (g4dn.xlarge, T4 16 GB)
  fallback: Qwen/Qwen2.5-VL-3B-Instruct
  # Default profile serves the 3B fallback; set false with the H100 profile.
  useFallback: true
```

- [ ] **Step 2: Longhorn weight cache**

Replace:

```yaml
cache:
  storageClassName: efs-sc
  size: 100Gi
```

with:

```yaml
cache:
  storageClassName: longhorn
  size: 100Gi
```

- [ ] **Step 3: GPU scheduling — gpu-small default, commented H100 profile**

Replace:

```yaml
nodeSelector:
  nodegroup: gpu

tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
```

with:

```yaml
# Default profile: gpu-small node group (g4dn.xlarge, T4 16 GB, scale-to-zero).
nodeSelector:
  eks.amazonaws.com/nodegroup: gpu-small

tolerations:
  - key: nvidia.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule

# --- H100 profile (gpu-large, p5.4xlarge) -----------------------------------
# To serve the 7B model: set model.useFallback=false and REPLACE the
# nodeSelector/tolerations above with the following (the gpu-large node group
# is tainted gpu.type=h100:NoSchedule):
#
# nodeSelector:
#   gpu.type: h100
# tolerations:
#   - key: gpu.type
#     operator: Equal
#     value: h100
#     effect: NoSchedule
```

- [ ] **Step 4: Right-size resources for g4dn.xlarge**

Replace:

```yaml
resources:
  requests:
    cpu: "3"
    memory: 10Gi
    nvidia.com/gpu: 1
  limits:
    memory: 13Gi
    nvidia.com/gpu: 1
```

with:

```yaml
# Sized for g4dn.xlarge (4 vCPU / 16 GiB).  The 4 Gi /dev/shm emptyDir
# (medium: Memory) counts against the container memory limit.
resources:
  requests:
    cpu: "2"
    memory: 8Gi
    nvidia.com/gpu: 1
  limits:
    memory: 12Gi
    nvidia.com/gpu: 1
```

- [ ] **Step 5: Verify default profile renders the 3B model on gpu-small**

Run:

```bash
helm template t charts/judge/ --set nebariapp.enabled=false | grep -E "Qwen|nodegroup|longhorn"
```

Expected output includes `Qwen/Qwen2.5-VL-3B-Instruct`, `eks.amazonaws.com/nodegroup: gpu-small`, and `storageClassName: longhorn`. It must NOT include `7B`.

- [ ] **Step 6: Verify the H100 profile renders cleanly (spec verification item)**

Run:

```bash
helm template t charts/judge/ \
  --set nebariapp.enabled=false \
  --set model.useFallback=false \
  --set 'nodeSelector.gpu\.type=h100' \
  --set 'nodeSelector.eks\.amazonaws\.com/nodegroup=null' \
  --set 'tolerations[0].key=gpu.type' \
  --set 'tolerations[0].operator=Equal' \
  --set 'tolerations[0].value=h100' \
  --set 'tolerations[0].effect=NoSchedule' \
  | grep -E "Qwen|gpu.type"
```

Expected output includes `Qwen/Qwen2.5-VL-7B-Instruct` and `gpu.type: h100`; no template errors.

- [ ] **Step 7: Run the chart checks**

Run: `cd dev && make lint-judge template-judge && cd ..`
Expected: lint passes, `judge: OK`.

- [ ] **Step 8: Commit**

```bash
git add charts/judge/values.yaml
git commit -m "feat(judge): default gpu-small/T4 profile serving 3B; commented H100 profile; longhorn cache; g4dn.xlarge-sized resources"
```

---

### Task 6: README overhaul

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
Nebari software pack repository for the **Mystic LOE-1 demo stack** — three
template-conformant Helm charts, one ArgoCD repo, three chart paths.
Langfuse is **not** deployed from this repo; the cluster GitOps repo
([openteams-ai/mystic.openteams.ai](https://github.com/openteams-ai/mystic.openteams.ai))
deploys `nebari-langfuse-pack` as release `langfuse` in namespace `langfuse`.
```

- [ ] **Step 2: Chart matrix — drop the langfuse row**

Delete the line:

```markdown
| `charts/langfuse` | `nebari-langfuse-pack` | langfuse-web + worker + deps | yes | Wraps official langfuse chart; /api/public/* bypasses Keycloak |
```

- [ ] **Step 3: ArgoCD section — three apps, mystic.openteams.ai hostnames**

Replace the sentence:

```markdown
Four ArgoCD `Application` resources point at the **same `repoURL`** with
different `path:` values:
```

with:

```markdown
Three ArgoCD `Application` resources (added to the cluster GitOps repo's
`apps/apps/` directory) point at the **same `repoURL`** with different
`path:` values:
```

In the YAML example, replace `agent.custody.demo.openteams.com` → `agent.mystic.openteams.ai`, `demo.custody.demo.openteams.com` → `demo.mystic.openteams.ai`, `checkmaite.custody.demo.openteams.com` → `checkmaite.mystic.openteams.ai`, and delete the entire `# langfuse` block (from the `# langfuse` comment line through the `...` line, inclusive).

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
# Lint all three charts
cd dev && make lint

# Render-check all three charts
make template

# Or individually
make lint-judge
make template-custody-demo
```
```

Then delete the trailing paragraph:

```markdown
For langfuse, `helm dependency update charts/langfuse/` must be run before
any lint or template operation (the langfuse subchart `.tgz` is not committed).
```

- [ ] **Step 5: Required secrets — drop the langfuse section, note key provenance**

Delete the `### charts/langfuse (namespace nebari-langfuse-pack)` heading and its table (the `langfuse-core` and `langfuse-init` rows). After the `charts/checkmaite` secrets table, add:

```markdown
The `LANGFUSE_PUBLIC_KEY` / `LANGFUSE_SECRET_KEY` values are generated inside
the cluster-provided Langfuse instance (`https://langfuse.mystic.openteams.ai`,
project settings → API keys).
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

Run: `grep -in "langfuse" README.md`
Expected: hits only in the intro note, the secrets-provenance note, and the key tables (`langfuse-keys` rows) — no chart-matrix row, no ArgoCD example, no `make *-langfuse` targets, no `charts/langfuse` path.
Run: `grep -in "four charts\|four chart paths\|custody.demo.openteams.com" README.md`
Expected: no output. (Note: "four-service chart" in the changelog and chart
matrix is correct — custody-demo really has four services — and must stay.)

- [ ] **Step 8: Commit**

```bash
git add README.md
git commit -m "docs: README for three-chart pack; cluster-provided Langfuse; mystic.openteams.ai examples"
```

---

### Task 7: Full-repo verification sweep

**Files:** none (verification only)

- [ ] **Step 1: Stale-reference sweep**

Run:

```bash
cd /Users/khan/openteams/mystic/mystic-software-pack
grep -rn "nebari-langfuse-pack\|custody\.demo\.openteams\.com\|efs-sc\|nodegroup: gpu$" \
  --include="*.yaml" --include="*.md" --include="Makefile" \
  . | grep -v "docs/superpowers/"
```

Expected: no output (the `docs/superpowers/` exclusion covers the historical spec/plan).

- [ ] **Step 2: Full make check**

Run: `cd dev && make all`
Expected: all three charts lint and template clean, exit 0.

- [ ] **Step 3: Commit (only if the sweep forced fixes)**

If Steps 1–2 required edits, commit them as `fix: <what>` with the standard footer; otherwise no commit.

---

### Task 8: Contract sync in the mystic program repo

**Files:**
- Modify: `/Users/khan/openteams/mystic/mystic/docs/shared-contracts.md` (§6 line ~185, §9 lines ~253, ~256, ~258)

**Note:** This is a *separate git checkout* (the program repo). Branch before editing.

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

- [ ] **Step 3: §9 — Namespaces row**

Replace:

```markdown
| Namespaces | one per pack: `nebari-checkmaite-pack`, `nebari-langfuse-pack`, `nebari-judge-pack`, `nebari-custody-demo-pack` |
```

with:

```markdown
| Namespaces | one per pack: `nebari-checkmaite-pack`, `nebari-judge-pack`, `nebari-custody-demo-pack`. Langfuse is cluster-provided: the GitOps repo (`openteams-ai/mystic.openteams.ai`) deploys `nebari-langfuse-pack` as release `langfuse` in namespace `langfuse` |
```

- [ ] **Step 4: §9 — Analytics store row**

Replace:

```markdown
| Analytics store path | EFS mount `/data/analytics` in checkmaite pack |
```

with:

```markdown
| Analytics store path | RWX PVC (longhorn) mounted at `/data/analytics` in checkmaite pack |
```

- [ ] **Step 5: §9 — Pack repo row**

In the `| Pack repo |` row, replace the phrase `ONE repo `mystic-software-pack` holding all four charts` with `ONE repo `mystic-software-pack` holding all three charts (custody-demo, checkmaite, judge; Langfuse is cluster-provided)` — leave the rest of the row unchanged.

- [ ] **Step 6: Verify**

Run: `grep -n "nebari-langfuse-pack.svc\|all four charts\|EFS mount" /Users/khan/openteams/mystic/mystic/docs/shared-contracts.md`
Expected: no output.

- [ ] **Step 7: Commit**

```bash
cd /Users/khan/openteams/mystic/mystic
git add docs/shared-contracts.md
git commit -m "docs(contracts): Langfuse is cluster-provided (ns langfuse); longhorn analytics PVC; three-chart pack"
```

(Standard footer.)
