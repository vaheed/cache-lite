# cache-lite

Production-grade transparent mirror CDN for Linux repositories and OCI registries under `*.repo.vaheed.net`.

```text
                         cache-lite
  ┌───────────────────────────────────────────────────────────────┐
  │ Clients                                                       │
  │ apt dnf yum apk pacman zypper docker containerd podman ...   │
  └───────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
  ┌───────────────────────────────────────────────────────────────┐
  │ Edge Layer                                                    │
  │ 1) Single host: Nginx reverse cache + TLS                    │
  │ 2) Swarm mode: ATS replicas + registry:2 mirrors             │
  └───────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
  ┌───────────────────────────────────────────────────────────────┐
  │ Upstreams                                                     │
  │ Linux repos: Ubuntu/Debian/Fedora/Arch/openSUSE/...          │
  │ OCI registries: Docker Hub/GHCR/Quay/GCR/K8s/MCR/custom OCI  │
  └───────────────────────────────────────────────────────────────┘
```

This repository now provides **two full deployment tracks**:

1. **Single host with Nginx + registry:2** (primary/recommended baseline)
2. **Docker Swarm with Apache Traffic Server (ATS) + registry:2** (scale-out variant)

Both are transparent mirror patterns where clients point directly to repository subdomains and do not use explicit HTTP proxy settings.

## Documentation Index

- [00-architecture-and-principles.md](./docs/00-architecture-and-principles.md)
- [01-single-host-nginx.md](./docs/01-single-host-nginx.md)
- [02-swarm-ats.md](./docs/02-swarm-ats.md)

## Domain Map (Reference)

Package mirrors:

- `ubuntu.repo.vaheed.net`
- `debian.repo.vaheed.net`
- `kali.repo.vaheed.net`
- `mint.repo.vaheed.net`
- `devuan.repo.vaheed.net`
- `alpine.repo.vaheed.net`
- `fedora.repo.vaheed.net`
- `rocky.repo.vaheed.net`
- `alma.repo.vaheed.net`
- `centos.repo.vaheed.net`
- `rhel.repo.vaheed.net`
- `opensuse.repo.vaheed.net`
- `arch.repo.vaheed.net`
- `manjaro.repo.vaheed.net`

OCI mirrors:

- `docker.repo.vaheed.net`
- `ghcr.repo.vaheed.net`
- `quay.repo.vaheed.net`
- `gcr.repo.vaheed.net`
- `k8s.repo.vaheed.net`
- `mcr.repo.vaheed.net`
- `oci.repo.vaheed.net` (generic BYO upstream registry endpoint)

## What Is Included

- Full DNS layout and sample zone
- Full filesystem layout
- Full Nginx and ATS config examples
- Full `registry:2` pull-through cache configs
- TLS (RSA manual deployment), OCSP, HSTS
- Kernel/sysctl and ulimit tuning
- Firewall and abuse controls
- Client-side mirror configuration examples for major Linux families
- Docker/containerd/CRI-O/Podman/nerdctl examples
- Operations runbook, monitoring, cache purge, backup, and DR

## Notes

- Start with [`01-single-host-nginx.md`](./docs/01-single-host-nginx.md) unless you already require clustered ingress and service scheduling.
- Use [`02-swarm-ats.md`](./docs/02-swarm-ats.md) when you need a Swarm-oriented edge design.
