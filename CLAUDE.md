# infra — home network IaC

Ansible code for my home server **rowan**. rowan was configured by hand first;
this repo codifies it. Detailed notes on the current setup: `docs/rowan.md`.

## Ground rules

- **rowan is live.** Never run a playbook for real until `--check --diff` shows
  only intended changes. Workflow for every role:
  1. Read the actual files on rowan over SSH (`ssh rowan.local sudo cat ...`).
     rowan is the source of truth; `docs/rowan.md` may be slightly off.
  2. Write the role to match the current state.
  3. `ansible-playbook site.yml --check --diff --tags <role>` until the diff
     is empty (or only shows changes I've agreed to).
- **Network and kernel roles are dangerous.** A bad bond0 or interfaces change
  cuts off SSH. Build these last, and assume I need the GL.iNet KVM console as a
  fallback. Never restart networking in a handler without asking.
- **Secrets** go in Ansible Vault (`host_vars/rowan/vault.yml`), never plaintext.
  Secrets: the Cloudflare API token and `ansible_become_password` (sudo).
  The vault password comes from 1Password: `.vault_pass` (gitignored) is a
  script running `op read`, wired up via `vault_password_file` in ansible.cfg.
  No `-K` or `--ask-vault-pass` needed.
- Run from my Mac (`brew install ansible`). No agent on rowan.

## Layout

```
ansible.cfg            # inventory = inventory.yml, roles_path = roles, pipelining
inventory.yml          # rowan: ansible_host rowan.local, ansible_user dpb
site.yml               # hosts: rowan, become: true, all roles below
host_vars/rowan/vars.yml
host_vars/rowan/vault.yml
roles/base kernel network nfs caddy channels
```

Old Ansible code from years ago may still be in the repo history/tags; it's
not a reference for anything here.

## rowan facts

- Beelink EQi 304 (Intel Wildcat Lake), Debian 13 trixie, user `dpb`
- IP **10.0.10.55** (UniFi fixed IP), `rowan.local` via avahi, `rowan.busse.net`
- NICs: `enp1s0` 10GbE Realtek (r8169), `enp88s0` 2.5GbE Intel (igc)
- NAS **gaia** = Synology RS422+ at 10.0.10.68

## Decisions already made (don't revisit without asking)

- **Kernel from trixie-backports** (stock 6.12 lacks support for this hardware:
  no 10GbE, black console). Plus backports Intel firmware
  (`firmware-intel-graphics`, `firmware-intel-misc`, `firmware-realtek`,
  `firmware-linux-nonfree`). Plan: drop backports after Debian 14 (forky).
- **bond0, active-backup**, primary enp1s0, ifupdown + ifenslave, DHCP.
- **busse.net**: registrar Hover, DNS on Cloudflare. Local names are UniFi
  local DNS records (CNAMEs → rowan.busse.net). DNS records are not managed by
  Ansible (maybe OpenTofu later).
- **Caddy runs natively, not in Docker** — critical infra stays out of the
  container/k3s experimentation layer. Installed from Caddy's official apt repo,
  then a **custom binary with the caddy-dns/cloudflare plugin** swapped in via
  Caddy's documented dpkg-divert + update-alternatives method
  (`/usr/bin/caddy.default` priority 10, `/usr/bin/caddy.custom` priority 50).
  apt never updates the custom binary.
- Caddy: one wildcard cert `*.busse.net` via DNS-01 (Cloudflare, zone-scoped
  token). Token in `/etc/caddy/caddy.env` (root, 600) loaded by a systemd
  drop-in `EnvironmentFile=`. Host matchers per service, final `handle { abort }`.
  Validating requires the env: `caddy validate` fails without CF_API_TOKEN.
- **NFS** from gaia: `10.0.10.68:/volume1/Media` → `/mnt/media`, NFSv4.1,
  **x-systemd.automount** (a boot-time mount races bond0/DHCP and times out).
  fstab options: `vers=4.1,_netdev,nofail,noatime,x-systemd.automount,x-systemd.mount-timeout=30`.
  DSM side squashes all users to admin (manual, not managed here).
- **Channels DVR**: native install (not Docker, not k3s), into
  `/usr/local/channels-dvr`, running as a **dedicated system user `channels`**
  (groups video, render; nologin shell) via a systemd drop-in that sets
  `User=`/`Group=` and `RequiresMountsFor=/mnt/media`. Channels self-updates.
  Migrating from the old server on Ursa (Mac mini). Clients are Apple TVs
  (direct play), so hardware transcoding is nice-to-have.
- SSH: key-only, no root login (`/etc/ssh/sshd_config.d/10-hardening.conf`).
- unattended-upgrades: security only, no auto-reboot.

## Status / next steps

1. Scaffold the repo and confirm `ansible rowan -m ping`.
2. Roles in this order: base → nfs → caddy → channels → network → kernel.
3. Channels DVR is **not installed yet** — write the `channels` role to do the
   install fresh (official setup script, then the user/ownership/drop-in changes),
   rather than codifying a hand install.
4. Later: Docker, Home Assistant, k3s (GitOps for in-cluster stuff), Tailscale
   split DNS for remote access, maybe OpenTofu for Cloudflare/UniFi DNS.



















