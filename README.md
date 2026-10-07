# banas

Ansible automation for the `banas` NAS host. Services run as Docker Compose stacks behind Traefik;
bulk data lives on a Btrfs RAID10 array at `/data`.

## Setup

Ansible lives in a dedicated pyenv virtualenv named `ansible` (Python 3.13.15). `.python-version`
in the repo root activates it automatically on `cd` via `pyenv virtualenv-init`.

To recreate it from scratch:

```
pyenv virtualenv 3.13.15 ansible
pyenv activate ansible
pip install ansible
ansible-galaxy collection install -r requirements.yaml
```

`ansible.cfg` lives in the repo root and is picked up automatically when you run from this
directory — there is no longer any need to hand-write `/etc/ansible/ansible.cfg`.

## Connecting (local or remote)

`inventory.yaml` points at the **`banas` alias from `~/.ssh/config`**, not at a raw IP. That block
supplies the `HostName` and the `ProxyJump` through the VPS, so the same command works on the LAN
and from anywhere:

```
host banas
        HostName 172.16.1.2
        User robert
        IdentityFile ~/.ssh/id_rsa
        ForwardAgent true
        ProxyJump belanecvps
```

To bypass the jump and go straight over the LAN: `-e ansible_host=172.16.1.2`.

The inventory connects as the `ansible` user by default. If that user does not exist yet on the
host, connect as a real sudo user instead and supply the become password:

```
ansible-playbook banas.yaml -e banas_user=robert -K
```

## Bootstrap

Creates the `ansible` user and installs the packages needed to run the rest of the playbooks.
Put the public key at `roles/bootstrap/files/ansible_rsa.pub` first (gitignored via `*pub`), and
put its private half at `~/.ssh/ansible_rsa` (mode `600`) — `ansible.cfg` points
`private_key_file` there, since the `banas` block in `~/.ssh/config` only offers robert's key.

```
ansible-playbook bootstrap.yaml -e target=banas -e banas_user=robert -K
```

For a host that is not in the inventory yet, pass it inline instead:

```
ansible-playbook bootstrap.yaml -u <USER> -e target=<IP> --ask-become-pass -i <IP>,
```

## Running

`OP_API_TOKEN` must be in the environment — it is read via `lookup('env', ...)` and used for every
1Password Connect lookup. Keep it in `.env` (gitignored; `cp .env.example .env` and fill it in) and
load it into the shell before running:

```
set -a; . ./.env; set +a
ansible-playbook banas.yaml
```

Ansible does not read `.env` on its own, so the `set -a` line is required — without it the lookup
returns an empty string and every `onepassword.connect` task fails to authenticate.

`-i inventory.yaml` and `-e target=banas` are no longer needed; both are defaults. Pass
`-e target=<other>` to aim the play somewhere else.

Dry run: `ansible-playbook banas.yaml --check --diff`

### Checking the 1Password lookup

`check_op.yaml` verifies the whole Connect path (token present -> module runs on the host ->
Connect server answers) without printing any secret values:

```
set -a; . ./.env; set +a
ansible-playbook check_op.yaml -e banas_user=robert
```

Use `-e check_item=immich-db` to test a different item. Expected failures:

- `OP_API_TOKEN is not set in this shell` - `.env` was not sourced, or still holds `REPLACE_ME`.
- `Invalid bearer token` - the Connect server was reached, but the token is wrong or expired.
- `Unreachable`/timeout - SSH or the ProxyJump is the problem, not 1Password.


## Btrfs RAID10

Destructive — runs `mkfs.btrfs` across `btrfs_disks`. `target` is deliberately required here:

```
ansible-playbook btrfs_raid10.yaml -e target=banas
```
