# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Ansible automation for a single home NAS host (`banas`, 172.16.1.2, Ubuntu). Every user-facing
service runs as a Docker Compose stack behind Traefik; data lives on a Btrfs RAID10 array mounted
at `/data`.

## Commands

`ansible.cfg` is in the repo root and is auto-loaded when running from this directory; it sets
`inventory = inventory.yaml`, `roles_path = roles`, `remote_user = ansible` and global `become`.
Collections come from `requirements.yaml` (`community.docker`, `onepassword.connect`), installed to
`~/.ansible/collections`.

Ansible is **not** on the system PATH — it lives in the pyenv virtualenv `ansible` (3.13.15,
ansible-core 2.21). `.python-version` activates it on `cd`; from a non-interactive shell call
`~/.pyenv/versions/3.13.15/envs/ansible/bin/ansible-playbook` directly.

```bash
set -a; . ./.env; set +a         # OP_API_TOKEN; .env is gitignored, .env.example is the template
ansible-playbook banas.yaml      # converge everything; -i and -e target=banas are defaults now

ansible-playbook banas.yaml --check --diff
ansible-playbook check_op.yaml               # verify the 1Password Connect path end to end
ansible-playbook banas.yaml --start-at-task="Start Mealie container with Docker Compose"

# Destructive, and `target` is intentionally NOT defaulted here (runs mkfs.btrfs)
ansible-playbook btrfs_raid10.yaml -e target=banas

# Creates the `ansible` user; needs roles/bootstrap/files/ansible_rsa.pub (gitignored)
ansible-playbook bootstrap.yaml -e target=banas -e banas_user=robert -K
```

## Connection model

`inventory.yaml` sets `ansible_host: banas` — an **`~/.ssh/config` alias, not an IP**. OpenSSH
resolves it to `172.16.1.2` behind `ProxyJump` via the VPS (the `belanecvps` alias), which is what makes off-LAN runs work.
Do not "fix" this back to a bare IP; that silently drops the jump host. Direct LAN runs override it
with `-e ansible_host=172.16.1.2`.

That alias only offers robert's key, so `ansible.cfg` sets `private_key_file = ~/.ssh/ansible_rsa`
for the `ansible` user (it must be mode `600` or SSH silently skips it).

`ansible_user` is `{{ banas_user | default('ansible') }}`. Inventory vars outrank the `-u` flag in
Ansible's precedence, so `-e banas_user=robert -K` is the supported way to connect as a real sudo
user — needed whenever the `ansible` user has not been bootstrapped on the target.

## Architecture

**`banas.yaml` is the source of truth for what is deployed.** Roles exist in `roles/` that are *not*
in that play (`nextcloud`, `librephotos`, `obsidian-livesync`, `monitoring` — the last is an empty
stub). Adding a role directory does nothing until it is listed in `banas.yaml`.

**Per-service role pattern.** Almost every service role repeats the same shape, and new services
should copy it:

1. `file:` create `{{ <service>_config_dir }}` (conventionally `/opt/<service>`, set in `defaults/main.yaml`).
2. `command: btrfs subvolume create /data/{{ btrfs_subvolume }}` with `args.creates:` for idempotency
   — only for services that store bulk data. `btrfs_subvolume` is defined per role as `@<service>`.
3. If secrets are needed: `onepassword.connect.item_info` against item `<service>-db` in vault
   `{{ ansible_vault }}`, then `set_fact` to unpack `op_item.op_item.fields.<x>.value` into
   `dbName`/`dbUser`/`dbPass`/`dbHost`.
4. `template:` the `.env` from `templates/<service>.env.j2` into the config dir — this is how secrets
   reach the container without landing in git.
5. `copy:` the static `files/docker-compose.yaml` into the config dir.
6. `community.docker.docker_compose_v2` with `project_src: {{ <service>_config_dir }}`, `state: present`.

Because compose files are copied verbatim (`copy`, not `template`), **they must not contain Jinja**.
Values that vary go through the `.env` file and are referenced as `${dbUser}`-style shell
interpolation inside the compose YAML. Note this means host paths in compose files are hardcoded
(e.g. `/opt/mealie/data`) even though the role has a `*_config_dir` variable — keep both in sync.

