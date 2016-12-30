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

TAG?=0.9
PREFIX?=gcr.io/google_containers/nginx-ingress-controller
REPO_INFO?=$(shell git config --get remote.origin.url)
VERSION?=git-$(shell git rev-parse --short HEAD)
ARCH?=amd64
BASEIMAGE?=gcr.io/google_containers/nginx-slim-$(ARCH):0.12
TEMP_DIR:=$(shell mktemp -d)
QEMUVERSION=v2.7.0
GOLANG_VERSION=1.7

# Set QEMUARCH based on arch
ifeq ($(ARCH),arm)
	QEMUARCH=arm
endif
ifeq ($(ARCH),arm64)
	QEMUARCH=aarch64
endif
ifeq ($(ARCH),ppc64le)
	QEMUARCH=ppc64le
endif
ifeq ($(ARCH),s390x)
	QEMUARCH=s390x
endif

all: push
controller: controller.go
	CGO_ENABLED=0 GOARCH=$(ARCH) go build -a -installsuffix cgo -ldflags \
		"-s -w -X main.version=$(VERSION) -X main.gitRepo=$(REPO_INFO)" \
		-o nginx-ingress-controller

container:
	# Copy the whole directory to a temporary dir and set the base image
	cp -r ./* $(TEMP_DIR)
	cd $(TEMP_DIR) && sed -i "s|BASEIMAGE|$(BASEIMAGE)|g" Dockerfile
	cd $(TEMP_DIR) && sed -i "s|ARCH|$(QEMUARCH)|g" Dockerfile

	# Compile the binary inside a container for reproducible builds
	docker run -it -e GOARCH=$(ARCH) -v $(TEMP_DIR):/build golang:$(GOLANG_VERSION) /bin/bash -c "\
		CGO_ENABLED=0 go build -a -installsuffix cgo -ldflags \
			\"-s -w -X main.version=$(VERSION) -X main.gitRepo=$(REPO_INFO)\" \
			-o /build/nginx-ingress-controller"

ifeq ($(ARCH),amd64)
	# When building "normally" for amd64, remove the whole line, it has no part in the amd64 image
	cd $(TEMP_DIR) && sed -i "/CROSS_BUILD_/d" Dockerfile
else
	# When cross-building, only the placeholder "CROSS_BUILD_" should be removed
	# Register /usr/bin/qemu-ARCH-static as the handler for ARM binaries in the kernel
	docker run --rm --privileged multiarch/qemu-user-static:register --reset
	curl -sSL https://github.com/multiarch/qemu-user-static/releases/download/$(QEMUVERSION)/x86_64_qemu-$(QEMUARCH)-static.tar.gz | tar -xz -C $(TEMP_DIR)
	cd $(TEMP_DIR) && sed -i "s/CROSS_BUILD_//g" Dockerfile
endif
	docker build -t $(PREFIX)-$(ARCH):$(TAG) $(TEMP_DIR)

push: container
	gcloud docker push $(PREFIX)-$(ARCH):$(TAG)

