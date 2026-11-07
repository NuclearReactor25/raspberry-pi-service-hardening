# Evidence to add

Add redacted evidence from your own lab runs here. Do not add private keys, passwords, hotspot credentials, or unredacted personal network details.

Suggested files:

- `nmap-initial.txt` — initial external TCP scan showing SSH reachable.
- `services-before.txt` — relevant `ss -tulnp` output before disabling services (redact usernames or paths if desired).
- `ufw-policy.txt` — `sudo ufw status verbose` output.
- `ssh-effective-settings.txt` — filtered `sudo sshd -T` output after hardening.
- `ssh-key-auth-success.txt` — evidence that key-only authentication succeeded (do not include passphrases).
- `ssh-password-auth-failure.txt` — evidence that password-only authentication failed.
- `nmap-final.txt` — final external Nmap scan.
- `services-after.txt` — final `sudo ss -tulnp` output.

When saving terminal output, review it first and replace the target IP with `<PI_IP>` if you do not want to publish your private network address.
