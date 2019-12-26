SHELL := bash
.ONESHELL:
.SHELLFLAGS := -eu -o pipefail -c -x
.DELETE_ON_ERROR:
MAKEFLAGS += --warn-undefined-variables
MAKEFLAGS += --no-builtin-rules

MKFILE_PATH := $(abspath $(lastword $(MAKEFILE_LIST)))
CURRENT_DIR := $(notdir $(patsubst %/,%,$(dir $(MKFILE_PATH))))

HAS_K3D := $(shell command -v k3d;)

CLUSTER_NAMES ?= left right
WAIT ?= 360

CURRENT_CLUSTER ?=
CMD ?=

export KUBECONFIG := $(shell k3d get-kubeconfig --name='$(CURRENT_CLUSTER)' 2>/dev/null || true)

define kubectl
KUBECONFIG=$$(k3d get-kubeconfig --name='$(1)') && kubectl
endef

.PHONY: bootstrap
bootstrap:
	@# Bootstrap the local dependencies
ifndef HAS_K3D
	curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | bash
endif

.PHONY: get-port
get-port:
	@netstat -aln | awk ' \
		$$6 == "LISTEN" { \
			if ($$4 ~ "[.:][0-9]+$$") { \
				split($$4, a, /[:.]/); \
				port = a[length(a)]; \
				p[port] = 1 \
			} \
		} \
		END { \
			for (i = 3000; i < 65000 && p[i]; i++){}; \
			if (i == 65000) {exit 1}; \
			print i \
		} \
	'

.PHONY: create-cluster
create-cluster: $(addprefix create-cluster-,$(CLUSTER_NAMES))

.PHONY: delete-cluster
delete-cluster: $(addprefix delete-cluster-,$(CLUSTER_NAMES))

create-cluster-%:
	PORT=$$($(MAKE) get-port); k3d create \
		--name $* \
		--api-port $${PORT} \
		--publish $$(($${PORT} + 80)):80 $$(($${PORT} + 443)):443
	grep -m 1 "Successfully registered node" \
		<(docker logs $$(docker ps -f "label=cluster=$*" -q) -f 2>&1)

delete-cluster-%:
	k3d delete --name $* || true

.PHONY: install-deps
install-deps: $(addprefix install-deps-,$(CLUSTER_NAMES))

install-deps-%:
	$(call kubectl,$*) apply -f k8s

.PHONY: clean
clean: delete-cluster

.PHONY: help
help:
	@# Output all targets available.
	@ echo "usage: make [target] ..."
	@ echo ""
	@eval "echo \"$$(grep -h -B1 $$'^\t@#' $(MAKEFILE_LIST) \
		| sed 's/@#//' \
		| awk \
			-v NO_COLOR="$(NO_COLOR)" \
			-v OK_COLOR="$(OK_COLOR)" \
			-v RS="--\n" \
			-v FS="\n" \
			-v OFS="@@" \
			'{ split($$1,target,":"); $$1=""; print OK_COLOR target[1] NO_COLOR $$0 }' \
		| sort \
		| awk \
			-v FS="@@" \
			-v OFS="\n" \
			'{ CMD=$$1; $$1=""; print CMD $$0 }')\""

.DEFAULT_GOAL := help
