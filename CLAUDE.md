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
  apt never updates the custom binary. Its version is pinned by
  `caddy_custom_version` in the caddy role; bump it to upgrade.
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
  (groups video, render; nologin shell). The official `setup.sh` is run with
  `DOWNLOAD_ONLY=1` (binaries + layout only); Ansible owns the systemd unit,
  which sets `User=`/`Group=` and `RequiresMountsFor=/mnt/media`. Channels
  self-updates inside its own directory.
  Migrating from the old server on Ursa (Mac mini). Clients are Apple TVs
  (direct play), so hardware transcoding is nice-to-have.
- SSH: key-only, no root login (`/etc/ssh/sshd_config.d/10-hardening.conf`).
- unattended-upgrades: security only, no auto-reboot.

## Status / next steps

1. All six roles are written and converged: a full `--check --diff` run is
   clean. `site.yml` runs them in rebuild order: base → kernel → network →
   nfs → caddy → channels. A rebuild needs a reboot after kernel and again
   after network; neither role reboots on its own.
2. Channels DVR is installed and served at `channels.busse.net`. Remaining
   setup is in its web UI: recordings directory under `/mnt/media`, migration
   from Ursa.
3. Open question: unattended-upgrades is on Debian's stock origins (security
   plus point-release updates), not security-only as decided above. Tighten
   via a `52unattended-upgrades-local` drop-in if wanted.
4. Later: Docker, Home Assistant, k3s (GitOps for in-cluster stuff), Tailscale
   split DNS for remote access, maybe OpenTofu for Cloudflare/UniFi DNS.



















