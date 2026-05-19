# 01 - Single Host with Nginx + registry:2

This document is a full step-by-step production deployment for Ubuntu Server LTS.

## 1) Architecture Overview

- One Linux host runs:
  - Nginx as TLS terminator + reverse proxy cache for package repositories and registry frontends
  - Multiple official `registry:2` containers in pull-through cache mode (one per upstream registry)
- Public endpoints:
  - Package repos: `*.repo.vaheed.net`
  - OCI mirrors: `docker.repo.vaheed.net`, `ghcr.repo.vaheed.net`, `quay.repo.vaheed.net`, `gcr.repo.vaheed.net`, `k8s.repo.vaheed.net`, `mcr.repo.vaheed.net`, `oci.repo.vaheed.net`

## 2) DNS Design and Hosts File

Use the DNS zone from `00-architecture-and-principles.md`.

Host `/etc/hosts` example for validation:

```hosts
203.0.113.10 ubuntu.repo.vaheed.net debian.repo.vaheed.net alpine.repo.vaheed.net arch.repo.vaheed.net
203.0.113.10 fedora.repo.vaheed.net rocky.repo.vaheed.net opensuse.repo.vaheed.net docker.repo.vaheed.net
```

## 3) Directory Layout

```text
/opt/repo-cdn/
├── compose/
│   └── docker-compose.yml
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
│   ├── privkey.rsa.pem
│   └── ca-chain.pem
├── scripts/
│   ├── deploy.sh
│   ├── cache-purge.sh
│   └── registry-gc.sh
└── backups/

/etc/nginx/
├── nginx.conf
├── conf.d/
│   ├── maps.conf
│   ├── package-mirrors.conf
│   ├── oci-frontends.conf
│   └── status.conf
└── snippets/
    ├── tls.conf
    ├── proxy-common.conf
    └── security-headers.conf

/var/cache/repo-cdn/
├── pkg/
└── oci/

/var/lib/repo-cdn/registry/
├── dockerhub/
├── ghcr/
├── quay/
├── gcr/
├── k8s/
├── mcr/
└── oci/
```

## 4) Nginx Global Config (`/etc/nginx/nginx.conf`)

```nginx
user www-data;
worker_processes auto;
pid /run/nginx.pid;
worker_rlimit_nofile 1000000;

events {
    worker_connections 65535;
    use epoll;
    multi_accept on;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    aio threads;
    directio 8m;
    keepalive_timeout 65;
    keepalive_requests 100000;
    reset_timedout_connection on;

    server_tokens off;
    types_hash_max_size 4096;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" rt=$request_time uct=$upstream_connect_time '
                    'uht=$upstream_header_time urt=$upstream_response_time cache=$upstream_cache_status';

    access_log /var/log/nginx/access.log main;
    error_log /var/log/nginx/error.log warn;

    open_file_cache max=500000 inactive=60s;
    open_file_cache_valid 120s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    proxy_cache_path /var/cache/repo-cdn/pkg levels=1:2 keys_zone=pkg_cache:1024m max_size=3t inactive=45d use_temp_path=off;
    proxy_cache_path /var/cache/repo-cdn/oci levels=1:2 keys_zone=oci_cache:1024m max_size=6t inactive=90d use_temp_path=off;

    proxy_temp_path /var/cache/repo-cdn/tmp 1 2;

    proxy_connect_timeout 15s;
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;

    resolver 1.1.1.1 1.0.0.1 8.8.8.8 9.9.9.9 valid=300s ipv6=on;
    resolver_timeout 5s;

    limit_req_zone $binary_remote_addr zone=pkg_req_zone:50m rate=120r/s;
    limit_req_zone $binary_remote_addr zone=oci_req_zone:50m rate=80r/s;
    limit_conn_zone $binary_remote_addr zone=addr_conn_zone:50m;

    map $request_method $method_ok {
        default 0;
        GET 1;
        HEAD 1;
        OPTIONS 1;
    }

    map $http_user_agent $bad_ua {
        default 0;
        ~*(sqlmap|nikto|masscan|zgrab|nmap|acunetix) 1;
    }

    include /etc/nginx/conf.d/*.conf;
}
```

## 5) Cache Configuration and Shared Snippets

`/etc/nginx/snippets/tls.conf`

