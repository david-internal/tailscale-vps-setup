# Tailscale VPS Setup — Claude skill

A Claude Code skill that connects your PC and your VPS over [Tailscale](https://tailscale.com) — a private, encrypted WireGuard network. It installs Tailscale on both machines, enables Tailscale SSH, and verifies the connection end to end. No public IPs, no open ports, no SSH key juggling.

## Install

1. Download or clone this repo.
2. Drop the folder into your agent's skills directory (e.g. `~/.claude/skills/tailscale-vps-setup/`).
3. Ask your agent: *"Set up Tailscale between my PC and my VPS."*

## What's inside

- [`SKILL.md`](SKILL.md) — the skill instructions (step-by-step setup with lockout-prevention rules)
- [`references/troubleshooting.md`](references/troubleshooting.md) — fix table for common issues

## Companion skill

[`agent-fleet-manager`](https://github.com/david-internal/agent-fleet-manager) — once your machines are on one tailnet, one agent can manage and update every other agent and model on the network.

---

From [David Ondrej's video](https://www.davidondrej.com/tailscale-agent-network) — grab both skills (plus a copy in your inbox) there.
