# Copyright 2021 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# This Dockerfile is used to build various container images. There are a few
# targets:
#
# go-protoc: image with Golang, protoc and protoc-go pluggin
# base: image with Go modules download and protobuf built
# server: image for recon server
# test: image to run all the tests

# =========  protoc stage ==========

FROM golang:1.16 AS go-protoc
# Install protoc
RUN apt-get update && \
  apt-get upgrade -y && \
  apt-get install -y protobuf-compiler
# Install protobuf go plugin
ENV GO111MODULE=on
RUN go get google.golang.org/protobuf/cmd/protoc-gen-go@v1.23.0
RUN go get google.golang.org/grpc/cmd/protoc-gen-go-grpc@v0.0.0-20200824180931-410880dd7d91


# =========  base stage ==========

# Use pre-built go-protoc image from last step.
FROM gcr.io/datcom-ci/go-protoc:latest AS base
WORKDIR /recon
# Docker cache: Download modules
COPY go.mod go.sum /recon/
RUN go mod download -x

# Copy files
COPY proto/ proto
COPY internal/ internal
# Build protobuf
RUN protoc \
  --proto_path=proto \
  --go_out=internal \
  --go-grpc_out=internal \
  --go-grpc_opt=requireUnimplementedServers=false \
  proto/*.proto


# =========  server stage ==========

FROM base as server
WORKDIR /recon
# Adding the grpc_health_probe
RUN GRPC_HEALTH_PROBE_VERSION=v0.3.5 && \
  wget -qO/bin/grpc_health_probe https://github.com/grpc-ecosystem/grpc-health-probe/releases/download/${GRPC_HEALTH_PROBE_VERSION}/grpc_health_probe-linux-amd64 && \
  chmod +x /bin/grpc_health_probe
# Build binary
COPY cmd/ cmd
RUN go build -o /go/bin/recon cmd/main.go
ENTRYPOINT ["/go/bin/recon"]


# =========  test stage ==========

FROM base as test
WORKDIR /recon
# Turn off GO111MODULE to allow for golangci-lint to run.
ENV GO111MODULE=off
# Golint
RUN wget -O- -nv https://raw.githubusercontent.com/golangci/golangci-lint/master/install.sh | sh -s v1.41.1
RUN ./bin/golangci-lint run --timeout=600s
# Test
COPY test/ test
COPY deploy/ deploy
CMD ["go", "test", "./..."]


