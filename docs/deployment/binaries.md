# Binaries and containers

When a push to `main` produces a release (semantic-release decides from the
commit messages), the `release-assets` CI job in `.github/workflows/ci.yml`
builds the three binaries for the new tag, packages them, and publishes these
release assets:

- `etcfuse-<version>-linux-amd64.tar.gz`, `etcfuse-meta-<version>-linux-amd64.tar.gz`,
  `etcfsctl-<version>-linux-amd64.tar.gz`
- `etcfuse_<version>_amd64.deb`, `etcfuse-meta_<version>_amd64.deb`, `etcfsctl_<version>_amd64.deb`
- `etcfuse-<version>-1.x86_64.rpm`, `etcfuse-meta-<version>-1.x86_64.rpm`, `etcfsctl-<version>-1.x86_64.rpm`
- `checksums.txt` (sha256, one line per artifact above)

It also pushes three container images, tagged `<version>` and `latest`:
`ghcr.io/etcfs/etcfuse`, `ghcr.io/etcfs/etcfuse-meta` and
`ghcr.io/etcfs/etcfs-csi` (the [CSI driver](kubernetes-csi.md)). `etcfsctl`
has no image, because it is a command-line client rather than a service.

## Install from a release

```bash
VERSION=$(curl -fsSL https://api.github.com/repos/etcfs/etcfs/releases/latest | jq -r .tag_name | tr -d v)

curl -fsSLO "https://github.com/etcfs/etcfs/releases/download/v${VERSION}/checksums.txt"
curl -fsSLO "https://github.com/etcfs/etcfs/releases/download/v${VERSION}/etcfuse-meta_${VERSION}_amd64.deb"
sha256sum --ignore-missing -c checksums.txt

sudo apt install ./etcfuse-meta_${VERSION}_amd64.deb
```

The `etcfuse` and `etcfuse-meta` packages install the binary to
`/usr/local/bin/`, install the matching systemd unit from `deploy/systemd/` to
`/usr/lib/systemd/system/`, and run `systemctl daemon-reload`. Neither package
enables or starts the service, because both need cluster-specific settings
first; see [Configuration](configuration.md).

The `etcfsctl` package contains only the binary.

RPM-based distros: `sudo dnf install ./etcfuse-meta-${VERSION}-1.x86_64.rpm`.

## Containers

```bash
docker pull ghcr.io/etcfs/etcfuse-meta:latest
docker pull ghcr.io/etcfs/etcfuse:latest
```

The images are built from `deploy/docker/Dockerfile.etcfuse-meta` and
`deploy/docker/Dockerfile.etcfuse`. Each is a multi-stage build: the Go daemon
compiles in a Go image and the C daemon in Amazon Linux 2023 with
`fuse3-devel`, and each binary is then copied into a minimal runtime image.

`deploy/docker/docker-compose.yml` builds these images locally instead of
pulling them. To run a released version with compose, replace each service's
`build:` block with an `image:` line naming the published tag.

## Package specs and local testing

Each binary has an [nfpm](https://nfpm.goreleaser.com) spec in
`packaging/nfpm/` (`etcfuse.yaml`, `etcfuse-meta.yaml`, `etcfsctl.yaml`). The
two daemon packages share `packaging/nfpm/scripts/postinstall.sh`, which only
reloads systemd's unit files.

Build a package locally without CI:

```bash
mkdir -p dist
CGO_ENABLED=0 go build -o dist/etcfuse-meta ./cmd/etcfuse-meta
VERSION=0.0.0-dev nfpm package -f packaging/nfpm/etcfuse-meta.yaml -p deb -t dist/
```

## Building from source

```bash
make all      # bin/etcfuse, bin/etcfuse-meta, bin/etcfsctl
make check    # lint + test — CI's entry point
```

The C daemon needs `libfuse3-dev` (Debian) or `fuse3-devel` (RPM). The Go
binaries are built with `CGO_ENABLED=0`, statically linked, and need nothing
else. The `Makefile` lists every target.
