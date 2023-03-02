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

# We use Alpine as the source for default CA certificates
ARG ALPINE_IMAGE=alpine:3.15.4

ARG COMPONENT=barbican-kms-plugin

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

# barbican-kms-plugin uses ALPINE instead of distroless because its entrypoint
# uses a shell for environment substitution. If there are no other uses this
# could be replaced by callers passing arguments explicitly.

FROM --platform=${TARGETPLATFORM} ${ALPINE_IMAGE}

ARG COMPONENT

COPY --from=builder /build/$COMPONENT /bin/$COMPONENT
COPY --from=certs /etc/ssl/certs /etc/ssl/certs

# Fill out the labels
LABEL name="barbican-kms-plugin" \
      license="Apache Version 2.0" \
      maintainers="Kubernetes Authors" \
      description="Barbican kms plugin" \
      distribution-scope="public" \
      summary="Barbican kms plugin" \
      help="none"

CMD ["sh", "-c", "/bin/barbican-kms-plugin --socketpath ${socketpath} --cloud-config ${cloudconfig}"]
