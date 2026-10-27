# mac-link

Shell access and a filesystem mount from an Ubuntu workstation ("u26") to a **stock**
MacBook Pro over a direct **Thunderbolt** cable. Nothing is installed on the Mac: no sudo,
no admin rights, no Homebrew, no Python, and macOS Remote Login stays off. The Mac side
only uses tools that ship with macOS (`ssh`, `sshd`, `crontab`, `shlock`).

```
 u26 (Ubuntu, sudo)                                   Mac (no sudo)
 ─────────────────                                    ─────────────
 thunderbolt0 10.77.0.1/24  ◄── Thunderbolt cable ──► Thunderbolt Bridge (DHCP from u26)
 sshd :22  ◄──────────── reverse tunnel (Mac dials out as "mactunnel") ──── ssh -N -R
 127.0.0.1:2222 ═══════════════════════════════════════════════════════► 127.0.0.1:2222
   ▲                                                                     user-level sshd
   ├── ssh mac          (shell)                                          (runs as you)
   └── sshfs → ~/mac    (Mac home dir)
```

- **u26**: NetworkManager runs the Thunderbolt link in `shared` mode, so the Mac's
  Thunderbolt Bridge (DHCP by default) gets an address with no admin rights. sshd gets
  a locked-down `mactunnel` account: key only, from the Thunderbolt subnet only, one
  reverse forward to `127.0.0.1:2222`, no shell.
- **Mac**: `/usr/sbin/sshd` runs as your own user on loopback with its own host key.
  `ssh -R` publishes it on the u26.
- **Recovery**: a cron job runs every minute (and at boot) on each machine.
  - On the Mac it restarts the sshd and tunnel when they die. `ServerAliveInterval`
    and `ExitOnForwardFailure` make a broken tunnel exit instead of hanging.
  - On the u26 it detaches stale sshfs mounts and remounts once the tunnel is back.
  - sshd on the u26 drops dead tunnel sessions after about 45 s, which frees port 2222.

## How the automation is split

Stock macOS has no Python, so Ansible can't run *on* the Mac. Instead:

| Playbook  | Runs on | Does |
|-----------|---------|------|
| `u26.yml` | u26 | Configures the u26: Thunderbolt link, sshd and the `mactunnel` account, sshfs and its cron watchdog. Also **generates the Mac's keys** and renders `exchange/mac-install.sh`, a self-contained bash 3.2 installer for the Mac. |
| `mac.yml` | u26, against the Mac through the tunnel | Pushes the current installer to the Mac and runs it. It uses only the `script` module, which needs no Python on the Mac. Use it for later changes. |

`mac-install.sh` is idempotent. It writes everything to `~/.u26link`, installs the
crontab entries and starts the user-level sshd and the tunnel.

## Setup

1. **On the u26.** Set `mac_user` in `group_vars/all.yml` to your short user name on
   the Mac (run `id -un` there). Then run (prompts for your sudo password):
   ```bash
   ansible-playbook u26.yml -K
   ```
   Plug in the Thunderbolt cable. `ip -br addr show thunderbolt0` should show `10.77.0.1`.

2. **On the Mac, once.** In Terminal, using your normal u26 login over the cable:
   ```bash
   scp <u26-user>@10.77.0.1:<repo>/exchange/mac-install.sh ~/ && bash ~/mac-install.sh && rm ~/mac-install.sh
   ```
   The playbook prints this exact line at the end. If macOS asks whether Terminal may
   "administer your computer" when the crontab is installed, click **OK**. It doesn't
   need an admin password.

3. **Back on the u26**, within about a minute: `ssh mac` and `ls ~/mac`.

Later changes: edit vars or templates, then run `ansible-playbook u26.yml -K` followed
by `ansible-playbook mac.yml`.

## Use

```bash
ssh mac                        # shell on the Mac
cp bigfile ~/mac/Downloads/    # through the sshfs mount
rsync -aP dir/ mac:somewhere/  # faster for large or bulk copies
```

## Notes

- **Protected Mac folders.** Desktop, Documents and Downloads are guarded by macOS
  privacy (TCC). A cron-launched sshd may get `Operation not permitted` in them. Either
  use another folder, for example `~/u26-inbox` (set `mac_share_path`), or add
  `/usr/sbin/cron` under *System Settings → Privacy & Security → Full Disk Access*.
  Managed Macs may need an admin for that.
- **Sleep.** When the Mac sleeps, the link drops. It reconnects within about a minute of waking.
- **Default route.** NetworkManager's shared mode offers the u26 as a gateway over
  Thunderbolt. macOS normally keeps Wi-Fi first in the service order, so this has no
  effect. If it does, move Thunderbolt Bridge down in *Network → … → Set Service Order*.
- **Speed.** Traffic is encrypted twice: once by the tunnel and once by the inner ssh.
  AES-GCM is preferred to keep that cheap.
- **Logs.** u26: `~/.local/state/mac-link/mount.log`. Mac: `~/.u26link/*.log`.
- **Undo.**
  - u26: `crontab -e`, `nmcli con delete tb-mac`, `sudo userdel -r mactunnel`, and
    remove the mac-link blocks from `/etc/ssh/sshd_config` and `~/.ssh/config`.
  - Mac: `crontab -e`, `kill $(cat ~/.u26link/*.pid)`, `rm -rf ~/.u26link`.

`exchange/` holds generated keys, including the **Mac's private keys** and the installer
that embeds them. It is git-ignored. Keep it to yourself.
