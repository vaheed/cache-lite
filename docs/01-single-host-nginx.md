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
│   ├── fullchain.pem
│   ├── privkey.pem
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

    proxy_cache_path /var/cache/repo-cdn/pkg levels=1:2 keys_zone=pkg_cache:1024m max_size=3072g inactive=45d use_temp_path=off;
    proxy_cache_path /var/cache/repo-cdn/oci levels=1:2 keys_zone=oci_cache:1024m max_size=6144g inactive=90d use_temp_path=off;

    proxy_temp_path /var/cache/repo-cdn/tmp 1 2;

    proxy_connect_timeout 15s;
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;

    resolver 1.1.1.1 1.0.0.1 8.8.8.8 9.9.9.9 valid=300s ipv6=off;
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

ssl_certificate     /opt/repo-cdn/tls/fullchain.pem;
ssl_certificate_key /opt/repo-cdn/tls/privkey.pem;

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
proxy_ssl_server_name on;
proxy_ssl_name $proxy_host;
proxy_buffers 64 256k;
proxy_busy_buffers_size 512k;
proxy_max_temp_file_size 0;
proxy_cache_lock on;
proxy_cache_lock_age 15s;
proxy_cache_lock_timeout 20s;
proxy_cache_revalidate on;
proxy_cache_background_update on;
proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
proxy_cache_key "$scheme$proxy_host$request_uri";
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
    server_name ~^(?<repo>[a-z0-9-]+)\.repo\.vaheed\.net$;
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
        proxy_cache_valid 200 206 301 302 45d;
        proxy_cache_valid 404 5m;
        proxy_ignore_headers Set-Cookie Cache-Control Expires;
        proxy_hide_header Set-Cookie;
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
        slice 1m;
        proxy_set_header Range $slice_range;
        proxy_cache_key "$scheme$proxy_host$request_uri$slice_range";

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
openssl x509 -in /opt/repo-cdn/tls/fullchain.pem -noout -text | head
openssl rsa -in /opt/repo-cdn/tls/privkey.pem -check -noout
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

Disable default Ubuntu entries to avoid duplicate targets:

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%F)
sed -i 's/^[[:space:]]*deb /# deb /g' /etc/apt/sources.list
apt update
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

Deploy full observability stack for metrics + logs + dashboards.

Monitoring components:

- `prometheus`: metrics storage and alert evaluation
- `grafana`: dashboards and activity views
- `loki`: log backend
- `promtail`: log shipping from Nginx/system logs
- `node-exporter`: host metrics (CPU, RAM, disk, inode, network)
- `cadvisor`: container metrics
- `nginx-prometheus-exporter`: Nginx traffic/connection metrics

Create monitoring directories:

```bash
mkdir -p /opt/repo-cdn/monitoring/{prometheus,loki,promtail,grafana/provisioning/datasources,grafana/provisioning/dashboards,grafana/dashboards}
```

Add Nginx stub status endpoint (`/etc/nginx/conf.d/status.conf`):

```nginx
server {
    listen 127.0.0.1:8080;
    server_name 127.0.0.1;

    location /nginx_status {
        stub_status;
        allow 127.0.0.1;
        deny all;
    }
}
```

Monitoring compose (`/opt/repo-cdn/monitoring/docker-compose.monitoring.yml`):

