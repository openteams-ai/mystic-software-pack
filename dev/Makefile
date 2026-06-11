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
		--set nebariapp.agent.hostname=agent.custody.local > /dev/null
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
