ANTITHESIS_ROOT :=$(dir $(realpath $(lastword $(MAKEFILE_LIST))))
REPOSITORY_ROOT := $(shell git rev-parse --show-toplevel)
USER_ID := $(shell id -u)
GROUP_ID := $(shell id -g)
ARCH ?= $(shell go env GOARCH)
REF = main
IMAGE_TAG = latest

.PHONY: antithesis-build-client-image
antithesis-build-client-image:
	# This is a release image using go build without our tooling. Using the
	# version defined in .go-version is the correct image tag.
	docker build \
		--build-arg GO_IMAGE_TAG=$(shell cat $(REPOSITORY_ROOT)/.go-version) \
		--file $(ANTITHESIS_ROOT)/test-template/Dockerfile \
		--tag etcd-client:latest \
		$(REPOSITORY_ROOT)

.PHONY: antithesis-build-etcd-image
antithesis-build-etcd-image:
	# Use the Workspace Go version from the go directive. This is the base image
	# later in the Dockerfile, we'll set the correct toolchain.
	docker build \
		--build-arg GO_IMAGE_TAG=$(shell go work edit -json | jq .Go) \
		--build-arg REF=$(REF) \
		--tag etcd-server:latest \
		-f ./server/Dockerfile \
		$(ANTITHESIS_ROOT)

.PHONY: antithesis-build-etcd-image-release-3.4
antithesis-build-etcd-image-release-3.4: REF=release-3.4
antithesis-build-etcd-image-release-3.4: antithesis-build-etcd-image

.PHONY: antithesis-build-etcd-image-release-3.5
antithesis-build-etcd-image-release-3.5: REF=release-3.5
antithesis-build-etcd-image-release-3.5: antithesis-build-etcd-image

.PHONY: antithesis-build-etcd-image-release-3.6
antithesis-build-etcd-image-release-3.6: REF=release-3.6
antithesis-build-etcd-image-release-3.6: antithesis-build-etcd-image

.PHONY: antithesis-build-etcd-image-main
antithesis-build-etcd-image-main: REF=main
antithesis-build-etcd-image-main: antithesis-build-etcd-image

# docker
.PHONY: antithesis-build-docker-config-image
antithesis-build-docker-config-image:
	docker build -f config/docker/Dockerfile config/docker -t etcd-config:latest \
		--build-arg IMAGE_TAG=$(IMAGE_TAG)

.PHONY: antithesis-docker-up
antithesis-docker-up:
	export USER_ID=$(USER_ID) && export GROUP_ID=$(GROUP_ID) && \
		docker compose -f config/docker/docker-composes/docker-compose.yaml up

.PHONY: antithesis-docker-wait-healthy
antithesis-docker-wait-healthy:
	@timeout=120; interval=10; \
	compose_file=config/docker/docker-composes/docker-compose.yaml; \
	end=$$(( $$(date +%s) + timeout )); \
	while [ $$(date +%s) -lt $$end ]; do \
		if docker compose -f $$compose_file logs client 2>/dev/null | grep -q 'cluster is healthy!'; then \
			echo "Cluster is healthy!"; \
			exit 0; \
		fi; \
		echo "Waiting for cluster to become healthy..."; \
		sleep $$interval; \
	done; \
	echo "Cluster did not become healthy in $$timeout seconds."; \
	docker compose -f $$compose_file logs; \
	exit 1

.PHONY: antithesis-run-docker-traffic
antithesis-run-docker-traffic:
	export USER_ID=$(USER_ID) && export GROUP_ID=$(GROUP_ID) && \
		docker compose -f config/docker/docker-composes/docker-compose.yaml exec client /opt/antithesis/test/v1/robustness/singleton_driver_traffic

.PHONY: antithesis-run-docker-validation
antithesis-run-docker-validation:
	export USER_ID=$(USER_ID) && export GROUP_ID=$(GROUP_ID) && \
		docker compose -f config/docker/docker-composes/docker-compose.yaml exec client /opt/antithesis/test/v1/robustness/finally_validation