```yaml
services:
  prometheus:
    image: prom/prometheus:v2.54.1
    container_name: repo-prometheus
    restart: unless-stopped
    ports: ["127.0.0.1:9090:9090"]
    volumes:
      - /opt/repo-cdn/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - /opt/repo-cdn/monitoring/prometheus/data:/prometheus

  grafana:
    image: grafana/grafana:11.2.0
    container_name: repo-grafana
    restart: unless-stopped
    ports: ["127.0.0.1:3000:3000"]
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=ChangeThisStrongPassword
      - GF_SERVER_ROOT_URL=https://status.repo.vaheed.net/grafana/
      - GF_SERVER_SERVE_FROM_SUB_PATH=true
    volumes:
      - /opt/repo-cdn/monitoring/grafana/data:/var/lib/grafana
      - /opt/repo-cdn/monitoring/grafana/provisioning:/etc/grafana/provisioning:ro
      - /opt/repo-cdn/monitoring/grafana/dashboards:/var/lib/grafana/dashboards:ro

  loki:
    image: grafana/loki:3.1.1
    container_name: repo-loki
    restart: unless-stopped
    ports: ["127.0.0.1:3100:3100"]
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - /opt/repo-cdn/monitoring/loki/config.yml:/etc/loki/local-config.yaml:ro
      - /opt/repo-cdn/monitoring/loki/data:/loki

  promtail:
    image: grafana/promtail:3.1.1
    container_name: repo-promtail
    restart: unless-stopped
    command: -config.file=/etc/promtail/config.yml
    volumes:
      - /opt/repo-cdn/monitoring/promtail/config.yml:/etc/promtail/config.yml:ro
      - /var/log:/var/log:ro
      - /opt/repo-cdn/monitoring/promtail/positions:/positions

  node-exporter:
    image: prom/node-exporter:v1.8.2
    container_name: repo-node-exporter
    restart: unless-stopped
    network_mode: host
    pid: host
    command:
      - --path.rootfs=/host
    volumes:
      - /:/host:ro,rslave

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: repo-cadvisor
    restart: unless-stopped
    ports: ["127.0.0.1:8088:8080"]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:1.3.0
    container_name: repo-nginx-exporter
    restart: unless-stopped
    command:
      - -nginx.scrape-uri=http://127.0.0.1:8080/nginx_status
    network_mode: host
```

Prometheus config (`/opt/repo-cdn/monitoring/prometheus/prometheus.yml`):

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: nginx
    static_configs:
      - targets: ['192.168.20.130:9113']

  - job_name: node
    static_configs:
      - targets: ['192.168.20.130:9100']

  - job_name: cadvisor
    static_configs:
      - targets: ['cadvisor:8080']
```

Important:
- `nginx-exporter` and `node-exporter` run with `network_mode: host`, so Prometheus must scrape them via host IP (example above uses `192.168.20.130`).
- Replace `192.168.20.130` with your cache server IP if different.

Loki config (`/opt/repo-cdn/monitoring/loki/config.yml`):

```yaml
auth_enabled: false
server:
  http_listen_port: 3100
common:
  path_prefix: /loki
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory
schema_config:
  configs:
    - from: 2025-01-01
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h
storage_config:
  filesystem:
    directory: /loki/chunks
```

Promtail config (`/opt/repo-cdn/monitoring/promtail/config.yml`):

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0
positions:
  filename: /positions/positions.yaml
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: nginx-access
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx-access
          host: repo-edge
          __path__: /var/log/nginx/access.log
  - job_name: nginx-error
    static_configs:
      - targets: [localhost]
        labels:
          job: nginx-error
          host: repo-edge
          __path__: /var/log/nginx/error.log
  - job_name: syslog
    static_configs:
      - targets: [localhost]
        labels:
          job: syslog
          host: repo-edge
          __path__: /var/log/syslog
```

Grafana datasources (`/opt/repo-cdn/monitoring/grafana/provisioning/datasources/datasources.yml`):

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
```

Start monitoring stack:

```bash
mkdir -p /opt/repo-cdn/monitoring/prometheus/data
mkdir -p /opt/repo-cdn/monitoring/loki/data
mkdir -p /opt/repo-cdn/monitoring/grafana/data
mkdir -p /opt/repo-cdn/monitoring/promtail/positions

# required ownership for container runtime users
chown -R 65534:65534 /opt/repo-cdn/monitoring/prometheus/data
chown -R 10001:10001 /opt/repo-cdn/monitoring/loki/data
chown -R 472:472 /opt/repo-cdn/monitoring/grafana/data
chown -R 0:0 /opt/repo-cdn/monitoring/promtail/positions

chmod -R 750 /opt/repo-cdn/monitoring/prometheus/data
chmod -R 750 /opt/repo-cdn/monitoring/loki/data
chmod -R 750 /opt/repo-cdn/monitoring/grafana/data
chmod -R 755 /opt/repo-cdn/monitoring/promtail/positions

