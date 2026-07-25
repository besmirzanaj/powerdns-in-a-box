# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A `podman-compose` stack that runs a full PowerDNS authoritative DNS service
behind `dnsdist`, with PowerDNS-AuthAdmin (`ghcr.io/powerdns-authadmin/powerdns-authadmin`)
as the management UI and MariaDB 10.11 as the PowerDNS backend. The target
deployment host is **Rocky Linux 9** and the canonical install path on that
host is `/root/git/powerdns-in-a-box/`. This is config-only — there is nothing
to build, lint, or test.

The four containers (`mariadb`, `pdns`, `dnsdist`, `powerdns-authadmin`) are
wired together by service name on the default compose network. Service names
act as DNS hostnames inside the network, so changing a service name in
`compose.yml` requires updating every other config that references it.

PowerDNS-AuthAdmin is a Next.js app that stores its own data in SQLite on a
named volume. It never writes to the `pdns` database directly — every zone or
record change goes through the PowerDNS HTTP API on `http://pdns:8081`.

## How the pieces are wired

A working stack requires values to stay byte-for-byte identical across files.
The most common failure mode is one of them drifting:

| Value             | `.env`                  | `pdns/pdns.conf`       | `mysql-init/01-init.sql`           | `provisioning.yaml` |
| ----------------- | ----------------------- | ---------------------- | ---------------------------------- | ------------------- |
| PowerDNS DB pass  | `PDNS_DB_PASS`          | `gmysql-password=`     | `'pdns'@'%' IDENTIFIED BY ...`     | —                   |
| PowerDNS API key  | `PDNS_API_KEY`          | `api-key=`             | —                                  | `api_key:`          |
| DB hostname       | implicit via compose    | `gmysql-host=mariadb`  | —                                  | —                   |

If anything in `pdns/pdns.conf` says `gmysql-host=mysql` instead of `mariadb`,
the authoritative server will crash with `Unknown MySQL server host 'mysql'`.
That is a leftover from an earlier iteration of the design.

`DNSDIST_BIND_IP` in `.env` is the host IP that dnsdist binds port 53 to. Set
this to the server's public IPv4 when another process is already on
`0.0.0.0:53` (commonly `aardvark-dns` from a co-tenant podman stack). Leave it
blank to let dnsdist bind `0.0.0.0:53`.

## The mysql-init trap

`mysql-init/01-init.sql` is executed by the MariaDB container **only on first
initialization of an empty `mysql-data/` data directory**. Editing it after
the first start has no effect on the running database. Two ways forward when
the schema or credentials need to change:

- Destructive (loses all DNS data): `podman-compose down && rm -rf mysql-data
  && podman-compose up -d`.
- Non-destructive: `podman exec -it pdns-mariadb mariadb -uroot -p` and run
  `ALTER USER` / `CREATE USER IF NOT EXISTS` / `GRANT` / `CREATE TABLE`
  manually.

The init script uses `CHANGE_ME_*` placeholders for passwords — deployment
rewrites them in place to match the secrets in `.env` before the container's
first boot. If you ever wipe `mysql-data/` and replay the init, make sure
those placeholders have been substituted with the same values that are in
`.env` and `pdns/pdns.conf`.

