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
sudo tailscale up
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

- Close public SSH on the VPS so port 22 is reachable **only** over the tailnet:
  ```bash
  sudo ufw allow in on tailscale0 to any port 22 proto tcp
  sudo ufw delete allow 22/tcp   # or the provider-firewall equivalent
  ```
  Warn the user: after this, losing Tailscale means falling back to the provider's web console.
- In the Tailscale admin console, review ACLs if the tailnet is shared with other people.

## Troubleshooting

See [references/troubleshooting.md](references/troubleshooting.md) for the fix table (expired login URL, relayed connections, MagicDNS not resolving, `tailscale: command not found` on macOS, SSH permission denied).

## Done checklist

- [ ] Both machines visible in `tailscale status`
- [ ] `tailscale ping` succeeds
- [ ] SSH over the tailnet works in a fresh terminal
- [ ] Key expiry disabled on the VPS (if it's a long-running server)
- [ ] User knows the VPS tailnet name and how to connect: `ssh <user>@<vps-name>`

Next step after this skill: the **agent-fleet-manager** skill uses this connection to manage and update every agent on the tailnet.
