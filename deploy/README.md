# VM setup

How the VM that hosts this project was prepared. Steps are listed in the order
they were done.

Placeholders used below:

| Placeholder        | Meaning                                  |
| ------------------ | ---------------------------------------- |
| `<user>`           | Non-root admin user on the VM            |
| `<vm-public-ip>`   | Public IP address of the VM              |
| `<vm-hostname>`    | Hostname of the VM                       |
| `<tailscale-ip>`   | The VM's Tailscale IP (`100.x.y.z`)      |
| `<port>`           | Port a stack's Caddy listens on          |

## Context: shared VM

Another app already runs on this VM behind Traefik, which binds ports 80 and
443 on the host network. This project must **not** use 80/443. Instead, each
stack runs its own Caddy that publishes only on the Tailscale IP, on its own
port (e.g. `<tailscale-ip>:<port>`), so it is reachable only from the tailnet.

## 1. Update the OS, create a sudo user, install an SSH key

*Why: start from patched packages and stop working as root day to day.*

```sh
# as root
apt update && apt full-upgrade -y
adduser <user>
usermod -aG sudo <user>

# from your local machine
ssh-copy-id <user>@<vm-public-ip>
```

Confirm you can log in as `<user>` with the key and run `sudo` before moving on.

## 2. Harden SSH

*Why: key-only logins remove password brute-forcing as an attack vector.*

Create `/etc/ssh/sshd_config.d/00-hardening.conf`:

```
PasswordAuthentication no
PermitRootLogin prohibit-password
```

The `00-` prefix makes it load first; sshd uses the first value it sees for
each option, so these win over later drop-ins (e.g. cloud-init's).

```sh
sudo sshd -t && sudo systemctl reload ssh
```

Keep the current session open and test a fresh login in a second terminal.

## 3. Firewall (ufw)

*Why: deny everything inbound except what the VM actually needs to serve.*

```sh
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp     # Traefik (other app)
sudo ufw allow 443/tcp    # Traefik (other app)
sudo ufw allow in on tailscale0
sudo ufw enable
sudo ufw status verbose
```

Ports used by this project's Caddy instances are not opened publicly; they are
reachable through the `tailscale0` rule only.

> Note: Docker-published ports bypass ufw via iptables. Binding Caddy to
> `<tailscale-ip>` (not `0.0.0.0`) is what keeps it off the public interface.

## 4. Docker access

*Why: lets `<user>` run `docker` / `docker compose` without `sudo`.*

Docker was already installed on the VM.

```sh
sudo usermod -aG docker <user>
# log out and back in, then:
docker ps
```

## 5. Tailscale

*Why: private, authenticated access to the VM and to this project's services
without exposing new public ports.*

```sh
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
tailscale ip -4   # -> <tailscale-ip>
```

Then, in the Tailscale admin console, **disable key expiry** for
`<vm-hostname>` so the server doesn't silently drop off the tailnet when its
node key expires.

`--ssh` enables Tailscale SSH, so you can also connect with
`ssh <user>@<vm-hostname>` over the tailnet, governed by tailnet ACLs.
