# Raspberry Pi Linux Service Hardening & Network Exposure Assessment

A hands-on defensive security lab using a Raspberry Pi 5 to examine exposed services, validate firewall behaviour, harden SSH authentication, and verify the result from a separate machine.

> **Project type:** Defensive system hardening and network exposure assessment  
> **Status:** Initial assessment completed; further improvements are listed under Future Work.

## 1. Objectives

- Establish which network services are listening on the Raspberry Pi.
- Identify services that are not required for the lab and reduce the exposed attack surface.
- Validate the host firewall from a separate machine rather than relying only on local inspection.
- Strengthen SSH authentication while preserving remote administration.
- Retest and record the resulting configuration.

## 2. Lab environment and scope

| Component | Details |
|---|---|
| Target | Raspberry Pi 5 (4 GB RAM) |
| Operating system | Debian GNU/Linux 13 (Trixie), ARM64 |
| Firewall | UFW |
| Remote administration | OpenSSH server |
| Reconnaissance | Nmap |
| Client systems | Windows PowerShell and Ubuntu on WSL |
| Network | Private mobile-hotspot lab network |

**Authorised scope:** The assessment was limited to my own Raspberry Pi and the client systems I used to administer and test it. The target's local IP address is intentionally omitted because it is dynamically assigned and not relevant to reproducing the methodology.

No third-party systems or networks were scanned.

## 3. Methodology

The work followed a PTES-inspired workflow adapted for a small defensive hardening assessment:

1. Define the target and scope.
2. Perform reconnaissance and service enumeration.
3. Review the firewall and service configuration.
4. Identify unnecessary exposure and weak configuration.
5. Apply controlled hardening changes.
6. Validate access and firewall behaviour from a separate client.
7. Record findings, limitations, and future improvements.

This was **not** a full penetration test: no exploitation of a third-party target, privilege-escalation assessment, or post-exploitation activity was performed.

## 4. Reconnaissance and baseline observations

### 4.1 External TCP scan

From the WSL client, the initial full TCP port scan used:

```bash
sudo nmap -sS -sV -sC -p- <PI_IP>
```

The scan identified:

| Port | State | Service | Observation |
|---|---|---|---|
| 22/tcp | Open | SSH | OpenSSH was reachable from the client. |
| Other scanned TCP ports | Closed | — | No TCP listener was accepting connections at the time of this scan. |

The service fingerprint reported OpenSSH `10.0p2` on Debian. Service versions are observations from the scan, not proof that a service is vulnerable.

### 4.2 Internal service and socket review

The Pi was inspected with:

```bash
sudo ss -tulnp
```

The earlier service inventory identified SSH, RPCbind, Avahi, and a Juice Shop container port bound to the Pi's loopback address. The latter was configured to listen on `127.0.0.1:3000`, rather than being directly exposed on the hotspot interface.

The service inventory was used alongside Nmap because the tools answer different questions:

- `ss` shows local listening sockets and their owning processes.
- Nmap shows what a separate machine can observe across the network.

## 5. Changes made

### 5.1 Disabled services not required for this lab

RPCbind and Avahi were not required for the intended lab workflow, so their service and socket units were disabled:

```bash
sudo systemctl disable --now rpcbind.service rpcbind.socket
sudo systemctl disable --now avahi-daemon.service avahi-daemon.socket
```

This reduced unnecessary local service exposure. These services should only be disabled on other systems after checking whether applications or workflows depend on them.

### 5.2 Reviewed UFW policy

The observed UFW policy was:

- Incoming traffic: deny by default.
- Outgoing traffic: allow by default.
- Routed traffic: deny by default.
- Explicit inbound exception: TCP port 22 for SSH (IPv4 and IPv6).

The policy was inspected with:

```bash
sudo ufw status verbose
```

The SSH allow rule was retained so remote administration remained possible.

### 5.3 Tested firewall behaviour with a temporary HTTP service

A temporary Python HTTP server was bound to the Pi's hotspot-facing address on TCP port 8080:

```bash
python3 -m http.server 8080 --bind <PI_IP> --directory /tmp
```

While the server was running, a scan from the Pi itself saw the port as open. A scan from WSL saw it as filtered, and a Windows TCP connection test failed. This demonstrated why a local scan is not a substitute for testing from a separate host.

The temporary server was stopped with `Ctrl+C` after the test. No permanent firewall allow rule was added for port 8080.

### 5.4 Hardened SSH authentication

The effective SSH configuration was reviewed with `sshd -T`. The cloud-init drop-in contained `PasswordAuthentication yes`, so a separate earlier-loading configuration file was created:

`/etc/ssh/sshd_config.d/00-lab-hardening.conf`

Contents:

