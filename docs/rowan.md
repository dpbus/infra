# rowan — home server notes

Always-on Linux box: TLS-terminating reverse proxy for the LAN, future home for
Docker / Home Assistant / k3s. Named for the rowan tree, traditionally planted
by gates to guard the way in.

---

## Hardware

| Item | Detail |
|---|---|
| Machine | Beelink EQi 304 (Intel Core 3 304, Wildcat Lake) |
| RAM | 24 GB DDR5 SO-DIMM, **single slot**, max 64 GB (upgrade = replace stick; go to 48/64 GB) |
| Boot disk | Samsung 990 PRO 1 TB NVMe in the **PCIe 4.0 x2** slot |
| Second M.2 | PCIe 4.0 x1, empty (future mirror / data disk) |
| Onboard UFS | 512 GB UFS 3.1 — **not used by Linux** (Windows left on it for BIOS/Samsung Magician updates, if kept) |
| NICs | `enp1s0` = Realtek RTL8127 **10GbE** (`r8169`), `enp88s0` = Intel i226-V **2.5GbE** (`igc`), `wlp2s0` = MediaTek Wi-Fi (unused) |
| Console | GL.iNet KVM |

Warranty: 3 years from Beelink. Check before opening the case for RAM swaps.

---

## OS

- **Debian 13 (trixie)**, headless, SSH server + standard utilities only.
- Disk: guided LVM, whole disk used by root. **No free space in the VG.**
  Growing is easy; shrinking root requires booting rescue/live media via the KVM
  (`lvreduce -r`), so back up first.
- `/tmp` is tmpfs (trixie default).

### Kernel — trixie-backports

Stock trixie kernel (6.12) is too old for this hardware (no 10GbE, broken GPU).
Running the backports kernel (7.2.x+bpo at install time).

`/etc/apt/sources.list.d/debian-backports.sources`:

```
Types: deb
URIs: http://deb.debian.org/debian
Suites: trixie-backports
Components: main contrib non-free non-free-firmware
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

Installed from backports:

```bash
sudo apt install -t trixie-backports linux-image-amd64 linux-headers-amd64 \
  firmware-linux-nonfree firmware-realtek firmware-intel-graphics firmware-intel-misc