```nginx
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:100m;
ssl_session_timeout 1d;
ssl_session_tickets off;

ssl_certificate     /opt/repo-cdn/tls/fullchain.ecdsa.pem;
ssl_certificate_key /opt/repo-cdn/tls/privkey.ecdsa.pem;
ssl_certificate     /opt/repo-cdn/tls/fullchain.rsa.pem;
ssl_certificate_key /opt/repo-cdn/tls/privkey.rsa.pem;

ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /opt/repo-cdn/tls/ca-chain.pem;

add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
```

`/etc/nginx/snippets/proxy-common.conf`

```nginx
proxy_http_version 1.1;
proxy_set_header Connection "";
proxy_set_header Host $proxy_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_buffering on;
proxy_request_buffering on;
proxy_buffers 64 256k;
proxy_busy_buffers_size 512k;
proxy_max_temp_file_size 0;
proxy_cache_lock on;
proxy_cache_lock_age 15s;
proxy_cache_lock_timeout 20s;
proxy_cache_revalidate on;
proxy_cache_background_update on;
proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
slice 1m;
proxy_set_header Range $slice_range;
proxy_cache_key "$scheme$proxy_host$request_uri$slice_range";
add_header X-Cache-Status $upstream_cache_status always;
```

`/etc/nginx/snippets/security-headers.conf`

```nginx
add_header X-Frame-Options DENY always;
add_header X-Content-Type-Options nosniff always;
add_header Referrer-Policy no-referrer always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
```

## 6) Per-Distribution Mirror Configs (`/etc/nginx/conf.d/package-mirrors.conf`)

```nginx
server {
    listen 80;
    server_name ~^(?<repo>[a-z0-9-]+)\.repo\.vaheed\.ir$;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name ubuntu.repo.vaheed.net debian.repo.vaheed.net kali.repo.vaheed.net mint.repo.vaheed.net devuan.repo.vaheed.net alpine.repo.vaheed.net fedora.repo.vaheed.net rocky.repo.vaheed.net alma.repo.vaheed.net centos.repo.vaheed.net rhel.repo.vaheed.net opensuse.repo.vaheed.net arch.repo.vaheed.net manjaro.repo.vaheed.net;

    include /etc/nginx/snippets/tls.conf;
    include /etc/nginx/snippets/security-headers.conf;

    if ($method_ok = 0) { return 405; }
    if ($bad_ua = 1) { return 403; }

    limit_req zone=pkg_req_zone burst=800 nodelay;
    limit_conn addr_conn_zone 200;

    set $pkg_upstream "";

    if ($host = ubuntu.repo.vaheed.net)   { set $pkg_upstream archive.ubuntu.com; }
    if ($host = debian.repo.vaheed.net)   { set $pkg_upstream deb.debian.org; }
    if ($host = kali.repo.vaheed.net)     { set $pkg_upstream http.kali.org; }
    if ($host = mint.repo.vaheed.net)     { set $pkg_upstream packages.linuxmint.com; }
    if ($host = devuan.repo.vaheed.net)   { set $pkg_upstream pkgmaster.devuan.org; }
    if ($host = alpine.repo.vaheed.net)   { set $pkg_upstream dl-cdn.alpinelinux.org; }
    if ($host = fedora.repo.vaheed.net)   { set $pkg_upstream dl.fedoraproject.org; }
    if ($host = rocky.repo.vaheed.net)    { set $pkg_upstream dl.rockylinux.org; }
    if ($host = alma.repo.vaheed.net)     { set $pkg_upstream repo.almalinux.org; }
    if ($host = centos.repo.vaheed.net)   { set $pkg_upstream mirror.stream.centos.org; }
    if ($host = rhel.repo.vaheed.net)     { set $pkg_upstream cdn.redhat.com; }
    if ($host = opensuse.repo.vaheed.net) { set $pkg_upstream download.opensuse.org; }
    if ($host = arch.repo.vaheed.net)     { set $pkg_upstream geo.mirror.pkgbuild.com; }
    if ($host = manjaro.repo.vaheed.net)  { set $pkg_upstream repo.manjaro.org; }

    location / {
        include /etc/nginx/snippets/proxy-common.conf;
        proxy_cache pkg_cache;
        proxy_cache_valid 200 301 302 45d;
        proxy_cache_valid 404 5m;
        proxy_pass https://$pkg_upstream$request_uri;
    }
}
```

## 7) OCI Registry Frontend Config (`/etc/nginx/conf.d/oci-frontends.conf`)