docker compose -f /opt/repo-cdn/monitoring/docker-compose.monitoring.yml up -d
```

Expose Grafana through Nginx (`/etc/nginx/conf.d/status.conf` add-on):

```nginx
server {
    listen 443 ssl http2;
    server_name status.repo.vaheed.net;
    include /etc/nginx/snippets/tls.conf;
    include /etc/nginx/snippets/security-headers.conf;

    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd-status;

    location = / {
        return 302 /grafana/;
    }

    location /grafana/ {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Prefix /grafana;
    }

    location /prometheus/ {
        proxy_pass http://127.0.0.1:9090;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
        proxy_set_header X-Forwarded-Port 443;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Prefix /prometheus;
        proxy_redirect ~^/(.*)$ /prometheus/$1;
    }
}
```

Create dashboard access user:

```bash
apt-get install -y apache2-utils
htpasswd -c /etc/nginx/.htpasswd-status admin
nginx -t && systemctl reload nginx
```

Dashboard panels to create in Grafana (full activity view):

1. Total requests/sec (Nginx):
`sum(rate(nginx_http_requests_total[1m]))`
2. 4xx rate:
`sum(rate(nginx_http_requests_total{status=~"4.."}[5m]))`
3. 5xx rate:
`sum(rate(nginx_http_requests_total{status=~"5.."}[5m]))`
4. Active connections:
`nginx_connections_active`
5. Package cache HIT ratio (from logs via Loki):
`sum(rate({job="nginx-access"} |= "cache=HIT" [5m])) / (sum(rate({job="nginx-access"} |= "cache=HIT" [5m])) + sum(rate({job="nginx-access"} |= "cache=MISS" [5m])))`
6. Top client IPs:
Loki query: `{job="nginx-access"} | pattern "<ip> - - [<time>] \"<method> <path> <proto>\" <status> <bytes> <_> <_> rt=<rt> <_>" | topk(20, count_over_time({job="nginx-access"}[5m]))`
7. Top requested hosts:
`sum by (host) (count_over_time({job="nginx-access"}[5m]))`
8. Upstream error log stream:
`{job="nginx-error"} |= "upstream"`
9. Disk usage (`/var/cache/repo-cdn`):
`100 - ((node_filesystem_avail_bytes{mountpoint="/var"} * 100) / node_filesystem_size_bytes{mountpoint="/var"})`
10. Docker registry container health:
`sum by (name) (rate(container_cpu_usage_seconds_total{name=~"registry-.*"}[5m]))`

Automatic dashboard provisioning:

- The dashboard `cache-lite Overview` is auto-loaded from:
  - `/opt/repo-cdn/monitoring/grafana/dashboards/cache-lite-overview.json`
- Provisioning file:
  - `/opt/repo-cdn/monitoring/grafana/provisioning/dashboards/dashboards.yml`

After changing dashboard JSON:

```bash
docker restart repo-grafana
```

Operational live activity commands:

```bash
# live request stream
tail -f /var/log/nginx/access.log

# cache HIT/MISS counters
awk '{for(i=1;i<=NF;i++) if($i ~ /^cache=/) print $i}' /var/log/nginx/access.log | sort | uniq -c

# current top endpoints
awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -20
```

If monitoring containers fail with `permission denied`:

```bash
docker compose -f /opt/repo-cdn/monitoring/docker-compose.monitoring.yml down

chown -R 65534:65534 /opt/repo-cdn/monitoring/prometheus/data
chown -R 10001:10001 /opt/repo-cdn/monitoring/loki/data
chown -R 472:472 /opt/repo-cdn/monitoring/grafana/data
chown -R 0:0 /opt/repo-cdn/monitoring/promtail/positions

