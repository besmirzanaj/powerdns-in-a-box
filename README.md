# powerdns-in-a-box

A self-contained PowerDNS Authoritative + dnsdist + PowerDNS-AuthAdmin stack
that runs on Rocky Linux 9 with `podman-compose`. Intended deployment path is
`/root/git/powerdns-in-a-box/` on the target server.

Three containers, no external database — PowerDNS uses the built-in gsqlite3
backend and PowerDNS-AuthAdmin stores its own data in SQLite (named volume).

For production deployments that prefer a shared database, a **PostgreSQL
option** is also available — see [Postgres backend](#postgres-backend) below.

## Stack

| Service                | Image                                              | Role                                       | Host port        |
| ---------------------- | -------------------------------------------------- | ------------------------------------------ | ---------------- |
| `dnsdist`              | `powerdns/dnsdist-19:latest`                       | Frontend ACL / load balancer on port 53    | `53/tcp+udp`     |
| `pdns`                 | `powerdns/pdns-auth-51:latest`                     | Authoritative DNS, gsqlite3 backend        | internal 5300    |
| `powerdns-authadmin`   | `ghcr.io/powerdns-authadmin/powerdns-authadmin`    | Next.js web UI; talks to PowerDNS HTTP API | `9090/tcp`       |

DNS queries hit `dnsdist` on port 53, which forwards everything to `pdns` on
internal port 5300. PowerDNS-AuthAdmin talks to PowerDNS over the compose
network via the HTTP API at `http://pdns:8081` and stores its own data in
SQLite (named volume). PowerDNS itself uses the gsqlite3 backend — no
external database needed.

Neither service writes to the other's storage — all zone changes go through
the PowerDNS HTTP API.

## Layout

```
.
├── compose.yml                  # podman-compose stack definition (SQLite default)
├── docker-compose.postgres.yml  # override: swap SQLite → PostgreSQL
├── .env                         # API key, secrets, dnsdist bind IP (edit before deploy)
├── provisioning.yaml            # AuthAdmin first-boot provisioning
├── pdns/
│   ├── pdns.conf                # gsqlite3 backend config
│   ├── pdns.postgres.conf       # gpgsql backend config (for Postgres override)
│   ├── schema.sqlite3.sql       # SQLite schema (used by pdns-init)
│   └── schema.pgsql.sql         # PostgreSQL schema (for Postgres override)
└── dnsdist/
    └── dnsdist.conf             # dnsdist frontend config
```

`pdns-data` (named compose volume) holds the PowerDNS SQLite database.
`app-data` (named compose volume) holds the AuthAdmin SQLite database.
`pg-data` (named compose volume) holds the Postgres database when using
the Postgres override.

## Prerequisites

- Rocky Linux 9 (or RHEL/Alma 9)
- `podman` and `podman-compose` installed
- Port 53 free on the IP that `DNSDIST_BIND_IP` points at (see "Port 53" below)
- A Tailscale-joined host if you want to reach the management UIs in a
  browser without an SSH tunnel — optional, see "Network model" below

## Quickstart

Nothing real is committed — every secret in the repo is a `CHANGE_ME_*`
placeholder, and `.env.example` is a template. Run this on the target
server as root to generate strong random secrets and substitute them in
lockstep across all files that need them.

```bash
git clone https://github.com/besmirzanaj/powerdns-in-a-box /root/git/powerdns-in-a-box
cd /root/git/powerdns-in-a-box

cp .env.example .env
chmod 600 .env

# 1. Generate random secrets (only three needed — no database passwords)
PDNS_API_KEY=$(openssl rand -hex 24)
PDA_SECRET_KEY=$(openssl rand -base64 32)
PDA_ENCRYPTION_KEY=$(openssl rand -base64 32)
PDA_ADMIN_PASSWORD=$(openssl rand -hex 16)
DNSDIST_WEBSERVER_PASSWORD=$(openssl rand -hex 16)
DNSDIST_API_KEY=$(openssl rand -hex 24)

# 2. Set host-specific values (edit these for your environment)
DNSDIST_BIND_IP="$(curl -s4 ifconfig.me)"     # or hard-code your public IPv4
TAILSCALE_BIND_IP="$(tailscale ip -4 2>/dev/null || echo '')"

# 3. Substitute into .env, pdns.conf and provisioning.yaml
sed -i \
  -e "s|CHANGE_ME_API_KEY|${PDNS_API_KEY}|"             \
  -e "s|CHANGE_ME_PDA_SECRET|${PDA_SECRET_KEY}|"        \
  -e "s|CHANGE_ME_PDA_ENCRYPTION|${PDA_ENCRYPTION_KEY}|" \
  -e "s|CHANGE_ME_PDA_ADMIN|${PDA_ADMIN_PASSWORD}|"     \
  -e "s|CHANGE_ME_DNSDIST_WEB|${DNSDIST_WEBSERVER_PASSWORD}|" \
  -e "s|CHANGE_ME_DNSDIST_API|${DNSDIST_API_KEY}|"      \
  -e "s|^DNSDIST_BIND_IP=.*|DNSDIST_BIND_IP=${DNSDIST_BIND_IP}|" \
  -e "s|^TAILSCALE_BIND_IP=.*|TAILSCALE_BIND_IP=${TAILSCALE_BIND_IP}|" \
  .env

sed -i "s|CHANGE_ME_API_KEY|${PDNS_API_KEY}|g" pdns/pdns.conf
sed -i "s|CHANGE_ME_API_KEY|${PDNS_API_KEY}|g" provisioning.yaml

# 4. Free port 53 (see "Port 53" below). Then:
podman-compose up -d

# 5. After ~30s, the admin UI is ready. PowerDNS-AuthAdmin bootstraps the
#    admin user automatically (BOOTSTRAP_ADMIN_EMAIL / BOOTSTRAP_ADMIN_PASSWORD)
#    and provisions the PDNS backend connection (provisioning.yaml).
#    No manual steps needed.

# 6. Firewall: DNS to the world, plus Tailscale interface in the trusted zone.
firewall-cmd --permanent --add-service=dns
firewall-cmd --zone=trusted --add-interface=tailscale0            # runtime
firewall-cmd --permanent --zone=trusted --add-interface=tailscale0
firewall-cmd --reload
# IMPORTANT: a firewalld reload wipes podman's per-container DNAT rules.
# Re-establish them by bouncing every running container on the host:
podman ps -q | xargs -r podman restart

# 7. Print the credentials you need to save:
echo "AuthAdmin URL : http://${TAILSCALE_BIND_IP:-127.0.0.1}:9090/"
echo "AuthAdmin user: ${PDA_ADMIN_EMAIL:-admin@example.com}"
echo "AuthAdmin pass: ${PDA_ADMIN_PASSWORD}"
echo "PDNS API key  : ${PDNS_API_KEY}"
echo "dnsdist UI    : http://${TAILSCALE_BIND_IP:-127.0.0.1}:8083/  (admin / ${DNSDIST_WEBSERVER_PASSWORD})"
```

That's the whole install. Your zone-management UI is now reachable from
any tailnet device at `http://${TAILSCALE_BIND_IP}:9090/`, or locally
via an SSH tunnel.

## Postgres backend

By default the stack uses SQLite for both PowerDNS (gsqlite3) and
PowerDNS-AuthAdmin — zero configuration, minimum resource usage. When you
want a shared database for backup tooling, monitoring, or future multi-replica
scaling, a Postgres override is provided.

The override swaps SQLite for a single Postgres 16 instance that serves both
services: PowerDNS uses the `gpgsql` backend and AuthAdmin connects via
`DATABASE_URL=postgres://...`. They share the same database (`powerdns`)
with separate table schemas — pdns zone tables and AuthAdmin user/settings
tables live side by side.

### Quickstart (Postgres)

The same `.env` is used — just add the `PG_PASS` secret and pass the override
file to `docker compose`:

```bash
git clone https://github.com/besmirzanaj/powerdns-in-a-box /root/git/powerdns-in-a-box
cd /root/git/powerdns-in-a-box

cp .env.example .env
chmod 600 .env

# 1. Generate random secrets — same as SQLite plus PG_PASS
PDNS_API_KEY=$(openssl rand -hex 24)
PDA_SECRET_KEY=$(openssl rand -base64 32)
PDA_ENCRYPTION_KEY=$(openssl rand -base64 32)
PDA_ADMIN_PASSWORD=$(openssl rand -hex 16)
PG_PASS=$(openssl rand -hex 16)
DNSDIST_WEBSERVER_PASSWORD=$(openssl rand -hex 16)
DNSDIST_API_KEY=$(openssl rand -hex 24)

# 2. Set host-specific values
DNSDIST_BIND_IP="$(curl -s4 ifconfig.me)"
TAILSCALE_BIND_IP="$(tailscale ip -4 2>/dev/null || echo '')"

# 3. Substitute into .env, pdns config and provisioning.yaml
sed -i \
  -e "s|CHANGE_ME_API_KEY|${PDNS_API_KEY}|"             \
  -e "s|CHANGE_ME_PDA_SECRET|${PDA_SECRET_KEY}|"        \
  -e "s|CHANGE_ME_PDA_ENCRYPTION|${PDA_ENCRYPTION_KEY}|" \
  -e "s|CHANGE_ME_PDA_ADMIN|${PDA_ADMIN_PASSWORD}|"     \
  -e "s|CHANGE_ME_PG_PASS|${PG_PASS}|"                  \
  -e "s|CHANGE_ME_DNSDIST_WEB|${DNSDIST_WEBSERVER_PASSWORD}|" \
  -e "s|CHANGE_ME_DNSDIST_API|${DNSDIST_API_KEY}|"      \
  -e "s|^DNSDIST_BIND_IP=.*|DNSDIST_BIND_IP=${DNSDIST_BIND_IP}|" \
  -e "s|^TAILSCALE_BIND_IP=.*|TAILSCALE_BIND_IP=${TAILSCALE_BIND_IP}|" \
  .env

sed -i "s|CHANGE_ME_API_KEY|${PDNS_API_KEY}|g; s|CHANGE_ME_PG_PASS|${PG_PASS}|g" pdns/pdns.postgres.conf
sed -i "s|CHANGE_ME_API_KEY|${PDNS_API_KEY}|g"                                   provisioning.yaml

# 4. Start with the Postgres override
podman-compose -f compose.yml -f docker-compose.postgres.yml up -d
```

The rest is identical to the SQLite stack — admin bootstrap, firewall rules,
DNS validation.

To switch back to SQLite: `podman-compose -f compose.yml -f docker-compose.postgres.yml down -v`,
then `podman-compose up -d` (without the override).

## Network model

Two categories of host port:

| Port      | Bind IP                                  | Visibility                                          |
| --------- | ---------------------------------------- | --------------------------------------------------- |
| `53` TCP+UDP (dnsdist) | `${DNSDIST_BIND_IP}` (public) | Open to the world — must be, for authoritative DNS. |
| `8081` (pdns API), `8083` (dnsdist console), `9090` (PowerDNS-AuthAdmin) | `127.0.0.1` and `${TAILSCALE_BIND_IP}` | Loopback (for SSH tunnel) + tailnet only. **Never** the public IP. |

If you don't have Tailscale, leave `TAILSCALE_BIND_IP` empty and remove
the matching `"${TAILSCALE_BIND_IP}:..."` port lines from `compose.yml`.
Then reach the UIs from your laptop with:

```bash
ssh -L 9090:127.0.0.1:9090 -L 8081:127.0.0.1:8081 -L 8083:127.0.0.1:8083 root@<host>
# then open http://localhost:9090/ in your browser
```

## Configuration cheatsheet

Only one secret spans multiple files in the default SQLite stack — everything
else is single-file. The Postgres override adds a second multi-file secret:

| Value               | `.env`             | `pdns/pdns.conf`       | `pdns/pdns.postgres.conf` | `provisioning.yaml` |
| ------------------- | ------------------ | ---------------------- | ------------------------- | ------------------- |
| PowerDNS API key    | `PDNS_API_KEY`     | `api-key`              | `api-key`                 | `api_key:`          |
| Postgres password   | `PG_PASS`          | —                      | `gpgsql-password`         | —                   |

## Port 53

The `dnsdist` container needs to bind port 53 on the host. Two things commonly
hold it on a Rocky 9 host:

### systemd-resolved

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved
```

Edit `/etc/systemd/resolved.conf` and set `DNSStubListener=no` under
`[Resolve]`. If `/etc/resolv.conf` is a symlink to systemd-resolved, replace it
with a regular file pointing at an upstream resolver so the host keeps
internet access.

### Podman's aardvark-dns

Podman runs `aardvark-dns` on the gateway of every bridge network it manages,
and that daemon binds port 53 on each network's gateway IP. There are two ways
to handle this:

- **Global move**: set `[network]\ndns_bind_port = 5353` in
  `/etc/containers/containers.conf` and restart podman / the affected
  containers. Cleanest, but every existing container network has to come
  back up.
- **Per-IP bind**: set `DNSDIST_BIND_IP` in `.env` to the host's public IP so
  that dnsdist binds only on that address. `aardvark-dns` keeps binding its
  bridge gateway (e.g. `10.89.0.1:53`) and the two coexist. This is the path
  used when you don't want to disturb other podman stacks on the same host.

Verify after starting:

```bash
ss -lptn 'sport = :53'
```

## Verification

Process- and container-level checks. For DNS resolution checks, see the
next section.

```bash
podman-compose ps                                # all three containers Up / healthy
podman logs -f pdns-auth                         # no DB errors, API on :8081
podman logs -f powerdns-authadmin                 # migrations complete, ready on :3000
ss -lptn 'sport = :53'                           # confirms dnsdist holds the bind IP
```

## DNS validation with dig

Once a zone exists in PowerDNS, walk through these checks. They climb from
"the local stack works" up to "the world resolves this zone correctly".
Substitute your values for `<your.zone>` and `<DNSDIST_BIND_IP>`.

### 1. Direct query against this server (authoritative answer)

Confirms that dnsdist forwards to pdns and pdns serves the zone:

```bash
dig @<DNSDIST_BIND_IP> <your.zone> +short
dig @<DNSDIST_BIND_IP> www.<your.zone> +short
```

The `aa` flag must be set on the response — that's the marker that the
answer came from an authoritative server, not a cache:

```bash
dig @<DNSDIST_BIND_IP> <your.zone> | grep -E 'flags|status'
# Expected: status: NOERROR  /  flags: qr aa rd
```

### 2. SOA and NS

```bash
dig @<DNSDIST_BIND_IP> <your.zone> SOA +short
dig @<DNSDIST_BIND_IP> <your.zone> NS  +short
```

The SOA's first field (MNAME) should be the primary nameserver. The NS list
should contain at least one in-bailiwick name (e.g. `ns.<your.zone>.`) — that
name must also have a glue A record at the **parent** zone, otherwise no
recursive resolver can find your server from the root.

### 3. PowerDNS HTTP API (proves pdns is healthy)

```bash
cd /root/git/powerdns-in-a-box
API_KEY=$(grep ^PDNS_API_KEY .env | cut -d= -f2)
curl -sS -H "X-API-Key: ${API_KEY}" \
  http://10.89.1.10:8081/api/v1/servers/localhost/zones | python3 -m json.tool
```

A JSON array of zones (possibly empty) means the API is up and the backend
DB is reachable.

### 4. Delegation trace from the root

This only works once the parent zone has been updated to delegate to your
server. It's the canonical "is this zone really on the public internet" test:

```bash
dig +trace +nodnssec <your.zone>
```

You want the trace to descend `.` → `<tld>.` → `<parent>.` → `<your.zone>.`,
with the final hop answered by your VPS's IP. The last line should look like:

```
<your.zone>. 300 IN A <ip>
;; Received N bytes from <DNSDIST_BIND_IP>#53(ns.<your.zone>) in M ms
```

### 5. Through a public recursive resolver

Confirms the world sees it the same way you do:

```bash
dig @1.1.1.1 <your.zone> +short
dig @8.8.8.8 <your.zone> +short
```

If these match what you see when querying `@<DNSDIST_BIND_IP>` directly,
the delegation and glue at the parent are correctly in place.

### 6. Common dig failures

| `dig` output                                   | What it means                                                                                          |
| ---------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| `status: REFUSED` from `@<DNSDIST_BIND_IP>`    | The zone doesn't exist on this server. Create it in PowerDNS-AuthAdmin or via the API first.            |
| `status: SERVFAIL` from `@<DNSDIST_BIND_IP>`   | pdns is up but can't query its backend — check `podman logs pdns-auth` for gsqlite3 errors.            |
| `status: NXDOMAIN`                             | The record name isn't in the zone. Verify it was added with the right name and trailing dot.           |
| Correct locally, NXDOMAIN via `@1.1.1.1`       | Parent delegation isn't published. Update the parent zone's NS + glue and wait for its TTL to expire.  |
| Local answer has no `aa` flag                  | Reply came from a cache, not the authoritative server. Always query the auth IP for validation.        |
| `+trace` stops at `<parent>.`                  | The parent has no NS records pointing at your server, or they point at an unresolvable name (no glue). |

## Troubleshooting

| Symptom                                                | Fix                                                                                                                                       |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `bind: address already in use` on port 53              | See "Port 53" — systemd-resolved or aardvark-dns is holding it. Either move them off 53 or bind dnsdist to a specific host IP.            |
| `Unable to open database` in pdns logs                 | The `pdns-data` volume ownership wasn't set. `podman-compose down && podman volume rm powerdns-in-a-box_pdns-data && podman-compose up -d`. |
| PowerDNS-AuthAdmin won't start, `ERR_MODULE_NOT_FOUND` | The SQLite volume is empty on first boot — that's normal. Check `podman logs powerdns-authadmin` for migration errors.                   |
| PowerDNS-AuthAdmin login fails with CSRF error         | `APP_URL` in `.env` must exactly match the browser URL (scheme + host + port). Session cookies are scoped to this value.                 |

## Notes

- The `:Z` volume labels are SELinux relabeling hints for Podman on RHEL-family
  hosts. Leave them in place.
- The example `.env` ships with `CHANGE_ME_*` placeholders. Replace them
  before any deployment that's reachable from the network. Only `pdns.conf`
  and `provisioning.yaml` share the `CHANGE_ME_API_KEY` placeholder with
  `.env`.
- To reset PowerDNS zone data without affecting AuthAdmin:
  `podman-compose down && podman volume rm powerdns-in-a-box_pdns-data && podman-compose up -d`.
- To reset AuthAdmin without affecting DNS data:
  `podman-compose down && podman volume rm powerdns-in-a-box_app-data && podman-compose up -d`.
- The `pdns-init` container creates the gsqlite3 schema on first boot — no
  init scripts needed.