```

- Backports packages keep updating via normal `apt upgrade` once installed.
- The 6.12 kernel stays installed as a fallback: GRUB → **Advanced options**
  (press Esc at boot if the menu is hidden).
- **Missing GPU firmware = black screen on the console** (system still boots and
  SSH works). Fix was the Intel firmware packages above + `sudo update-initramfs -u`.
  Find which package has a file: `apt-file search <filename>`.
- Check firmware health: `sudo dmesg | grep -i firmware`

### Plan for Debian 14 (forky), expected ~summer 2027

Forky's stock kernel will support this hardware. After upgrading:
remove the backports `.sources` file, reinstall `linux-image-amd64` from stable,
and drop the backports firmware pins.

---

## Network

| | |
|---|---|
| Interface | `bond0` (active-backup) |
| IP | **10.0.10.55** (fixed IP in UniFi, keyed to bond0's MAC) |
| Primary | `enp1s0` 10GbE |
| Backup | `enp88s0` 2.5GbE (takes over automatically if 10G drops) |
| mDNS | `rowan.local` (avahi-daemon) |
| DNS | `rowan.busse.net` (UniFi local A record) |

`/etc/network/interfaces` (backup of the original at `interfaces.bak`):

```
source /etc/network/interfaces.d/*

auto lo
iface lo inet loopback

auto bond0
iface bond0 inet dhcp
    bond-slaves enp1s0 enp88s0
    bond-mode active-backup
    bond-primary enp1s0
    bond-miimon 100
```

Requires the `ifenslave` package.

Check bond state:

```bash
cat /proc/net/bonding/bond0     # active slave, link speeds, failure counts
```

Watch `Link Failure Count` on enp1s0 — if it keeps climbing, the Realtek
link or driver is flapping.

Future idea: repurpose `enp88s0` for a separate VLAN (IoT / k8s) instead of
bond backup.

---

## Domain & DNS

- **busse.net** — registrar: **Hover**. DNS hosting: **Cloudflare** (nameservers
  moved from Hover). Email and the GitHub Pages site records live in Cloudflare.
  GitHub Pages records are DNS-only (grey cloud).
- **Cloudflare API token**: "Edit zone DNS" template, scoped to the busse.net zone
  only. Stored on rowan in `/etc/caddy/caddy.env`. If it leaks, revoke it in
  Cloudflare → My Profile → API Tokens and create a new one.
- **Local names** are UniFi local DNS records (Settings → Policy Table → DNS on
  Network 9.4+). UniFi has no wildcard local records, so each name needs an entry:

| Name | Type | Target |
|---|---|---|
| `rowan.busse.net` | A | 10.0.10.55 |
| `gaia.busse.net` | CNAME | rowan.busse.net |
| `kvm.busse.net` | CNAME | rowan.busse.net |
| `channels.busse.net` | CNAME | rowan.busse.net |

Every service name points at **rowan**, not at the device. Caddy routes by hostname.

These names only resolve for clients using the UDM for DNS.

---

## Caddy (reverse proxy + certificates)

### How it's installed

1. Caddy's official apt repo (Cloudsmith) — provides the systemd service,
   `caddy` user, `/etc/caddy`, `/var/lib/caddy`.
2. A **custom binary with the Cloudflare DNS plugin** swapped in using Caddy's
   documented method (caddyserver.com/docs/build → "Package support files"):

```bash
sudo dpkg-divert --divert /usr/bin/caddy.default --rename /usr/bin/caddy
sudo mv ./caddy /usr/bin/caddy.custom
sudo update-alternatives --install /usr/bin/caddy caddy /usr/bin/caddy.default 10
sudo update-alternatives --install /usr/bin/caddy caddy /usr/bin/caddy.custom 50
```

- `/usr/bin/caddy` → symlink → `caddy.custom`
- apt updates only `caddy.default`. **apt never updates the custom binary.**
- Switch binaries: `sudo update-alternatives --config caddy`, then restart.

### Files

| Path | What |
|---|---|
| `/etc/caddy/Caddyfile` | Config (`Caddyfile.orig` = package default) |
| `/etc/caddy/caddy.env` | `CF_API_TOKEN=...` — root:root, mode 600, no quotes/spaces |
| `/etc/systemd/system/caddy.service.d/override.conf` | `EnvironmentFile=/etc/caddy/caddy.env` |
| `/var/lib/caddy/` | Certificates + ACME account (recreatable) |

### Current Caddyfile

```
{
	email you@busse.net
}

*.busse.net {
	tls {
		dns cloudflare {env.CF_API_TOKEN}
		resolvers 1.1.1.1
	}

	@gaia host gaia.busse.net
	handle @gaia {
		reverse_proxy https://10.0.10.68:5001 {
			transport http {
				tls_insecure_skip_verify
			}
		}
	}

	@kvm host kvm.busse.net
	handle @kvm {
		reverse_proxy https://<KVM-IP> {
			transport http {
				tls_insecure_skip_verify
			}
		}
	}

	handle {
		abort
	}
}
```

- One **wildcard cert** for `*.busse.net` via DNS-01 (Let's Encrypt). Caddy renews
  it automatically ~30 days before expiry.
- `tls_insecure_skip_verify` = don't validate the device's self-signed cert on the
  LAN hop. Browsers only ever see Caddy's real cert.
- `resolvers 1.1.1.1` = check challenge propagation against public DNS, not the UDM.
- Unknown names hit `abort` (connection dropped).

### Common commands

```bash
# Validate (must load the token, or it errors with "API token '' appears invalid")
sudo bash -c 'set -a; source /etc/caddy/caddy.env; caddy validate --config /etc/caddy/Caddyfile'

sudo caddy fmt --overwrite /etc/caddy/Caddyfile   # fix formatting/indentation
sudo systemctl reload caddy                        # apply config, no downtime; keeps old config if new one is bad
sudo journalctl -u caddy -f                        # live log
sudo journalctl -u caddy -n 50 --no-pager          # recent log
```

### Add a new service

1. UniFi: CNAME `<name>.busse.net` → `rowan.busse.net`
2. Caddyfile: add a block inside `*.busse.net`, above the final `handle { abort }`:
   ```
   @name host name.busse.net
   handle @name {
   	reverse_proxy http://10.0.10.X:PORT
   }
   ```
   (Use `https://` + the `tls_insecure_skip_verify` transport if the device only serves HTTPS.)
3. Validate (with env) → reload.
4. No new cert needed — the wildcard covers it.

### Update the custom Caddy binary (every few months)

Bump `caddy_custom_version` in `roles/caddy/defaults/main.yml` to the new
release tag, then run the caddy role. It downloads the pinned version, verifies
the version and the Cloudflare module, and restarts Caddy.

```bash
ansible-playbook site.yml --check --diff --tags caddy
ansible-playbook site.yml --tags caddy
```

---

## Channels DVR

Native install in `/usr/local/channels-dvr`, running as the `channels` system
user (groups video, render). Web UI at `https://channels.busse.net` (Caddy →
`127.0.0.1:8089`). No authentication on the local API or UI; LAN is trusted.

| Unit | What |
|---|---|
| `channels-dvr.service` | the DVR; requires `/mnt/media` to be mounted |
| `channels-refresh.timer` | hourly: fetch guide updates, then refresh virtual channels |

Channels self-updates inside its own directory. The binaries live in
`<version>/` subdirectories with a `latest` symlink.

```bash
sudo systemctl start channels-refresh.service     # run the refresh now
systemctl list-timers channels-refresh.timer      # last/next run
journalctl -u channels-refresh.service -n 20      # refresh output
curl -s http://127.0.0.1:8089/log | tail -50      # DVR log
```

Undocumented API calls the web UI's quick actions make (all unauthenticated):

```bash
curl -X PUT http://127.0.0.1:8089/dvr/guide                   # Fetch Guide Updates
curl -X PUT "http://127.0.0.1:8089/dvr/guide?force=true"      # Re-Download Entire Guide
curl -X PUT http://127.0.0.1:8089/dvr/lineups/X-VIRTUAL       # Refresh Virtual Channels
curl -X PUT http://127.0.0.1:8089/dvr/scanner/scan            # Scan Personal Media
curl -X PUT http://127.0.0.1:8089/dvr/scanner/imports/prune   # Prune Removed Media
curl -X DELETE http://127.0.0.1:8089/dvr/cache                # Clear Streaming Cache
```

---

## Troubleshooting

**A new name works with `dig` but not in curl/browser (Mac)**
`dig` bypasses the macOS resolver cache. Flush it:
```bash
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
dscacheutil -q host -a name <name>.busse.net    # tests the system resolver
```
Then restart the browser. Still broken? Check `scutil --dns` for Tailscale /
Private Relay / browser secure-DNS taking over.

**Name resolves to the device's IP instead of rowan's**
UniFi auto-creates DNS from DHCP client hostnames. Use an explicit A record to
10.0.10.55 for that name.

**Cert errors in Caddy log**
- 403 / auth / invalid token → token wrong, revoked, or not scoped to busse.net.
- "timed out waiting for record to fully propagate" → usually retries fine; wait.

**Console (KVM) is black after boot**
GPU firmware missing — see Kernel section. SSH still works.

**Locked out of SSH**
Use the KVM console. Password auth is disabled; root login is disabled.

**Network broken after editing interfaces**
From the KVM console: `sudo cp /etc/network/interfaces.bak /etc/network/interfaces && sudo reboot`

---

## Base system config

- **sudo**: user is in the `sudo` group.
- **SSH**: key-only. `/etc/ssh/sshd_config.d/10-hardening.conf`:
  ```
  PasswordAuthentication no
  KbdInteractiveAuthentication no
  PermitRootLogin no
  ```
  Test with `sudo sshd -t` before `sudo systemctl reload ssh`.
- **Security updates**: `unattended-upgrades` (security only, no auto-reboot).
  Kernel/backports updates: manual `sudo apt update && sudo apt upgrade` + reboot.
- **Editor**: `sudo update-alternatives --config editor` → vim.basic.
  Edit root-owned files with `sudoedit <file>` (uses my vim config).
- **Admin tools PATH**: `/usr/sbin` and `/sbin` added to PATH in zshrc.
- **Ghostty**: `shell-integration-features = ssh-terminfo` in the Ghostty config
  installs the terminfo on SSH hosts (fixes backspace/arrows in zsh).
- **zsh**: `bindkey -e` in zshrc (EDITOR=vim otherwise flips zsh to vi keys).
- **Dotfiles**: github.com/dpbus/dotfiles

---

## Backups worth having

- `/etc/caddy/` (Caddyfile + caddy.env) — the hard-to-recreate part
- `/etc/network/interfaces`
- `/etc/ssh/sshd_config.d/`
- this file

---

## Future plans

- Docker (from Docker's own apt repo) — consider a separate LV for `/var/lib/docker`
- Home Assistant: container on Debian, or HAOS in a VM (HAOS can't see the UFS drive)
- k3s
- Remote access: Tailscale split DNS for `busse.net` → UDM, plus a subnet route
- RAM upgrade to 48/64 GB when k8s needs it
- Debian 14 upgrade → drop backports
