# Copyright 2016 The Kubernetes Authors All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

VERSION ?= $(shell git describe --tags --always --dirty)
REGISTRY ?= gcr.io/k8s-staging-dns
ARCH ?= amd64
DNSMASQ_VERSION ?= dnsmasq-2.90
CONTAINER_PREFIX ?= k8s-dns

ALL_ARCH := amd64 arm arm64 ppc64le s390x
MANIFEST_IMAGE := $(REGISTRY)/$(CONTAINER_PREFIX)-dnsmasq
IMAGE := $(CONTAINER_PREFIX)-dnsmasq-$(ARCH)
COMPILE_IMAGE := registry.k8s.io/kube-cross:v1.7.6-k8s1.6-0
OUTPUT_DIR := _output/$(ARCH)

# Ensure that the docker command line supports the manifest images
export DOCKER_CLI_EXPERIMENTAL=enabled

# Multiarch image
# Uploaded: Mar 28, 2025
BASEIMAGE ?= gcr.io/distroless/base-debian12@sha256:27769871031f67460f1545a52dfacead6d18a9f197db77110cfc649ca2a91f44
ifeq ($(ARCH),amd64)
	COMPILE_IMAGE := registry.k8s.io/build-image/debian-base-$(ARCH):bookworm-v1.0.4
else ifeq ($(ARCH),arm)
	TRIPLE    ?= arm-linux-gnueabihf
	QEMUARCH  := arm
else ifeq ($(ARCH),arm64)
	TRIPLE    ?= aarch64-linux-gnu
	QEMUARCH  := aarch64
else ifeq ($(ARCH),ppc64le)
	TRIPLE    ?= powerpc64le-linux-gnu
	QEMUARCH  := ppc64le
else ifeq ($(ARCH),s390x)
	TRIPLE    ?= s390x-linux-gnu
	QEMUARCH  := s390x
else
$(error Unsupported ARCH: $(ARCH))
endif

DNSMASQ_URL := http://www.thekelleys.org.uk/dnsmasq/$(DNSMASQ_VERSION).tar.xz
# SHA-256 computed from GPG-verified download:
# $ gpg --recv-keys --keyserver keyring.debian.org D6EACBD6EE46B834248D111215CDDA6AE19135A2
# ...
# $ gpg --verify dnsmasq-2.90.tar.xz.asc dnsmasq-2.90.tar.xz
# $ gpg: Signature made Tue 13 Feb 2024 01:50:43 PM UTC
# $ gpg:                using RSA key D6EACBD6EE46B834248D111215CDDA6AE19135A2
# $ gpg: Good signature from "Simon Kelley <simon@thekelleys.org.uk>" [unknown]
# $ gpg:                 aka "Simon Kelley <srk@debian.org>" [unknown]
# $ gpg: WARNING: This key is not certified with a trusted signature!
# ...
# $ sha256sum dnsmasq-2.90.tar.xz
DNSMASQ_SHA256 := 8e50309bd837bfec9649a812e066c09b6988b73d749b7d293c06c57d46a109e4
DNSMASQ_ARCHIVE := $(OUTPUT_DIR)/dnsmasq.tar.xz

MULTIARCH_CONTAINER := multiarch/qemu-user-static:register
MULTIARCH_RELEASE := https://github.com/multiarch/qemu-user-static/releases/download/v5.2.0-11/x86_64_qemu-$(QEMUARCH)-static.tar.gz

PATCHES := $(shell find patches/ -name *.patch)
BINARY := $(OUTPUT_DIR)/docker/dnsmasq

BUILDSTAMP := $(subst :,_,$(subst /,_,$(REGISTRY))_$(IMAGE)_$(VERSION))
CONTAINER_STAMP := .$(BUILDSTAMP)-container
PUSH_STAMP := .$(BUILDSTAMP)-push

ifeq ($(VERBOSE), 1)
	VERBOSE_OUTPUT := >&1
else
	VERBOSE_OUTPUT := >/dev/null 2>/dev/null
endif

all: build

build-%:
	@$(MAKE) --no-print-directory ARCH=$* build

containers-%:
	@$(MAKE) --no-print-directory ARCH=$* containers

test-%:
	@$(MAKE) --no-print-directory ARCH=$* test

push-%:
	@$(MAKE) --no-print-directory ARCH=$* push

.PHONY: all-build
all-build: $(addprefix build-, $(ALL_ARCH))

.PHONY: all-containers
all-containers: $(addprefix containers-, $(ALL_ARCH))

.PHONY: all-test
all-test: $(addprefix test-, $(ALL_ARCH))

