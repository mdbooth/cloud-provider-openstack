# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

################################################################################
##                               BUILD ARGS                                   ##
################################################################################
# This build arg allows the specification of a custom Golang image.
ARG GOLANG_IMAGE=golang:1.19.6

# The distroless image on which the CPI manager image is built.
#
# Please do not use "latest". Explicit tags should be used to provide
# deterministic builds. Follow what kubernetes uses to build
# kube-controller-manager, for example for 1.23.x:
# https://github.com/kubernetes/kubernetes/blob/release-1.24/build/common.sh#L94
ARG DISTROLESS_IMAGE=k8s.gcr.io/build-image/go-runner:v2.3.1-go1.19.6-bullseye.0

# We use Alpine as the source for default CA certificates
ARG ALPINE_IMAGE=alpine:3.15.4

ARG COMPONENT=openstack-cloud-controller-manager

################################################################################
##                              BUILD STAGE                                   ##
################################################################################
# Fetch ca-certificates from Alpine
FROM ${ALPINE_IMAGE} as certs
RUN apk add --no-cache ca-certificates

# Build the manager as a statically compiled binary so it has no dependencies
# libc, muscl, etc.
FROM --platform=linux/amd64 ${GOLANG_IMAGE} as builder

ARG GOPROXY=https://goproxy.io,direct
ARG TARGETOS
ARG TARGETARCH
ARG VERSION
ARG COMPONENT

WORKDIR /build
COPY go.mod go.sum ./
COPY cmd/ cmd/
COPY pkg/ pkg/
RUN GO111MODULE=on CGO_ENABLED=0 GOOS=${TARGETOS} GOARCH=${TARGETARCH} GOPROXY=${GOPROXY} \
                go build \
                -trimpath \
                -ldflags="-w -s -X k8s.io/component-base/version.gitVersion=${VERSION}" \
                -o=$COMPONENT \
                cmd/$COMPONENT/main.go

################################################################################
##                               MAIN STAGE                                   ##
################################################################################
FROM --platform=${TARGETPLATFORM} ${DISTROLESS_IMAGE}

ARG COMPONENT

COPY --from=builder /build/$COMPONENT /bin/$COMPONENT
COPY --from=certs /etc/ssl/certs /etc/ssl/certs

# Fill out the labels
LABEL name="openstack-cloud-controller-manager" \
      license="Apache Version 2.0" \
      maintainers="Kubernetes Authors" \
      description="OpenStack cloud controller manager" \
      distribution-scope="public" \
      summary="OpenStack cloud controller manager" \
      help="none"

CMD [ "/bin/openstack-cloud-controller-manager" ]
