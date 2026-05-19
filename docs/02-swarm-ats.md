# 02 - Swarm Mode with Apache Traffic Server (ATS) + registry:2

This is the scale-out variant requested: Docker Swarm scheduling with ATS edge cache and official `registry:2` pull-through mirrors.

## 1) Architecture Overview

- 3+ Swarm nodes recommended.
- ATS service runs as replicated edge cache.
- Registry mirrors run as replicated services with persistent volumes.
- TLS termination can be at ATS with mounted certs, or at an L4 LB in front.

## 2) DNS Design and Hosts

Use same public subdomains as single-host design. Point all records to Swarm ingress VIP or external LB IP.

## 3) Directory Layout

```text
/opt/repo-cdn-swarm/
├── stack.yml
├── ats/
│   ├── records.yaml
│   ├── remap.config
│   ├── storage.config
│   ├── volume.config
│   └── ssl_multicert.config
├── registry/
│   ├── dockerhub/config.yml
│   ├── ghcr/config.yml
│   ├── quay/config.yml
│   ├── gcr/config.yml
│   ├── k8s/config.yml
│   ├── mcr/config.yml
│   └── oci/config.yml
├── tls/
│   ├── fullchain.ecdsa.pem
│   ├── privkey.ecdsa.pem
│   ├── fullchain.rsa.pem
│   └── privkey.rsa.pem
└── scripts/
    ├── deploy-stack.sh
    ├── gc-all.sh
    └── rotate-certs.sh
```

## 4) ATS Global Config (`/opt/repo-cdn-swarm/ats/records.yaml`)

```yaml
CONFIG proxy.config.diags.debug.enabled INT 0
CONFIG proxy.config.http.server_ports STRING 80 443:ssl
CONFIG proxy.config.reverse_proxy.enabled INT 1
CONFIG proxy.config.url_remap.required INT 1
CONFIG proxy.config.http.cache.http INT 1
CONFIG proxy.config.cache.ram_cache.size INT 8589934592
CONFIG proxy.config.net.connections_throttle INT 250000
CONFIG proxy.config.http.keep_alive_enabled_in INT 1
CONFIG proxy.config.http.keep_alive_enabled_out INT 1
CONFIG proxy.config.http.keep_alive_no_activity_timeout_in INT 60
CONFIG proxy.config.http.keep_alive_no_activity_timeout_out INT 60
CONFIG proxy.config.http.transaction_active_timeout_in INT 600
CONFIG proxy.config.http.transaction_active_timeout_out INT 600
CONFIG proxy.config.ssl.server.cert.path STRING /etc/trafficserver/tls
CONFIG proxy.config.ssl.server.private_key.path STRING /etc/trafficserver/tls
CONFIG proxy.config.ssl.client.verify.server.policy STRING PERMISSIVE
CONFIG proxy.config.ssl.hsts_include_subdomains INT 1
CONFIG proxy.config.ssl.hsts_max_age INT 31536000
CONFIG proxy.config.http.response_server_str STRING repo-vaheed-edge
CONFIG proxy.config.exec_thread.autoconfig INT 1
CONFIG proxy.config.accept_threads INT 8
CONFIG proxy.config.thread.default.stacksize INT 1048576
```

## 5) Cache Configuration

`/opt/repo-cdn-swarm/ats/storage.config`

```text
/var/cache/trafficserver 500G
```

`/opt/repo-cdn-swarm/ats/volume.config`

```text
volume=1 scheme=http size=100%
```

## 6) Per-Distribution Mirror Mapping (`/opt/repo-cdn-swarm/ats/remap.config`)

