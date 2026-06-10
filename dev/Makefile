# dev/Makefile — local lint / template targets for mystic-software-pack
#
# Prerequisites: helm 3.8+
#
# Usage:
#   make dep           Fetch all chart dependencies (langfuse subchart)
#   make lint          helm lint all four charts
#   make template      helm template render-check for all four charts
#   make all           dep + lint + template
#
# Individual chart targets:
#   make lint-custody-demo
#   make lint-checkmaite
#   make lint-judge
#   make lint-langfuse
#   make template-custody-demo
#   make template-checkmaite
#   make template-judge
#   make template-langfuse

CHARTS_DIR := ../charts

.PHONY: all dep lint template \
        lint-custody-demo lint-checkmaite lint-judge lint-langfuse \
        template-custody-demo template-checkmaite template-judge template-langfuse

all: dep lint template

# --------------------------------------------------------------------------
# dep — fetch upstream chart dependencies (langfuse subchart)
# --------------------------------------------------------------------------
dep:
	helm dependency update $(CHARTS_DIR)/langfuse/

# --------------------------------------------------------------------------
# lint — validate all four charts
# --------------------------------------------------------------------------
lint: lint-custody-demo lint-checkmaite lint-judge lint-langfuse

lint-custody-demo:
	helm lint $(CHARTS_DIR)/custody-demo/

lint-checkmaite:
	helm lint $(CHARTS_DIR)/checkmaite/

lint-judge:
	helm lint $(CHARTS_DIR)/judge/

lint-langfuse: dep
	helm lint $(CHARTS_DIR)/langfuse/

# --------------------------------------------------------------------------
# template — render-check all four charts (both enabled/disabled NebariApp)
# --------------------------------------------------------------------------
template: template-custody-demo template-checkmaite template-judge template-langfuse

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
	@echo "--- judge ---"
	helm template t $(CHARTS_DIR)/judge/ --set nebariapp.enabled=false > /dev/null
	@echo "judge: OK"

template-langfuse: dep
	@echo "--- langfuse (nebariapp.enabled=false) ---"
	helm template t $(CHARTS_DIR)/langfuse/ \
		--set langfuse.postgresql.auth.password=dev \
		--set langfuse.clickhouse.auth.password=dev \
		--set langfuse.redis.auth.password=dev \
		--set langfuse.minio.auth.rootPassword=dev > /dev/null
	@echo "--- langfuse (nebariapp.enabled=true) ---"
	helm template t $(CHARTS_DIR)/langfuse/ \
		--set nebariapp.enabled=true \
		--set nebariapp.hostname=langfuse.custody.local \
		--set langfuse.langfuse.nextauth.url=https://langfuse.custody.local \
		--set langfuse.postgresql.auth.password=dev \
		--set langfuse.clickhouse.auth.password=dev \
		--set langfuse.redis.auth.password=dev \
		--set langfuse.minio.auth.rootPassword=dev > /dev/null
	@echo "langfuse: OK"