```nginx
upstream reg_dockerhub { server 127.0.0.1:5001; keepalive 128; }
upstream reg_ghcr      { server 127.0.0.1:5002; keepalive 128; }
upstream reg_quay      { server 127.0.0.1:5003; keepalive 128; }
upstream reg_gcr       { server 127.0.0.1:5004; keepalive 128; }
upstream reg_k8s       { server 127.0.0.1:5005; keepalive 128; }
upstream reg_mcr       { server 127.0.0.1:5006; keepalive 128; }
upstream reg_oci       { server 127.0.0.1:5007; keepalive 128; }

server {
    listen 443 ssl http2;
    server_name docker.repo.vaheed.net ghcr.repo.vaheed.net quay.repo.vaheed.net gcr.repo.vaheed.net k8s.repo.vaheed.net mcr.repo.vaheed.net oci.repo.vaheed.net;

    client_max_body_size 0;
    include /etc/nginx/snippets/tls.conf;
    include /etc/nginx/snippets/security-headers.conf;

    limit_req zone=oci_req_zone burst=500 nodelay;
    limit_conn addr_conn_zone 120;

    location /v2/ {
        include /etc/nginx/snippets/proxy-common.conf;
        proxy_request_buffering off;
        proxy_buffering off;

        if ($host = docker.repo.vaheed.net) { proxy_pass http://reg_dockerhub; }
        if ($host = ghcr.repo.vaheed.net)   { proxy_pass http://reg_ghcr; }
        if ($host = quay.repo.vaheed.net)   { proxy_pass http://reg_quay; }
        if ($host = gcr.repo.vaheed.net)    { proxy_pass http://reg_gcr; }
        if ($host = k8s.repo.vaheed.net)    { proxy_pass http://reg_k8s; }
        if ($host = mcr.repo.vaheed.net)    { proxy_pass http://reg_mcr; }
        if ($host = oci.repo.vaheed.net)    { proxy_pass http://reg_oci; }
    }
}
```

## 8) Docker Compose (`/opt/repo-cdn/compose/docker-compose.yml`)

```yaml
services:
  registry-dockerhub:
    image: registry:2
    container_name: registry-dockerhub
    restart: unless-stopped
    network_mode: bridge
    ports: ["127.0.0.1:5001:5000"]
    volumes:
      - /opt/repo-cdn/registry/dockerhub/config.yml:/etc/docker/registry/config.yml:ro
      - /var/lib/repo-cdn/registry/dockerhub:/var/lib/registry

  registry-ghcr:
    image: registry:2
    container_name: registry-ghcr
    restart: unless-stopped
    ports: ["127.0.0.1:5002:5000"]
    volumes:
      - /opt/repo-cdn/registry/ghcr/config.yml:/etc/docker/registry/config.yml:ro
      - /var/lib/repo-cdn/registry/ghcr:/var/lib/registry

  registry-quay:
    image: registry:2
    container_name: registry-quay
    restart: unless-stopped
    ports: ["127.0.0.1:5003:5000"]
    volumes:
      - /opt/repo-cdn/registry/quay/config.yml:/etc/docker/registry/config.yml:ro
      - /var/lib/repo-cdn/registry/quay:/var/lib/registry

  registry-gcr:
    image: registry:2
    container_name: registry-gcr
    restart: unless-stopped
    ports: ["127.0.0.1:5004:5000"]
    volumes:
      - /opt/repo-cdn/registry/gcr/config.yml:/etc/docker/registry/config.yml:ro
      - /var/lib/repo-cdn/registry/gcr:/var/lib/registry

  registry-k8s:
    image: registry:2
    container_name: registry-k8s
    restart: unless-stopped
    ports: ["127.0.0.1:5005:5000"]
    volumes:
      - /opt/repo-cdn/registry/k8s/config.yml:/etc/docker/registry/config.yml:ro
      - /var/lib/repo-cdn/registry/k8s:/var/lib/registry

  registry-mcr:
    image: registry:2
    container_name: registry-mcr
    restart: unless-stopped
    ports: ["127.0.0.1:5006:5000"]
    volumes:
      - /opt/repo-cdn/registry/mcr/config.yml:/etc/docker/registry/config.yml:ro
      - /var/lib/repo-cdn/registry/mcr:/var/lib/registry

  registry-oci:
    image: registry:2
    container_name: registry-oci
    restart: unless-stopped
    ports: ["127.0.0.1:5007:5000"]
    volumes:
      - /opt/repo-cdn/registry/oci/config.yml:/etc/docker/registry/config.yml:ro
      - /var/lib/repo-cdn/registry/oci:/var/lib/registry
```

