# service-bunker

Bunkerweb reverse proxy (WAF, rate limiting, Let's Encrypt) in a rootless Podman
pod under the `proxy` user.

## Architecture

```
  Internet → bunker-nginx (80/443) → 169.254.1.2:8080 (Nextcloud pod)
```

Rootless users have separate container networks, so the upstream is the
Nextcloud pod's host-published port, not a container name. It is an address,
not `host.containers.internal`: nginx resolves upstreams through its `resolver`
directive, which never consults `/etc/hosts` where podman puts that name.

| Container | Image | Purpose |
|---|---|---|
| bunker-nginx | bunkerity/bunkerweb:1.6.11 | Proxy, WAF, TLS (image HEALTHCHECK honored) |
| bunker-scheduler | bunkerity/bunkerweb-scheduler:1.6.11 | Config generation, cert renewal |

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `bunker_service_server_name` | required | Public hostname |
| `bunker_service_letsencrypt_email` | required | ACME account |
| `bunker_service_host_address` | `169.254.1.2` | The host as seen from the pod |
| `bunker_service_nextcloud_upstream_url` | `http://{{ bunker_service_host_address }}:8080` | Upstream |
| `bunker_service_dns_resolvers` | `169.254.1.1 10.0.2.3` | nginx resolvers (not Docker's 127.0.0.11) |
| `bunker_service_whitelist_ip` | `127.0.0.1` | Lets local health checks past the geo filter |
| `bunker_service_generate_self_signed_ssl` | `no` | Fallback cert; mutually exclusive with ACME |
| `bunker_service_whitelist_country` | `DE CH AT` | Geo allowlist |
| `bunker_service_limit_req_rate` | `3r/s` | Rate limit |
| `bunker_service_use_modsecurity` | `no` | ModSecurity (RAM heavy) |
| `bunker_service_max_client_size` | `10G` | Upload limit |
| `bunker_service_*_extra_args` | `--memory=...` | Per-container ceilings |
| `bunker_service_auto_update` | `registry` | Podman auto-update |

Both containers keep the pinned `1.6.11` tag; bump both together.

## Role Contract

Inherited from `site.yml`: `service_name`, `service_user`, `service_home`,
`service_repo`. File tasks notify `proxy quadlets changed`.

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
