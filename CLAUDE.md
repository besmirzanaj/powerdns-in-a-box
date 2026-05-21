# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A `podman-compose` stack that runs a full PowerDNS authoritative DNS service
behind `dnsdist`, with PowerDNS-Admin (`powerdnsadmin/pda-legacy`) as the
management UI and MariaDB 10.11 as the shared backend. The target deployment
host is **Rocky Linux 9** and the canonical install path on that host is
`/root/git/powerdns-in-a-box/`. This is config-only — there is nothing to
build, lint, or test.

The four containers (`mariadb`, `pdns`, `dnsdist`, `powerdns-admin`) are wired
together by service name on the default compose network. Service names act as
DNS hostnames inside the network, so changing a service name in `compose.yml`
requires updating every other config that references it.

PowerDNS-Admin keeps its own users/settings/history in the `powerdns_admin`
database via Flask-SQLAlchemy migrations (auto-runs on first start). It never
writes to the `pdns` database directly — every zone or record change goes
through the PowerDNS HTTP API on `http://pdns:8081`.

## How the pieces are wired

A working stack requires three sets of values to stay byte-for-byte identical
across three files. The most common failure mode is one of them drifting:

| Value             | `.env`                  | `pdns/pdns.conf`       | `mysql-init/01-init.sql`           |
| ----------------- | ----------------------- | ---------------------- | ---------------------------------- |
| PowerDNS DB pass  | `PDNS_DB_PASS`          | `gmysql-password=`     | `'pdns'@'%' IDENTIFIED BY ...`     |
| PDA DB pass       | `PDA_DB_PASS`           | —                      | `'pda'@'%' IDENTIFIED BY ...`      |
| PowerDNS API key  | `PDNS_API_KEY`          | `api-key=`             | —                                  |
| DB hostname       | implicit via compose    | `gmysql-host=mariadb`  | —                                  |

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

## PowerDNS-Admin specifics

- The image (`powerdnsadmin/pda-legacy`) does **not** create an admin user
  from environment variables. Either let the first signup at the web UI claim
  admin, or run `flask user create-admin --username ... --password ...
  --email ... --firstname ... --lastname ...` inside the container after the
  schema has migrated.
- API integration (URL, key, version) is **not** wired from env vars in this
  image. After first login, set it under Settings → PDNS: URL
  `http://pdns:8081`, API key from `.env` (`PDNS_API_KEY`), version `4.x`.
- `SECRET_KEY` in `.env` is the Flask session signing key. Rotating it
  invalidates every active session.

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
podman-compose restart powerdns-admin

# Watch a container come up
podman logs -f pdns-auth
podman logs -f powerdns-admin

# Inspect or repair the DB
podman exec -it pdns-mariadb mariadb -uroot -p
```

When changing credentials, change them in `.env` AND `pdns/pdns.conf` AND run
the equivalent `ALTER USER` against the live MariaDB (or wipe `mysql-data/`).
Restarting `powerdns-admin` alone is not enough; PowerDNS reads `pdns.conf` at
startup, and PowerDNS-Admin re-reads `.env` at container restart.

## Firewall posture

DNS (port 53 tcp/udp) needs to be open to the world for an authoritative
server to do its job. The PowerDNS-Admin port (`9090/tcp`) must NOT be — it
should always be restricted to a trusted source list via a firewalld rich
rule. There is no app-level IP allowlist inside PDA; the firewall is the only
gate.
