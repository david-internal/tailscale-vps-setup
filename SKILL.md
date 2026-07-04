---
name: tailscale-vps-setup
description: Connect the user's PC and VPS over Tailscale (a private encrypted WireGuard network), enable Tailscale SSH, and verify the machines can reach each other — no public IPs, no open ports, no SSH key juggling. Use when the user wants to connect a computer to a VPS, set up Tailscale, reach a server securely from anywhere, or can't reach their VPS.
---

# Tailscale VPS Setup

Put the user's PC and VPS on one private network (a "tailnet"). After this skill runs, the user can SSH into the VPS from anywhere using a stable name like `ssh root@my-vps`, with all traffic encrypted end-to-end — even if the VPS firewall blocks every public port.

## What you are setting up (plain English)

- **Tailscale** is a mesh VPN built on WireGuard. Every machine that joins the user's tailnet gets a stable private IP (`100.x.y.z`) and a name (MagicDNS).
- **Tailscale SSH** lets machines SSH to each other over the tailnet with access controlled by Tailscale — no copying key pairs around.
- Free for personal use (up to 100 devices, 3 users).

## Ground rules (read before acting)

1. **Never lock the user out.** Do not disable public SSH or change firewall rules until Tailscale SSH is proven working in a separate, second connection. Keep the original SSH session open the whole time.
2. **Never print or store auth keys or login URLs in files.** Show login URLs to the user directly and let them authenticate in their browser.
3. **Ask before anything destructive** — firewall changes, sshd config edits, killing sessions.
4. One machine at a time: PC first, then VPS, then verify both directions.
5. **`sudo` needs a real terminal.** If you (the agent) can't answer a password prompt, don't fail silently — either hand the exact command to the user to run in their terminal, or on macOS wrap it in `osascript -e 'do shell script "..." with administrator privileges'` so the user gets a password dialog while you keep driving. On macOS, join with `sudo tailscale up --operator=$USER` once — after that, `tailscale` commands (status, serve, up) work without sudo.

## Prerequisites — check these first