MariaDB 10.11 inherits MySQL 8's stricter user rules: `GRANT ... TO
'user'@'host'` fails with `ERROR 1410` if the user/host pair does not already
exist. Always `CREATE USER IF NOT EXISTS` before granting. MySQL also treats
`'user'@'%'` and `'user'@'localhost'` as distinct accounts — both must exist
and have matching passwords if you want both container-network and host-local
logins to work.

## PowerDNS-AuthAdmin specifics

- The image (`ghcr.io/powerdns-authadmin/powerdns-authadmin`) boots admin
  user + PDNS backend connection automatically from environment variables and
  a provisioning YAML file. No manual UI configuration needed.
- SQLite database lives on the `app-data` named volume. To reset without
  affecting MariaDB: `podman-compose down && podman volume rm powerdns-in-a-box_app-data && podman-compose up -d`.
- `APP_URL` in the container environment (set from `PDA_SECRET_KEY` / `TAILSCALE_BIND_IP`
  in `.env`) must exactly match the URL operators type in their browser
  (scheme + host + port). Session cookies are scoped to this value.
- `PDA_SECRET_KEY` (maps to AuthAdmin's `APP_SECRET_KEY`) is the session
  signing key. Rotating it invalidates every active session.
- `PDA_ENCRYPTION_KEY` (maps to AuthAdmin's `APP_ENCRYPTION_KEY`) is the
  AES-256 encryption key. Rotating it makes stored PDNS API keys, OIDC
  secrets, and MFA secrets undecryptable — generate once, back the `.env`
  up, never change them.
- First-boot provisioning is driven by `provisioning.yaml` at the repo
  root, mounted into the container at `/etc/powerdns-authadmin/provisioning.yaml`.
  To re-apply provisioning, delete the `provisioned_at` row from the
  settings table in the SQLite DB.
- On MariaDB side this stack now only creates the `pdns` database and user.
  The `powerdns_admin` database and `pda` user are gone — AuthAdmin uses its
  own SQLite file.

## Host prep for port 53

The stack cannot start unless port 53 is free on whatever address dnsdist
binds. On a fresh Rocky 9 host two things commonly hold it:

1. **`systemd-resolved`** — stop and disable it, and set `DNSStubListener=no`
   in `/etc/systemd/resolved.conf`. If `/etc/resolv.conf` is a symlink, replace
   it with a real file pointing at an upstream resolver so the host keeps DNS.
2. **`aardvark-dns`** — Podman's per-network DNS daemon binds 53 on each
   bridge gateway. To globally move it: add `[network]\ndns_bind_port = 5353`
   to `/etc/containers/containers.conf` and restart Podman (this disrupts all
   running podman stacks). To work around it without disruption: set
   `DNSDIST_BIND_IP` in `.env` to the host's public IP so dnsdist binds only
   that address while aardvark keeps its bridge gateway IP.

The README's "Port 53" section has the exact commands.

## Working in this repo

There are no commands to memorize, but these are the ones you'll need when
making changes:

```bash
# After editing any config, restart the affected service
podman-compose restart pdns
podman-compose restart powerdns-authadmin

# Watch a container come up
podman logs -f pdns-auth
podman logs -f powerdns-authadmin

# Reset AuthAdmin without affecting DNS data (SQLite volume)
podman-compose down
podman volume rm powerdns-in-a-box_app-data
podman-compose up -d

# Inspect or repair the MariaDB backend
podman exec -it pdns-mariadb mariadb -uroot -p

# Inspect the AuthAdmin SQLite database (inside the container)
podman exec -it powerdns-authadmin sqlite3 /data/powerdns_authadmin.db
```

## Network model

DNS (port 53 tcp/udp) is bound to `${DNSDIST_BIND_IP}` and open to the world
— it must be, for an authoritative server. Everything else (PowerDNS HTTP
API on `8081`, dnsdist console on `8083`, PowerDNS-AuthAdmin on `9090`) is
bound to **`127.0.0.1` + `${TAILSCALE_BIND_IP}`** in `compose.yml`. The
public IP never sees them.

Why not firewalld rich rules? They apply to the INPUT chain, but traffic
to a podman-published port gets DNAT'd to the container and traverses
FORWARD instead, so rich rules silently fail to match. Direct FORWARD
rules technically work but also caused a fragility: `firewall-cmd
--reload` flushes the entire nftables ruleset and rebuilds from
firewalld's permanent XML — netavark's per-container DNAT rules are not
in that XML and get wiped, taking down every podman-published port on
the host. Recovery is `podman ps -q | xargs podman restart`, but the
right answer is to avoid the firewalld/netavark fight: bind sensitive
ports to non-public IPs and put `tailscale0` in firewalld's `trusted`
zone.