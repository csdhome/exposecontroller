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

# Build the hyperkube image.
#
# Usage:
#   [ARCH=amd64] [REGISTRY="gcr.io/google_containers"] make (build|push) VERSION={some_version_number e.g. v1.2.0}

REGISTRY?=gcr.io/google_containers
ARCH?=amd64
TEMP_DIR:=$(shell mktemp -d)
CNI_RELEASE=c864f0e1ea73719b8f4582402b0847064f9883b0

UNAME_S:=$(shell uname -s)
ifeq ($(UNAME_S),Darwin)
	SED_CMD?=sed -i ""
endif
ifeq ($(UNAME_S),Linux)
	SED_CMD?=sed -i
endif

ifeq ($(ARCH),amd64)
	BASEIMAGE?=debian:jessie
endif
ifeq ($(ARCH),arm)
	BASEIMAGE?=armel/debian:jessie
	QEMUARCH=arm
endif
ifeq ($(ARCH),arm64)
	BASEIMAGE?=aarch64/debian:jessie
	QEMUARCH=aarch64
endif
ifeq ($(ARCH),ppc64le)
	BASEIMAGE?=ppc64le/debian:jessie
	QEMUARCH=ppc64le
endif

all: build

build:

ifndef VERSION
    $(error VERSION is undefined)
endif
	cp -r ./* ${TEMP_DIR}
	mkdir -p ${TEMP_DIR}/cni ${TEMP_DIR}/addons
	cp ../../saltbase/salt/helpers/safe_format_and_mount ${TEMP_DIR}
	cp ../../saltbase/salt/generate-cert/make-ca-cert.sh ${TEMP_DIR}
	cp ../../saltbase/salt/kube-dns/skydns-rc.yaml.base ${TEMP_DIR}/addons/skydns-rc.yaml
	cp ../../saltbase/salt/kube-dns/skydns-svc.yaml.base ${TEMP_DIR}/addons/skydns-svc.yaml
	cp ../../addons/dashboard/dashboard-controller.yaml ${TEMP_DIR}/addons
	cp ../../addons/dashboard/dashboard-service.yaml ${TEMP_DIR}/addons
	
	# TODO: Move out kube-proxy to a DaemonSet again
	#cp kube-proxy-ds.yaml ${TEMP_DIR}/addons/kube-proxy.yaml
	cp ../../../_output/dockerized/bin/linux/${ARCH}/hyperkube ${TEMP_DIR}

	cd ${TEMP_DIR} && sed -i.back "s|VERSION|${VERSION}|g" addons/*.yaml static-pods/*.json
	cd ${TEMP_DIR} && sed -i.back "s|REGISTRY|${REGISTRY}|g" addons/*.yaml static-pods/*.json
	cd ${TEMP_DIR} && sed -i.back "s|ARCH|${ARCH}|g" addons/*.yaml static-pods/*.json
	cd ${TEMP_DIR} && sed -i.back "s|ARCH|${QEMUARCH}|g" Dockerfile
	cd ${TEMP_DIR} && sed -i.back "s|BASEIMAGE|${BASEIMAGE}|g" Dockerfile
	cd ${TEMP_DIR} && sed -i.back "s|-amd64|-${ARCH}|g" addons/*.yaml
	cd ${TEMP_DIR} && sed -i.back "s|__PILLAR__DNS__REPLICAS__|1|g;s|__PILLAR__DNS__SERVER__|10.0.0.10|g;" addons/skydns*.yaml
	cd ${TEMP_DIR} && sed -i.back "s|__PILLAR__DNS__DOMAIN__|cluster.local|g;s|__PILLAR__FEDERATIONS__DOMAIN__MAP__||g;" addons/skydns*.yaml
	rm ${TEMP_DIR}/addons/*.back

	# Make scripts executable before they are copied into the Docker image. If we make them executable later, in another layer
	# they'll take up twice the space because the new executable binary differs from the old one, but everything is cached in layers.
	cd ${TEMP_DIR} && chmod a+rx \
		hyperkube \
		safe_format_and_mount \
		setup-files.sh \
		make-ca-cert.sh \
		copy-addons.sh

ifeq ($(ARCH),amd64)
	# When building "normally" for amd64, remove the whole line, it has no part in the amd64 image
	cd ${TEMP_DIR} && ${SED_CMD} "/CROSS_BUILD_/d" Dockerfile
	# Download CNI
	curl -sSL --retry 5 https://storage.googleapis.com/kubernetes-release/network-plugins/cni-${CNI_RELEASE}.tar.gz | tar -xz -C ${TEMP_DIR}/cni
else
	# When cross-building, only the placeholder "CROSS_BUILD_" should be removed
	# Register /usr/bin/qemu-ARCH-static as the handler for ARM binaries in the kernel
	docker run --rm --privileged multiarch/qemu-user-static:register --reset
	curl -sSL --retry 5 https://github.com/multiarch/qemu-user-static/releases/download/v2.5.0/x86_64_qemu-${QEMUARCH}-static.tar.xz | tar -xJ -C ${TEMP_DIR}
	cd ${TEMP_DIR} && ${SED_CMD} "s/CROSS_BUILD_//g" Dockerfile
endif

	docker build -t ${REGISTRY}/hyperkube-${ARCH}:${VERSION} ${TEMP_DIR}

push: build
	gcloud docker push ${REGISTRY}/hyperkube-${ARCH}:${VERSION}
ifeq ($(ARCH),amd64)
	docker tag -f ${REGISTRY}/hyperkube-${ARCH}:${VERSION} ${REGISTRY}/hyperkube:${VERSION}
	gcloud docker push ${REGISTRY}/hyperkube:${VERSION}
endif

.PHONY: all
