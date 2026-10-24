# Two-Node Proxmox Home Lab

A practical self-hosted infrastructure and cybersecurity training environment built using two Intel N200 mini PCs, Proxmox VE, Ubuntu, Docker, OPNsense, and Kali Linux.

> Documentation is based on the recorded build walkthrough. Commands marked **Example** are reusable references and are not asserted to have been executed exactly as written. GUI operations are documented separately. All testing is restricted to systems owned or explicitly authorized by the operator.

The full chronological build record, with step-by-step procedures and test results, is in [docs/build-log.md](docs/build-log.md).

## Goals
- Keep production home networking separate from experimental security VMs.
- Host reliable DNS and uptime monitoring.
- Practice virtualization, Linux administration, firewalls, troubleshooting, reconnaissance, and SSH hardening.
- Maintain recoverable VM snapshots and document operational changes.

## Hardware and network architecture

| Component | Purpose | Private lab address |
|---|---|---|
| ISP router | Production gateway and DHCP | `192.168.50.1` |
| Managed switch | Production switching | `192.168.50.2` |
| Proxmox node 1 | Production infrastructure | `192.168.50.10` |
| Ubuntu VM 100 | AdGuard Home + Uptime Kuma | `192.168.50.20` |
| Proxmox node 2 | Isolated cybersecurity lab | `192.168.50.11` |
| Ubuntu VM 101 | Secondary Uptime Kuma + AdGuard Home replica | `192.168.50.21` |
| OPNsense VM 102 | Lab firewall/DHCP/DNS | LAN `10.20.10.1`, WAN `192.168.50.5` (static) |
| Ubuntu Desktop VM 103 | Lab administration | `10.20.10.157` |
| Kali VM 104 | Security testing | `10.20.10.173` (DHCP; may change) |
| Target VM 105 (`target-01`) | Practice target on the TARGETS network | `10.20.20.x` (DHCP from OPNsense) |

- Node 2 `vmbr0`: production physical bridge, attached to home LAN.
- Node 2 `vmbr1`: virtual-only isolated lab bridge; **no physical port, host IP, or gateway**.
- OPNsense VM102: `net0` on `vmbr1` (LAN), `net1` on `vmbr0` (WAN).
- Ubuntu Desktop VM103 and Kali VM104: **one NIC each, on `vmbr1` only**.
- Node 2 `vmbr2`: second virtual-only bridge for lab targets (**no physical port, host IP, or gateway**). OPNsense has a third NIC (`net2`) on it, serving the TARGETS network `10.20.20.0/24`. Target VMs have one NIC, on `vmbr2` only.
- OPNsense routes lab internet traffic via the home LAN while enforcing lab firewall rules. This is logical isolation, not a physically air-gapped network.

**Note on addresses:** the IP addresses and device names in this repository are examples, not real values. Adjust them for your own network. Do not publish real MAC addresses, credentials, public IPs, host keys, or router configuration exports.

## Proxmox VE host administration

Run on the intended **Proxmox host shell** as root (or with appropriate privileges):

```bash
pveversion -v                 # Proxmox component versions
hostnamectl                  # Hostname and OS
ip -br addr                  # Interface/address summary
ip route                     # Routing table
cat /etc/network/interfaces  # Proxmox bridge configuration (read-only)
pvesm status                 # Storage availability
lsblk                        # Block devices
vgs && lvs                  # LVM volume groups and logical volumes
free -h                      # Memory
lscpu                        # CPU capabilities
ls -l /dev/kvm               # Hardware virtualization device
qm list                      # Virtual machines on this node
pct list                     # Containers on this node
```

Connectivity and package maintenance:

```bash
ping -c 4 192.168.50.1
ping -c 4 1.1.1.1
getent hosts debian.org
apt update
apt list --upgradable
# Review upgrade changes before running:
apt full-upgrade
```

**Example — inspect a VM and guest agent (node 2):**

```bash
qm config 104
qm status 104
qm agent 104 ping
qm agent 104 network-get-interfaces
```

**Example — list VM snapshots:**

```bash
qm listsnapshot 102
qm listsnapshot 104
```

**GUI instructions:** Open `https://<PROXMOX_NODE_IP>:8006`. Create VM disks on `local-lvm`, upload official OS ISOs to `local` ISO storage, enable QEMU Guest Agent under VM Options, and configure start-at-boot only for VMs that must run continuously. Proxmox nodes remain standalone; do not assume cluster features or live migration are available.

