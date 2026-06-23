# service-bunker

Bunkerweb — security-focused reverse proxy (WAF, rate limiting, TLS) — deployed via Podman Quadlet. Runs rootless under the `proxy` user.

## Architecture

```
  Internet → Bunkerweb (443) → nextcloud-web:80 (upstream)
```

| Service | Image | Purpose |
|---|---|---|
| `bunker-nginx` | `bunkerity/bunkerweb:1.6.11` | Reverse proxy, WAF, rate limiting, TLS |
| `bunker-scheduler` | `bunkerity/bunkerweb-scheduler:1.6.11` | Let's Encrypt renewal, config generation |
| `promtail-proxy` | `grafana/promtail:3` | Log shipping |

Both bunker containers use `shared-network` (10.89.0.0/24) to reach Nextcloud and other upstreams.

## Task Reference (8 tasks)

| # | Module | Purpose | Rationale |
|---|--------|---------|-----------|
| 1 | `copy` (loop 3) | Deploy static Quadlet files (network, pods) | Network and pod definitions are static |
| 2 | `template` (loop 3) | Render `.container.j2` → `.container` | Image tags, auto-update policy injected via vars |
| 3 | `file` | Ensure `configs/` directory | Host path for bind-mounted configs |
| 4 | `copy` (loop 2) | Deploy static configs (promtail yaml, json_analytics.conf) | Log shipping + log format |
| 5 | `template` (loop 2) | Render env files (mode 0600) | Server name, secrets, rate limits, upstream |
| 6 | `command` | `machinectl shell ... systemctl --user daemon-reload` | Re-read Quadlet files |
| 7 | `systemd` | Restart `user@<uid>.service` | Triggers Quadlet generator |

## Role Contract

Inherited from `site.yml`:

| Var | Description |
|---|---|
| `service_name` | `proxy` |
| `service_user` | `proxy` |
| `service_uid` | 1001 (default) |
| `service_home` | `/var/services/proxy` |
| `service_repo` | `../service-bunker` |

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `bunker_service_auto_update` | `registry` | Podman auto-update policy |
| `bunker_nginx_image` | `bunkerity/bunkerweb:1.6.11` | Nginx image tag |
| `bunker_scheduler_image` | `bunkerity/bunkerweb-scheduler:1.6.11` | Scheduler image tag |
| `bunker_promtail_image` | `grafana/promtail:3` | Promtail image tag |
| `bunker_server_name` | (required) | Domain name |
| `bunker_letsencrypt_email` | (required) | LE registration |
| `bunker_whitelist_country` | `DE CH AT` | Geo-allowlist |
| `bunker_limit_req_rate` | `3r/s` | Rate limit |
| `bunker_max_client_size` | `10G` | Max upload size |
| `bunker_use_modsecurity` | `no` | Enable WAF |
| `bunker_auto_lets_encrypt` | `yes` | Auto TLS |

## Generalization Gaps

| What | Where | Hardcoded |
|---|---|---|
| Upstream backend | `bunkerized_nginx.env.j2` | `http://nextcloud-web:80` |
| Network subnet | `shared-network.network` | `10.89.0.0/24` |
| Host ports | `proxy.pod` | `80:8080`, `443:8443` |
| Loki endpoint | `promtail-proxy.yaml` | `http://10.0.2.2:3100` |
| Multi-site config | `bunkerized_nginx.env.j2` | Single upstream, `MULTISITE=yes` but only one site |
| Bunkernet | `bunkerized_nginx.env.j2` | `USE_BUNKERNET=no` hardcoded |

## Bugs / Known Issues

- **Missing `Restart=` on `promtail-proxy.container.j2`** — crash won't auto-restart
- **Duplicate log format** — `json_analytics.conf` (static nginx conf) and `json_analytics.env.j2` (Bunkerweb env) both define the log format; likely one is dead
- **`bunker_letsencrypt_email` is required** but was missing in `vars.yml` — TLS provisioning will fail

## Files

```
service-bunker/
  ansible-role/bunker_service/
    defaults/main.yml         # 4 vars: images + auto_update
    tasks/main.yml            # 8 tasks
    templates/                # env templates (0600)
  quadlets/
    proxy.pod                 # Pod: 80:8080, 443:8443
    shared-network.network    # Bridge 10.89.0.0/24
    bunker-nginx.container.j2
    bunker-scheduler.container.j2
    promtail-proxy.container.j2    # (missing Restart=)
    configs/                  # Promtail yaml, log format conf
  .github/workflows/          # CI/CD
```

## License

MIT