**Traefik routing.** Every stack joins the external Docker network `proxy` (created by the Traefik
compose file) and exposes itself purely through container labels:

```yaml
- "traefik.enable=true"
- "traefik.http.routers.<name>.rule=Host(`<sub>.belanec.eu`)"
- "traefik.http.routers.<name>.entrypoints=websecure"
- "traefik.http.routers.<name>.tls.domains[0].main=<sub>.belanec.eu"
- "traefik.http.routers.<name>.tls.domains[0].sans=*.belanec.eu"
- "traefik.http.routers.<name>.tls=true"
- "traefik.http.services.<name>.loadbalancer.server.port=<container port>"
```

TLS is *not* ACME-on-demand. The `certbot` role installs certbot + the Cloudflare DNS plugin, and
the `traefik` role copies `/etc/letsencrypt/live/belanec.eu/{fullchain,privkey}.pem` into
`/opt/traefik/certs/` where Traefik's file provider (`providers.file.directory: /certs`, watched)
picks them up via `files/certs_tls.yml`. Certificate issuance/renewal itself is not automated here.

**`opengym` deviates from the pattern** in three ways worth knowing before copying it: it needs no
1Password lookup (no database, no secrets), it defines a second `internal` network so only `web`
joins `proxy` — which is why its labels carry `traefik.docker.network=proxy`; `traefik.yml` sets no
default network, so any service on more than one network needs that label or Traefik may route via
a network it isn't on (504) — and it publishes no host port (upstream's default `WEB_PORT=8080` would collide with
the Traefik dashboard). Its `.env` is templated from plain role variables rather than `set_fact`.

Its public URL differs from its Traefik Host, the same way as every service fronted by the VPS
(`belanecvps`): nginx there terminates `gym.belanec.eu` and proxies to `https://172.16.1.2` with
`Host: gym-ba.belanec.eu`. Passkeys bind to the public name (`opengym_hostname` → `RP_ID`/`ORIGIN`),
so the label keeps `gym-ba` while the env says `gym.belanec.eu`; don't "sync" one to the other.

Its signup lockdown is deliberately two-phase: `INVITE_ONLY=1` on a fresh instance locks everyone
out, because invite codes are minted from the admin dashboard and that requires a registered admin
(the `INVITE_ONLY` signup gate in upstream `api/server.js`). Deploy open, register, read the uid from
`/data/@opengym/data/db.json`, set `opengym_admin_uids`, re-run, then set `opengym_invite_only`.

**`pihole` publishes a host port on purpose.** DNS can't go through Traefik, so `53/tcp+udp` is
published — on `172.16.1.2` only (`pihole_dns_bind`), because systemd-resolved already holds `:53`
on `127.0.0.53/54` and the wildcard bind would collide. The web UI goes through Traefik as
`pihole-ba.belanec.eu` and is deliberately LAN-only: there is no VPS proxy for it, and there should
not be one. `FTLCONF_dns_listeningMode: ALL` is required, since LAN clients are not on the bridge
subnet. Its 1Password item (`pihole`, field `password`) is created by hand — the Connect token is
read-only (`generic_item` creates fail with "token does not have permission to perform create").

**1Password Connect (`op`) must be running for any role with a lookup.** Its compose file had no
`restart:` policy, so the containers stayed down after a reboot; it now has `unless-stopped`. If
lookups fail, check `docker ps -a | grep op-` before suspecting the token.

**Ordering matters in `banas.yaml`:** `common` installs Docker, `traefik` creates the `proxy`
network, `certbot`/`op` must precede anything doing 1Password lookups at runtime. New service roles
belong after `op`.

## Conventions

- All YAML uses the `.yaml` extension (not `.yml`), except the two Traefik config files it ships.
- Timezone is `Europe/Bratislava` everywhere.
- `.gitignore` excludes `*pub` — `roles/bootstrap/files/ansible_rsa.pub` must be placed manually
  before running `bootstrap.yaml`.