## 9) Registry Config Files (`/opt/repo-cdn/registry/*/config.yml`)

Use this template and only change `remoteurl`.

```yaml
version: 0.1
log:
  level: info
storage:
  cache:
    blobdescriptor: inmemory
  filesystem:
    rootdirectory: /var/lib/registry
  delete:
    enabled: true
http:
  addr: :5000
  headers:
    X-Content-Type-Options: [nosniff]
proxy:
  remoteurl: https://registry-1.docker.io
```

Per service `remoteurl`:

- dockerhub: `https://registry-1.docker.io`
- ghcr: `https://ghcr.io`
- quay: `https://quay.io`
- gcr: `https://gcr.io`
- k8s: `https://registry.k8s.io`
- mcr: `https://mcr.microsoft.com`
- oci: `https://REGISTRY_FQDN` (set your own)

## 10) TLS Setup Steps

1. Place certs under `/opt/repo-cdn/tls/`.
2. Verify:

```bash
openssl x509 -in /opt/repo-cdn/tls/fullchain.ecdsa.pem -noout -text | head
openssl rsa -in /opt/repo-cdn/tls/privkey.rsa.pem -check -noout
```

3. Test Nginx and reload:

```bash
nginx -t && systemctl reload nginx
```

Optional mTLS snippet (restricted paths only):

```nginx
ssl_client_certificate /opt/repo-cdn/tls/client-ca.pem;
ssl_verify_client optional;
```

## 11) System Tuning

`/etc/sysctl.d/99-repo-cdn.conf`

```conf
fs.file-max = 2097152
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 250000
net.core.rmem_max = 33554432
net.core.wmem_max = 33554432
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.ip_local_port_range = 10240 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
vm.swappiness = 10
vm.max_map_count = 262144
```

Apply:

```bash
sysctl --system
```

`/etc/security/limits.d/repo-cdn.conf`

```conf
* soft nofile 1000000
* hard nofile 1000000
root soft nofile 1000000
root hard nofile 1000000
```

## 12) Firewall Configuration (`nftables`)

`/etc/nftables.conf`

```nft
table inet filter {
  chain input {
    type filter hook input priority 0;
    policy drop;

    iif lo accept
    ct state established,related accept

    tcp dport {22,80,443} accept
    ip protocol icmp accept
    ip6 nexthdr ipv6-icmp accept

    counter drop
  }

  chain forward {
    type filter hook forward priority 0;
    policy drop;
  }

  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}
```

## 13) Client Configuration Examples

APT (`/etc/apt/sources.list.d/vaheed.sources`):

```deb822
Types: deb
URIs: https://ubuntu.repo.vaheed.net/ubuntu
Suites: noble noble-updates noble-security
Components: main universe restricted multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

DNF/YUM (`/etc/yum.repos.d/vaheed.repo`):

```ini
[vaheed-baseos]
name=VAHEED BaseOS
baseurl=https://rocky.repo.vaheed.net/pub/rocky/$releasever/BaseOS/$basearch/os/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-rockyofficial
```

APK (`/etc/apk/repositories`):

```text
https://alpine.repo.vaheed.net/alpine/v3.21/main
https://alpine.repo.vaheed.net/alpine/v3.21/community
```

Pacman (`/etc/pacman.d/mirrorlist`):

```text
Server = https://arch.repo.vaheed.net/$repo/os/$arch
```

Zypper:

```bash
zypper ar -f https://opensuse.repo.vaheed.net/distribution/leap/15.6/repo/oss/ vaheed-oss
zypper ar -f https://opensuse.repo.vaheed.net/update/leap/15.6/oss/ vaheed-update
```

Docker (`/etc/docker/daemon.json`):

```json
{
  "registry-mirrors": ["https://docker.repo.vaheed.net"],
  "max-concurrent-downloads": 20
}
```

containerd (`/etc/containerd/certs.d/docker.io/hosts.toml`):

```toml
server = "https://docker.io"
[host."https://docker.repo.vaheed.net"]
  capabilities = ["pull", "resolve"]
