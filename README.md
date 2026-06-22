# service-bunker

Bunkerweb — security-focused reverse proxy (WAF, rate limiting, TLS) — deployed via Podman Quadlet. Runs rootless under the `proxy` user.

## Structure

```
service-bunker/
├── ansible-role/bunker_service/
│   ├── defaults/main.yml               Role default variables (auto_update)
│   ├── tasks/main.yml                  Deployment tasks
│   └── templates/
│       ├── bunkerized_nginx.env.j2     Bunkerweb environment template
│       └── json_analytics.env.j2       Analytics format template
├── quadlets/
│   ├── proxy.pod                       Pod definition (publishes 80, 443)
│   ├── bunker-nginx.container          Main Bunkerweb nginx instance
│   ├── bunker-scheduler.container      Let's Encrypt renewal scheduler
│   ├── shared-network.network          Bridge network (10.89.0.0/24)
│   ├── promtail-proxy.pod              Log shipping pod
│   ├── promtail-proxy.container        Log shipping container
│   └── configs/
│       ├── bunkerized_nginx.env.example
│       ├── json_analytics.env.example
│       └── promtail-proxy.yaml         Log shipping config
└── .github/workflows/                  CI/CD
```

## Architecture

```
  Internet
     |
     v
  Bunkerweb (443)    ← TLS termination, WAF, rate limiting
     |
     v
  nextcloud-web (80) ← backend upstream
```

| Service | Image | Purpose |
|---|---|---|
| `bunker-nginx` | `bunkerity/bunkerweb:1.6.11` | Reverse proxy, WAF, rate limiting, TLS |
| `bunker-scheduler` | `bunkerity/bunkerweb-scheduler:1.6.11` | Let's Encrypt renewal, config generation |

Both containers use `shared-network` (10.89.0.0/24) to reach Nextcloud and other upstreams.

## Role Contract

Inherited variables from `site.yml`:

| Var | Description |
|-----|-------------|
| `service_name` | Service name (`proxy`) |
| `service_user` | System user (`proxy`) |
| `service_uid` | User UID (default: 1001) |
| `service_home` | Home dir (`/var/services/proxy`) |
| `service_repo` | Repo path (`../service-bunker`) |

Role tasks:
1. Copy static Quadlet files
2. Copy config files to `configs/`
3. Template environment files to `configs/` (mode `0600`)

## Configuration

Environment is supplied via `EnvironmentFile=` in Quadlet units:

| Variable | Default | Description |
|---|---|---|
| `SERVER_NAME` | (required) | Your domain |
| `AUTO_LETS_ENCRYPT` | `yes` | Automatic Let's Encrypt TLS |
| `LIMIT_REQ_RATE` | `3r/s` | Rate-limit requests |
| `WHITELIST_COUNTRY` | `DE CH AT` | Geo whitelist |
| `API_WHITELIST_IP` | `127.0.0.1 10.0.0.0/8` | API access control |
| `USE_MODSECURITY` | `no` | Enable ModSecurity WAF |
| `MAX_CLIENT_SIZE` | `10G` | Max upload size |
| `REVERSE_PROXY_HOST` | (per-domain) | Upstream backend |

Override via `secrets/vars.yml` with `bunker_*` prefixed variables.

## Deployment

```bash
ansible-playbook -i inventory site.yml --tags bunker_service
```

## Requirements

- Podman 4.0+ (Quadlet)
- systemd user instances
- Ansible

## License

MIT