.PHONY: all-push
all-push: $(addprefix push-, $(ALL_ARCH))
	docker manifest create --amend $(MANIFEST_IMAGE):$(VERSION) $(shell echo $(ALL_ARCH) | sed -e "s~[^ ]*~$(MANIFEST_IMAGE)\-&:$(VERSION)~g")
	@for arch in $(ALL_ARCH); do docker manifest annotate --arch $${arch} ${MANIFEST_IMAGE}:${VERSION} ${MANIFEST_IMAGE}-$${arch}:${VERSION}; done
	docker manifest push --purge ${MANIFEST_IMAGE}:${VERSION}

.PHONY: build
build: $(BINARY)

$(BINARY): Dockerfile.cross $(DNSMASQ_ARCHIVE) $(PATCHES) dnsmasq.conf
	@echo "building :" $(BINARY)
	@mkdir -p $(@D)
	@mkdir -p $(@D)/docker
	@cp Dockerfile.cross $(OUTPUT_DIR)/docker/Dockerfile
	@cd $(OUTPUT_DIR)/docker && sed -i- "s|__BASEIMAGE__|$(BASEIMAGE)|g" Dockerfile && rm Dockerfile-
	@cd $(OUTPUT_DIR) && tar -xJf dnsmasq.tar.xz
	@for patch in $(PATCHES); do patch -p0 -d $(OUTPUT_DIR)/$(DNSMASQ_VERSION) < $${patch}; done
	@cp dnsmasq.conf $(OUTPUT_DIR)/docker

ifeq ($(ARCH),amd64)
	@cd $(OUTPUT_DIR)/docker && \
		sed -i- "/__CROSS_BUILD_COPY__/d" Dockerfile && rm Dockerfile-
	@docker pull $(COMPILE_IMAGE)
	@docker run --rm --sig-proxy=true \
		-v `pwd`/$(OUTPUT_DIR):/build \
		-v `pwd`:/src                 \
		$(COMPILE_IMAGE)              \
		/bin/sh -c                    \
		"apt-get update                           \
			&& apt-get -y install build-essential \
			&& cd /build/$(DNSMASQ_VERSION)     \
			&& make -j                          \
			&& cp src/dnsmasq /build/docker/dnsmasq" $(VERBOSE_OUTPUT)
else
	@cd $(OUTPUT_DIR)/docker                                                 \
		&& sed -i- "s|__ARCH__|$(QEMUARCH)|g" Dockerfile && rm Dockerfile-     \
		&& sed -i- "s/__CROSS_BUILD_COPY__/COPY/g" Dockerfile && rm Dockerfile-
	@cd $(OUTPUT_DIR)/docker \
		&& curl -sSL $(MULTIARCH_RELEASE) | tar -xz
	@docker run --sig-proxy=true --rm      \
		--privileged $(MULTIARCH_CONTAINER)  \
		--reset $(VERBOSE_OUTPUT)
	@docker pull $(COMPILE_IMAGE)
	@docker run --rm --sig-proxy=true \
		-v `pwd`/$(OUTPUT_DIR):/build   \
		-v `pwd`:/src                   \
		$(COMPILE_IMAGE)                \
		/bin/bash -c                    \
		"cd /build/$(DNSMASQ_VERSION)   \
			&& CC=$(TRIPLE)-gcc make -j   \
			&& cp src/dnsmasq /build/docker/dnsmasq" $(VERBOSE_OUTPUT)
endif

$(DNSMASQ_ARCHIVE):
	@mkdir -p $(@D)
	@curl -sSL $(DNSMASQ_URL) > $@
	@echo "$(DNSMASQ_SHA256)  $(DNSMASQ_ARCHIVE)" > $(@D)/sha256.txt
	@sha256sum -c $(@D)/sha256.txt

.PHONY: containers
containers: $(CONTAINER_STAMP)

$(CONTAINER_STAMP): $(BINARY)
	@echo "container:" $(REGISTRY)/$(IMAGE):$(VERSION)
	@docker buildx build --pull --load --platform linux/$(ARCH) \
		-q -t $(REGISTRY)/$(IMAGE):$(VERSION) $(OUTPUT_DIR)/docker > $@

.PHONY: test
test: containers
	@ARCH=$(ARCH) IMAGE=$(REGISTRY)/$(IMAGE):$(VERSION) ./validate.sh


.PHONY: push
push: $(PUSH_STAMP)

$(PUSH_STAMP): $(CONTAINER_STAMP)
	@echo "pushing  :" $(REGISTRY)/$(IMAGE):$(VERSION)
	@docker push $(REGISTRY)/$(IMAGE):$(VERSION)
	@cat $< > $@

.PHONY: clean
clean:
	rm -f .*-container
	rm -f .*-push
	rm -rf _output/

.PHONY: version
version:
	@echo $(VERSION)