.PHONY: antithesis-docker-clean
antithesis-docker-clean:
	export USER_id=$(USER_ID) && export GROUP_ID=$(GROUP_ID) && \
		docker compose -f config/docker/docker-composes/docker-compose.yaml down --remove-orphans
	rm -rf /tmp/etcddata0 /tmp/etcddata1 /tmp/etcddata2 /tmp/etcdreport

# kubernetes
.PHONY: antithesis-build-k8s-config-image
antithesis-build-k8s-config-image:
	docker build -f config/k8s/Dockerfile config/k8s -t etcd-config:latest \
		--build-arg IMAGE_TAG=$(IMAGE_TAG)

.PHONY: antithesis-k8s-up
antithesis-k8s-up: check-k8s-tools
	kind create cluster --name etcd-antithesis
	kind load docker-image etcd-client:latest --name etcd-antithesis
	kind load docker-image etcd-server:latest --name etcd-antithesis
	kubectl apply -f config/k8s/manifests

.PHONY: antithesis-k8s-wait-healthy
antithesis-k8s-wait-healthy:
	@timeout=120; interval=10; \
	kubectl wait --for=condition=ready pod -l app=etcd --timeout=$${timeout}s && \
	kubectl wait --for=condition=ready pod -l app=etcd-client --timeout=$${timeout}s && \
	end=$$(( $$(date +%s) + timeout )); \
	while [ $$(date +%s) -lt $$end ]; do \
		if kubectl logs deployment/etcd-client 2>/dev/null | grep -q 'cluster is healthy!'; then \
			echo "Cluster is healthy!"; \
			exit 0; \
		fi; \
		echo "Waiting for cluster to become healthy..."; \
		sleep $$interval; \
	done; \
	echo "Cluster did not become healthy in $$timeout seconds."; \
	kubectl get pods; \
	kubectl logs deployment/etcd-client; \
	exit 1

.PHONY: antithesis-run-k8s-traffic
antithesis-run-k8s-traffic:
	kubectl exec deployment/etcd-client -- /opt/antithesis/test/v1/robustness/singleton_driver_traffic

.PHONY: antithesis-run-k8s-validation
antithesis-run-k8s-validation:
	kubectl exec deployment/etcd-client -- /opt/antithesis/test/v1/robustness/finally_validation

.PHONY: antithesis-k8s-clean
antithesis-k8s-clean: check-k8s-tools
	kind delete cluster --name etcd-antithesis
	rm -rf /tmp/etcddata /tmp/etcdreport


.PHONY: antithesis-run-local-traffic
antithesis-run-local-traffic:
	export ETCD_ROBUSTNESS_DATA_PATHS=/tmp/etcddata/etcd-0,/tmp/etcddata/etcd-1,/tmp/etcddata/etcd-2 && export ETCD_ROBUSTNESS_REPORT_PATH=reports/report && export ETCD_ROBUSTNESS_ENDPOINTS=127.0.0.1:12379,127.0.0.1:22379,127.0.0.1:32379 && \
		go run --race ./test-template/robustness/traffic/main.go

.PHONY: antithesis-run-local-validation
antithesis-run-local-validation:
	export ETCD_ROBUSTNESS_DATA_PATHS=/tmp/etcddata/etcd-0,/tmp/etcddata/etcd-1,/tmp/etcddata/etcd-2 && export ETCD_ROBUSTNESS_REPORT_PATH=reports/report && export ETCD_ROBUSTNESS_ENDPOINTS=127.0.0.1:12379,127.0.0.1:22379,127.0.0.1:32379 && \
		go run --race ./test-template/robustness/finally/main.go

REQUIRED_K8S_TOOLS := kubectl kind

.PHONY: check-k8s-tools
check-k8s-tools:
	@for tool in $(REQUIRED_K8S_TOOLS); do \
		if ! command -v $$tool >/dev/null 2>&1; then \
			echo "Error: '$$tool' is missing.  Please install it before continuing." >&2; \
			exit 1; \
		fi \
	done
