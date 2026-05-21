# powerdns-in-a-box

A self-contained PowerDNS Authoritative + dnsdist + PowerDNS-Admin + MariaDB
stack that runs on Rocky Linux 9 with `podman-compose`. Intended deployment
path is `/root/git/powerdns-in-a-box/` on the target server.

## Stack

| Service          | Image                                   | Role                                       | Host port        |
| ---------------- | --------------------------------------- | ------------------------------------------ | ---------------- |
| `dnsdist`        | `powerdns/dnsdist-19:latest`            | Frontend ACL / load balancer on port 53    | `53/tcp+udp`     |
| `pdns`           | `powerdns/pdns-auth-49:latest`          | Authoritative DNS, gmysql backend          | internal 5300    |
| `powerdns-admin` | `powerdnsadmin/pda-legacy:latest`       | Flask web UI; talks to PowerDNS HTTP API   | `9090/tcp`       |
| `mariadb`        | `mariadb:10.11`                         | Backend DB for PowerDNS and PowerDNS-Admin | internal 3306    |

DNS queries hit `dnsdist` on port 53, which forwards everything to `pdns` on
internal port 5300. PowerDNS-Admin talks to PowerDNS over the compose network
via the HTTP API at `http://pdns:8081` and stores its own users/settings in a
separate `powerdns_admin` database in MariaDB. PowerDNS itself uses the `pdns`
database via the gmysql backend.

Two databases, two SQL users:

- `pdns@%` → owns the `pdns` database (zones, records, DNSSEC keys, ...)
- `pda@%`  → owns the `powerdns_admin` database (PDA users, settings, history)

PowerDNS-Admin never touches the `pdns` database directly — all zone changes
go through the PowerDNS HTTP API.

## Layout

```
.
├── compose.yml              # podman-compose stack definition
├── .env                     # passwords, API key, dnsdist bind IP (edit before deploy)
├── pdns/pdns.conf           # PowerDNS authoritative config (gmysql + API)
├── dnsdist/dnsdist.conf     # dnsdist frontend config
└── mysql-init/01-init.sql   # one-shot DB init: databases + users + PowerDNS schema
```

`mysql-data/` is created on first start and holds the MariaDB data directory.

## Prerequisites

- Rocky Linux 9 (or RHEL/Alma 9)
- `podman` and `podman-compose` installed
- Port 53 free on the IP that `DNSDIST_BIND_IP` points at (see "Port 53" below)

## Deploy

1. Copy this directory to the target server:

   ```bash
   scp -r . root@server:/root/git/powerdns-in-a-box
   ssh root@server
   cd /root/git/powerdns-in-a-box
   ```

2. **Edit `.env`** and replace every `CHANGE_ME_*` placeholder with strong
   secrets. The same values must also appear in `pdns/pdns.conf` and
   `mysql-init/01-init.sql` — mismatches are the #1 source of `ERROR 1045`
   (MySQL access denied) at startup. Set `DNSDIST_BIND_IP` to the public
   IPv4 that should answer DNS queries.

   Values that must match across files:

   | `.env`              | `pdns/pdns.conf`     | `mysql-init/01-init.sql`           |
   | ------------------- | -------------------- | ---------------------------------- |
   | `PDNS_DB_PASS`      | `gmysql-password`    | `'pdns'@'%' IDENTIFIED BY ...`     |
   | `PDNS_API_KEY`      | `api-key`            | —                                  |
   | `PDA_DB_PASS`       | —                    | `'pda'@'%' IDENTIFIED BY ...`      |

3. Free up port 53 on the bind IP (see next section).

4. Start the stack:

   ```bash
   podman-compose up -d
   podman-compose ps
   ```

5. Wait for PowerDNS-Admin to migrate its schema (first run only, ~30s), then
   create the first admin user:

   ```bash
   podman exec -it powerdns-admin flask user create-admin \
     --username "$PDA_ADMIN_USERNAME" \
     --password "$PDA_ADMIN_PASSWORD" \
     --email    "$PDA_ADMIN_EMAIL" \
     --firstname Admin --lastname User
   ```

