# powerdns-in-a-box

A self-contained PowerDNS Authoritative + dnsdist + Poweradmin + MariaDB stack
that runs on Rocky Linux 9 with `podman-compose`. Intended deployment path is
`/root/powerdns/` on the target server.

## Stack

| Service      | Image                                | Role                                      | Host port      |
| ------------ | ------------------------------------ | ----------------------------------------- | -------------- |
| `dnsdist`    | `powerdns/dnsdist-19:latest`         | Frontend load balancer / ACL on port 53   | `53/tcp+udp`   |
| `pdns`       | `powerdns/pdns-auth-49:latest`       | Authoritative DNS, gmysql backend         | internal 5300  |
| `poweradmin` | `poweradmin/poweradmin:stable`       | Web admin UI for PowerDNS                 | `8090/tcp`     |
| `mariadb`    | `mariadb:10.11`                      | Backend DB for both PowerDNS and Poweradmin | internal 3306 |

DNS queries hit `dnsdist` on port 53, which forwards everything to `pdns` on
internal port 5300. Poweradmin talks to PowerDNS over the compose network via
the HTTP API at `http://pdns:8081` and writes records directly to the `pdns`
database in MariaDB.

## Layout

```
.
├── compose.yml              # podman-compose stack definition
├── .env                     # passwords, API key, NS records (edit before deploy)
├── pdns/pdns.conf           # PowerDNS authoritative config (gmysql + API)
├── dnsdist/dnsdist.conf     # dnsdist frontend config
└── mysql-init/01-init.sql   # one-shot DB init: databases, users, schema
```

`mysql-data/` is created on first start and holds the MariaDB data directory.

## Prerequisites

- Rocky Linux 9 (or RHEL/Alma 9)
- `podman` and `podman-compose` installed
- Port 53 free on the host (see "Free up port 53" below)

## Deploy

1. Copy this directory to the target server as `/root/powerdns`:

   ```bash
   scp -r . root@server:/root/powerdns
   ssh root@server
   cd /root/powerdns
   ```

2. **Edit `.env`** and replace every `Change*` placeholder. The same passwords
   and API key must also appear in `pdns/pdns.conf` and `mysql-init/01-init.sql`
   — mismatches are the #1 source of `ERROR 1045` (MySQL access denied) at
   startup.

   Values that must match:

   | `.env`             | `pdns/pdns.conf`     | `mysql-init/01-init.sql`            |
   | ------------------ | -------------------- | ----------------------------------- |
   | `PDNS_DB_PASS`     | `gmysql-password`    | `'pdns'@'%' IDENTIFIED BY ...`      |
   | `PDNS_API_KEY`     | `api-key`            | —                                   |
   | `PA_DB_PASS`       | —                    | `'poweradmin'@'%' IDENTIFIED BY ...` |

3. Free up port 53 (see next section).

4. Start the stack:

   ```bash
   podman-compose up -d
   podman-compose ps
   ```

5. Open the firewall:

   ```bash
   firewall-cmd --permanent --add-service=dns
   firewall-cmd --permanent --add-port=8090/tcp
   firewall-cmd --reload
   ```

6. Poweradmin should be reachable at `http://<server>:8090`, log in with
   `PA_ADMIN_USERNAME` / `PA_ADMIN_PASSWORD` from `.env`.

## Free up port 53

Rocky 9 hosts usually have two things competing for port 53:

### systemd-resolved

```bash
systemctl stop systemd-resolved
systemctl disable systemd-resolved
```

Edit `/etc/systemd/resolved.conf` and set:

```ini
[Resolve]
DNSStubListener=no
```

If `/etc/resolv.conf` is a symlink to systemd-resolved, replace it with a
regular file pointing at an upstream resolver (e.g. `nameserver 1.1.1.1`) so
the host keeps internet access.

### Podman's aardvark-dns

By default `aardvark-dns` binds port 53 on each bridge network. Edit
`/etc/containers/containers.conf`:

```ini
[network]
dns_bind_port = 5353
```

Then restart Podman:

```bash
systemctl restart podman
```

Verify port 53 is free before starting the stack:

```bash
ss -lptn 'sport = :53'
```

## Verification

```bash
podman-compose ps                # all four containers should be Up / healthy
podman logs -f pdns-auth         # should connect to mariadb, no schema errors
podman logs -f poweradmin        # should reach mariadb and PowerDNS API
dig @127.0.0.1 -p 53 example.com # should reach dnsdist → pdns
```

## Troubleshooting

| Symptom                                                | Fix                                                                                                                                  |
| ------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `bind: address already in use` on port 53              | See "Free up port 53" — systemd-resolved or aardvark-dns is holding it.                                                              |
| `ERROR 1045 ... 'poweradmin'@'10.89.0.x'`              | Password mismatch between `.env` and the user row in MariaDB. `ALTER USER 'poweradmin'@'%' IDENTIFIED BY '...';` to the `.env` value. |
| `Unknown MySQL server host 'mysql'`                    | `pdns.conf` still references the old `mysql` service name. It must say `gmysql-host=mariadb`.                                        |
| `Table 'pdns.domains' doesn't exist`                   | DB volume was created before the schema was added. `podman-compose down && rm -rf mysql-data && podman-compose up -d`.                |
| `ERROR 1410 (42000): not allowed to create user with GRANT` | MySQL 8 / MariaDB requires explicit `CREATE USER` before `GRANT`. Run `CREATE USER IF NOT EXISTS ...` first.                         |

## Notes

- The `:Z` volume labels are SELinux relabeling hints for Podman on RHEL-family
  hosts. Leave them in place.
- `mysql-init/*.sql` runs **only on first MariaDB initialization** (empty data
  directory). Edits made after the first start do not retroactively apply —
  either run them manually with `mariadb -uroot -p` or wipe `mysql-data/` and
  recreate.
- The example `.env` ships with weak placeholder passwords. Replace them before
  any deployment that's reachable from the network.
