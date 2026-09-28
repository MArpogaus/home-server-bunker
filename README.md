# home-server-bunker

BunkerWeb in a rootless Podman pod, with an Ansible role that deploys it.
BunkerWeb is the reverse proxy: it terminates TLS, applies a web application
firewall and bans a client that misbehaves.

| Container | Job | Default memory ceiling |
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

The service follows the configuration interface in
`home-server-template/README.md`, "Configuration interface". It has no public
hostname of its own.

| Variable | Default | Controls |
|---|---|---|
| `bunker_service_sites` | `{}` | Site settings per service; see "Sites" |
| `bunker_service_certificates` | `letsencrypt` | `letsencrypt`, or `self-signed` for a host without public DNS |
| `bunker_service_letsencrypt_email` | empty | ACME contact |
| `bunker_service_config` | `{}` | BunkerWeb's global settings, merged over `bunker_service_config_defaults` |
| `bunker_service_memory` | `{}` | Memory ceilings per container |
| `bunker_service_host_loopback_address` | `169.254.1.3` | Host loopback as the pod sees it |
| `bunker_service_nginx_image`, `bunker_service_scheduler_image` | see `defaults/main.yml` | The images; both keep the same tag |

`bunker_service_config` takes BunkerWeb's own keys, such as
`WHITELIST_COUNTRY: DE CH AT` or `MODSECURITY_SEC_RULE_ENGINE: DetectionOnly`.
`true` and `false` become `yes` and `no`; quote `On`, `Off` and a country code
such as `NO`, because YAML reads them as booleans. The role keeps the
certificate settings, `SERVER_NAME`, `MULTISITE`, `SERVE_FILES`, the reverse
proxy switches, `DISABLE_DEFAULT_SERVER`, `LOG_FORMAT` and the audit log
parts; neither the config nor a site's options can change them.

### Sites

`bunker_service_sites` maps a service name to the settings of its site:
`options`, BunkerWeb settings for this site, and `limit_req_urls`, a list of
`url` and `rate` pairs.

```yaml
bunker_service_sites:
  nextcloud:
    options:
      MAX_CLIENT_SIZE: 15G
    limit_req_urls:
      - {url: /remote.php/, rate: 8r/s}
```

- The site exists while the service has an entry here, is in
  `base_setup_services` with a `port`, and `<name>_service_hostname` is not
  empty. Its name is that hostname and
  its upstream `http://169.254.1.3:<port>`.
- The template writes `<hostname>_<KEY>=<value>` for each option, so a site
  can carry any BunkerWeb setting. BunkerWeb ignores a key that it does not
  know. The deploy asserts upper-case keys, values without a newline or `=`,
  and no option that sets a key the role keeps.
- The deploy refuses a hostname that appears twice. Such a site loses its
  settings.
- `LIMIT_REQ_RATE`, `USE_MODSECURITY` and the geo allowlist are global. A site
  overrides them in `options` or `limit_req_urls`.
- Each site needs a public DNS record, because BunkerWeb requests its
  certificate.

## Specifics

- `USE_BUNKERNET` is off by default, because BunkerNet reports blocked
  requests to Bunkerity.
- The pod keeps Podman's journald log driver, because the image links its logs
  to `/proc/1/fd/1` and `/proc/1/fd/2`. Every stderr line is therefore `err`.
- `LOG_FORMAT` adds `$request_time` and `$upstream_response_time` to the image
  default. It writes `-` for the user and the referer, which can carry a share
  token. The dashboard parses this format. The query string stays, so
  `monitoring/alloy-redact.txt` redacts every query value, in every line.
- The error and ban rules match only lines that start with nginx's own
  timestamp and level. An access line starts with the client's `Host` header,
  so a match anywhere in the line lets a client raise an alert.
- The ModSecurity audit log keeps parts `A`, `H` and `Z`: the rule messages
  without the request and response headers and bodies, which carry cookies,
  `Authorization` and form passwords. A rule message still quotes the value
  that matched, so `monitoring/alloy-redact.txt` redacts it.
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