**Safety:** Back up `/etc/network/interfaces` and ensure console access before modifying bridges. Never attach `vmbr1` to a physical port while treating it as isolated.

### Proxmox access hardening (implemented on both nodes)

Because the nodes are standalone, each node has its own user list, so this was done on each node separately.

1. **Datacenter → Permissions → Users → Add:** a named user in the **Proxmox VE authentication server** realm (`<name>@pve`) with a long unique password stored in a password manager.
2. **Datacenter → Permissions → Add → User Permission:** path `/`, that user, role **Administrator**, propagate checked.
3. **Datacenter → Permissions → Two Factor → Add → TOTP** for that user (scan the QR code with an authenticator app, verify the code), then **Add → Recovery Keys** and store the one-time codes in the password manager.
4. Test the login in a private browser window (realm "Proxmox VE authentication server", password, then the TOTP code) before relying on it.
5. Use the named account for everyday work. Keep `root@pam` as the emergency account with a long unique password; some host-level tasks (such as the node shell) may still require it. Add two-factor to `root` only after its recovery keys are saved.

If TOTP codes are rejected even though they are correct, check the node's clock (Node → System → Time).

## Ubuntu VM setup and troubleshooting

Commands below run **inside the Ubuntu VM**, not on the Proxmox host.

```bash
hostnamectl
lsb_release -a
ip -br addr
ip route
resolvectl status
ping -c 4 192.168.50.1   # Production Ubuntu VMs only
ping -c 4 1.1.1.1
getent hosts example.com
sudo apt update
sudo apt upgrade
```

**Example — DHCP renewal (NetworkManager-managed desktop):**

```bash
nmcli device status
nmcli connection show
# Replace the connection name with the exact name shown above:
sudo nmcli connection down '<CONNECTION_NAME>'
sudo nmcli connection up '<CONNECTION_NAME>'
```

**Example — check systemd-networkd (server installs):**

```bash
networkctl status
sudo netplan get
# Apply netplan only after validating the configuration and ensuring console access.
```

Install and check QEMU Guest Agent:

```bash
sudo apt update
sudo apt install -y qemu-guest-agent
systemctl is-active qemu-guest-agent
systemctl status qemu-guest-agent
```

**Note:** The guest-agent unit can be static or socket/device activated. A message that it cannot be `enabled` is not by itself a failure; check that it is active and that Proxmox reports guest information.

**GUI instructions:** Create Ubuntu VM100 on node 1 and VM101/103 on node 2. Use router DHCP reservations for production Ubuntu IPs; OPNsense Dnsmasq provides the reservation for Ubuntu VM103. Do not configure a second production DHCP server.

## Docker Engine and Compose on Ubuntu

