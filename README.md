# service-bunker

Bunkerweb reverse proxy (WAF, rate limiting, Let's Encrypt) in a rootless Podman
pod under the `proxy` user.

## Architecture

```
  Internet → bunker-nginx (80/443) → host.containers.internal:8080 (Nextcloud pod)
```

Rootless users have separate container networks, so the upstream is the
Nextcloud pod's host-published port, not a container name.

| Container | Image | Purpose |
|---|---|---|
| bunker-nginx | bunkerity/bunkerweb:1.6.11 | Proxy, WAF, TLS (image HEALTHCHECK honored) |
| bunker-scheduler | bunkerity/bunkerweb-scheduler:1.6.11 | Config generation, cert renewal |

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `bunker_server_name` | required | Public hostname |
| `bunker_letsencrypt_email` | required | ACME account |
| `bunker_nextcloud_upstream_url` | `http://host.containers.internal:8080` | Upstream |
| `bunker_whitelist_country` | `DE CH AT` | Geo allowlist |
| `bunker_limit_req_rate` | `3r/s` | Rate limit |
| `bunker_use_modsecurity` | `no` | ModSecurity (RAM heavy) |
| `bunker_max_client_size` | `10G` | Upload limit |
| `bunker_service_*_extra_args` | `--memory=...` | Per-container ceilings |
| `bunker_service_auto_update` | `registry` | Podman auto-update |

Both containers keep the pinned `1.6.11` tag; bump both together.

## Role Contract

Inherited from `site.yml`: `service_name`, `service_user`, `service_uid`,
`service_home`, `service_repo`. File tasks notify `proxy quadlets changed`.

## Development

```bash
pre-commit install --install-hooks -t pre-commit -t commit-msg -t pre-push
```

Plain `pre-commit install` wires up only the pre-commit stage, so the
commitizen message and branch checks stay dormant. Hooks: shellcheck,
ansible-lint (which owns YAML style here), commitizen for conventional commits.
CI runs the same set on push and pull request. Actions are pinned to SHAs, and
dependabot updates actions and hook revisions weekly against `dev`.

## License

MIT
