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
| `tailscale set --ssh` aborts: "will result in your session disconnecting" | You're enabling Tailscale SSH over a tailnet SSH session | Expected. Re-run with `--accept-risk=lose-ssh`, let the session drop, reconnect |
| Host key warning ("REMOTE HOST IDENTIFICATION HAS CHANGED") right after enabling Tailscale SSH | tailscaled now answers SSH instead of sshd — different host key, not an attack | `ssh-keygen -R <vps-ip-or-name>`, then reconnect |
| SSH to a **tagged** device refused, untagged devices fine | Default ACL `ssh` rule only covers devices you own (`autogroup:self`), which excludes tagged nodes | Admin console → Access Controls → add an `ssh` rule with the tag as `dst` |
| Password auth still works after setting `PasswordAuthentication no` in a drop-in | Drop-in sorts after `50-cloud-init.conf`; OpenSSH keeps the **first** value it sees | Name the file `00-hardening.conf`, restart ssh, verify with `sudo sshd -T \| grep -i passwordauthentication` |
| Connections went relayed (DERP) after enabling ufw | Inbound UDP 41641 blocked, direct path lost | `sudo ufw allow 41641/udp` |
| Agent stuck: `sudo: a terminal is required to read the password` | Agent ran sudo without a TTY | Hand the exact command to the user's terminal, or on macOS wrap it in `osascript ... with administrator privileges` (gives the user a password dialog) |

## Useful commands

```bash
tailscale status            # who's on the tailnet, and how you're connected
tailscale ip -4             # this machine's tailnet IPv4
tailscale ping <machine>    # reachability + direct vs relayed
tailscale netcheck          # NAT/firewall diagnostics
sudo tailscale up --reset   # reapply settings from scratch (keeps the machine on the tailnet)
```