These are **example installation commands** for supported Ubuntu releases; consult the current Docker installation documentation before executing on a new system. The original build successfully installed Docker Engine and Compose, but its exact repository setup commands were not preserved.

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
. /etc/os-release
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu ${VERSION_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl status docker
sudo docker run --rm hello-world
```

Docker operational commands:

```bash
sudo docker ps
sudo docker ps -a
sudo docker images
sudo docker logs --tail 100 <CONTAINER_NAME>
sudo docker inspect <CONTAINER_NAME>
sudo docker compose ps
sudo docker compose logs --tail 100
sudo docker compose pull
sudo docker compose up -d
sudo docker stats --no-stream
```

**Caution:** Membership in the `docker` group grants root-equivalent privileges. Never commit Docker `.env` files containing secrets.

## AdGuard Home — VM100 on node 1

Deployment: Docker on Ubuntu `192.168.50.20`; DNS on port `53`. The dashboard was configured on port `8080` in the recorded environment. Persist AdGuard work/config directories across container recreation.

**Example — inspect container and DNS listeners:**

```bash
sudo docker ps
sudo docker logs --tail 100 adguardhome
sudo ss -lntup | grep -E ':(53|8080)\b'
```

**Example — test DNS explicitly against AdGuard:**

```bash
dig @192.168.50.20 example.com
nslookup example.com 192.168.50.20
```

**Windows PowerShell examples:**

```powershell
Resolve-DnsName example.com -Server 192.168.50.20
nslookup example.com 192.168.50.20
ipconfig /all
ipconfig /flushdns
```

**GUI instructions:** Configure AdGuard via its initial setup wizard, verify queries in Query Log, and choose browsing security options appropriate for the household. Browser secure DNS, VPN clients, and IPv6 DNS can bypass expected IPv4 DNS settings; test each path independently. The original troubleshooting found a third-party VPN client interfered with some DNS tests.

### Redundant DNS — AdGuard Home replica on VM101

A second AdGuard Home runs in Docker on VM101 (`192.168.50.21`, web UI port `8080`). [AdGuardHome-sync](https://github.com/bakito/adguardhome-sync) copies settings **one way** from the primary on VM100 to the replica (at startup and every 10 minutes). Make configuration changes on the primary only; edits on the replica are overwritten. Query logs and statistics are per instance and are not copied.

Setup summary (Example commands; run on VM101):

```bash
# Free port 53 (disable only the systemd-resolved stub listener):
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nDNSStubListener=no\n' | sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
getent hosts example.com          # the VM itself must still resolve names
sudo ss -lntup | grep ':53 '      # expect no output before AdGuard starts

# From the replica project folder (compose file defines adguard-replica + adguardhome-sync):
sudo docker compose up -d adguard      # then finish the setup wizard on port 3000
sudo docker compose up -d              # starts the sync container
sudo docker compose logs --tail 50 adguardhome-sync
```

Do not set the VM's own DNS to `127.0.0.1` while setting this up; it would point at a container that is not running yet and image pulls would fail. Keep the compose `.env` file (primary and replica credentials) out of GitHub.

**Confirmed:** the replica answers DNS directly and blocks the same test domain as the primary (`dig @192.168.50.21 doubleclick.net`).

**Router behavior (ISP-supplied consumer router):** the router's DNS setting under the broadband connection sets the DNS the *router* uses upstream; it did not change what LAN clients receive. Clients still use the router (`192.168.50.1`) as their DNS server, and the router forwards queries to `192.168.50.20` and `192.168.50.21`. Consequences: filtering works for IPv4 and IPv6 lookups to the router (tested with a blocked domain on both), but AdGuard sees queries as coming from the router, so per-device statistics and rules are not available. To get per-device visibility, set DNS manually on individual devices to both AdGuard addresses.

**Failover test (confirmed):** with the primary container stopped on VM100, a client flushed its DNS cache and lookups still resolved through the replica. Restart the primary afterward (`sudo docker start adguardhome`).

**Do not:** Change router-wide DNS without a verified fallback. Test the replica directly before pointing the router at it.

## Uptime Kuma — monitoring on both nodes

- VM100: `http://192.168.50.20:3001`
- VM101: `http://192.168.50.21:3001`
- Monitors included AdGuard dashboard, DNS lookups, Proxmox HTTPS, Ubuntu SSH, router ping, and switch ping.
- DNS-type monitors for both AdGuard servers (`192.168.50.20` and `192.168.50.21`) exist in both Kuma instances, so a server that is up but not answering queries is detected.
- Each Kuma instance watches the Proxmox node that the other one runs on (Kuma #1 watches node 2; Kuma #2 watches node 1), so if a node goes down the Kuma on the surviving node sends the alert.
- A ping monitor for the OPNsense WAN address (`192.168.50.5`) exists in both Kuma instances, made possible by the narrow WAN ICMP rule described in the OPNsense section.
- **Second alert channel (Discord):** each Kuma instance posts to a private Discord channel through its own webhook (named for the instance, so alerts show which Kuma sent them), in addition to the email SMTP alerts. In Kuma: Settings → Notifications → Setup Notification → Discord, paste the webhook, enable "Default enabled" and "Apply on all existing monitors", and use Test. **Confirmed:** Discord messages arrived for the test message and for a temporary monitor's DOWN and UP transitions on both instances. A Discord webhook URL is a secret: anyone who has it can post to the channel. Never publish it; if it leaks, delete the webhook in Discord and create a new one.
- Email (SMTP) alerts were configured and both DOWN and UP notifications were tested.

**Example — inspect container:**

```bash
sudo docker ps
sudo docker logs --tail 100 uptime-kuma
sudo ss -lntp | grep ':3001'
```

**GUI instructions:** Create HTTP(s), TCP port, DNS, and ping monitors as appropriate. Configure email SMTP credentials in the notification settings, send a test notification, then test a temporary DOWN→UP transition. Remove temporary test monitors afterward.

**Security:** Never publish email app passwords, SMTP tokens, or notification URLs. Monitoring dashboards should not be exposed to the public internet without access controls.

## OPNsense isolated firewall — VM102 on node 2

**GUI build procedure:**
1. Create `vmbr1` on Proxmox node 2 with no physical ports, host IP, or gateway.
2. Create OPNsense VM102 (2 vCPU, 4GB RAM, 32GB disk) with two VirtIO NICs.
3. Assign OPNsense LAN `vtnet0` / `vmbr1` = `10.20.10.1/24`.
4. Assign OPNsense WAN `vtnet1` / `vmbr0` = static `192.168.50.5/24` with upstream gateway `192.168.50.1` (ISP router). Make sure this address is outside the router DHCP pool or reserved on the router, so no other device is handed the same address.
5. Configure Dnsmasq for lab DHCP `10.20.10.100–10.20.10.200` and DNS.
6. Add static DHCP reservation `10.20.10.157` for Ubuntu Desktop VM103.
7. Since WAN is private RFC1918, uncheck WAN **Block private networks**; keep **Block bogon networks** enabled as tested.
8. Set restrictive LAN rules: allow needed management HTTPS to firewall from VM103, DNS to firewall, DHCP, ICMP echo to firewall, block IPv6, block lab-to-private networks, and allow remaining IPv4 internet access.
9. Verify `10.20.10.0/24` clients reach internet while attempts to access `192.168.50.1` are blocked in OPNsense Live View.
10. Save OPNsense snapshot `baseline-secure-lab` (without RAM).

**Important rule-order notes:** Rules are evaluated top-down. The recorded configuration includes an IPv6 block; targeted HTTPS/DNS/DHCP/ICMP allowances; a disabled broad 'LAN net → This Firewall' rule retained for recovery; RFC1918 and production-LAN blocks; and an outbound internet allowance. DHCP initial broadcasts can require a broad source/destination match for UDP port 67. This describes the recorded lab, **not** a generic production-hardened ruleset.

**WAN rule for monitoring (added later):** OPNsense blocks all inbound traffic on the WAN by default, including ping, so the Uptime Kuma servers could not check that the firewall was alive. A narrow rule now allows it: **Pass**, interface WAN, direction in, IPv4, protocol ICMP, type Echo Request, source alias `KUMA_HOSTS` (`192.168.50.20`, `192.168.50.21`), destination WAN address. Take a VM102 snapshot before changing firewall rules (`pre-kuma-icmp`). **Confirmed:** `ping -c 3 192.168.50.5` replied from VM100 and VM101; from the Windows laptop the ping timed out and the tests against TCP ports 443 and 22 on the WAN address failed. This confirms the firewall is reachable from the home side only; it does not test the lab LAN.

**Example — inspect client networking:**

```bash
ip -br addr
ip route
nslookup example.com 10.20.10.1
ping -c 4 10.20.10.1
ping -c 4 1.1.1.1
curl -I https://www.kali.org
# Expected to be blocked by the lab firewall:
ping -c 4 192.168.50.1
```

**Caution:** OPNsense firewall rules do not filter traffic between Kali and Ubuntu Desktop VM103 because both are on the same L2 subnet (`vmbr1`). Lab *targets* are on a separate network (see "Target network" below), so Kali-to-target traffic does pass through the firewall; the Kali/VM103 pair remains a shared segment.

**Anti-lockout rule (disabled):** OPNsense adds an automatic "anti-lockout" rule that is evaluated before user rules and lets any LAN host reach the firewall's web interface. Live View showed Kali (`10.20.10.173`) reaching `https://10.20.20.1` through it, so it was disabled (System → Settings → Administration). Management access now depends on the explicit rule allowing VM103 (`10.20.10.157`) to port 443. **Confirmed:** VM103 can still log in to the OPNsense web interface after the change. Before changing this, take a VM102 snapshot and know the recovery path: the Proxmox console of VM102, then the shell option and `pfctl -d` (temporarily disables the firewall) and `pfctl -e` (re-enables it).

## Target network (TARGETS) — `vmbr2` on node 2

A separate network behind OPNsense for machines to practice against, so traffic from Kali passes through the firewall instead of staying on one segment.

**Build summary (GUI):**
1. Snapshot VM102 first (`pre-targets-network`, no RAM).
2. Node 2 → System → Network → Create → Linux Bridge `vmbr2` with Bridge ports, IPv4/CIDR, Gateway and IPv6 all empty, then click **Apply Configuration**. If it is not applied, starting VM102 fails with "bridge 'vmbr2' does not exist".
3. Shut down VM102, add a third VirtIO NIC on `vmbr2`, and start it. Confirm the lab still works.
4. OPNsense: Interfaces → Assignments → add `vtnet2` as `TARGETS`; enable it with static IPv4 `10.20.20.1/24`.
5. Services → Dnsmasq DNS & DHCP: make sure TARGETS is a listening interface and add a DHCP range `10.20.20.100–10.20.20.200`, then save and apply.
6. Create target VM 105 (`target-01`, Ubuntu Server with OpenSSH, no QEMU guest agent) with its only NIC on `vmbr2`.

**TARGETS rules (top-down, all logged):**

| # | Action | Protocol | Source | Destination | Purpose |
|---|---|---|---|---|---|
| 1 | Block | IPv6 | any | any | No IPv6 |
| 2 | Pass | UDP | any | any, port 67 | DHCP |
| 3 | Pass | TCP/UDP | TARGETS net | TARGETS address, port 53 | DNS |
| 4 | Pass | ICMP echo request | TARGETS net | TARGETS address | Ping gateway |
| 5 | Block | IPv4 | any | any | Everything else, visible in the log |

Targets therefore get an address and DNS but cannot reach the Kali network, the home network, or the internet. For updates, temporarily add a pass rule from TARGETS net to any and disable it afterwards.

**Rules added on LAN (above "Block LAN net → PRIVATE_NETWORKS"):** a block for LAN net → the firewall's TARGETS address `10.20.20.1` (log), then directly below it a pass for LAN net → `10.20.20.0/24` ("Lab to targets", log). The block must sit above the pass, otherwise Kali could reach the firewall's web interface through the pass rule. Reverse-shell exercises (target connecting back to Kali) are blocked by design; add a narrow, logged pass rule on a specific port only when needed.

**DHCP troubleshooting (target shows "autoconfiguration failed"):** check that the VM's NIC is on `vmbr2` and not disconnected; that Interfaces → Overview shows TARGETS up with `10.20.20.1/24`; that Dnsmasq lists TARGETS and has a DHCP range for it (save and apply to restart the service); that the DHCP rule has destination port 67 and no source port; and watch Firewall → Log Files → Live View filtered to TARGETS while the target retries.

**Confirmed tests:**
- On the target: DHCP address `10.20.20.x`, default route via `10.20.20.1`, ping to `10.20.20.1` and DNS lookup via `10.20.20.1` work; pings to `1.1.1.1`, `10.20.10.1` and `192.168.50.1` fail.
- From Kali: the target is reachable (ping, port 22). Before the anti-lockout rule was disabled, `curl -kI --max-time 5 https://10.20.20.1` returned a response from the firewall; after disabling it, both `https://10.20.20.1` and `https://10.20.10.1` time out from Kali.
- VM103 can still open the OPNsense web interface after the anti-lockout rule was disabled. The VM102 snapshot `targets-network-verified` was taken after the tests passed.

## Kali Linux — VM104 on node 2

Installed from official `kali-linux-2026.3-installer-amd64.iso`, with SHA-256 checked against official published sums. VM104: 2 vCPU (host CPU), 4GB RAM, 40GiB disk, Xfce, one VirtIO NIC attached **only to `vmbr1`**. DHCP address observed `10.20.10.173`. QEMU Guest Agent installed and running. Snapshot: `kali-clean-install`.

**Example — verify downloaded Kali ISO checksum:**

```bash
sha256sum kali-linux-2026.3-installer-amd64.iso
# Compare the result manually to Kali's official SHA256SUMS for that exact release.
```

**Commands executed/verified during lab setup:**

```bash
ip -br addr
ip route
ping -c 4 10.20.10.1
ping -c 4 1.1.1.1
nslookup example.com 10.20.10.1
curl -I https://www.kali.org
sudo apt update
sudo apt install qemu-guest-agent -y
sudo systemctl enable --now qemu-guest-agent
systemctl is-active qemu-guest-agent
nmap --version
```

The `systemctl enable --now` command produced a static-unit warning; `systemctl is-active` returned `active`.

**GUI instructions:** Enable guest agent in Proxmox VM Options; verify VM Summary shows the guest IP; snapshot before further experimentation.

## Authorized Nmap discovery and service testing

**Only scan lab VMs you own or are explicitly permitted to test.** The following targets are confined to the recorded lab subnet.

```bash
# Confirm Ubuntu VM103 is online without a port scan:
sudo nmap -sn 10.20.10.157

# Scan five specific TCP ports on Ubuntu VM103:
sudo nmap -sS -p 22,53,80,443,3389 10.20.10.157

# Rescan SSH after starting the service:
sudo nmap -sS -p 22 10.20.10.157

# Detect SSH service version:
sudo nmap -sV -p 22 10.20.10.157
```

**Observed results:** Ubuntu VM103 initially returned `closed` for all five ports. After installing and starting OpenSSH Server, port 22 was `open`. Nmap detected OpenSSH 10.2, corroborated by a local version check. A detected version is **not** proof of a vulnerability.

**Interpretation:** `open` = a listener accepts connections; `closed` = host responds but no listener; `filtered` = Nmap cannot determine state because of filtering or another network condition.

## OpenSSH service, remote login, and key-based authentication

**Ubuntu VM103 — recorded setup and checks:**

```bash
systemctl status ssh             # Initially: unit not found
sudo apt update
sudo apt install openssh-server -y
sudo systemctl status ssh        # Initially inactive after install
sudo systemctl start ssh
systemctl is-active ssh         # active
ssh -V                          # OpenSSH 10.2 reported
sudo sshd -T | grep -E '^(passwordauthentication|permitrootlogin|pubkeyauthentication) '
```

**Observed effective SSH settings (initial install):**
- `permitrootlogin prohibit-password`
- `passwordauthentication yes`
- `pubkeyauthentication yes`

These were later hardened; see the key-based authentication section below.

**Kali VM104 — connect to Ubuntu VM103:**

```bash
ssh <UBUNTU_USERNAME>@10.20.10.157
# On first connection, verify host fingerprint on Ubuntu before accepting.
```

**Ubuntu VM103 — check host fingerprint:**

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

**Inside SSH session — verify remote host, then disconnect:**

```bash
hostname
hostname -I
exit
```

**SSH key authentication (completed):**

```bash
# Run on Kali. Check for an existing key first; never overwrite one.
ls -l ~/.ssh/
ssh-keygen -t ed25519 -C 'kali-lab'        # strong passphrase

# Copy only the public key to Ubuntu VM103 (password login still enabled at this point):
ssh-copy-id <UBUNTU_USERNAME>@10.20.10.157
ssh -o PreferredAuthentications=publickey <UBUNTU_USERNAME>@10.20.10.157
```

**Confirmed:** key login from Kali to VM103 succeeded and prompted for the key passphrase, not the account password.

**Disabling password authentication (completed on VM103):**

Safety steps first: take a Proxmox snapshot of VM103 (`pre-ssh-hardening`), keep one existing SSH session open, and know that the Proxmox console is the recovery path. On Ubuntu, the first value read wins and a `50-cloud-init.conf` file may set `PasswordAuthentication yes`, so the drop-in is named to sort first.

```bash
# Ubuntu VM103 — see which files already set the option:
sudo grep -ri PasswordAuthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/

# Create the drop-in, validate, apply, and verify:
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t                                # no output means the config is valid
sudo systemctl reload ssh
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin) '
```

Test from Kali in a **new** session before closing the original one:

```bash
ssh <UBUNTU_USERNAME>@10.20.10.157                                                   # key login works
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <UBUNTU_USERNAME>@10.20.10.157   # refused
```

**Observed effective settings (after hardening):** `passwordauthentication no`, `kbdinteractiveauthentication no`, `pubkeyauthentication yes`, `permitrootlogin prohibit-password`. Key login worked and password login was refused.

**Recovery:** from the Proxmox console of VM103, `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl reload ssh`, or roll back to the snapshot. The Kali private key is now the only way in over the network.

Never upload `~/.ssh/id_ed25519`, passwords, or `authorized_keys` containing personal identifiers to GitHub.

### Key-only SSH on the production VMs (VM100 and VM101)

The production Ubuntu VMs are on the home LAN, so the key comes from the Windows laptop, not Kali (Kali is isolated and cannot reach them). VM101 was done first as a trial, then VM100. Take a Proxmox snapshot of each VM first (`pre-ssh-hardening`, no RAM) and keep the Proxmox console as the recovery path.

```powershell
# Windows laptop (PowerShell): create a key only if one does not already exist
ssh -V
dir $env:USERPROFILE\.ssh
ssh-keygen -t ed25519 -C "laptop-homelab"

# Copy only the public key to each VM (Windows has no ssh-copy-id):
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <VM_USER>@192.168.50.21 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
ssh -o PreferredAuthentications=publickey <VM_USER>@192.168.50.21
```

```bash
# On each VM, in an open key-authenticated session:
sudo grep -ri PasswordAuthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t
sudo systemctl reload ssh
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin) '
```

Test from a **new** PowerShell window before closing the original session: key login works, and `ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <VM_USER>@<VM_IP>` is refused. **Confirmed:** completed successfully on both VM101 and VM100.

The laptop's private key (`id_ed25519`, no `.pub`) is now the only way into these VMs over the network; keep an encrypted backup copy (for example in a password manager) and never commit it. Recovery uses the Proxmox console: remove `/etc/ssh/sshd_config.d/10-hardening.conf` and reload `ssh`, or roll back the snapshot. Uptime Kuma's SSH monitors only check that port 22 is open, so they are unaffected.

## Validation and recovery checklist

### After Proxmox maintenance
```bash
pveversion -v
qm list
pvesm status
ip route
ping -c 4 192.168.50.1
```

### After Docker changes (run in the applicable Ubuntu VM)
```bash
sudo docker ps
sudo docker compose ps
sudo docker logs --tail 50 <CONTAINER_NAME>
```

### After lab firewall changes (run on a lab client)
```bash
ip route
nslookup example.com 10.20.10.1
ping -c 4 1.1.1.1
ping -c 4 192.168.50.1  # Expected to fail; confirm block in OPNsense logs
# ICMP is only one protocol. Also confirm TCP to production hosts is not reachable
# (expect filtered or closed; scan only your own hosts):
nmap -Pn -p 22,53,80,443,8006 192.168.50.1,192.168.50.10,192.168.50.11,192.168.50.20,192.168.50.21
```

Also confirm on the Proxmox node that `vmbr1` and `vmbr2` have no host IP (`ip -br addr`), and confirm the static OPNsense WAN address (`192.168.50.5`) cannot be handed out by the ISP router's DHCP pool.

### Lab isolation verification (confirmed)

Run after any change to OPNsense rules or Proxmox bridges. Nmap takes targets separated by **spaces**, not commas. Only scan your own systems.

```bash
# From Kali: Proxmox nodes (expect every port "filtered"):
nmap -Pn -p 22,111,3128,8006 192.168.50.10 192.168.50.11 --max-retries 1 --host-timeout 60s
curl -kI --max-time 5 https://192.168.50.10:8006     # expect timeout
curl -kI --max-time 5 https://192.168.50.11:8006     # expect timeout

# From Kali: other home-LAN hosts (expect "filtered"):
nmap -Pn -p 22,53,80,443,3001,8080 192.168.50.1 192.168.50.20 192.168.50.21 --max-retries 1 --host-timeout 60s

# From the target VM (expect "blocked" for all three):
timeout 5 bash -c '</dev/tcp/192.168.50.10/8006' && echo OPEN || echo blocked
timeout 5 bash -c '</dev/tcp/192.168.50.11/8006' && echo OPEN || echo blocked
timeout 5 bash -c '</dev/tcp/10.20.10.1/443' && echo OPEN || echo blocked
```

```bash
# On Proxmox node 2: the lab bridges must have no host address:
ip -br addr show vmbr1
ip -br addr show vmbr2
```

```powershell
# From the home-LAN laptop: the lab must not be routable (both should fail):
ping 10.20.10.1
Test-NetConnection 10.20.20.1 -Port 443
```

In OPNsense Live View a blocked attempt from Kali should show as a block under the "Block LAN net → PRIVATE_NETWORKS" rule. **Confirmed:** all of these tests produced the expected results with no exceptions. The Proxmox host firewall is not enabled; protection of the web interface relies on the OPNsense rules and the lab having no host address on `vmbr1` or `vmbr2`.

### Snapshot / backup practices
- Snapshot `baseline-secure-lab` on OPNsense VM102 and `kali-clean-install` on Kali VM104 already exist.
- Snapshots are **not backups**; they live on the same SSD as the VM. Off-host backups are configured (below).

**Backup design (implemented):**
- Storage: an SMB/CIFS share named `ProxmoxBackups` on a 500 GB external drive attached to a Windows laptop (`<BACKUP_HOST_IP>`, DHCP reservation). A dedicated local Windows account has access to that share only. The storage ID `laptop-backup` was added on **each** node (the nodes are standalone, so each keeps its own storage list), with content type **Backup** only.
- Jobs: scheduled per node under **Datacenter → Backup**: node 1 backs up VM100; node 2 backs up VMs 101–104. Mode Snapshot, compression ZSTD, retention Keep Last 3, staggered evening start times. **Confirmed:** the scheduled jobs were created on both nodes and ran as scheduled.
- The laptop must be awake and the drive connected at backup time; check **Datacenter → Tasks** for failures.

```powershell
# Windows laptop (PowerShell as administrator) — example; adjust the drive letter:
net user <BACKUP_ACCOUNT> * /add
New-SmbShare -Name ProxmoxBackups -Path D:\ProxmoxBackups -FullAccess <BACKUP_ACCOUNT>
icacls D:\ProxmoxBackups /grant "<BACKUP_ACCOUNT>:(OI)(CI)M"
```

```bash
# Proxmox host — example one-off backup and restore test:
pvesm status                                   # Find the backup storage ID
vzdump 100 --storage laptop-backup --mode snapshot --compress zstd
# Restore test: use an unused VM ID, do not start the restored VM (it would clash with the original):
qmrestore <PATH_TO_VMA_FILE> <NEW_VMID> --storage local-lvm
```

- **Confirmed:** a manual backup of VM100 completed without errors, and a restore test to a new VM ID succeeded (the test VM was not started and was then removed).
- Export and securely store OPNsense config outside the public repository. **Confirmed:** an OPNsense config export was saved to private storage.
- Record version, date, purpose, rollback instructions, and observed result for every significant change.

## Project status and next steps

**Implemented:** Two standalone Proxmox nodes; Ubuntu production VMs; AdGuard DNS with a synced replica and tested failover; two Uptime Kuma instances; isolated OPNsense lab with DHCP/DNS and rules; Ubuntu lab admin VM; Kali VM; guest agent; snapshots; host discovery, targeted Nmap scan; SSH service setup, key-based login, and password authentication disabled on VM103; scheduled off-host backups to an SMB share with a verified restore; OPNsense config export; DNS monitors for both AdGuard servers in both Uptime Kuma instances; cross-node Proxmox monitoring; an OPNsense WAN ping monitor backed by a narrow WAN ICMP rule limited to the Kuma hosts; named Proxmox admin accounts with TOTP two-factor on both nodes; key-only SSH on VM100, VM101 and VM103; Discord webhook alerts in both Kuma instances as a second alert channel; a separate TARGETS network (`vmbr2`, `10.20.20.0/24`) behind OPNsense with a first target VM and verified isolation; the OPNsense anti-lockout rule disabled so firewall management is limited to VM103.

**Planned / not yet verified:** further security monitoring tools, and eventual carefully planned home-network firewall migration.

**Additional planned improvements:**
- **Alerting:** both alert channels (email and Discord) depend on the internet connection, so a full outage will not alert. Consider a local alert path if that matters.
- **Per-device DNS visibility:** the ISP-supplied router forwards DNS itself, so AdGuard sees one client. Set DNS manually on selected devices to see them individually and fail over on the device.
- **Backups:** keep checking Datacenter → Tasks for missed runs, since the laptop must be awake and the drive connected; consider a more always-on backup target if that becomes a problem.- **More targets:** add further target VMs (for example intentionally vulnerable machines) to `vmbr2` only, keep the TARGETS rules restrictive, and snapshot each before testing.
- **Proxmox repositories:** without a subscription, switch from the enterprise repository to the no-subscription repository so `apt update` completes cleanly.

## References
- Proxmox VE documentation: https://pve.proxmox.com/pve-docs/
- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- OPNsense documentation: https://docs.opnsense.org/
- Kali Linux documentation: https://www.kali.org/docs/
- Nmap reference guide: https://nmap.org/book/man.html
- OpenSSH manual pages: https://www.openssh.com/manual.html
