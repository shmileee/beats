# Name can be overwritten, as Metricbeat is also a library
BEAT_NAME?=metricbeat
BEAT_TITLE?=Metricbeat
SYSTEM_TESTS?=true
TEST_ENVIRONMENT?=true
BEATS_DOCKER_INTEGRATION_TEST_ENV?=true
ES_BEATS?=..

# Metricbeat can only be cross-compiled on platforms not requiring CGO.
GOX_OS=netbsd linux windows
GOX_FLAGS=-arch="amd64 386 arm ppc64 ppc64le"

DOCS_BRANCH=$(shell grep doc-branch ../libbeat/docs/version.asciidoc | cut -c 14-)

include ${ES_BEATS}/libbeat/scripts/Makefile

# Collects all module dashboards
.PHONY: kibana
kibana:
	@rm -rf _meta/kibana.generated
	@mkdir -p _meta/kibana.generated
	@-cp -pr module/*/_meta/kibana/* _meta/kibana.generated

# Collects all module docs
.PHONY: collect-docs
collect-docs:
	mage CollectAll

# Collects all module configs
.PHONY: configs
configs: python-env
	@mkdir -p _meta
	@cp ${ES_BEATS}/metricbeat/_meta/common.yml _meta/beat.yml
	@cat ${ES_BEATS}/metricbeat/_meta/setup.yml >> _meta/beat.yml
	@cat ${ES_BEATS}/metricbeat/_meta/common.reference.yml > _meta/beat.reference.yml
	@${PYTHON_ENV_EXE} ${ES_BEATS}/script/config_collector.py --beat ${BEAT_NAME} --full $(PWD) >> _meta/beat.reference.yml
	@rm -rf modules.d
	mage config
	@chmod go-w modules.d/*
	@# Enable system by default:
	@if [ -f modules.d/system.yml.disabled ]; then mv modules.d/system.yml.disabled modules.d/system.yml; fi

# Generates imports for all modules and metricsets
.PHONY: imports
imports:
	@mkdir -p include
	mage imports

# Runs all collection steps and updates afterwards
.PHONY: collect
collect: assets collect-docs configs kibana imports

# Creates a new metricset. Requires the params MODULE and METRICSET
.PHONY: create-metricset
create-metricset: python-env
	@${PYTHON_ENV_EXE} ${ES_BEATS}/metricbeat/scripts/create_metricset.py --path=$(PWD) --es_beats=$(ES_BEATS) --module=$(MODULE) --metricset=$(METRICSET)

# Generates the data.json example documents
.PHONY: generate-json
generate-json: build-image
	 ${DOCKER_COMPOSE} run beat go test -tags=integration github.com/elastic/beats/metricbeat/module/... -data

.PHONY: run-module
run-module: ## @testing Runs the given module with exposing the port. Needs $MODULE and $PORT as param
run-module:
	${DOCKER_COMPOSE} build ${MODULE}
	${DOCKER_COMPOSE} run -p ${PORT}:${PORT} ${MODULE}

.PHONY: test-module
test-module: ## @testing Tests the given module. Needs $MODULE as param an run-module must be started first.
test-module: python-env update metricbeat.test
	go test -tags=integration ${BEAT_PATH}/module/${MODULE}/... -v
	. ${PYTHON_ENV}/bin/activate && INTEGRATION_TESTS=1 nosetests module/${MODULE}

.PHONY: assets
assets:
	go run ${INSTALL_FLAG} ${ES_BEATS}/metricbeat/scripts/assets/assets.go ${ES_BEATS}/metricbeat/module
	mkdir -p include/fields
	go run ${INSTALL_FLAG} ${ES_BEATS}/libbeat/scripts/cmd/global_fields/main.go -es_beats_path ${ES_BEATS} -beat_path ${PWD} | go run  ${ES_BEATS}/dev-tools/cmd/asset/asset.go -license ${LICENSE} -out ./include/fields/fields.go -pkg include -priority asset.LibbeatFieldsPri ${ES_BEATS}/libbeat/fields.yml $(BEAT_NAME)

.PHONY: integration-tests
integration-tests: ## @testing Run golang integration tests.
integration-tests: prepare-tests mage
	rm -f docker-compose.yml.lock
	mage goIntegTest
