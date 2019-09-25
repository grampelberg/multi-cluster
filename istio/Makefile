# Note: this doesn't work for 1.3 yet, see https://github.com/istio/istio/issues/16890
export ISTIO_VERSION ?= 1.2.6
export REMOTE_ENDPOINT ?=

CLUSTER_NAME ?= gke-thomas-dev
ALL_CLUSTERS ?= gke-thomas-dev gke-reference-dev
.PHONY: all $(ALL_CLUSTERS)

define kubectl
	kubectl $(FLAGS) --context=$(CLUSTER_NAME)
endef

define cert-opts
	--no-password \
	--insecure \
	--not-before="-24h" \
	--kty RSA \
	--size 2048
endef

define endpoint
$$($(call kubectl) -n istio-system get svc --selector=app=istio-ingressgateway \
	-o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
endef

define sleep_pod
$$($(call kubectl) get po -l app=sleep \
	-o jsonpath='{.items[0].metadata.name}')
endef

define dns_ip
$$($(call kubectl) get svc istiocoredns -o jsonpath={.spec.clusterIP})
endef

TMP = tmp

$(TMP):
	mkdir -p $(TMP)

# ------------------------------------------------------------------------------
# Istio configuration and installation
# ------------------------------------------------------------------------------

$(TMP)/istio-$(ISTIO_VERSION): | tmp
	cd $(TMP) && \
		curl -L https://git.io/getLatestIstio | sh -

$(TMP)/cp.yaml: $(TMP)/istio-$(ISTIO_VERSION)
	helm template tmp/istio-*/install/kubernetes/helm/istio \
		--name istio \
		--namespace istio-system \
		-f helm/values.yaml \
		> tmp/cp.yaml

$(TMP)/init.yaml: $(TMP)/istio-$(ISTIO_VERSION)
	helm template tmp/istio-*/install/kubernetes/helm/istio-init \
		--name istio-init \
		--namespace istio-system \
		-f helm/values-init.yaml \
		> tmp/init.yaml

# ------------------------------------------------------------------------------
# Certificate creation
# ------------------------------------------------------------------------------

CRYPTO = $(TMP)/crypto

$(CRYPTO): tmp
	mkdir -p tmp/crypto

$(CRYPTO)/root-cert.pem: | $(CRYPTO)
	cd $(CRYPTO) && \
	step certificate create root.istio.io \
		root-cert.pem \
		root-key.pem \
		--profile root-ca \
		$(call cert-opts)

$(CRYPTO)/ca-cert.pem: $(CRYPTO)/root-cert.pem
	cd $(CRYPTO) && \
	step certificate create ca.istio.io \
		ca-cert.pem \
		ca-key.pem \
		--ca root-cert.pem \
		--ca-key root-key.pem \
		--profile intermediate-ca \
		$(call cert-opts)

$(CRYPTO)/cert-chain.pem: $(CRYPTO)/ca-cert.pem
	cd $(CRYPTO) && \
	step certificate bundle \
		ca-cert.pem \
		root-cert.pem \
		cert-chain.pem

.PHONY: cacerts
cacerts: FLAGS = -n istio-system
cacerts: $(CRYPTO)/ca-cert.pem $(CRYPTO)/cert-chain.pem
	$(call kubectl) create ns istio-system || true
	cd $(CRYPTO) && \
	$(call kubectl) create secret generic cacerts \
		--from-file ca-cert.pem \
		--from-file ca-key.pem \
		--from-file cert-chain.pem \
		--from-file root-cert.pem

# ------------------------------------------------------------------------------
# Control plane setup
# ------------------------------------------------------------------------------

.PHONY: install
install: FLAGS = -n istio-system
install: $(TMP)/cp.yaml $(TMP)/init.yaml cacerts
	$(call kubectl) apply -f tmp/init.yaml
	$(call kubectl) wait job --all --for=condition=Complete
	$(call kubectl) apply -f $(TMP)/cp.yaml
	$(call kubectl) wait deploy \
		-l release=istio \
		--for=condition=Available \
		--timeout=300s
	cat k8s/dns.yaml | \
		DNS=$(call dns_ip) envsubst | \
		kubectl --context=$(CLUSTER_NAME) apply -f -

.PHONY: global-svc
global-svc:
ifndef REMOTE_ENDPOINT
	@echo "must include the remote endpoint." && exit 1
endif
	cat k8s/service-entry.yaml | envsubst | $(call kubectl) apply -f -

.PHONY: other-cluster
other-cluster:
	for i in $$(echo $(ALL_CLUSTERS) | sed "s/$(CLUSTER_NAME)//"); do \
		$(MAKE) global-svc CLUSTER_NAME=$$i REMOTE_ENDPOINT=$(call endpoint); \
	done

.PHONY: app
app:
	$(call kubectl) apply -f k8s/app.yaml

all: $(ALL_CLUSTERS)
	for i in $(ALL_CLUSTERS); do \
		$(MAKE) other-cluster CLUSTER_NAME=$$i; \
	done

$(ALL_CLUSTERS): %:
	$(MAKE) install app CLUSTER_NAME=$*

# ------------------------------------------------------------------------------
# Testing
# ------------------------------------------------------------------------------

TEST_URL ?= httpbin.test.global:8000/headers

.PHONY: test
test: FLAGS = -n test
test:
	$(call kubectl) exec $(call sleep_pod) -c sleep \
		-- curl -I $(TEST_URL)

# ------------------------------------------------------------------------------
# Cleanup
# ------------------------------------------------------------------------------

CLEAN_TARGETS = $(addprefix clean-,$(ALL_CLUSTERS))
.PHONY: $(CLEAN_TARGETS)

$(CLEAN_TARGETS): clean-%:
	$(MAKE) clean-k8s CLUSTER_NAME=$*

clean-k8s:
	$(call kubectl) delete -f tmp/init.yaml -f tmp/cp.yaml || true
	$(call kubectl) delete ns istio-system test || true
	kubectl --context=$(CLUSTER_NAME) delete -f k8s/dns.yaml || true

.PHONY: clean
clean: $(CLEAN_TARGETS)
	rm -rf tmp