```text
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
MaxAuthTries 3
X11Forwarding no
```

The configuration was syntax-checked and the effective settings verified before reloading SSH:

```bash
sudo sshd -t
sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|pubkeyauthentication|maxauthtries|x11forwarding|kbdinteractiveauthentication)'
sudo systemctl reload ssh
```

Key-based access was configured independently from both Windows and WSL. Each client has its own Ed25519 key pair; the corresponding public keys were added to the Pi user's `authorized_keys` file. Private keys remain on their respective client systems.

## 6. Validation and results

### 6.1 SSH authentication tests

The following tests were completed:

| Test | Result |
|---|---|
| WSL connection using public-key authentication only | Successful |
| Windows PowerShell connection using public-key authentication only | Successful |
| Password-only SSH authentication attempt | Failed, as intended |
| Existing SSH session after configuration reload | Remained usable |

The key passphrase may still be requested by the client. This is separate from the Raspberry Pi account password: disabling SSH password authentication does not remove encryption on the private key.

### 6.2 Final external Nmap scan

The final full TCP scan was run from WSL:

```bash
sudo nmap -sS -sV -sC -vvv -p- <PI_IP>
```

Observed result:

```text
Not shown: 65534 filtered tcp ports (no-response)
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack         OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
```

The final scan found SSH as the only open TCP port. The other scanned TCP ports were reported as filtered (no response). This is consistent with the default-deny inbound firewall policy. Nmap's `filtered` state does not, by itself, prove which firewall or network device performed the filtering; the result was interpreted together with the UFW policy and the controlled port-8080 test.

### 6.3 Final local socket review

After the service changes, the Pi's `ss -tulnp` output showed SSH listening on IPv4 and IPv6 port 22. No other listening sockets appeared in that captured output.

## 7. Findings and security impact

| Finding | Action | Validation |
|---|---|---|
| RPCbind was running but was not required for this lab | Disabled its service and socket units | Later local socket inventory no longer showed RPCbind; the final TCP scan found no open TCP/111 |
| Avahi was running but was not required for this lab | Disabled its service and socket units | Later local socket inventory no longer showed Avahi; an external UDP scan was not performed |
| SSH allowed password authentication | Disabled password and keyboard-interactive authentication; enabled key-based access | Password-only test failed; key-only tests succeeded from Windows and WSL |
| SSH root login was not fully disabled | Set `PermitRootLogin no` | Effective configuration reported `permitrootlogin no` |
| SSH allowed six authentication attempts | Reduced `MaxAuthTries` to 3 | Effective configuration verified with `sshd -T` |
| X11 forwarding was enabled although not needed for this headless lab | Set `X11Forwarding no` | Effective configuration verified with `sshd -T` |
| Inbound firewall policy needed external validation | Retained deny-by-default policy and tested a temporary service from a separate client | Port 8080 was filtered from WSL and the Windows TCP connection test failed |

## 8. Limitations

- The assessment focused on listening services, firewall behaviour, and SSH configuration; it was not a comprehensive vulnerability assessment.
- UDP exposure was inspected locally with `ss`, but a complete external UDP Nmap assessment was not performed.
- The hotspot network and target IP are dynamic, so the exact network addressing is not included.
- The SSH allow rule permits connections from `Anywhere` at the host-firewall rule level. Public-key-only authentication reduces password-guessing risk, but a future improvement is to restrict SSH source addresses where the network design permits it.
- The final Nmap scan reports network-visible port states; it does not establish that the SSH version has no known vulnerabilities or that the whole operating system is secure.

## 9. Future work

- Restrict the SSH firewall rule to trusted source addresses or a trusted subnet where practical.
- Perform a deliberate external UDP scan and compare it with the local socket inventory.
- Review remaining enabled systemd services and justify each one against the lab's requirements.
- Add repeatable before-and-after evidence files and a lab network diagram.
- Extend the lab with a separate deliberately vulnerable application, keeping it isolated from the normal administration surface.

## 10. Reproducibility and safety notes

Replace `<PI_IP>` with the Raspberry Pi's current private address when repeating commands. Do not publish private SSH keys, passwords, hotspot credentials, personal IP details, or unredacted host-specific secrets.

Useful commands from this assessment:

```bash
# Local listening sockets
sudo ss -tulnp

# UFW policy
sudo ufw status verbose

# Effective SSH settings
sudo sshd -T

# Validate SSH configuration syntax
sudo sshd -t

# External full TCP scan (run from a separate client)
sudo nmap -sS -sV -sC -p- <PI_IP>
```

---

**Key takeaway:** Hardening is not just changing settings. It is checking the system from the administrator's perspective, testing it from a separate machine, and confirming that the intended controls actually work.