6. Open the firewall — DNS to the world, admin UI and PowerDNS API
   restricted by source IP:

   ```bash
   firewall-cmd --permanent --add-service=dns
   # PowerDNS-Admin UI
   firewall-cmd --permanent --add-rich-rule='rule family="ipv4" \
     source address="YOUR_TRUSTED_CIDR" port port="9090" protocol="tcp" accept'
   # PowerDNS HTTP API / webserver (only published on the public IP via compose)
   firewall-cmd --permanent --add-rich-rule='rule family="ipv4" \
     source address="YOUR_TRUSTED_CIDR" port port="8081" protocol="tcp" accept'
   firewall-cmd --reload
   ```

7. PowerDNS-Admin is now reachable at `http://<server>:9090` from the
   allowlisted IPs. Log in, then under **Settings → PDNS** point it at the
   API: URL `http://pdns:8081`, version `4.x`, and the API key from `.env`.

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
podman-compose ps                                # all four containers Up / healthy
podman logs -f pdns-auth                         # no gmysql errors, API on :8081
podman logs -f powerdns-admin                    # migrations complete, gunicorn ready
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

### 3. PowerDNS HTTP API (proves pdns ↔ MariaDB is healthy)

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
| `status: REFUSED` from `@<DNSDIST_BIND_IP>`    | The zone doesn't exist on this server. Create it in PowerDNS-Admin or via the API first.               |
| `status: SERVFAIL` from `@<DNSDIST_BIND_IP>`   | pdns is up but can't query its backend — check `podman logs pdns-auth` for gmysql errors.              |
| `status: NXDOMAIN`                             | The record name isn't in the zone. Verify it was added with the right name and trailing dot.           |
| Correct locally, NXDOMAIN via `@1.1.1.1`       | Parent delegation isn't published. Update the parent zone's NS + glue and wait for its TTL to expire.  |
| Local answer has no `aa` flag                  | Reply came from a cache, not the authoritative server. Always query the auth IP for validation.        |
| `+trace` stops at `<parent>.`                  | The parent has no NS records pointing at your server, or they point at an unresolvable name (no glue). |

## Troubleshooting

| Symptom                                                | Fix                                                                                                                                       |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `bind: address already in use` on port 53              | See "Port 53" — systemd-resolved or aardvark-dns is holding it. Either move them off 53 or bind dnsdist to a specific host IP.            |
| `ERROR 1045 ... 'pda'@'10.89.0.x'`                     | Password mismatch between `.env` and the user row in MariaDB. `ALTER USER 'pda'@'%' IDENTIFIED BY '...';` to the `.env` value, then restart powerdns-admin. |
| `Unknown MySQL server host 'mysql'`                    | `pdns.conf` still references the old `mysql` service name. It must say `gmysql-host=mariadb`.                                              |
| `Table 'pdns.domains' doesn't exist`                   | DB volume was created before the schema was added. `podman-compose down && rm -rf mysql-data && podman-compose up -d`.                    |
| `ERROR 1410 (42000): not allowed to create user with GRANT` | MariaDB 10.11 inherits MySQL 8's rule that `GRANT` won't auto-create a user. Run `CREATE USER IF NOT EXISTS ...` first.              |
| PowerDNS-Admin login page works but zone list is empty | Settings → PDNS not configured. Set API URL `http://pdns:8081`, API key from `.env`, version `4.x`, save.                                 |

## Notes

- The `:Z` volume labels are SELinux relabeling hints for Podman on RHEL-family
  hosts. Leave them in place.
- `mysql-init/*.sql` runs **only on first MariaDB initialization** (empty data
  directory). Edits made after the first start do not retroactively apply —
  either run them manually with `mariadb -uroot -p` or wipe `mysql-data/` and
  recreate.
- The example `.env` ships with `CHANGE_ME_*` placeholders. Replace them
  before any deployment that's reachable from the network. The `pdns.conf` and
  `01-init.sql` files contain the same placeholders and must be rewritten in
  lockstep.
