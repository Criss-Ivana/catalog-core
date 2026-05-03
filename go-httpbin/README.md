# Unikraft ELF Loader (Networking): go-httpbin

Build and run the [Unikraft ELF Loader](https://github.com/unikraft/app-elfloader) with networking support, loading a Linux ELF build of [mccutchen/go-httpbin](https://github.com/mccutchen/go-httpbin) v2.13.4 (same upstream version as catalog `library/httpbingo/2.13.4`).

The ELF loader uses the [Unikraft binary-compatibility layer](https://unikraft.org/docs/concepts/compatibility) to run a native Linux binary from an initrd. The default build matches catalog **httpbingo**: **CGO static-pie** (`-buildmode=pie` with `-static-pie`), which is **PIE** and has **no** `PT_INTERP` (plain **`CGO_ENABLED=0` + `-buildmode=pie`** would still request **`/lib64/ld-linux-x86-64.so.2`**, which the initrd does not include). The server listens on **port 8080** (default for `go-httpbin`), matching `kraft run -p 8080:8080` in the catalog README.

Follow the instructions below to set up, configure, build and run. Install the [requirements](../README.md#requirements). The ELF loader targets x86_64 on KVM (QEMU and Firecracker) in line with `elfloader-net`.

## Quick setup

Run the [top-level `setup.sh`](../setup.sh) once, then from this directory:

```console
chmod +x setup.sh
./setup.sh
make distclean
wget -O /tmp/defconfig https://raw.githubusercontent.com/unikraft/catalog-core/refs/heads/scripts/elfloader-net/scripts/defconfig/qemu.x86_64
UK_DEFCONFIG=/tmp/defconfig make defconfig
make -j $(nproc)
sudo ip link set dev virbr0 down
sudo ip link del dev virbr0
sudo ip link add dev virbr0 type bridge
sudo ip address add 172.44.0.1/24 dev virbr0
sudo ip link set dev virbr0 up
make -C rootfs/
test -f initrd.cpio || ./workdir/unikraft/support/scripts/mkcpio initrd.cpio ./rootfs/
sudo qemu-system-x86_64 \
    -nographic \
    -m 256 \
    -cpu max \
    -netdev bridge,id=en0,br=virbr0 -device virtio-net-pci,netdev=en0 \
    -append "elfloader_qemu-x86_64 netdev.ip=172.44.0.2/24:172.44.0.1::: vfs.fstab=[ \"initrd0:/:extract::ramfs=1:\" ] -- /go-httpbin" \
    -kernel workdir/build/elfloader_qemu-x86_64 \
    -initrd ./initrd.cpio
```

For QEMU bridged networking prerequisites, see the [top-level README](../README.md#qemu) (`/etc/qemu/bridge.conf`).

**Memory:** Firecracker is configured with **256 MiB** (`fc.x86_64.json`); use **`-m 256`** for QEMU so behavior matches. Raise both if the Go runtime reports out-of-memory during bring-up.

## Build the Linux ELF (rootfs)

**Default:** `make -C rootfs/` runs Docker with the same flags as [catalog `library/httpbingo/2.13.4/Dockerfile`](https://github.com/unikraft/catalog/blob/staging/library/httpbingo/2.13.4/Dockerfile): **`CGO_ENABLED=1`**, **`-buildmode=pie`**, **`-ldflags="-linkmode external -extldflags -static-pie"`**, **`-tags netgo`**, producing one static-pie ELF at **`rootfs/go-httpbin`** → **`/go-httpbin`** in the guest.

**Host build (same as default):** needs a C toolchain (`gcc`, `libc6-dev`) on Linux amd64:

```console
git clone --depth=1 --branch v2.13.4 https://github.com/mccutchen/go-httpbin.git /tmp/go-httpbin-src
cd /tmp/go-httpbin-src
CGO_ENABLED=1 go build -buildmode=pie -ldflags="-linkmode external -extldflags -static-pie" -tags netgo -trimpath -o go-httpbin ./cmd/go-httpbin
install -m755 go-httpbin /path/to/catalog-core/go-httpbin/rootfs/go-httpbin
```

## Configure, build kernel, initrd

Same flow as [elfloader-net](../elfloader-net/README.md): `make menuconfig`, `make -j $(nproc)`, then pack the initrd:

```console
rm -f initrd.cpio
./workdir/unikraft/support/scripts/mkcpio initrd.cpio ./rootfs/
```

## Run on QEMU/x86_64

Bridge setup (same pattern as `elfloader-net`):

```console
sudo ip link set dev virbr0 down
sudo ip link del dev virbr0
sudo ip link set dev tap0 down
sudo ip link del dev tap0
sudo ip link add dev virbr0 type bridge
sudo ip address add 172.44.0.1/24 dev virbr0
sudo ip link set dev virbr0 up
```

```console
sudo qemu-system-x86_64 \
    -nographic \
    -m 256 \
    -cpu max \
    -netdev bridge,id=en0,br=virbr0 -device virtio-net-pci,netdev=en0 \
    -append "elfloader_qemu-x86_64 netdev.ip=172.44.0.2/24:172.44.0.1::: vfs.fstab=[ \"initrd0:/:extract::ramfs=1:\" ] -- /go-httpbin" \
    -kernel workdir/build/elfloader_qemu-x86_64 \
    -initrd ./initrd.cpio
```

## Run on Firecracker/x86_64

Tap setup:

```console
sudo ip link set dev virbr0 down
sudo ip link del dev virbr0
sudo ip link set dev tap0 down
sudo ip link del dev tap0
sudo ip tuntap add dev tap0 mode tap
sudo ip address add 172.44.0.1/24 dev tap0
sudo ip link set dev tap0 up
```

```console
rm -f firecracker.socket
firecracker-x86_64 --config-file fc.x86_64.json --api-sock firecracker.socket
```

The user must have KVM access (e.g. `kvm` group).

## Test

From the host after the guest has booted and configured **172.44.0.2** (same checks as catalog):

```console
curl -sS http://172.44.0.2:8080/ | head
curl -I http://172.44.0.2:8080/status/418
```

Expect HTML from `/` and **`HTTP/1.1 418 I'm a teapot`** (or equivalent) for `/status/418`.

## Close the VM

The ELF loader does **not** auto-shutdown while `go-httpbin` keeps serving: the VM stays running until you stop it explicitly (same as `elfloader-net`).

### QEMU

Use **`Ctrl+a`** then **`x`** in the QEMU console.

### Firecracker

In another terminal:

```console
sudo pkill -f firecracker
```

## Clean

```console
make clean
make properclean
make distclean
```

`make -C rootfs/ clean` removes the extracted `rootfs/go-httpbin` binary.

## Customize

To run a different binary path or arguments, change the segment after `--` in `-append` / `fc.x86_64.json` `boot_args`, and place the ELF under `rootfs/` at the matching path.
