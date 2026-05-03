# Build rootfs/go-httpbin (Linux amd64 static PIE) via Docker by default.
IMAGE ?= go-httpbin-catalog-core-rootfs

.PHONY: all clean

all: go-httpbin

go-httpbin: Dockerfile
	-docker rm -f go-httpbin-catalog-core-extract >/dev/null 2>&1 || true
	docker build -t $(IMAGE) .
	docker create --name go-httpbin-catalog-core-extract $(IMAGE) /go-httpbin
	docker cp go-httpbin-catalog-core-extract:/go-httpbin ./go-httpbin
	docker rm -f go-httpbin-catalog-core-extract
	chmod +x go-httpbin

clean:
	rm -f go-httpbin
