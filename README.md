# home-server-bunker

BunkerWeb in a rootless Podman pod, with an Ansible role that deploys it.
BunkerWeb is the reverse proxy: it terminates TLS, applies a web application
firewall and bans a client that misbehaves.

Each site it serves is one entry in `bunker_service_sites`, which the
deployment sets. The role names no service.
`home-server` prepares the host.

## Architecture

```
  Internet → bunker-nginx (80/443) → 169.254.1.3:8080 (Nextcloud pod)
                                   → 169.254.1.3:8081 (ntfy, monitoring pod)
```

| Container | Purpose |
|---|---|
| bunker-nginx | Proxy, WAF, TLS (image HEALTHCHECK honored) |
| bunker-scheduler | Config generation, cert renewal |

Both images keep the same pinned tag. Bump both together.

## How the proxy reaches the other pods

Each service runs as its own rootless user. Two rootless users do not share a
container network, so the proxy cannot use a container name for the upstream.
It goes through the host instead.

The Nextcloud pod publishes port 8080 on the host loopback only. pasta gives a
pod the address `169.254.1.2` for the host, but that address reaches the
routable addresses of the host, not the loopback. A request to `127.0.0.1` on
the host therefore fails with 502. The `--map-host-loopback` option in
`bunker.pod` adds a second address, `169.254.1.3`, which maps to the host
loopback. The upstream URLs use it. The address is a literal, not
`host.containers.internal`, because nginx resolves an upstream name through
its `resolver` directive, which never reads `/etc/hosts`.

Publishing on loopback is defence in depth: firewalld refuses 8080 from the
LAN anyway.

`bunker_service_dns_resolvers` must name resolvers the pod can reach. The
BunkerWeb default is the Docker resolver `127.0.0.11`, which does not exist
here. The role default is pasta's gateway `169.254.1.1`, which forwards to the
host's resolver on the VM and on real hardware alike. A resolver that does not
exist shows up as `failed to receive reply from UDP server` on every DNSBL and
reverse lookup.

## Configuration

The variables that every deployment sets are in `home-server/README.md`,
"Variables". The role's own defaults:

| Variable | Default | Controls |
|---|---|---|
| `bunker_service_nginx_image`, `bunker_service_scheduler_image` | see `defaults/main.yml` | Proxy and scheduler images; Renovate bumps both tags together |
| `bunker_service_host_loopback_address` | `169.254.1.3` | Host loopback as seen from the pod |
| `bunker_service_sites` | required | The sites; see "Sites" |
| `bunker_service_dns_resolvers` | `169.254.1.1` | nginx resolvers |
| `bunker_service_whitelist_country` | `DE CH AT` | Geo allowlist |
| `bunker_service_whitelist_ip` | `127.0.0.1` | The whitelist plugin: a match skips every check, not the geo filter alone. Loopback is the pod itself |
| `bunker_service_use_modsecurity` | `yes` | ModSecurity WAF |
| `bunker_service_modsecurity_sec_rule_engine` | `On` | `DetectionOnly` logs matches and blocks nothing |
| `bunker_service_limit_req_rate` | `3r/s` | Default rate limit |
| `bunker_service_log_level` | `notice` | nginx `error_log` level |

### The ntfy site

`home-server/inventory/group_vars/homeserver.yml` sets two sites:
Nextcloud and ntfy. Alerts must reach the phone while it is away from home, so
ntfy gets a site of its own, and not a path under Nextcloud. ntfy serves its API
and its web app from the root, and does not work under a subpath. Its hostname,
`ntfy_hostname`, is in the secrets.