docker compose -f /opt/repo-cdn/monitoring/docker-compose.monitoring.yml up -d
docker logs --tail=80 repo-loki
docker logs --tail=80 repo-prometheus
```

## 16) Troubleshooting

Runbook commands for common startup failures:

1. Duplicate proxy directives (`proxy_buffering` / `proxy_request_buffering`)

```bash
# Keep proxy_buffering/proxy_request_buffering only in OCI location block.
# Remove duplicates from shared snippet:
sed -i '/proxy_buffering /d;/proxy_request_buffering /d' /etc/nginx/snippets/proxy-common.conf
nginx -t && systemctl reload nginx
```

2. Missing trusted chain file

```bash
# If ca-chain.pem does not exist, use fullchain.pem as trusted certificate source
sed -i 's#ssl_trusted_certificate /opt/repo-cdn/tls/ca-chain.pem;#ssl_trusted_certificate /opt/repo-cdn/tls/fullchain.pem;#' /etc/nginx/snippets/tls.conf
nginx -t && systemctl reload nginx
```

3. OCSP stapling warning (`no OCSP responder URL`)

```bash
# Non-fatal warning. Disable stapling if cert has no OCSP URI.
sed -i 's/ssl_stapling on;/ssl_stapling off;/; s/ssl_stapling_verify on;/ssl_stapling_verify off;/' /etc/nginx/snippets/tls.conf
nginx -t && systemctl reload nginx
```

4. Missing cache directories

```bash
mkdir -p /var/cache/repo-cdn/{pkg,oci,tmp}
chown -R www-data:www-data /var/cache/repo-cdn
chmod 755 /var/cache/repo-cdn /var/cache/repo-cdn/pkg /var/cache/repo-cdn/oci /var/cache/repo-cdn/tmp
nginx -t && systemctl reload nginx
```

5. Invalid cache size unit

```bash
# Use g/m/k only; do not use 't'
grep -n 'proxy_cache_path' /etc/nginx/nginx.conf
# expected: max_size=3072g and max_size=6144g
```

6. `apt update` returns many `configured multiple times` warnings

```bash
cp /etc/apt/sources.list /etc/apt/sources.list.bak.$(date +%F-%H%M%S)
sed -i 's/^[[:space:]]*deb /# deb /g' /etc/apt/sources.list
grep -R --line-number '^deb ' /etc/apt/sources.list /etc/apt/sources.list.d || true
apt clean
apt update
```

7. `502` with upstream IPv6 / TLS handshake errors

```bash
# Force IPv4 resolver behavior and enable upstream SNI for CDN-backed mirrors
sed -i 's/ipv6=on/ipv6=off/' /etc/nginx/nginx.conf
grep -q 'proxy_ssl_server_name on;' /etc/nginx/snippets/proxy-common.conf || \
  printf '\nproxy_ssl_server_name on;\nproxy_ssl_name $proxy_host;\n' >> /etc/nginx/snippets/proxy-common.conf
nginx -t && systemctl reload nginx

# Validate direct origin reachability from server
curl -4I https://archive.ubuntu.com/ubuntu/dists/jammy/InRelease
curl -4I https://security.ubuntu.com/ubuntu/dists/jammy-security/InRelease
```

8. Package files not cached for any distro (`X-Cache-Status` stays `MISS`)

```bash
# Ensure package mirrors do NOT use slice range cache key
sed -i '/slice 1m;/d;/\\$slice_range/d' /etc/nginx/snippets/proxy-common.conf

# Ensure package location caches metadata/content even when upstream sets cookie/cache-control headers
grep -n 'proxy_ignore_headers Set-Cookie Cache-Control Expires;' /etc/nginx/conf.d/package-mirrors.conf
grep -n 'proxy_hide_header Set-Cookie;' /etc/nginx/conf.d/package-mirrors.conf
grep -n 'proxy_cache_valid 200 206 301 302' /etc/nginx/conf.d/package-mirrors.conf

nginx -t && systemctl reload nginx

# Test across distro endpoints (2nd request should be HIT)
for u in \
  https://ubuntu.repo.vaheed.net/ubuntu/dists/jammy/InRelease \
  https://debian.repo.vaheed.net/debian/dists/bookworm/InRelease \
  https://alpine.repo.vaheed.net/alpine/v3.21/main/x86_64/APKINDEX.tar.gz \
  https://arch.repo.vaheed.net/core/os/x86_64/core.db \
  https://fedora.repo.vaheed.net/pub/fedora/linux/releases/40/Everything/x86_64/os/repodata/repomd.xml \
  https://opensuse.repo.vaheed.net/distribution/leap/15.6/repo/oss/repodata/repomd.xml; do
  curl -sI "$u" | egrep -i 'HTTP/|x-cache-status|content-length' || true
  curl -sI "$u" | egrep -i 'HTTP/|x-cache-status|content-length' || true
done
```

Additional checks:

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
ss -ltnp | grep -E ':(80|443|5001|5002|5003|5004|5005|5006|5007)\\b'
tail -n 100 /var/log/nginx/error.log
```

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
