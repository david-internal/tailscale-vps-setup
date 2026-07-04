# Tailscale Setup — Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Login URL expired / nothing happened | User took too long to open it | Re-run `sudo tailscale up` — it prints a fresh URL |
| `tailscale: command not found` on macOS after brew install | brew installs the daemon but it isn't started | `sudo brew services start tailscale`, or use the GUI app from tailscale.com; App Store version puts the CLI at `/Applications/Tailscale.app/Contents/MacOS/Tailscale` |
| `tailscale ping` says "via DERP" | Direct UDP path blocked (NAT/firewall), traffic relays through Tailscale servers | Still secure and works; for direct paths allow UDP 41641 outbound. Not worth fighting on locked-down networks |
| MagicDNS name doesn't resolve | MagicDNS disabled on the tailnet | Admin console → DNS → enable MagicDNS. Meanwhile use the `100.x.y.z` IP from `tailscale status` |
| `ssh` over tailnet: permission denied | VPS wasn't brought up with `--ssh`, or ACLs block SSH | Re-run `sudo tailscale up --ssh` on the VPS. Check admin console → Access controls (default policy allows SSH between your own devices) |
| VPS shows offline in `tailscale status` | tailscaled not running, or machine key expired | On the VPS: `sudo systemctl enable --now tailscaled`, then `sudo tailscale up --ssh`. Disable key expiry in the admin console for servers |
| Two machines with the same hostname | Cloned VPS images | `sudo tailscale up --ssh --hostname=<unique-name>` |
| Locked out of VPS after firewall change | Public SSH closed before tailnet SSH was verified | Use the VPS provider's web console to undo the firewall rule. This is why Step 5 comes after Step 4 |

## Useful commands

```bash
tailscale status            # who's on the tailnet, and how you're connected
tailscale ip -4             # this machine's tailnet IPv4
tailscale ping <machine>    # reachability + direct vs relayed
tailscale netcheck          # NAT/firewall diagnostics
sudo tailscale up --reset   # reapply settings from scratch (keeps the machine on the tailnet)
```
