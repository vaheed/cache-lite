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

## 2) Ubuntu 24.04 LTS (APT)

Create `/etc/apt/sources.list.d/vaheed.sources`:

```deb822
Types: deb
URIs: https://ubuntu.repo.vaheed.net/ubuntu
Suites: noble noble-updates noble-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

Ubuntu 22.04 LTS uses `jammy jammy-updates jammy-security`.

Make sure the keyring exists, then disable duplicate default Ubuntu entries:

```bash
apt-get install -y ubuntu-keyring
[ -f /etc/apt/sources.list ] && cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%F-%H%M%S)
[ -f /etc/apt/sources.list ] && sed -i 's/^[[:space:]]*deb /# deb /g' /etc/apt/sources.list
[ -f /etc/apt/sources.list.d/ubuntu.sources ] && mv /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.disabled
apt clean
apt update
```

If `apt update` reports `File has unexpected size` or `Hash Sum mismatch`, the mirror cache has stale repository metadata. Fix the server-side Nginx metadata bypass in `01-single-host-nginx.md`, then run:

```bash
apt clean
rm -rf /var/lib/apt/lists/*
apt update
```

## 3) Debian 12 (APT)

Create `/etc/apt/sources.list.d/vaheed.sources`:

```deb822
Types: deb
URIs: https://debian.repo.vaheed.net/debian
Suites: bookworm bookworm-updates
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://debian.repo.vaheed.net/debian-security
Suites: bookworm-security
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

Make sure the keyring exists:

```bash
apt-get install -y debian-archive-keyring
apt clean
apt update
```

Do not use `ubuntu.repo.vaheed.net` or `ubuntu-archive-keyring.gpg` on Debian. `NO_PUBKEY 871920D1991BC93C` on Debian usually means Ubuntu sources were configured on a Debian host or the Ubuntu keyring path does not exist.

## 4) Rocky Linux / AlmaLinux (DNF/YUM)

Create `/etc/yum.repos.d/vaheed-rocky.repo` on Rocky Linux:

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

For AlmaLinux, use `alma.repo.vaheed.net/almalinux/$releasever/...` and the AlmaLinux GPG key shipped by the OS.

## 5) CentOS Stream (DNF/YUM)

Create `/etc/yum.repos.d/vaheed-centos-stream.repo`:

```ini
[vaheed-baseos]
name=vaheed centos stream baseos mirror
baseurl=https://centos.repo.vaheed.net/10-stream/BaseOS/$basearch/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-centosofficial
```

Use `9-stream` or `10-stream` to match the installed CentOS Stream release.

CentOS Linux 7 is EOL and does not have a Rocky-style `7/BaseOS` path. Use `vault.centos.org` for legacy CentOS 7 systems or migrate those clients to Rocky/Alma 8+.

## 6) Alpine (APK)

Set `/etc/apk/repositories`:

```text
https://alpine.repo.vaheed.net/alpine/v3.21/main
https://alpine.repo.vaheed.net/alpine/v3.21/community
```

Update:

```bash
apk update
```

## 7) Arch / Manjaro (Pacman)

Set `/etc/pacman.d/mirrorlist`:

```text
Server = https://arch.repo.vaheed.net/$repo/os/$arch
```

Update:

```bash
pacman -Syy
```

## 8) openSUSE (Zypper)

```bash
zypper ar -f https://opensuse.repo.vaheed.net/distribution/leap/15.6/repo/oss/ vaheed-oss
zypper ar -f https://opensuse.repo.vaheed.net/update/leap/15.6/oss/ vaheed-update
zypper ref
```

## 9) Docker

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

## 10) containerd / nerdctl

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

## 11) CRI-O / Podman

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

## 12) Kubernetes (k3s / RKE2) Registry Mirrors

For Kubernetes nodes that use containerd via k3s or RKE2, configure mirrors in `registries.yaml`.

k3s path:

- `/etc/rancher/k3s/registries.yaml`

RKE2 path:

- `/etc/rancher/rke2/registries.yaml`

Example:

```yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.repo.vaheed.net"
  gcr.io:
    endpoint:
      - "https://gcr.repo.vaheed.net"
  ghcr.io:
    endpoint:
      - "https://ghcr.repo.vaheed.net"
  quay.io:
    endpoint:
      - "https://quay.repo.vaheed.net"
  registry.k8s.io:
    endpoint:
      - "https://k8s.repo.vaheed.net"
  mcr.microsoft.com:
    endpoint:
      - "https://mcr.repo.vaheed.net"
```

Restart service after update:

```bash
# k3s
systemctl restart k3s

# RKE2
systemctl restart rke2-server
systemctl restart rke2-agent
```

## 13) Validation Commands

APT metadata:

```bash
curl -I https://ubuntu.repo.vaheed.net/ubuntu/dists/jammy/InRelease
curl -I https://ubuntu.repo.vaheed.net/ubuntu/dists/noble/InRelease
curl -I https://debian.repo.vaheed.net/debian/dists/bookworm/InRelease
curl -I https://debian.repo.vaheed.net/debian-security/dists/bookworm-security/InRelease
```

Docker registry endpoint:

```bash
curl -I https://docker.repo.vaheed.net/v2/
```

Expected for `/v2/`: `200` or `401` with `WWW-Authenticate`.

## 14) Common Issues

1. `apt update` shows duplicate target warnings:
Disable duplicate lines in `/etc/apt/sources.list` as shown in section 2.

2. `Hash Sum mismatch` or `File has unexpected size` in APT:
Fix server-side metadata caching first, then clear client cache and retry:

```bash
apt clean
rm -rf /var/lib/apt/lists/*
apt update
```

3. Docker pulls bypass mirror:
Confirm `daemon.json` is valid JSON and Docker was restarted.

4. `curl: (6) Could not resolve host`:
Fix DNS on the client or server before adding Docker repositories:

```bash
resolvectl status || cat /etc/resolv.conf
getent hosts download.docker.com
getent hosts ubuntu.repo.vaheed.net
```

5. `sudo: unable to resolve host`:
Add the machine hostname to `/etc/hosts` or set a normal hostname:

```bash
hostname
printf '127.0.1.1 %s\n' "$(hostname)" >> /etc/hosts
```

6. `sudo: Account or password is expired` while already root:
Do not use `sudo` as root. Repair the expired root password state if this is a template image:

```bash
passwd -u root
chage -d "$(date +%F)" root
```

## 15) Support Data to Provide

If a client has issues, provide:

```bash
cat /etc/os-release
ip -4 route get 1.1.1.1
getent hosts ubuntu.repo.vaheed.net debian.repo.vaheed.net docker.repo.vaheed.net
curl -I https://ubuntu.repo.vaheed.net/ubuntu/dists/noble/InRelease
curl -I https://debian.repo.vaheed.net/debian/dists/bookworm/InRelease
curl -I https://debian.repo.vaheed.net/debian-security/dists/bookworm-security/InRelease
curl -I https://docker.repo.vaheed.net/v2/
```
