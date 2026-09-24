# home-server-bunker

BunkerWeb in a rootless Podman pod, with an Ansible role that deploys it.
BunkerWeb is the reverse proxy: it terminates TLS, applies a web application
firewall and bans a client that misbehaves.

| Container | Job | Memory ceiling |
|---|---|---|
| bunker-nginx | Proxy, WAF, TLS on 80 and 443 | 768M |
| bunker-scheduler | Config generation, certificate renewal | 512M |

## How the proxy reaches the other pods

The proxy reaches an upstream on a port that the upstream pod publishes on the
host loopback, because two rootless users share no container network.

- pasta's host address `169.254.1.2` does not reach the host loopback.
  `--map-host-loopback` in `bunker.pod` adds `169.254.1.3`, which does. The
  upstream URLs use it.
- The upstream is a literal address, because nginx resolves a name through its
  `resolver` directive and never reads `/etc/hosts`.
- The resolver is pasta's gateway `169.254.1.1`. The BunkerWeb default,
  `127.0.0.11`, does not exist here.

## Configuration

| Variable | Default | Controls |
|---|---|---|
| `bunker_service_nginx_image`, `bunker_service_scheduler_image` | see `defaults/main.yml` | The images; both keep the same tag |
| `bunker_service_sites` | required | The sites; see "Sites" |
| `bunker_service_host_loopback_address` | `169.254.1.3` | Host loopback as the pod sees it |
| `bunker_service_dns_resolvers` | `169.254.1.1` | nginx resolvers |
| `bunker_service_whitelist_country` | `DE CH AT` | Geo allowlist |
| `bunker_service_whitelist_ip` | `127.0.0.1` | Addresses that skip every check |
| `bunker_service_use_modsecurity` | `yes` | ModSecurity WAF |
| `bunker_service_modsecurity_sec_rule_engine` | `On` | `DetectionOnly` logs matches and blocks nothing |
| `bunker_service_limit_req_rate` | `3r/s` | Default rate limit |
| `bunker_service_auto_lets_encrypt` | `yes` | Let's Encrypt certificates |
| `bunker_service_letsencrypt_email` | empty | ACME contact |
| `bunker_service_generate_self_signed_ssl` | `no` | Self-signed certificates; excludes Let's Encrypt |
| `bunker_service_log_level` | `notice` | nginx `error_log` level |

### Sites

`home-server/inventory/group_vars/homeserver.yml` sets `bunker_service_sites`.
An entry has a `name` (the hostname), an `upstream` URL, `options` and
`limit_req_urls`, a list of `url` and `rate` pairs.

- The template writes `<name>_<KEY>=<value>` for each option, so a site can
  carry any BunkerWeb setting.
- BunkerWeb ignores an option key that it does not know. The deploy asserts
  only upper-case keys and values without a newline or `=`.
- The deploy refuses a name that is not a hostname or that appears twice. Such
  a name makes the site lose its settings.
- `LIMIT_REQ_RATE`, `USE_MODSECURITY` and the geo allowlist are global. A site
  overrides them in `options` or `limit_req_urls`.
- Each site needs a public DNS record, because BunkerWeb requests its
  certificate.

## Specifics

- The pod keeps Podman's journald log driver, because the image links its logs
  to `/proc/1/fd/1` and `/proc/1/fd/2`. Every stderr line is therefore `err`.
- `LOG_FORMAT` adds `$request_time` and `$upstream_response_time` to the image
  default. It writes `-` for the user and the referer, which can carry a share
  token. The dashboard parses this format.
- BunkerNet is off, because it reports blocked requests to Bunkerity.
- `DISABLE_DEFAULT_SERVER=yes` drops a request whose SNI matches no site.
- The containers have no `HealthOnFailure=kill`: Podman accepts it only with a
  `HealthCmd`.
- A read-only mount inside `/etc/nginx` makes the scheduler's config push fail.
- The Nextcloud site needs the CRS plugin `nextcloud-rule-exclusions`, or CRS
  blocks WebDAV verbs and large uploads. A client that gets 429 needs its path
  in the site's `limit_req_urls`.
- The Nextcloud site counts no 404 as bad behaviour, because Nextcloud answers
  404 to normal requests. Its threshold is 25, because a DAV client gets one
  401 per collection before it authenticates.
- The Nextcloud site sets `REFERRER_POLICY`, because BunkerWeb replaces the
  upstream's header.
- ntfy has a site of its own, because it does not work under a subpath. Its
  `deny-all` authorization is the only gate. ModSecurity is off there, because
  the phone's polls match CRS rule 920440. 401 is no bad behaviour there,
  because the phone answers a 401 challenge on every connection.

## Alerts

The dashboard follows `home-server-monitoring/README.md`, "Dashboards".

| Alert | Severity | Fires when |
|---|---|---|
| `BunkerWebError` | warning | BunkerWeb logs `crit`, `alert`, `emerg` or `ERROR` |
| `BunkerWebBanSpike` | warning | More than 20 bans in 1 h |
| `BunkerWeb5xx` | warning | More than 5 % of more than 50 requests to one site answer 5xx in 10 min |

## Role contract

The contract is in `home-server-template/README.md`. The role templates
`bunker.pod.j2`, because it carries the host loopback address.

## LLM coding tools

This project is developed with LLM-based coding tools. They write most of the
code and documentation. The maintainer sets the goals and the design, reviews
every change and is responsible for it. Changes are tested on a VM before they
reach a host.

## License

MIT