```

nerdctl uses containerd `certs.d` entries above.

CRI-O (`/etc/containers/registries.conf.d/10-vaheed.conf`):

```toml
[[registry]]
prefix = "docker.io"
location = "docker.repo.vaheed.net"
insecure = false
blocked = false
```

Podman (`/etc/containers/registries.conf`):

```toml
unqualified-search-registries = ["docker.repo.vaheed.net", "ghcr.repo.vaheed.net"]
[[registry]]
prefix = "docker.io"
location = "docker.repo.vaheed.net"
```

## 14) Operational Procedures

Deploy:

```bash
mkdir -p /opt/repo-cdn/{compose,registry/{dockerhub,ghcr,quay,gcr,k8s,mcr,oci},tls,scripts,backups}
mkdir -p /var/cache/repo-cdn/{pkg,oci,tmp}
mkdir -p /var/lib/repo-cdn/registry/{dockerhub,ghcr,quay,gcr,k8s,mcr,oci}

docker compose -f /opt/repo-cdn/compose/docker-compose.yml up -d
nginx -t && systemctl enable --now nginx
```

Optional systemd unit (`/etc/systemd/system/repo-registry.service`):

```ini
[Unit]
Description=Repo CDN Registry Mirrors
After=docker.service network-online.target
Requires=docker.service

[Service]
Type=oneshot
WorkingDirectory=/opt/repo-cdn/compose
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
RemainAfterExit=yes
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```

Cache purge by host/path:

```bash
#!/usr/bin/env bash
set -euo pipefail
KEY="$1"
echo "$KEY" | xargs -I{} curl -s -X PURGE "https://status.repo.vaheed.net/purge/{}"
```

Registry GC weekly:

```bash
docker exec registry-dockerhub registry garbage-collect /etc/docker/registry/config.yml --delete-untagged
```

Logrotate (`/etc/logrotate.d/repo-cdn-nginx`):

```conf
/var/log/nginx/*.log {
    daily
    rotate 30
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        [ -s /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid)
    endscript
}
```

Fail2ban (`/etc/fail2ban/jail.d/repo-cdn.conf`):

```ini
[nginx-http-auth]
enabled = true

[nginx-botsearch]
enabled = true
port = http,https
logpath = /var/log/nginx/access.log
maxretry = 10
findtime = 600
bantime = 3600
```

## 15) Monitoring Guidance

- Nginx log scrape: cache hit/miss ratio, upstream errors, latency.
- Node exporter for disk/inode/network.
- Container health probes for each registry service.

Fast health checks:

```bash
curl -I https://ubuntu.repo.vaheed.net/ubuntu/dists/noble/Release
curl -s https://docker.repo.vaheed.net/v2/
```

## 16) Troubleshooting

- `502` on OCI path: check container port binding and local firewall.
- poor hit ratio: verify client URLs are consistent and avoid query-string variance.
- TLS chain error: rebuild `ca-chain.pem` and reload.
- disk full: reduce `inactive` TTL and run GC.

## 17) Scaling Considerations

- Add second host behind anycast or DNS round-robin.
- Shard heavy domains (`docker1`, `docker2`) if one cache volume saturates.
- Separate package and OCI traffic to distinct NVMe pools.
- Keep backup upstream hostname mappings ready, then switch map entries and hot-reload Nginx during upstream outages.

## 18) Security Hardening

- Run Docker daemon with user namespace remap.
- Set registry containers `read_only: true` except storage mount.
- Restrict SSH to mgmt CIDRs.
- Add fail2ban jail for repeated 403/429 abuse patterns.

## 19) Maintenance Procedures

Weekly:

1. Verify cert expiry window >30 days
2. Verify disk usage <80%
3. Run registry GC during low traffic
4. Review top 4xx/5xx patterns

Monthly:

1. Update host OS
2. Update `registry:2` image
3. Reload Nginx after staged config tests

## 20) Backup and DR Guidance

Backup set:

- `/etc/nginx`
- `/opt/repo-cdn/registry`
- `/opt/repo-cdn/tls` (encrypted)
- `/opt/repo-cdn/compose/docker-compose.yml`

Disaster recovery procedure:

1. Provision new host
2. Restore config + TLS + compose files
3. Start registry containers
4. Start Nginx and validate endpoints
5. Restore cache data optionally; platform can warm from upstream automatically