```text
map https://ubuntu.repo.vaheed.net/ https://archive.ubuntu.com/
map https://debian.repo.vaheed.net/ https://deb.debian.org/
map https://kali.repo.vaheed.net/ https://http.kali.org/
map https://mint.repo.vaheed.net/ https://packages.linuxmint.com/
map https://devuan.repo.vaheed.net/ https://pkgmaster.devuan.org/
map https://alpine.repo.vaheed.net/ https://dl-cdn.alpinelinux.org/
map https://fedora.repo.vaheed.net/ https://dl.fedoraproject.org/
map https://rocky.repo.vaheed.net/ https://dl.rockylinux.org/
map https://alma.repo.vaheed.net/ https://repo.almalinux.org/
map https://centos.repo.vaheed.net/ https://mirror.stream.centos.org/
map https://rhel.repo.vaheed.net/ https://cdn.redhat.com/
map https://opensuse.repo.vaheed.net/ https://download.opensuse.org/
map https://arch.repo.vaheed.net/ https://geo.mirror.pkgbuild.com/
map https://manjaro.repo.vaheed.net/ https://repo.manjaro.org/

map https://docker.repo.vaheed.net/v2/ http://registry-dockerhub:5000/v2/
map https://ghcr.repo.vaheed.net/v2/   http://registry-ghcr:5000/v2/
map https://quay.repo.vaheed.net/v2/   http://registry-quay:5000/v2/
map https://gcr.repo.vaheed.net/v2/    http://registry-gcr:5000/v2/
map https://k8s.repo.vaheed.net/v2/    http://registry-k8s:5000/v2/
map https://mcr.repo.vaheed.net/v2/    http://registry-mcr:5000/v2/
map https://oci.repo.vaheed.net/v2/    http://registry-oci:5000/v2/
```

## 7) OCI Mirror Registry Configs

Template for each `/opt/repo-cdn-swarm/registry/*/config.yml`:

```yaml
version: 0.1
log:
  level: info
storage:
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
http:
  addr: :5000
proxy:
  remoteurl: https://registry-1.docker.io
```

Set upstream per service same as single-host version.

## 8) Swarm Stack (`/opt/repo-cdn-swarm/stack.yml`)

```yaml
version: "3.9"

services:
  ats:
    image: apache/trafficserver:10.0.1
    ports:
      - target: 80
        published: 80
        protocol: tcp
        mode: ingress
      - target: 443
        published: 443
        protocol: tcp
        mode: ingress
    deploy:
      replicas: 3
      placement:
        constraints: ["node.role==worker"]
      restart_policy:
        condition: any
    volumes:
      - /opt/repo-cdn-swarm/ats/records.yaml:/etc/trafficserver/records.yaml:ro
      - /opt/repo-cdn-swarm/ats/remap.config:/etc/trafficserver/remap.config:ro
      - /opt/repo-cdn-swarm/ats/storage.config:/etc/trafficserver/storage.config:ro
      - /opt/repo-cdn-swarm/ats/volume.config:/etc/trafficserver/volume.config:ro
      - /opt/repo-cdn-swarm/tls:/etc/trafficserver/tls:ro
      - ats-cache:/var/cache/trafficserver
    networks: [repo-net]

  registry-dockerhub:
    image: registry:2
    deploy: { replicas: 2 }
    volumes:
      - /opt/repo-cdn-swarm/registry/dockerhub/config.yml:/etc/docker/registry/config.yml:ro
      - reg-dockerhub:/var/lib/registry
    networks: [repo-net]

  registry-ghcr:
    image: registry:2
    deploy: { replicas: 2 }
    volumes:
      - /opt/repo-cdn-swarm/registry/ghcr/config.yml:/etc/docker/registry/config.yml:ro
      - reg-ghcr:/var/lib/registry
    networks: [repo-net]

  registry-quay:
    image: registry:2
    deploy: { replicas: 2 }
    volumes:
      - /opt/repo-cdn-swarm/registry/quay/config.yml:/etc/docker/registry/config.yml:ro
      - reg-quay:/var/lib/registry
    networks: [repo-net]

  registry-gcr:
    image: registry:2
    deploy: { replicas: 2 }
    volumes:
      - /opt/repo-cdn-swarm/registry/gcr/config.yml:/etc/docker/registry/config.yml:ro
      - reg-gcr:/var/lib/registry
    networks: [repo-net]

  registry-k8s:
    image: registry:2
    deploy: { replicas: 2 }
    volumes:
      - /opt/repo-cdn-swarm/registry/k8s/config.yml:/etc/docker/registry/config.yml:ro
      - reg-k8s:/var/lib/registry
    networks: [repo-net]

  registry-mcr:
    image: registry:2
    deploy: { replicas: 2 }
    volumes:
      - /opt/repo-cdn-swarm/registry/mcr/config.yml:/etc/docker/registry/config.yml:ro
      - reg-mcr:/var/lib/registry
    networks: [repo-net]

  registry-oci:
    image: registry:2
    deploy: { replicas: 2 }
    volumes:
      - /opt/repo-cdn-swarm/registry/oci/config.yml:/etc/docker/registry/config.yml:ro
      - reg-oci:/var/lib/registry
    networks: [repo-net]

networks:
  repo-net:
    driver: overlay

volumes:
  ats-cache:
  reg-dockerhub:
  reg-ghcr:
  reg-quay:
  reg-gcr:
  reg-k8s:
  reg-mcr:
  reg-oci:
```

