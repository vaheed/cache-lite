# 00 - Architecture and Principles

## 1) Architecture Overview

This platform provides transparent mirror CDN behavior for package managers and container clients using direct repository endpoints on `*.repo.vaheed.net`.

- Clients are configured with repository base URLs and registry mirrors.
- No explicit `http_proxy` or forward-proxy mode is required.
- Edge cache does pull-through from upstreams and persists local cache.

Two supported patterns in this repo:

1. Single-host Nginx reverse cache + multiple official `registry:2` pull-through mirrors
2. Docker Swarm with Apache Traffic Server (ATS) edge cache + official `registry:2` mirrors

## 2) DNS Design and Host Layout

Create `A`/`AAAA` records to your public edge IP(s):

```dns
$ORIGIN repo.vaheed.net.
@                       3600 IN A      203.0.113.10
@                       3600 IN AAAA   2001:db8::10

ubuntu                  3600 IN CNAME  @
debian                  3600 IN CNAME  @
kali                    3600 IN CNAME  @
mint                    3600 IN CNAME  @
devuan                  3600 IN CNAME  @
alpine                  3600 IN CNAME  @
fedora                  3600 IN CNAME  @
rocky                   3600 IN CNAME  @
alma                    3600 IN CNAME  @
centos                  3600 IN CNAME  @
rhel                    3600 IN CNAME  @
opensuse                3600 IN CNAME  @
arch                    3600 IN CNAME  @
manjaro                 3600 IN CNAME  @

docker                  3600 IN CNAME  @
ghcr                    3600 IN CNAME  @
quay                    3600 IN CNAME  @
gcr                     3600 IN CNAME  @
k8s                     3600 IN CNAME  @
mcr                     3600 IN CNAME  @
oci                     3600 IN CNAME  @
status                  3600 IN CNAME  @
```

Optional staging hosts:

```dns
staging                 300 IN A       203.0.113.11
staging-ubuntu          300 IN CNAME   staging
staging-docker          300 IN CNAME   staging
```

## 3) TLS Certificate Strategy (Manual)

Supported key types:

- RSA 2048/3072 (`fullchain.pem`, `privkey.pem`)

Recommended:

- Use RSA certificate and key only.
- SAN wildcard cert: `*.repo.vaheed.net` + `repo.vaheed.net`.
- Use OCSP stapling resolver and refresh via scheduled deployment script.

## 4) Cache and Storage Principles

- Keep package cache and OCI cache on dedicated NVMe mount.
- Use XFS for large, long-lived cache partitions.
- Reserve 15-20% free space to avoid heavy eviction thrash.
- Separate logs from cache filesystem if possible.

Suggested capacity planning baseline:

- Package cache: 1-3 TB
- OCI cache: 2-8 TB
- Logs: 100-300 GB

## 5) Upstream and Failover Principles

- Primary upstreams mapped by distro/registry host.
- Use stale-while-revalidate where possible.
- Use backup mirrors in resolver/upstream definitions.
- Do not cache authentication tokens; cache immutable blobs aggressively.

## 6) Security Baseline

- Expose only 80/443 publicly.
- Protect admin metrics endpoints by source IP allowlist.
- Apply request rate limits and connection caps.
- Drop malformed methods and suspicious user agents.
- Run `registry:2` containers rootless where feasible and read-only except storage paths.

## 7) Anti-Abuse Controls

- Separate zones for package and OCI limits.
- Burst with delay for legitimate parallel fetch workloads.
- Per-path stricter limits for `/v2/*/blobs/uploads/`.
- Optional geo/IP ACL for institutional-only service.

## 8) Monitoring Baseline

Track:

- Nginx/ATS cache hit ratio
- Upstream latency and error rates
- Open file usage
- Disk usage/inodes
- Registry cache growth and GC outcomes

## 9) Backup and DR Principles

- Configuration files: daily backup + version control
- TLS material: encrypted offsite backup
- Registry metadata and cache: snapshot-based or cold backup
- RTO target example: <2h (single host), <30m (swarm)

## 10) Which Deployment to Use

- Use `01-single-host-nginx.md` for minimal complexity and highest operational simplicity.
- Use `02-swarm-ats.md` if you need service scheduling and multi-node edge behavior.
