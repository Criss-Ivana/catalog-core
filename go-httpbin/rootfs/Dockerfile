# SPDX-License-Identifier: BSD-3-Clause
# Matches catalog library/httpbingo/2.13.4/Dockerfile: CGO static-pie PIE with no PT_INTERP,
# which Unikraft's ELF loader requires (Go's CGO_ENABLED=0 -buildmode=pie leaves /lib64/ld-linux).

FROM golang:1.21-bookworm AS build

ARG VERSION=2.13.4

WORKDIR /src

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        ca-certificates \
        git \
        gcc \
        libc6-dev \
    && git clone --depth=1 --branch "v${VERSION}" https://github.com/mccutchen/go-httpbin.git . \
    && CGO_ENABLED=1 go build \
        -buildmode=pie \
        -ldflags="-linkmode external -extldflags -static-pie" \
        -tags netgo \
        -trimpath \
        -o /out/go-httpbin \
        ./cmd/go-httpbin \
    && apt-get purge -y --auto-remove git gcc libc6-dev \
    && rm -rf /var/lib/apt/lists/*

FROM scratch
COPY --from=build /out/go-httpbin /go-httpbin
