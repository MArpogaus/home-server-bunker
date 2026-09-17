# service-bunker

Bunkerweb reverse proxy (WAF, rate limiting, Let's Encrypt) in a rootless Podman
pod under the `proxy` user.

## Architecture

```
  Internet → bunker-nginx (80/443) → 169.254.1.3:8080 (Nextcloud pod)
```

| Container | Image | Purpose |
|---|---|---|
| bunker-nginx | bunkerity/bunkerweb:1.6.11 | Proxy, WAF, TLS (image HEALTHCHECK honored) |
| bunker-scheduler | bunkerity/bunkerweb-scheduler:1.6.11 | Config generation, cert renewal |

Both containers keep the pinned `1.6.11` tag. Bump both together.

## How the proxy reaches Nextcloud

Each service runs as its own rootless user. Two rootless users do not share a
container network, so the proxy cannot use a container name for the upstream.
It goes through the host instead.

The Nextcloud pod publishes port 8080 on the host loopback only. pasta gives a
pod the address `169.254.1.2` for the host, but that address reaches the
routable addresses of the host, not the loopback. A request to `127.0.0.1` on
the host therefore fails with 502.

The `--map-host-loopback` option in `proxy.pod` adds a second address,
`169.254.1.3`, which maps to the host loopback. The upstream URL uses this
address. The address is a literal, not `host.containers.internal`, because
nginx resolves an upstream name through its `resolver` directive. That
directive never reads `/etc/hosts`, where podman writes the name.

`bunker_service_dns_resolvers` must name the resolvers of the pod. The
BunkerWeb default is the Docker resolver `127.0.0.11`, which does not exist
here.

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `bunker_service_server_name` | required | Public hostname |
| `bunker_service_letsencrypt_email` | required | ACME account |
| `bunker_service_host_loopback_address` | `169.254.1.3` | Host loopback as seen from the pod |
| `bunker_service_nextcloud_upstream_url` | derived | Upstream |
| `bunker_service_dns_resolvers` | `169.254.1.1 10.0.2.3` | nginx resolvers |
| `bunker_service_whitelist_country` | `DE CH AT` | Geo allowlist |
| `bunker_service_whitelist_ip` | `127.0.0.1` | Lets local health checks past the geo filter |
| `bunker_service_api_whitelist_ip` | `127.0.0.1 10.0.0.0/8` | Who can call the BunkerWeb API |
| `bunker_service_bad_behavior_status_codes` | `400 401 403 405 444` | Codes that count toward a ban |
| `bunker_service_use_modsecurity` | `yes` | ModSecurity WAF |
| `bunker_service_modsecurity_sec_rule_engine` | `DetectionOnly` | Log matches, block nothing |
| `bunker_service_modsecurity_crs_plugins` | `nextcloud-rule-exclusions` | CRS plugin for Nextcloud |
| `bunker_service_limit_req_rate` | `3r/s` | Default rate limit |
| `bunker_service_limit_req_urls` | six paths | Per-path rate limits |
| `bunker_service_auto_lets_encrypt` | `yes` | ACME certificates |
| `bunker_service_generate_self_signed_ssl` | `no` | Fallback cert; mutually exclusive with ACME |
| `bunker_service_max_client_size` | `10G` | Upload limit |
| `bunker_service_*_extra_args` | `--memory=...` | Per-container ceilings |
| `bunker_service_auto_update` | `registry` | Podman auto-update |

### Why these defaults

ModSecurity runs in `DetectionOnly` mode. It writes a log line for every match
and blocks nothing. Read the log for some weeks. If no legitimate request
matches a rule, set `bunker_service_modsecurity_sec_rule_engine` to `On`.

The `nextcloud-rule-exclusions` plugin is necessary. The CRS core rules block
WebDAV verbs and large uploads without it.

The default rate limit of `3r/s` is too low for a Nextcloud client. The paths
in `bunker_service_limit_req_urls` get a higher limit: the app store, the
collaborative text editor, preview generation, WebDAV, the push websocket and
the Memories app. Add a path to this list when a client reports HTTP 429.

`bunker_service_bad_behavior_status_codes` omits 404. Nextcloud answers 404 for
many normal requests, such as a missing `.well-known` path. With 404 in the
list, a normal client gets a ban.

Let's Encrypt and the self-signed certificate exclude each other. Set
`bunker_service_generate_self_signed_ssl` to `yes` only for a host without a
public DNS name.

## Role Contract

Inherited from `site.yml`: `service_name`, `service_user`, `service_home`,
`service_repo`. File tasks notify `proxy quadlets changed`.

## Development

Read [AGENTS.md](../AGENTS.md) for the hook setup and the branch rules.

## License

MIT
