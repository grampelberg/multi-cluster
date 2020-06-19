SHELL := bash
.ONESHELL:
.SHELLFLAGS := -eu -o pipefail -c -x
.DELETE_ON_ERROR:
MAKEFLAGS += --warn-undefined-variables
MAKEFLAGS += --no-builtin-rules

MKFILE_PATH := $(abspath $(lastword $(MAKEFILE_LIST)))
CURRENT_DIR := $(notdir $(patsubst %/,%,$(dir $(MKFILE_PATH))))
TMP_DIR := /tmp/multicluster

export PATH := ~/.linkerd2/bin:$(PATH)

HAS_K3D := $(shell command -v k3d;)
HAS_LINKERD := $(shell ls ~/.linkerd2/bin/linkerd;)
HAS_STEP := $(shell command -v step;)

CLUSTER_NAMES ?= east west

.PHONY: bootstrap
bootstrap: | tmp
	@# Bootstrap the local dependencies
ifndef HAS_K3D
	curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | TAG=v3.0.0-rc.6 bash
endif
ifndef HAS_LINKERD
	curl -sL run.linkerd.io/install | bash
endif
ifndef HAS_STEP
	curl -sL https://github.com/smallstep/cli/releases/download/v0.14.4/step-cli_0.14.4_amd64.deb \
		-o $(TMP_DIR)/step.deb \
	&& sudo dpkg -i $(TMP_DIR)/step.deb
endif

.PHONY: get-port
get-port:
	@netstat -aln | awk ' \
		$$6 == "LISTEN" { \
			if ($$4 ~ "[.:][0-9]+$$") { \
				len = split($$4, a, /[:.]/); \
				port = a[len]; \
				p[port] = 1 \
			} \
		} \
		END { \
			for (i = 3000; i < 65000 && p[i]; i++){}; \
			if (i == 65000) {exit 1}; \
			print i \
		} \
	'

.PHONY: get-ip
get-ip-%:
	@docker inspect \
		-f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
		k3d-$*-master-0

.PHONY: create-cluster
create-cluster: $(addprefix create-cluster-,$(CLUSTER_NAMES))

.PHONY: delete-cluster
delete-cluster: $(addprefix delete-cluster-,$(CLUSTER_NAMES))

create-cluster-%:
	PORT=$$($(MAKE) --no-print-directory get-port); \
		k3d create cluster $* \
		--update-kubeconfig \
		--wait \
		--api-port $${PORT} \
		--port $$(($${PORT} + 80)):80@loadbalancer \
		--port $$(($${PORT} + 443)):443@loadbalancer \
	&& kubectl config set-cluster k3d-$* \
		--server=https://$$($(MAKE) --no-print-directory get-ip-$*):6443

delete-cluster-%:
	k3d delete cluster $* || true

.PHONY: install-deps
install-linkerd: cluster-issuer $(addprefix install-linkerd-,$(CLUSTER_NAMES))

install-linkerd-%:
	linkerd --context=k3d-$* install \
		--identity-trust-anchors-file=$(TMP_DIR)/anchor.cert \
		--identity-issuer-certificate-file=$(TMP_DIR)/$*.issuer.cert \
		--identity-issuer-key-file=$(TMP_DIR)/$*.issuer.key | \
	kubectl --context=k3d-$* apply -f - && \
	linkerd check

.PHONY: install-mc
install-mc: install-linkerd $(addprefix install-mc-,$(CLUSTER_NAMES))

install-mc-%:
	linkerd --context=k3d-$* mc install | \
		kubectl --context=k3d-$* apply -f - \
	&& linkerd check --multicluster

.PHONY: trust-anchor
trust-anchor: | tmp
	step certificate create \
		anchor.trust.cluster.local \
		$(TMP_DIR)/anchor.cert \
		$(TMP_DIR)/anchor.key \
		--profile root-ca \
		--no-password  --insecure --force

.PHONY: cluster-issuer
cluster-issuer: $(addprefix cluster-issuer-,$(CLUSTER_NAMES))

cluster-issuer-%: | trust-anchor
	step certificate create \
		identity.linkerd.cluster.local \
		$(TMP_DIR)/$*.issuer.cert \
		$(TMP_DIR)/$*.issuer.key \
		--ca=$(TMP_DIR)/anchor.cert \
		--ca-key=$(TMP_DIR)/anchor.key \
		--profile=intermediate-ca \
		--not-after 8760h --no-password --insecure --force

tmp:
	mkdir -p $(TMP_DIR)

.PHONY: clean
clean: delete-cluster
	rm -rf $(TMP_DIR)

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
