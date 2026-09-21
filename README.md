# service-bunker

BunkerWeb reverse proxy (WAF, rate limiting, Let's Encrypt) in a rootless
Podman pod under the `proxy` user. The host is set up by `ansible-base`, whose
README is the entry point for the project.

## Architecture

```
  Internet → bunker-nginx (80/443) → 169.254.1.3:8080 (Nextcloud pod)
                                   → 169.254.1.3:8081 (ntfy, monitoring pod)
```

| Container | Image | Purpose |
|---|---|---|
| bunker-nginx | bunkerity/bunkerweb:1.6.14 | Proxy, WAF, TLS (image HEALTHCHECK honored) |
| bunker-scheduler | bunkerity/bunkerweb-scheduler:1.6.14 | Config generation, cert renewal |

Both containers keep the same pinned tag. Bump both together.

## How the proxy reaches the other pods

Each service runs as its own rootless user. Two rootless users do not share a
container network, so the proxy cannot use a container name for the upstream.
It goes through the host instead.

The Nextcloud pod publishes port 8080 on the host loopback only. pasta gives a
pod the address `169.254.1.2` for the host, but that address reaches the
routable addresses of the host, not the loopback. A request to `127.0.0.1` on
the host therefore fails with 502. The `--map-host-loopback` option in
`proxy.pod` adds a second address, `169.254.1.3`, which maps to the host
loopback; the upstream URLs use it. The address is a literal, not
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

| Variable | Default | Controls |
|---|---|---|
| `bunker_service_server_name` | required | Public hostname |
| `bunker_service_letsencrypt_email` | `""` | ACME contact; empty registers `contact@<server name>` |
| `bunker_service_host_loopback_address` | `169.254.1.3` | Host loopback as seen from the pod |
| `bunker_service_nextcloud_upstream_url` | derived | Upstream |
| `bunker_service_ntfy_server_name` | `""` | Second site for ntfy; empty leaves it out |
| `bunker_service_ntfy_upstream_url` | derived | ntfy in the monitoring pod |
| `bunker_service_ntfy_auth_user` / `_password` | `ntfy` / required with the name | Basic auth on that site |
| `bunker_service_dns_resolvers` | `169.254.1.1` | nginx resolvers |
| `bunker_service_whitelist_country` | `DE CH AT` | Geo allowlist |
| `bunker_service_whitelist_ip` | `127.0.0.1` | Lets local health checks past the geo filter |
| `bunker_service_bad_behavior_status_codes` | `400 401 403 405 444` | Codes that count toward a ban |
| `bunker_service_bad_behavior_threshold` | `50` | Bad answers per minute before a 24 h ban |
| `bunker_service_use_modsecurity` | `yes` | ModSecurity WAF |
| `bunker_service_modsecurity_sec_rule_engine` | `On` | `DetectionOnly` logs matches and blocks nothing |
| `bunker_service_modsecurity_crs_plugins` | `nextcloud-rule-exclusions` | CRS plugin for Nextcloud |
| `bunker_service_limit_req_rate` | `3r/s` | Default rate limit |
| `bunker_service_limit_req_urls` | six paths | Per-path rate limits |
| `bunker_service_auto_lets_encrypt` | `yes` | ACME certificates |
| `bunker_service_generate_self_signed_ssl` | `no` | Fallback cert; mutually exclusive with ACME |
| `bunker_service_max_client_size` | `10G` | Upload limit |
| `bunker_service_log_level` | `notice` | nginx `error_log` level |
| `bunker_service_*_extra_args` | `--memory=...` | Per-container ceilings |
| `bunker_service_auto_update` | `registry` | Podman auto-update |

### The ntfy site

Alerts must reach the phone while it is away from home, so ntfy gets a second
site rather than a path under Nextcloud: ntfy serves its API and its web app
from the root and does not work under a subpath.

```yaml
bunker_service_ntfy_server_name: ntfy.example.org
bunker_service_ntfy_auth_password: "<a long random string>"
```

The name needs a DNS record of its own, because BunkerWeb requests a
certificate for it. Basic auth is enforced here rather than in ntfy: the phone
app sends the same header either way, and ntfy then needs no user database and
no volume. `401` is left out of the bad-behavior codes for this site, because
basic auth answers `401` before the phone sends its credentials.

Alertmanager reaches ntfy inside the monitoring pod and never passes through
the proxy, so alerts still arrive when the proxy is down. ModSecurity is off
for this site: the phone's polls of `/alerts/json` match CRS rule 920440, and
a basic-auth API with one client gains nothing from a WAF.

### Why these defaults

The proxy pod keeps Podman's journald log driver, unlike the other pods. The
BunkerWeb image symlinks its log files to `/proc/1/fd/1` and `/proc/1/fd/2`
and has no syslog setting. A journal stream is a socket, and a socket cannot
be opened by path. So every stderr line of this pod reaches the journal as
`err`; read it by unit, not by priority.

BunkerNet is off: it reports blocked requests to Bunkerity's servers, and this
project sends nothing to a third party.

ModSecurity blocks (`On`) since 2026-09-20 after two days in `DetectionOnly`,
in which every match on the Nextcloud site was a scanner probing `/.env`,
`/.git/config` and friends (rule 930130) and no client matched. A client that
gets HTTP 403 from the proxy is the sign of a false positive: read the
`ModSecurity` lines for the rule id, and set the engine back to
`DetectionOnly` while you add an exclusion.

The `nextcloud-rule-exclusions` plugin is necessary. The CRS core rules block
WebDAV verbs and large uploads without it.

The default rate limit of `3r/s` is too low for a Nextcloud client. The paths
in `bunker_service_limit_req_urls` get a higher limit: the app store, the
collaborative text editor, preview generation, WebDAV, the push websocket and
the Memories app. Add a path to this list when a client reports HTTP 429.

`bunker_service_bad_behavior_status_codes` omits 404. Nextcloud answers 404
for many normal requests, such as a missing `.well-known` path. 401 stays in
the list, so the threshold is 50 per minute instead of BunkerWeb's 10: a DAV
client asks for every calendar and address book without credentials first,
one 401 each, and Thunderbird's ten collections met the default in one second
(2026-09-21, home address banned for a day). Fifty wrong passwords a minute
is still a ban, and Nextcloud's brute-force throttle slows a guesser long
before that.

Let's Encrypt and the self-signed certificate exclude each other. Set
`bunker_service_generate_self_signed_ssl` to `yes` only for a host without a
public DNS name. Certificate expiry needs no alert of its own: the
`CertificateRenewalFailed` alert in `service-monitoring` reports a failed
renewal.

## When it breaks

**Nextcloud returns 502 through the proxy.** nginx cannot reach its upstream.
Test the path from inside the proxy:

```bash
podman exec bunker-nginx curl -sS -o /dev/null -w '%{http_code}\n' \
  http://169.254.1.3:8080/status.php
```

`000` means pasta does not map the address: make sure that `proxy.pod` has
`Network=pasta:--map-host-loopback,169.254.1.3` and that the address equals
`bunker_service_host_loopback_address`. `400` means the path works and the
Host header is wrong; that is Nextcloud's trusted domains, see
`service-nextcloud/README.md`.

**HTTPS does not answer at all.** Check in this order.

1. Is a certificate present? `podman exec bunker-scheduler find /data -name '*.pem'`
2. Did the scheduler push config? Look for `Successfully reloaded bunkerweb`
   in the proxy journal. `API request ... status = 500` means the push failed;
   a read-only mount inside `/etc/nginx` caused it once.
3. Are you testing with the right hostname? `DISABLE_DEFAULT_SERVER=yes` drops
   requests whose SNI matches no site, which looks identical to a dead server:
   `curl -k --resolve <domain>:443:127.0.0.1 https://<domain>/status.php`

**Watching a new certificate.** `journalctl _UID=$(id -u proxy) -f | grep -iE 'lets.?encrypt|certificate'`.
Nothing TLS works until the DNS record resolves from the internet.

## Role contract

The contract is in `service-template/README.md`. Specific here:
`bunkerized_nginx.env` is mode `0600` (`vars/main.yml`), and `proxy.pod.j2` is
templated because it carries the host loopback address.

## Development

Work on `dev`. Conventional commits. Hook setup: `ansible-base/README.md`.

## License

MIT
