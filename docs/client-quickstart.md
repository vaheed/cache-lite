# cache-lite Client Quickstart

This guide is for client machines that should use the mirror CDN endpoints at `*.repo.vaheed.net`.

## 1) Base Endpoints

Package mirrors:

- `https://ubuntu.repo.vaheed.net/ubuntu`
- `https://debian.repo.vaheed.net/debian`
- `https://alpine.repo.vaheed.net/alpine`
- `https://arch.repo.vaheed.net`
- `https://fedora.repo.vaheed.net`
- `https://rocky.repo.vaheed.net`
- `https://alma.repo.vaheed.net`
- `https://centos.repo.vaheed.net`
- `https://opensuse.repo.vaheed.net`

OCI mirrors:

- `https://docker.repo.vaheed.net`
- `https://ghcr.repo.vaheed.net`
- `https://quay.repo.vaheed.net`
- `https://gcr.repo.vaheed.net`
- `https://k8s.repo.vaheed.net`
- `https://mcr.repo.vaheed.net`
- `https://oci.repo.vaheed.net`

## 2) Ubuntu / Debian (APT)

Create `/etc/apt/sources.list.d/vaheed.sources`:

```deb822
Types: deb
URIs: https://ubuntu.repo.vaheed.net/ubuntu
Suites: jammy jammy-updates jammy-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

Disable duplicate default `deb` lines in `/etc/apt/sources.list`:

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%F-%H%M%S)
sed -i 's/^[[:space:]]*deb /# deb /g' /etc/apt/sources.list
apt clean
apt update
```

## 3) Fedora / Rocky / Alma / CentOS Stream (DNF/YUM)

Create `/etc/yum.repos.d/vaheed.repo`:

```ini
[vaheed-baseos]
name=vaheed baseos mirror
baseurl=https://rocky.repo.vaheed.net/pub/rocky/$releasever/BaseOS/$basearch/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rockyofficial
```

Refresh:

```bash
dnf clean all
dnf makecache
```

## 4) Alpine (APK)

Set `/etc/apk/repositories`:

```text
https://alpine.repo.vaheed.net/alpine/v3.21/main
https://alpine.repo.vaheed.net/alpine/v3.21/community
```

Update:

```bash
apk update
```

## 5) Arch / Manjaro (Pacman)

Set `/etc/pacman.d/mirrorlist`:

```text
Server = https://arch.repo.vaheed.net/$repo/os/$arch
```

Update:

```bash
pacman -Syy
```

## 6) openSUSE (Zypper)

```bash
zypper ar -f https://opensuse.repo.vaheed.net/distribution/leap/15.6/repo/oss/ vaheed-oss
zypper ar -f https://opensuse.repo.vaheed.net/update/leap/15.6/oss/ vaheed-update
zypper ref
```

## 7) Docker

Create or edit `/etc/docker/daemon.json`:

```json
{
  "registry-mirrors": ["https://docker.repo.vaheed.net"],
  "max-concurrent-downloads": 20
}
```

Restart Docker:

```bash
systemctl restart docker
```

Test:

```bash
docker pull alpine:latest
```

## 8) containerd / nerdctl

Create `/etc/containerd/certs.d/docker.io/hosts.toml`:

```toml
server = "https://docker.io"
[host."https://docker.repo.vaheed.net"]
  capabilities = ["pull", "resolve"]
```

Restart:

```bash
systemctl restart containerd
```

Test:

```bash
nerdctl pull docker.io/library/alpine:latest
```

## 9) CRI-O / Podman

Create `/etc/containers/registries.conf.d/10-vaheed.conf`:

```toml
[[registry]]
prefix = "docker.io"
location = "docker.repo.vaheed.net"
insecure = false
blocked = false
```

Podman test:

```bash
podman pull docker.io/library/alpine:latest
```

## 10) Validation Commands

APT metadata:

```bash
curl -I https://ubuntu.repo.vaheed.net/ubuntu/dists/jammy/InRelease
```

Docker registry endpoint:

```bash
curl -I https://docker.repo.vaheed.net/v2/
```

Expected for `/v2/`: `200` or `401` with `WWW-Authenticate`.

## 11) Common Issues

1. `apt update` shows duplicate target warnings:
Disable duplicate lines in `/etc/apt/sources.list` as shown in section 2.

2. `Hash Sum mismatch` in APT:
Clear client cache and retry:

```bash
apt clean
rm -rf /var/lib/apt/lists/*
apt update
```

3. Docker pulls bypass mirror:
Confirm `daemon.json` is valid JSON and Docker was restarted.

## 12) Support Data to Provide

If a client has issues, provide:

```bash
cat /etc/os-release
curl -I https://ubuntu.repo.vaheed.net/ubuntu/dists/jammy/InRelease
curl -I https://docker.repo.vaheed.net/v2/
```
