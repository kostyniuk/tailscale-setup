# Ubuntu: Tailscale + SSH (headless-ready)

Goal: SSH into this machine over Tailscale at any time.

## 1. Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo systemctl enable tailscaled
```

## 2. SSH server (not installed by default on Ubuntu desktop)
```bash
sudo apt update && sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```
- `ssh.service` may show inactive on newer Ubuntu: it starts on demand through `ssh.socket`. That is expected.
- Alternative: skip openssh and use `sudo tailscale up --ssh` (Tailscale SSH).

## 3. Never suspend
Locked or logged out is fine. Suspended or hibernated is not reachable.
```bash
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```
Run `gsettings` as your normal user, not with sudo.

## 4. Ignore lid close (laptop only)
```bash
sudo sed -i -E 's/^#?HandleLidSwitch=.*/HandleLidSwitch=ignore/; s/^#?HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
sudo systemctl kill -s HUP systemd-logind
```
Use `HUP`, not `restart`. A restart can log out the desktop session.

## 5. Firewall (only if ufw is enabled)
```bash
sudo ufw allow in on tailscale0
```

## Verify
```bash
tailscale status                                   # on any machine: host listed
systemctl status sleep.target                      # masked
grep HandleLidSwitch /etc/systemd/logind.conf      # =ignore, no leading #
```
From client: `tailscale ping <host>` then `ssh <user>@<host>`.

## Client shortcut (~/.ssh/config)
```
Host <host>
  HostName <host>
  User <user>
```

## Notes
- Wake-on-LAN does not work over Tailscale. Keep the machine awake.
- `<host>` resolves via MagicDNS. If not, use the 100.x IP from `tailscale status`.

## Undo
```bash
sudo systemctl unmask sleep.target suspend.target hibernate.target hybrid-sleep.target
gsettings reset org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type
gsettings reset org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type
```