- The user has a way into the VPS today (provider SSH over public IP, or the provider's web console as fallback).
- Root or sudo on both machines.
- A Tailscale account (free — they can sign in with Google, GitHub, or Microsoft during setup).

## Step 1 — Install Tailscale on this machine (the PC)

Detect the OS, then install:

| OS | Install |
|---|---|
| macOS | `brew install tailscale` (or the Mac App Store app) |
| Linux | `curl -fsSL https://tailscale.com/install.sh \| sh` |
| Windows | Download the installer from https://tailscale.com/download (or `winget install tailscale.tailscale`) |

If Tailscale is already installed, run `tailscale version` and `tailscale status` and skip ahead.

## Step 2 — Join the PC to the tailnet

```bash
sudo tailscale up                      # Linux / Windows
sudo tailscale up --operator=$USER     # macOS: also lets future tailscale commands run without sudo
```

This prints a login URL. **Show it to the user and wait** — they open it in a browser, sign in, and approve the machine. Then confirm:

```bash
tailscale status
```

The PC should appear with a `100.x.y.z` IP.

## Step 3 — Install Tailscale on the VPS

SSH into the VPS the way the user does today (public IP). Then:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
```

- `--ssh` enables **Tailscale SSH** so the tailnet handles SSH auth from now on.
- This also prints a login URL — show it to the user, wait for approval.
- If the VPS will run agents 24/7, tell the user to disable key expiry for it: Tailscale admin console → Machines → the VPS → "Disable key expiry". Otherwise the VPS silently drops off the tailnet after ~180 days.

**Headless / no-browser path (auth keys).** If nobody can click a login URL on the VPS, the user generates an auth key instead: admin console → Settings → Keys → "Generate auth key" (single-use is safest). Then:

```bash
sudo tailscale up --ssh --auth-key=tskey-auth-...
```

The user pastes the key directly into the terminal — remember ground rule 2: never store it in a file or echo it into shell history you keep. If the key is **tagged** (e.g. `tag:server`), key expiry is disabled automatically — no admin-console step needed. Note: Tailscale SSH *to a tagged device* needs an explicit `ssh` rule in the tailnet ACLs (the default rule only covers untagged devices you own) — if SSH is refused later, check Access Controls first.

**VPS already on the tailnet, just missing Tailscale SSH?** Don't re-run `up` — enable it in place:

```bash
sudo tailscale set --ssh --accept-risk=lose-ssh
```

Two things WILL happen (both fine): (1) if you ran this over a tailnet SSH session, that session drops — reconnect; (2) the host key changes, because tailscaled now answers SSH instead of sshd, so the next connection warns about a mismatched key. Clear it with `ssh-keygen -R <vps-ip-or-name>` and reconnect.

## Step 4 — Verify (do not skip)

From the PC:

```bash
tailscale status          # both machines listed, VPS shows "active" or idle (not offline)
tailscale ping <vps-name> # should get pong; "via DERP" is OK (relayed), direct is better
ssh <user>@<vps-name>     # MagicDNS name from `tailscale status`, e.g. ssh root@my-vps
```

**Only call the setup done when that SSH login works in a fresh terminal.** Report to the user: both machines' tailnet names and IPs, and whether the connection is direct or relayed.

## Step 5 — Optional hardening (ask first)

Only if the user wants it, and only after Step 4 passed:

**Level 1 — close public SSH** so port 22 is reachable **only** over the tailnet:

```bash
sudo ufw allow in on tailscale0 to any port 22 proto tcp
sudo ufw delete allow 22/tcp   # or the provider-firewall equivalent
```

**Level 2 — full lockdown**: the VPS accepts *nothing* from the public internet. This is the right default for an agent server that only ever talks over the tailnet:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 comment "tailnet"
sudo ufw allow 41641/udp comment "tailscale direct connections"
sudo ufw --force enable
```

Keep 41641/udp — that's Tailscale's own WireGuard port; without it connections may fall back to relays (slower). Verify from the PC afterwards: tailnet SSH still works, and `nc -z -w5 <public-ip> 22` times out.

**Level 3 — disable SSH password auth** (so a leaked root password is useless). On Ubuntu, a drop-in file works ONLY if it sorts *before* cloud-init's — OpenSSH takes the **first** value it sees, and `50-cloud-init.conf` often says `PasswordAuthentication yes`:

```bash
printf "PasswordAuthentication no\nKbdInteractiveAuthentication no\n" | \
  sudo tee /etc/ssh/sshd_config.d/00-hardening.conf     # 00- so it sorts before 50-cloud-init.conf
sudo sshd -t && sudo systemctl restart ssh
sudo sshd -T | grep -i passwordauthentication            # must print "passwordauthentication no"
```

Always run that last check — a `99-` file silently loses to cloud-init and password auth stays on.

Warn the user: after Level 2–3, losing Tailscale means falling back to the provider's web console, so they should keep the root password stored somewhere safe.

- In the Tailscale admin console, review ACLs if the tailnet is shared with other people.

## Bonus — expose a local service to the tailnet

Any machine can publish a local-only service (an Ollama server, a dev web app) to the whole tailnet without touching the service's own config — and without exposing it to the internet:

```bash
tailscale serve --bg --tcp=11434 tcp://127.0.0.1:11434   # example: Ollama
tailscale serve status                                    # confirm
tailscale serve --tcp=11434 off                           # undo
```

Other tailnet machines reach it at `http://<machine-name>:11434`. This is how a VPS agent can use a model running on the user's laptop.

## Troubleshooting

See [references/troubleshooting.md](references/troubleshooting.md) for the fix table (expired login URL, relayed connections, MagicDNS not resolving, `tailscale: command not found` on macOS, SSH permission denied).

## Done checklist

- [ ] Both machines visible in `tailscale status`
- [ ] `tailscale ping` succeeds
- [ ] SSH over the tailnet works in a fresh terminal
- [ ] Key expiry disabled on the VPS (if it's a long-running server)
- [ ] User knows the VPS tailnet name and how to connect: `ssh <user>@<vps-name>`
- [ ] If hardened: tailnet SSH re-verified, public port 22 times out from outside, and `sshd -T` shows password auth off

Next step after this skill: the **agent-fleet-manager** skill uses this connection to manage and update every agent on the tailnet.
