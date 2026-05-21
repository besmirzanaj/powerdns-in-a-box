# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A `podman-compose` stack that runs a full PowerDNS authoritative DNS service
behind `dnsdist`, with Poweradmin as the web UI and MariaDB 10.11 as the shared
backend. The target deployment host is **Rocky Linux 9** and the canonical
install path on that host is `/root/powerdns/`. This is config-only — there is
nothing to build, lint, or test.

The four containers (`mariadb`, `pdns`, `dnsdist`, `poweradmin`) are wired
together by service name on the default compose network. Service names act as
DNS hostnames inside the network, so changing a service name in `compose.yml`
requires updating every other config that references it.

## How the pieces are wired

A working stack requires three sets of values to stay byte-for-byte identical
across three files. The most common failure mode is one of them drifting:

| Value             | `.env`                  | `pdns/pdns.conf`       | `mysql-init/01-init.sql`               |
| ----------------- | ----------------------- | ---------------------- | -------------------------------------- |
| PowerDNS DB pass  | `PDNS_DB_PASS`          | `gmysql-password=`     | `'pdns'@'%' IDENTIFIED BY ...`         |
| Poweradmin DB pass | `PA_DB_PASS`           | —                      | `'poweradmin'@'%' IDENTIFIED BY ...`   |
| PowerDNS API key  | `PDNS_API_KEY`          | `api-key=`             | —                                      |
| DB hostname       | implicit via compose    | `gmysql-host=mariadb`  | —                                      |

If anything in `pdns/pdns.conf` says `gmysql-host=mysql` instead of `mariadb`,
the authoritative server will crash with `Unknown MySQL server host 'mysql'`.
That is a leftover from an earlier iteration of the design.

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

MariaDB 10.11 inherits MySQL 8's stricter user rules: `GRANT ... TO
'user'@'host'` fails with `ERROR 1410` if the user/host pair does not already
exist. Always `CREATE USER IF NOT EXISTS` before granting. MySQL also treats
`'user'@'%'` and `'user'@'localhost'` as distinct accounts — both must exist
and have matching passwords if you want both container-network and host-local
logins to work.

## Host prep for port 53

The stack cannot start unless port 53 is free on the Rocky 9 host. Two things
commonly hold it:

1. **`systemd-resolved`** — stop and disable it, and set `DNSStubListener=no`
   in `/etc/systemd/resolved.conf`. If `/etc/resolv.conf` is a symlink, replace
   it with a real file pointing at an upstream resolver so the host keeps DNS.
2. **`aardvark-dns`** — Podman's per-network DNS daemon binds 53 on bridge
   networks by default. Add `[network]\ndns_bind_port = 5353` to
   `/etc/containers/containers.conf` and restart Podman.

The README's "Free up port 53" section has the exact commands.

## Working in this repo

There are no commands to memorize, but these are the ones you'll need when
making changes:

```bash
# After editing any config, restart the affected service
podman-compose restart pdns
podman-compose restart poweradmin

# Watch a container come up
podman logs -f pdns-auth
podman logs -f poweradmin

# Inspect or repair the DB
podman exec -it pdns-mariadb mariadb -uroot -p
```

When changing credentials, change them in `.env` AND `pdns/pdns.conf` AND run
the equivalent `ALTER USER` against the live MariaDB (or wipe `mysql-data/`).
Restarting `poweradmin` alone is not enough; PowerDNS reads `pdns.conf` at
startup, and Poweradmin re-reads `.env` at container restart.