## 9) TLS Setup

Use manual cert deployment in `/opt/repo-cdn-swarm/tls/`.

Create `/opt/repo-cdn-swarm/ats/ssl_multicert.config`:

```text
dest_ip=* ssl_cert_name=fullchain.ecdsa.pem ssl_key_name=privkey.ecdsa.pem
dest_ip=* ssl_cert_name=fullchain.rsa.pem ssl_key_name=privkey.rsa.pem
```

## 10) System Tuning

Apply the same sysctl and limits baseline from the Nginx version on all Swarm nodes.

## 11) Firewall Configuration

Open only:

- TCP 22 (management)
- TCP 80/443 (public)
- Swarm control/data plane between nodes (`2377`, `7946`, `4789`) restricted to cluster CIDR

## 12) Client Configuration Examples

Client configs are identical to the single-host document because endpoints stay on `*.repo.vaheed.net`.

## 13) Operational Procedures

Initialize swarm:

```bash
docker swarm init --advertise-addr <MANAGER_IP>
```

Deploy stack:

```bash
docker stack deploy -c /opt/repo-cdn-swarm/stack.yml repo-cdn
```

Verify:

```bash
docker stack services repo-cdn
docker service logs -f repo-cdn_ats
```

## 14) Monitoring Guidance

- ATS metrics with `traffic_ctl metric match proxy.process.http`.
- Docker service health and replica drift.
- Per-node disk/inode/network saturation.

## 15) Troubleshooting

- `503`: verify overlay network DNS and registry service names.
- TLS mismatch: check `ssl_multicert.config` names and mounted paths.
- low cache efficiency: ensure clients are using one canonical URL per distro/registry.

## 16) Scaling Considerations

- Increase ATS replicas horizontally.
- Pin registry services with placement constraints near storage-heavy nodes.
- Separate OCI heavy traffic into dedicated worker pool.

## 17) Security Hardening

- Lock manager API to mgmt VLAN.
- Use Docker secrets for upstream credentials when needed.
- Restrict inter-node communication via security groups/firewall zones.

## 18) Maintenance Procedures

- Rolling service updates with `--update-parallelism 1`.
- Quarterly Swarm version upgrade window.
- Scheduled registry garbage collection with node drain strategy.

## 19) Backup Strategy

- Backup `stack.yml`, ATS configs, registry configs, TLS files.
- Snapshot volume backends if using external volume drivers.

## 20) Disaster Recovery Guidance

1. Rebuild manager and workers.
2. Recreate overlay network and deploy stack.
3. Restore config + TLS + registry volumes.
4. Validate package and OCI endpoint health.
