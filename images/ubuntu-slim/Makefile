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

# Build the ubuntu-slim image for amd64, arm, arm64, ppc64le and s390x
#
# Usage:
# 	[PREFIX=gcr.io/google_containers/ubuntu-slim] [ARCH=amd64] [TAG=0.6] make (container|push)

TAG?=0.6
PREFIX?=gcr.io/google_containers/ubuntu-slim
BUILD_IMAGE?=ubuntu-slim-build
TAR_FILE?=rootfs.tar
ARCH?=amd64
TEMP_DIR:=$(shell mktemp -d)
QEMUVERSION=v2.7.0

# Set default base image dynamically for each arch
ifeq ($(ARCH),amd64)
	BASEIMAGE?=ubuntu:16.04
endif
ifeq ($(ARCH),arm)
	BASEIMAGE?=armhf/ubuntu:16.04
	QEMUARCH=arm
endif
ifeq ($(ARCH),arm64)
	BASEIMAGE?=aarch64/ubuntu:16.04
	QEMUARCH=aarch64
endif
ifeq ($(ARCH),ppc64le)
	BASEIMAGE?=ppc64le/ubuntu:16.04
	QEMUARCH=ppc64le
endif
ifeq ($(ARCH),s390x)
	BASEIMAGE?=s390x/ubuntu:16.04
	QEMUARCH=s390x
endif

all: push

clean:
	#docker rmi -f $(BUILD_IMAGE) || true
	docker rm -f $(BUILD_IMAGE) || true

container: clean
	# Copy the whole directory to a temporary dir and set the base image
	cp -r ./* $(TEMP_DIR)
	cd $(TEMP_DIR) && sed -i "s|BASEIMAGE|$(BASEIMAGE)|g" Dockerfile.build
	cd $(TEMP_DIR) && sed -i "s|ARCH|$(QEMUARCH)|g" Dockerfile.build

ifeq ($(ARCH),amd64)
	# When building "normally" for amd64, remove the whole line, it has no part in the amd64 image
	cd $(TEMP_DIR) && sed -i "/CROSS_BUILD_/d" Dockerfile.build
else
	# When cross-building, only the placeholder "CROSS_BUILD_" should be removed
	# Register /usr/bin/qemu-ARCH-static as the handler for ARM binaries in the kernel
	docker run --rm --privileged multiarch/qemu-user-static:register --reset
	curl -sSL https://github.com/multiarch/qemu-user-static/releases/download/$(QEMUVERSION)/x86_64_qemu-$(QEMUARCH)-static.tar.gz | tar -xz -C $(TEMP_DIR)
	cd $(TEMP_DIR) && sed -i "s/CROSS_BUILD_//g" Dockerfile.build
endif

	docker build -t $(BUILD_IMAGE) -f $(TEMP_DIR)/Dockerfile.build $(TEMP_DIR)
	docker create --name $(BUILD_IMAGE) $(BUILD_IMAGE)
	docker export $(BUILD_IMAGE) > $(TEMP_DIR)/$(TAR_FILE)
	docker build -t $(PREFIX)-$(ARCH):$(TAG) $(TEMP_DIR)

	#rm -f $(TEMP_DIR)

push: container
	gcloud docker push $(PREFIX)-$(ARCH):$(TAG)

