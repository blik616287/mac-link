# mac-link

Shell access and a filesystem mount from an Ubuntu workstation ("u26") to a MacBook Pro
over a direct **Thunderbolt** cable. No sudo or admin rights are needed on the Mac, and
macOS Remote Login stays off.

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

## Setup

Edit `group_vars/all.yml` if you want a different subnet, ports or mount path.

1. **On the u26** (prompts for your sudo password):
   ```bash
   ansible-playbook u26.yml -K
   ```
   Plug in the Thunderbolt cable. `ip -br addr show thunderbolt0` should show `10.77.0.1`.

2. **Copy the repo to the Mac**, over the cable using your normal u26 login:
   ```bash
   scp -r <u26-user>@10.77.0.1:<path>/mac-link ~/
   ```

3. **On the Mac**, install Ansible without sudo and run the playbook:
   ```bash
   curl -LsSf https://astral.sh/uv/install.sh | sh   # or: python3 -m pip install --user ansible-core
   ~/.local/bin/uv tool install ansible-core
   cd ~/mac-link && ~/.local/bin/ansible-playbook mac.yml
   ```
   If macOS asks whether Terminal may "administer your computer" when the crontab is
   installed, click **OK**. That prompt does not need an admin password.

4. **Copy the Mac's info back and finish on the u26**:
   ```bash
   # on the Mac
   scp ~/mac-link/exchange/mac_info.yml <u26-user>@10.77.0.1:<path>/mac-link/exchange/
   # on the u26
   ansible-playbook u26.yml -K
   ```

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

`exchange/` holds per-machine public keys and is git-ignored.