The name needs a DNS record of its own, because BunkerWeb requests a certificate
for it. ntfy does its own authentication (`home-server-monitoring`, "Reaching
ntfy"). This site is a plain TLS proxy with ModSecurity off. The authorization
of ntfy is therefore the only gate on it. It is `deny-all` by default, with one
user and one token, and signup stays disabled. This site leaves `401` out of the
bad-behavior codes. The phone app answers ntfy's `401` challenge with its
credentials on every fresh connection.

Alertmanager reaches ntfy inside the monitoring pod and never passes through
the proxy, so alerts still arrive when the proxy is down. ModSecurity is off
for this site. The phone's polls of `/alerts/json` match CRS rule 920440. An
authenticated API with one client gains nothing from a WAF.

### Why these defaults

The proxy pod keeps Podman's journald log driver, unlike the other pods. The
BunkerWeb image symlinks its log files to `/proc/1/fd/1` and `/proc/1/fd/2`
and has no syslog setting. A journal stream is a socket, and you cannot open a
socket by path. Every stderr line of this pod therefore reaches the journal as
`err`. Read the log by unit, not by priority.

BunkerNet is off: it reports blocked requests to Bunkerity's servers, and this
project sends nothing to a third party.

The proxy containers carry no `HealthOnFailure=kill`. Podman rejects the key
unless the Quadlet also sets `HealthCmd`, and the BunkerWeb image's own
`HEALTHCHECK` does not satisfy that. `Restart=on-failure` from the shared
drop-in covers a real crash.

ModSecurity blocks (`On`). A client that gets HTTP 403 from the proxy is the
sign of a false positive. Read the `ModSecurity` lines for the rule id. Set the
engine to `DetectionOnly` while you add an exclusion.

The `nextcloud-rule-exclusions` plugin is necessary. The CRS core rules block
WebDAV verbs and large uploads without it.

The default rate limit of `3r/s` is too low for a Nextcloud client. The paths
in the Nextcloud site's `limit_req_urls` get a higher limit: the app store, the
collaborative text editor, preview generation, WebDAV, the push websocket and
the Memories app. When a client reports HTTP 429, add its path to this list.

The Nextcloud site's `BAD_BEHAVIOR_STATUS_CODES` omits 404. Nextcloud answers
404 for many normal requests, such as a missing `.well-known` path. 401 stays in
the list, so the threshold is 25 per minute instead of BunkerWeb's 10. A DAV
client asks for every calendar and address book without credentials first, one
401 each. A client with ten collections reaches ten in a second. Twenty-five
wrong passwords a minute is still a ban, and Nextcloud's brute-force throttle
slows a guesser long before that.

The certificate variables and the ACME contact are part of a deployment's
settings: `home-server/README.md`, "Variables". `CertificateExpiresSoon` in
`home-server-monitoring` reports a certificate that renewal does not keep fresh.

The Nextcloud site sets `REFERRER_POLICY=no-referrer`. BunkerWeb replaces the
upstream's `Referrer-Policy` with its own, so the proxy is the one layer that
sets it.

## Sites

Every proxied site is an entry in `bunker_service_sites`, and the deployment
sets the whole list in `home-server/inventory/group_vars/homeserver.yml`. The
template writes `<name>_<KEY>=<value>` for each key in the entry's `options`. A
site therefore carries any setting that BunkerWeb understands, without a change
to this role. A new service defines its site beside `nextcloud_site` and joins
it into the list:

```yaml
bunker_service_sites: >-
  {{ [nextcloud_site, immich_site] + ([ntfy_site] if ntfy_hostname | default('') | length > 0 else []) }}

immich_site:
  name: "{{ immich_hostname }}"
  upstream: "http://{{ bunker_service_host_loopback_address }}:8082"
  options:
    REVERSE_PROXY_WS: "yes"
    MAX_CLIENT_SIZE: "50G"
    ALLOWED_METHODS: "GET|POST|HEAD|PUT|DELETE|PATCH|OPTIONS"
  limit_req_urls:
    - {url: /api/, rate: 30r/s}
```

Three things to know:

- Option keys are BunkerWeb's own, and they are case sensitive. BunkerWeb
  ignores a key that it does not know. An option key must be upper case, and
  the deploy asserts that shape, so `max_client_size` fails the play. The
  assert does not know which keys BunkerWeb has. An option value must carry no
  newline and no `=`, or it writes a second, global setting.
- The settings above the per-site block are global, and they apply to every
  site: `LIMIT_REQ_RATE`, `USE_MODSECURITY`, the geo allowlist. A site sets its
  own exceptions with `limit_req_urls`, a list of `url` and `rate` pairs beside
  `options`.
- The name is a hostname and becomes a multisite key prefix. A name with a
  space, an `=` or a `/` makes BunkerWeb read the line as a different setting.
  The site then silently loses all of its own settings. The deploy refuses such
  a name, and it refuses a name that appears twice.

The proxy holds each site's settings, and not the service repository. The proxy
must already know every site that it fronts, because it issues their
certificates.

## Monitoring

`monitoring/` holds the rules and the dashboard that `home-server-monitoring`
collects. The label contract is in its README.

The dashboard follows `home-server-monitoring/README.md`, "Dashboards". Its
access-log panels parse the `LOG_FORMAT` that
`bunkerweb.env.j2` pins: the image default plus `$request_time` and
`$upstream_response_time`, with `-` in place of `$remote_user` and
`$http_referer`. Public WebDAV sends a share token as the user name, and a
referer can carry a share link.

| Alert | Severity | Fires when |
|---|---|---|
| `BunkerWebError` | warning | BunkerWeb logged `crit`, `alert`, `emerg` or `ERROR` |
| `BunkerWebBanSpike` | warning | More than 20 bans in 1 h |
| `BunkerWeb5xx` | warning | More than 5 % of more than 50 requests to one site answer 5xx in 10 min |

## Operations

### When it breaks

**Nextcloud returns 502 through the proxy.** nginx cannot reach its upstream.
Test the path from inside the proxy:

```bash
run0 --user=bunker -- bash -c "podman exec bunker-nginx curl -sS -o /dev/null -w '%{http_code}\n' http://169.254.1.3:8080/status.php"
```

- `000`: pasta does not map the address. `bunker.pod` must have
  `Network=pasta:--map-host-loopback,169.254.1.3`, and the address must equal
  `bunker_service_host_loopback_address`.
- `400`: the path works, and the Host header is not one of Nextcloud's
  trusted domains.

**HTTPS does not answer.** Check these in order:

1. A certificate exists:
   `run0 --user=bunker -- bash -c "podman exec bunker-scheduler find /data -name '*.pem'"`.
2. The scheduler pushed its config: the proxy journal shows
   `Successfully reloaded bunkerweb`. `API request ... status = 500` means the
   push failed, which a read-only mount inside `/etc/nginx` causes.
3. The test uses the right hostname. `DISABLE_DEFAULT_SERVER=yes` drops a
   request whose SNI matches no site:
   `curl -k --resolve <domain>:443:127.0.0.1 https://<domain>/status.php`.

To watch a new certificate, run
`run0 journalctl _UID=$(id -u bunker) -f | grep -iE 'lets.?encrypt|certificate'`.
TLS works only after the DNS record resolves from the internet.

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
