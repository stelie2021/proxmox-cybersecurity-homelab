# Two-Node Proxmox Home Lab — Complete Build Guide

Two NEOSMAY N200 nodes • Ubuntu • Docker • AdGuard Home • Uptime Kuma • OPNsense • Kali • SSH

Build record and reusable reference. Exact historical command transcripts were not retained for every stage. Example commands are identified; do not run commands blindly.

## Scope, evidence, and safety

This is a chronological reconstruction of the two-node home lab build, based on recorded configuration and troubleshooting.

- **Confirmed** means the outcome was reported or checked.
- **Example** means a reusable command or GUI setting, not proof of the exact historical keystrokes.
- **Planned** means not yet built or verified.

Never execute an entire block blindly.

Production network: ISP router `192.168.50.1`, managed switch `192.168.50.2`, Proxmox node 1 `192.168.50.10`, node 2 `192.168.50.11`. IP addresses and device names in this document are examples; adjust them for your own network, and never publish device MAC addresses.

Keep the isolated `vmbr1` without a physical uplink, host IP, or gateway. Never change production DHCP/DNS or Proxmox bridge settings without console access and a rollback plan. Do not publish passwords, SMTP secrets, private SSH keys, tokens, or firewall exports.

### Address summary

| System | Address | Role |
|---|---|---|
| ISP router | `192.168.50.1` | Production gateway and DHCP |
| managed switch | `192.168.50.2` | Managed switch |
| Proxmox node 1 | `192.168.50.10` | Production services |
| Ubuntu VM 100 | `192.168.50.20` (reserved) | AdGuard Home + Uptime Kuma |
| Proxmox node 2 | `192.168.50.11` | Lab virtualization |
| Ubuntu VM 101 | `192.168.50.21` (reserved) | Uptime Kuma #2 + AdGuard Home replica |
| OPNsense VM 102 | LAN `10.20.10.1`; WAN `192.168.50.5` (static) | Isolated lab firewall |
| Ubuntu Desktop VM 103 | `10.20.10.157` (reserved) | Lab administration |
| Kali VM 104 | `10.20.10.173` (DHCP; may change) | Security testing |

---

## 1. Prepare both NEOSMAY mini PCs and install Proxmox VE

### GUI / procedure

1. Connect each NEOSMAY AC8-N (Intel N200, 32 GB RAM, 1 TB NVMe) to power, Ethernet and a display/keyboard for initial setup.
2. Obtain a current official Proxmox VE installer ISO; write it to a USB installer with a trusted imaging utility.
3. Boot from USB, select Install Proxmox VE, accept licensing, choose the internal NVMe disk, set locale/timezone, and set a strong root password and administrator email.
4. Configure management network: Node 1 `192.168.50.10/24` and Node 2 `192.168.50.11/24`; gateway `192.168.50.1`; working DNS. Use unique hostnames.
5. Reboot, remove installer media, open `https://192.168.50.10:8006` and `https://192.168.50.11:8006` from your Windows laptop; log in as `root` via Linux PAM.
6. Confirm both nodes remain standalone (no cluster created).

### CLI commands

```bash
pveversion -v
hostnamectl
lscpu
free -h
lsblk
ls -l /dev/kvm
ip -br addr
ip route
ping -c 4 192.168.50.1
ping -c 4 1.1.1.1
getent hosts debian.org
```

**Confirmed:** N200 CPU, ~32 GB RAM, KVM virtualization support, network gateway/internet/DNS reachability and successful host reboot. GUI: Datacenter → each node → Summary, System → Network, and Disks.

---

## 2. Update Proxmox and verify storage

### GUI / procedure

1. On each node, open Node → Updates → Repositories. Without a paid subscription, disable the enterprise repositories and add the **No-Subscription** repository; the Proxmox documentation describes the exact format for your release. Avoid incompatible repository changes.
2. In Node → Updates, refresh and review updates; install applicable updates and reboot if required.
3. In Node → Disks → LVM-Thin and Datacenter → Storage, confirm `local` (ISO/backups) and `local-lvm` (VM disks) are enabled. On Node 1 the `pve/data` thin pool was approximately 852.88 GB.

### CLI commands

```bash
apt update
apt list --upgradable
# After reviewing upgrade impact:
apt full-upgrade
pvesm status
vgs
lvs
lsblk
qm list
```

**Confirmed:** both hosts completed upgrades, rebooted, and `local`/`local-lvm` storage checks succeeded. Exact repository edits and original installer disk layout were not preserved.

---

## 3. Configure physical home LAN and production addressing

### GUI / procedure

1. ISP router stays at `192.168.50.1` and remains the production DHCP gateway.
2. managed switch management IP is `192.168.50.2`. Leave VLAN settings unchanged for the initial build.
3. Verify the static addresses used in this lab do not conflict with the router's DHCP pool: Node 1 `192.168.50.10`, Node 2 `192.168.50.11`, and the OPNsense WAN `192.168.50.5`. Move the pool or add reservations if any of them fall inside it.
4. Plan DHCP reservations for the Ubuntu production VMs (`192.168.50.20` for VM100, `192.168.50.21` for VM101). These are created in steps 4 and 7, after each VM exists and its MAC is known.
5. Windows laptop accesses Proxmox GUIs via HTTPS port 8006.

### CLI commands

```bash
ip -br addr
ip route
ping -c 4 192.168.50.1
```

```powershell
# Windows PowerShell examples:
Test-NetConnection 192.168.50.10 -Port 8006
Test-NetConnection 192.168.50.11 -Port 8006
```

**Confirmed:** the Ubuntu production VM addresses were reserved and subsequently observed. DHCP reservation menus vary by ISP router firmware.

---

## 4. Build Ubuntu Server VM100 on Node 1

### GUI / procedure

1. Node 1 → `local` → ISO Images → Upload official Ubuntu Server ISO.
2. Create VM → VM ID 100, Ubuntu Server ISO, Linux guest; disk on `local-lvm` (~22 GB observed), RAM ~4 GB, NIC on `vmbr0`; complete installer.
3. Install Ubuntu, create a non-root admin account, configure networking initially via DHCP.
4. Node 1 → VM100 → Options → QEMU Guest Agent → Enabled; enable Start at boot and configure startup order/delay.
5. Create the router DHCP reservation for `192.168.50.20` (matching the VM's MAC), renew/reboot Ubuntu, verify address and DNS.
6. Install `qemu-guest-agent` inside Ubuntu; verify VM Summary reports the IP.

### CLI commands

```bash
hostnamectl
ip -br addr
ip route
ping -c 4 192.168.50.1
ping -c 4 1.1.1.1
getent hosts example.com
sudo apt update
sudo apt install -y qemu-guest-agent
systemctl is-active qemu-guest-agent
```

```bash
# Proxmox Node 1 host shell, example:
qm config 100
qm agent 100 ping
```

**Confirmed:** VM100 initially used DHCP `192.168.50.224`, then reserved `192.168.50.20`; QEMU agent, startup settings, and network tests worked.

---

## 5. Install Docker Engine and Compose on Ubuntu VM100

**Confirmed:** Docker Engine and Docker Compose were installed, and `hello-world` completed. The exact historical Docker repository commands were not preserved. The following is a reusable Ubuntu installation **example**; check current Docker instructions before using.

### CLI commands

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
sudo docker ps
sudo docker compose version
```

Do not add users to the `docker` group without understanding it grants root-equivalent access.

---

## 6. Deploy AdGuard Home on VM100

### GUI / procedure

1. Create persistent Docker directories for AdGuard configuration and working data.
2. Deploy the AdGuard Home container with DNS port 53 TCP/UDP and web administration ports mapped appropriately. The exact original compose file and port mapping were not preserved; do not paste an invented production compose file over your existing container.
3. Open the initial setup wizard at `http://192.168.50.20:3000` (observed during initial setup); the configured dashboard is subsequently accessible at `http://192.168.50.20:8080`.
4. Configure upstream DNS and protection choices; "Use AdGuard browsing security web service" was selected.
5. Test DNS directly against `192.168.50.20` and inspect the AdGuard Query Log.
6. Test Windows DNS with any VPN client disconnected; review Windows IPv4/IPv6 DNS and browser secure DNS when requests do not appear in AdGuard.

### CLI commands

```bash
sudo docker ps
sudo docker logs --tail 100 adguardhome
sudo ss -lntup | grep -E ':(53|8080)\b'
dig @192.168.50.20 example.com
nslookup example.com 192.168.50.20
```

```powershell
# Windows PowerShell:
Resolve-DnsName example.com -Server 192.168.50.20
nslookup example.com 192.168.50.20
ipconfig /all
ipconfig /flushdns
```

**Confirmed:** AdGuard answered DNS and displayed requests in its Query Log; a third-party VPN client caused earlier Windows test issues. VM100 remains on to provide DNS independently of the laptop.

**Limitation at this stage:** all DNS depends on VM100. A secondary AdGuard Home on VM101 was added later; see section 17.

---

## 7. Deploy Uptime Kuma on both Ubuntu production VMs

### GUI / procedure

1. On VM100 deploy a persistent Uptime Kuma container and open `http://192.168.50.20:3001`; configure the admin account.
2. Add monitors: AdGuard dashboard HTTP, DNS `google.com`, Proxmox node 1 HTTPS `192.168.50.10:8006`, Ubuntu SSH TCP `192.168.50.20:22`, router ping `192.168.50.1`, managed switch ping `192.168.50.2` (and other recorded checks).
3. Settings → Notifications → configure email SMTP; send a test email.
4. Temporarily create a failing monitor to test the DOWN email; correct its URL to trigger the UP email; remove the temporary monitor.
5. On Node 2 create Ubuntu Server VM101 (reserved `192.168.50.21`), install Docker, deploy Uptime Kuma at `http://192.168.50.21:3001`, configure its monitors and email alerts.
6. Confirm both Kuma instances are reachable and monitors show UP.

### CLI commands

```bash
sudo docker ps
sudo docker logs --tail 100 uptime-kuma
sudo ss -lntp | grep ':3001'

# Reusable Docker Compose operations (from the compose project directory):
sudo docker compose ps
sudo docker compose logs --tail 100
sudo docker compose pull
sudo docker compose up -d
```

**Confirmed:** Kuma #1 had eight monitors and Kuma #2 six monitors; both monitored successfully, with email DOWN/UP alerts tested. SMTP credentials must remain private.

**Later addition (confirmed):** after the AdGuard replica was built (section 17), a DNS-type monitor for each AdGuard server (`192.168.50.20` and `192.168.50.21`) was added in both Kuma instances.

**Cross-node monitoring (confirmed):** each Kuma instance watches the Proxmox node that the other one runs on (Kuma #1 watches node 2; Kuma #2 watches node 1), so if a node goes down, the Kuma on the surviving node sends the alert.

**OPNsense WAN ping (confirmed):** a Ping monitor for `192.168.50.5` exists in both Kuma instances. It depends on the narrow WAN ICMP rule described in section 11.

**Second alert channel, Discord (confirmed):** the email alerts have a second channel so an email or SMTP failure does not silence monitoring.

1. In Discord, create a private server (or use an existing one) with a text channel such as `homelab-alerts`.
2. Channel settings → Integrations → Webhooks → New Webhook: create one webhook per Kuma instance, named for the instance (for example `Kuma-VM100` and `Kuma-VM101`), so alerts show which Kuma sent them. Copy each webhook URL.
3. In each Kuma: profile menu → Settings → Notifications → Setup Notification → type **Discord**, friendly name `Discord alerts`, paste that instance's webhook URL, turn on **Default enabled** and **Apply on all existing monitors**, click **Test**, then Save.
4. Test a real alert: create a temporary TCP monitor against a closed port, confirm the DOWN alert arrives on both email and Discord, correct the port, confirm the UP alert, then delete the temporary monitor.
5. In the Discord phone app, set the server or channel notifications to **All Messages** and confirm the phone is not blocking them.

**Confirmed:** the Discord test messages arrived, and a temporary monitor's DOWN and UP alerts arrived on both Kuma instances.

A webhook URL is a secret: anyone who has it can post to the channel. Never put it in the repository, screenshots or documentation; if it leaks, delete the webhook in Discord and create a new one. Discord also depends on the internet connection, so a full outage will not alert on either channel.

---

## 8. Create Node 2 isolated Proxmox bridge `vmbr1`

### GUI / procedure

1. Proxmox Node 2 → System → Network → Create → Linux Bridge.
2. Name: `vmbr1`; Bridge ports: **empty**; IPv4/CIDR: **empty**; Gateway: **empty**; IPv6: **empty**.
3. Save and Apply Configuration with physical/console recovery available.
4. Confirm `vmbr0` remains attached to the production NIC with Node 2 management address `192.168.50.11/24`.
5. Do not add a physical switch port or VLAN uplink to `vmbr1`.

### CLI commands

```bash
ip -br addr
cat /etc/network/interfaces
ip route
# Confirm no host IP/default gateway on vmbr1
```

**Confirmed:** `vmbr1` was created without physical ports, IP or gateway; `vmbr0` continued to serve production management.

---

## 9. Install OPNsense VM102 and assign WAN/LAN

### GUI / procedure

1. Node 2 → `local` → ISO Images → upload the official OPNsense ISO.
2. Create VM ID 102 `opnsense-lab`; 2 vCPU, 4 GB RAM, 32 GB disk.
3. Add VirtIO NIC `net0` on `vmbr1` (LAN) and `net1` on `vmbr0` (WAN). Record NIC MACs and verify interface ordering inside OPNsense.
4. Install OPNsense to the VM disk, detach the ISO, reboot.
5. OPNsense console → Assign Interfaces: `vtnet0` LAN, `vtnet1` WAN.
6. Set LAN IPv4 static `10.20.10.1/24`.
7. Set WAN IPv4 **static** `192.168.50.5/24` with upstream gateway `192.168.50.1` (ISP router). Confirm this address is outside the router DHCP pool or otherwise reserved, so no other device is handed the same address.

Continue with step 10 to reach the OPNsense web GUI from a lab VM.

**Confirmed:** OPNsense VM102 LAN `10.20.10.1`, WAN static `192.168.50.5`, routing and NAT functioning.

---

## 10. Create Ubuntu Desktop VM103, configure OPNsense WAN, Dnsmasq and DHCP

### GUI / procedure

1. Node 2 → Create VM103 `ubuntu-lab-admin`, 2 vCPU, 4 GB RAM, 32 GB disk; attach **only one** NIC on `vmbr1`. Install Ubuntu Desktop.
2. Check VM103 obtained an address and default route `10.20.10.1`. If it did not (Dnsmasq DHCP is configured later in this step), set a temporary static address such as `10.20.10.50/24` with gateway `10.20.10.1`, then return it to DHCP afterward.
3. From VM103 open `https://10.20.10.1` to finish web configuration.
4. Interfaces → WAN: uncheck **Block private networks** (WAN is RFC1918); leave **Block bogon networks** checked as tested.
5. Services → Dnsmasq DNS & DHCP: enable the Dnsmasq service and DHCP on the lab LAN. Set DHCP pool `10.20.10.100–10.20.10.200`; use OPNsense `10.20.10.1` for gateway and DNS. Exact OPNsense 2026 menu labels may differ.
6. In Dnsmasq static leases, reserve `10.20.10.157` for the VM103 MAC (do not publish the MAC).
7. Renew VM103 DHCP via its GUI network toggle or NetworkManager; confirm IP `10.20.10.157`.

### CLI commands

```bash
ip -br addr
ip route
resolvectl status
nmcli device status
nmcli connection show
nslookup example.com 10.20.10.1

# Example only; replace the exact connection name:
sudo nmcli connection down "<CONNECTION_NAME>"
sudo nmcli connection up "<CONNECTION_NAME>"
```

**Confirmed:** VM103 DHCP reservation `10.20.10.157` and working management GUI, DNS and internet access.

---

## 11. Apply OPNsense lab firewall rules in verified order

### GUI / procedure

1. OPNsense → Firewall → Aliases: create `PRIVATE_NETWORKS` containing RFC1918 networks `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`.
2. Firewall → Rules → LAN: first confirm OPNsense's default LAN "allow to any" rules (IPv4 and IPv6) have been deleted or disabled. If they are still active, the rules below restrict nothing.
3. Add the following entries **top-down**, saving/applying changes and testing after each:

| # | Action | Protocol | Source | Destination | Notes |
|---|---|---|---|---|---|
| 1 | Block | IPv6 | any | any | Log |
| 2 | Pass | TCP | `10.20.10.157` | This Firewall / LAN address, port 443 | GUI management |
| 3 | Pass | TCP/UDP | LAN net | LAN address, port 53 | DNS |
| 4 | Pass | UDP | any | any, port 67 | DHCP broadcasts, as tested (broader than ideal) |
| 5 | Pass | ICMP echo request | LAN net | LAN address | Gateway ping |
| 6 | Pass (**disabled**) | any | LAN net | This Firewall | Kept disabled for recovery reference |
| 7 | Block | IPv4 | LAN net | `PRIVATE_NETWORKS` | Log |
| 8 | Block | IPv4 | LAN net | `192.168.50.0/24` | Log; redundant with alias |
| 9 | Pass | IPv4 | LAN net | any | Internet |

4. Firewall → Log Files → Live View: confirm denied traffic from lab clients to `192.168.50.1`.
5. Check OPNsense WAN TCP ports 443 and 22 are not reachable from the Windows laptop.
6. Node 2 → VM102 → Snapshots → Take Snapshot: `baseline-secure-lab`; Include RAM unchecked.

### CLI commands

```bash
# Run on a lab client, not on the Proxmox host:
ip route
nslookup example.com 10.20.10.1
ping -c 4 1.1.1.1
curl -I https://www.kali.org

# Expected blocked by OPNsense rules:
ping -c 4 192.168.50.1

# TCP checks toward production hosts (expect filtered or closed; scan only your own hosts):
nmap -Pn -p 22,53,80,443,8006 192.168.50.1,192.168.50.10,192.168.50.11,192.168.50.20,192.168.50.21
```

```powershell
# Windows PowerShell examples from the home LAN:
Test-NetConnection 192.168.50.5 -Port 443
Test-NetConnection 192.168.50.5 -Port 22
```

**Confirmed:** internet/DNS worked and access to the ISP router was blocked/logged.

**Important:** these firewall rules do **not** isolate VM103 and VM104 from each other on the same `vmbr1` L2 segment (lab targets are isolated on a separate network; see section 19). Firewall changes require caution: a permissive WAN or misordered LAN rule can undermine isolation.

### WAN rule for monitoring (added later)

OPNsense blocks all inbound traffic on the WAN by default, including ping, so the Uptime Kuma servers could not check that the firewall was alive. A narrow rule now allows it. Take a VM102 snapshot first (`pre-kuma-icmp`, no RAM).

1. Firewall → Aliases → Add: name `KUMA_HOSTS`, type Host(s), content `192.168.50.20` and `192.168.50.21`. Save and apply.
2. Firewall → Rules → **WAN** → Add: Action **Pass**, Interface WAN, Direction in, TCP/IP version IPv4, Protocol ICMP, ICMP type Echo Request, Source the alias `KUMA_HOSTS`, Destination **WAN address**, description `Allow Kuma ping to WAN`. Save and apply.
3. Verification (confirmed; all results matched):
   - From VM100 or VM101, `ping -c 3 192.168.50.5` replies.
   - From any other home-LAN host (for example the Windows laptop), `ping 192.168.50.5` times out.
   - From the laptop, `Test-NetConnection 192.168.50.5 -Port 443` and `-Port 22` still fail.
4. In both Kuma instances add a Ping monitor named `OPNsense WAN` for `192.168.50.5`, with retries `2` and the email notification enabled.

This only shows the firewall is reachable from the home side; it does not test the lab LAN.

---

## 12. Install Kali Linux VM104

### GUI / procedure

1. Download official `kali-linux-2026.3-installer-amd64.iso` and verify SHA-256 against Kali's official SHA256SUMS for the same release.
2. Node 2 → `local` → ISO Images: upload the installer ISO.
3. Create VM104 `kali-lab`: q35 machine, SeaBIOS, VirtIO SCSI single, 40 GiB `local-lvm` disk, SSD emulation/discard, 2 cores CPU type host, 4096 MiB RAM fixed, one VirtIO NIC **only** on `vmbr1`, Start at boot unchecked, QEMU Guest Agent enabled.
4. Boot the graphical installer; choose English / United States / American English; hostname `kali-lab`, blank domain; create an account (choose a private password); guided entire disk, one partition; Xfce and recommended/default tools; GRUB to `/dev/sda`.
5. Remove the ISO; boot and log in. Confirm DHCP `10.20.10.173/24` and default gateway `10.20.10.1`.
6. If ping to the gateway is blocked, add the narrow LAN ICMP echo rule from step 11 rather than broad firewall access.
7. Install QEMU Guest Agent and check VM104 Summary displays the IP.
8. Node 2 → VM104 → Snapshots → Take Snapshot: `kali-clean-install`; Include RAM unchecked.

### CLI commands

```bash
# Example checksum verification; compare with the official published digest:
sha256sum kali-linux-2026.3-installer-amd64.iso
```

```bash
# Confirmed Kali terminal checks:
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
```

```bash
# Proxmox Node 2 example:
qm agent 104 ping
qm listsnapshot 104
```

**Confirmed:** guest agent active despite the static-unit enable warning; Kali IP appeared in Proxmox Summary; snapshot `kali-clean-install` created.

---

## 13. Kali host discovery and Nmap port scanning

### GUI / procedure

1. On Kali VM104 open Terminal.
2. Verify Nmap is installed.
3. Run host discovery against only the owned Ubuntu VM103 `10.20.10.157`.
4. Scan TCP 22, 53, 80, 443, 3389; all were initially closed.
5. After enabling SSH on VM103, rescan port 22; it was open.
6. Use version detection to identify the SSH version; observed OpenSSH 10.2.
7. Do not scan the production home LAN or unknown hosts without authorization.

### CLI commands

```bash
nmap --version
sudo nmap -sn 10.20.10.157
sudo nmap -sS -p 22,53,80,443,3389 10.20.10.157
sudo nmap -sS -p 22 10.20.10.157
sudo nmap -sV -p 22 10.20.10.157
```

**Interpretation:** `open` = a listener accepts connections; `closed` = reachable but no listener; `filtered` = state not determined due to filtering. A version string does not establish vulnerability.

---

## 14. Install, start, and test OpenSSH on Ubuntu VM103

### GUI / procedure

1. Ubuntu VM103 → Terminal: initial `systemctl status ssh` returned "Unit ssh.service could not be found".
2. Install `openssh-server`.
3. First post-install check showed inactive; start the service manually.
4. Verify active and rescan port 22 from Kali (open).
5. Compare `ssh -V` locally with Nmap `-sV` remotely (both reported OpenSSH 10.2).
6. Inspect effective sshd settings: `permitrootlogin prohibit-password`, `passwordauthentication yes`, `pubkeyauthentication yes`.
7. From Kali, SSH to Ubuntu using the Ubuntu username; verify the host key fingerprint on Ubuntu before accepting.
8. In the remote session run `hostname` and `hostname -I`, then exit; the Kali prompt returned.

### CLI commands

```bash
# Ubuntu VM103:
systemctl status ssh
sudo apt update
sudo apt install openssh-server -y
sudo systemctl status ssh
sudo systemctl start ssh
systemctl is-active ssh
ssh -V
sudo sshd -T | grep -E '^(passwordauthentication|permitrootlogin|pubkeyauthentication) '
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

```bash
# Kali VM104 (replace the placeholder):
ssh <UBUNTU_USERNAME>@10.20.10.157

# In the remote Ubuntu session:
hostname
hostname -I
exit
```

**Confirmed:** SSH login succeeded and exit returned to Kali. Root password SSH login is prohibited; normal account password authentication is enabled. No public-key login was yet confirmed.

---

## 15. SSH key authentication and disabling password login on VM103

### GUI / procedure

1. On Kali, check for an existing key (`ls -l ~/.ssh/`); never overwrite one.
2. Generate an ED25519 key on Kali with a strong passphrase; keep the private key private.
3. Copy only the public key to Ubuntu VM103 `authorized_keys` using `ssh-copy-id` (password login is still enabled at this point).
4. Test public-key login; it should ask for the key passphrase, not the account password.
5. Take a Proxmox snapshot of VM103 (`pre-ssh-hardening`), keep one existing SSH session open, and remember the Proxmox console is the recovery path.
6. Create the hardening drop-in, validate, reload, and verify the effective settings.
7. Test in a **new** session: key login works and password login is refused. Only then close the original session.

### CLI commands

```bash
# Kali VM104:
ls -l ~/.ssh/
ssh-keygen -t ed25519 -C "kali-lab"
ssh-copy-id <UBUNTU_USERNAME>@10.20.10.157
ssh -o PreferredAuthentications=publickey <UBUNTU_USERNAME>@10.20.10.157
```

On Ubuntu, a file in `/etc/ssh/sshd_config.d/` can be overridden by another file that sets the same option earlier. The first value read wins, and `50-cloud-init.conf` (if present) may set `PasswordAuthentication yes`. Name the drop-in so it sorts first.

```bash
# Ubuntu VM103 — see which files already set the option:
sudo grep -ri PasswordAuthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/

# Create the drop-in, validate, apply, verify:
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t                           # no output means the config is valid
sudo systemctl reload ssh              # use "restart" if reload is rejected
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin) '
```

```bash
# Kali, in a NEW terminal, before closing the original session:
ssh <UBUNTU_USERNAME>@10.20.10.157        # key login works
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no <UBUNTU_USERNAME>@10.20.10.157   # refused
```

**Confirmed:** key login from Kali to VM103 succeeded. After hardening, the effective settings were `passwordauthentication no`, `kbdinteractiveauthentication no`, `pubkeyauthentication yes` and `permitrootlogin prohibit-password`; key login worked and password login was refused.

**Recovery:** from the Proxmox console of VM103, run `sudo rm /etc/ssh/sshd_config.d/10-hardening.conf && sudo systemctl reload ssh`, or roll back to the snapshot. The Kali private key is now the only way in over the network; if it is lost, use the Proxmox console.

Never commit `~/.ssh/id_ed25519`, passwords, tokens, or raw authentication logs to GitHub.

### Key-only SSH on the production VMs (VM100 and VM101)

The production Ubuntu VMs are on the home LAN, so the key comes from the Windows laptop, not Kali (Kali is isolated and cannot reach them). VM101 was done first as a trial, then VM100.

1. Take a Proxmox snapshot of each VM (`pre-ssh-hardening`, no RAM). The Proxmox console is the recovery path.
2. On the laptop, check for an existing key (`dir $env:USERPROFILE\.ssh`); create one only if none exists.
3. Copy only the public key to the VM, then test key login (it should ask for the key passphrase, not the VM's password).
4. In that open session, create the hardening drop-in, validate, reload and verify (same commands as VM103 above).
5. From a **new** window confirm key login works and password login is refused, then close the original session.

```powershell
# Windows laptop (PowerShell):
ssh -V
dir $env:USERPROFILE\.ssh
ssh-keygen -t ed25519 -C "laptop-homelab"          # only if no key exists

# Windows has no ssh-copy-id; copy only the public key:
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh <VM_USER>@192.168.50.21 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
ssh -o PreferredAuthentications=publickey <VM_USER>@192.168.50.21
```

```bash
# On each VM:
sudo grep -ri PasswordAuthentication /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\n' | sudo tee /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t
sudo systemctl reload ssh
sudo sshd -T | grep -E '^(passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication|permitrootlogin) '
```

**Confirmed:** completed successfully on both VM101 and VM100.

The laptop's private key (`id_ed25519`, no `.pub`) is now the only way into these VMs over the network; keep an encrypted backup copy (for example in a password manager) and never commit it. Recovery uses the Proxmox console: remove `/etc/ssh/sshd_config.d/10-hardening.conf` and reload `ssh`, or roll back the snapshot. Uptime Kuma's SSH monitors only check that port 22 is open, so they are unaffected.

---

## 16. Maintenance, backups and recovery

### GUI / procedure

1. On both Proxmox nodes regularly check VM Summary, storage, and backup status.
2. Before changes, snapshot appropriate VMs. Snapshots are not independent backups; they live on the same SSD as the VM.
3. Off-host backups are configured (see "Backup design" below). Check Datacenter → Tasks for failures, and test a restore after any change to the backup setup.
4. In AdGuard Query Log confirm DNS activity; in Uptime Kuma confirm monitors and email notifications.
5. In OPNsense Firewall → Log Files → Live View verify expected blocks; export the config to secure private storage.
6. Keep a change log of actual dates, commands, screenshots, observed results and rollback steps.

### CLI commands

```bash
# Proxmox host examples:
pveversion -v
qm list
pvesm status
qm listsnapshot 102
qm listsnapshot 104
```

#### Backup design (implemented)

- **Storage:** an SMB/CIFS share `ProxmoxBackups` on a 500 GB external drive attached to a Windows laptop (`<BACKUP_HOST_IP>`, DHCP reservation). A dedicated local Windows account has access to that share only. Wired Ethernet on the laptop is preferable to Wi-Fi.
- **Proxmox side:** the storage ID `laptop-backup` was added on **each** node (Datacenter → Storage → Add → SMB/CIFS), because the nodes are standalone and each keeps its own storage list. Content type: **Backup** only. Leave Domain and Subdirectory empty and leave "Allow Snapshots as Volume-Chain" unchecked (technology preview).
- **Jobs:** one scheduled job per node under Datacenter → Backup. Node 1 backs up VM100; node 2 backs up VMs 101–104. Mode Snapshot, compression ZSTD, retention Keep Last 3, with staggered evening start times so the two jobs do not write to the laptop at once.
- **Confirmed:** the scheduled jobs were created on both nodes and ran as scheduled.
- **Limitation:** the laptop must be awake and the drive connected at backup time.

```powershell
# Windows laptop (PowerShell as administrator) — example; adjust the drive letter:
net user <BACKUP_ACCOUNT> * /add
New-SmbShare -Name ProxmoxBackups -Path D:\ProxmoxBackups -FullAccess <BACKUP_ACCOUNT>
icacls D:\ProxmoxBackups /grant "<BACKUP_ACCOUNT>:(OI)(CI)M"
Get-NetConnectionProfile      # network category must be Private
```

```bash
# Proxmox host — example one-off backup, then a restore test.
# Restore to an unused VM ID and do NOT start the restored VM (it would clash with the original):
vzdump 100 --storage laptop-backup --mode snapshot --compress zstd
qmrestore <PATH_TO_BACKUP_FILE> <NEW_VMID> --storage local-lvm
```

**Confirmed:** a manual backup of VM100 completed without errors and the file appeared on the external drive. A restore test to a new VM ID succeeded; the test VM was not started and was then removed. An OPNsense config export (System → Configuration → Backups → Download) was saved to private storage, outside the repository.

```bash
# Ubuntu Docker examples:
sudo docker ps
sudo docker compose ps
sudo docker logs --tail 50 <CONTAINER_NAME>
```

**Note:** initial Ubuntu/Docker compose file contents, exact installation click paths for some software versions, and all historical terminal output were not preserved. This is a detailed build record plus reproducible reference, not an exact shell history transcript.

---

## 17. Redundant DNS: AdGuard Home replica on VM101

A second AdGuard Home runs in Docker on VM101 (`192.168.50.21`, web UI port `8080`). [AdGuardHome-sync](https://github.com/bakito/adguardhome-sync) replicates settings **one way** from the primary on VM100 to the replica at startup and every 10 minutes. Make configuration changes on the primary only. Query logs and statistics are per instance and are not copied.

### GUI / procedure

1. Snapshot VM101 (`pre-adguard-replica`, no RAM).
2. On VM101, confirm Docker works and the primary is reachable (`curl -I http://192.168.50.20:8080`).
3. Free port 53 (CLI below).
4. Create a project folder (separate from the Uptime Kuma project) with a compose file defining `adguard` (container `adguard-replica`; ports 53/tcp, 53/udp, 3000 for the wizard, and host 8080 mapped to container port 80) and `adguardhome-sync`, plus a private `.env` holding the primary and replica credentials (`chmod 600`; never commit it).
5. Start the replica (`sudo docker compose up -d adguard`) and finish the wizard on port 3000: admin interface on all interfaces port 80, DNS on all interfaces port 53, and create the replica login.
6. Start the sync (`sudo docker compose up -d`) and check its log for errors.
7. Test the replica directly before involving the router.
8. Set the router's DNS servers to `192.168.50.20` then `192.168.50.21` and test failover.

Docker commands on these VMs need `sudo` unless the account is in the `docker` group (which grants root-equivalent access).

### CLI commands

Ubuntu's `systemd-resolved` listens on port 53 and must be released first:

```bash
sudo mkdir -p /etc/systemd/resolved.conf.d
printf '[Resolve]\nDNSStubListener=no\n' | sudo tee /etc/systemd/resolved.conf.d/adguardhome.conf
# Point resolv.conf at the real upstream list instead of the disabled stub (127.0.0.53):
sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
sudo systemctl restart systemd-resolved
getent hosts example.com               # the VM itself must still resolve names
sudo ss -lntup | grep ':53 '           # expect no output before AdGuard starts
```

Do not set the VM's own DNS to `127.0.0.1` here. It would point at a container that is not running yet, and image pulls would fail.

```bash
# From the replica project folder:
sudo docker compose up -d adguard
sudo docker compose ps
sudo docker compose up -d
sudo docker compose logs --tail 50 adguardhome-sync

# Test the replica directly:
dig @192.168.50.21 example.com
dig @192.168.50.21 doubleclick.net      # blocked if the lists block it, same as the primary
```

**Confirmed:** the replica answers DNS and blocked the same test domain as the primary.

**ISP-supplied router behavior:** the router's DNS setting under the broadband connection sets the DNS the *router* uses upstream. It did not change what LAN clients receive: clients still use the router (`192.168.50.1`) as their DNS server, and the router forwards queries to `192.168.50.20` and `192.168.50.21`. Filtering worked for both IPv4 and IPv6 lookups sent to the router (a blocked test domain was blocked on both paths). The trade-off is that AdGuard sees every query as coming from the router, so per-device statistics and rules are not available. To see individual devices, set DNS manually on those devices to both AdGuard addresses. Do not add a public resolver as a third entry, or some clients will bypass filtering.

**Failover test (confirmed):** with the primary container stopped on VM100, a client flushed its DNS cache and lookups still resolved through the replica. Restart the primary afterward (`sudo docker start adguardhome`).

---

## 18. Proxmox access hardening: named admin accounts and two-factor login

Because the nodes are standalone, each node has its own user list, so this was done on each node separately (`https://192.168.50.10:8006` and `https://192.168.50.11:8006`).

### GUI / procedure

1. **Datacenter → Permissions → Users → Add:** a named user in the **Proxmox VE authentication server** realm (`<name>@pve`; no Linux account needed) with a long unique password stored in a password manager.
2. **Datacenter → Permissions → Add → User Permission:** path `/`, that user, role **Administrator**, propagate checked.
3. **Datacenter → Permissions → Two Factor → Add → TOTP** for that user: scan the QR code with an authenticator app and verify the code. Then **Add → Recovery Keys** and store the one-time codes in the password manager.
4. Test the login in a private browser window (realm "Proxmox VE authentication server", password, then the TOTP code) before relying on it.
5. Use the named account for everyday work. Keep `root@pam` as the emergency account with a long unique password; some host-level tasks (such as the node shell) may still require it. Add two-factor to `root` only after its recovery keys are saved.

**Confirmed:** completed on both nodes. If TOTP codes are rejected even though they are correct, check the node's clock (Node → System → Time).

---

## 19. Target network: TARGETS interface on `vmbr2`

A separate network behind OPNsense for machines to practice against, so traffic from Kali passes through the firewall instead of staying on one segment. Kali and VM103 still share `vmbr1`; the targets do not.

### GUI / procedure

1. Snapshot VM102 first (`pre-targets-network`, no RAM).
2. Node 2 → System → Network → Create → Linux Bridge: name `vmbr2`, with Bridge ports, IPv4/CIDR, Gateway and IPv6 all **empty**, then click **Apply Configuration**. If the configuration is not applied, starting VM102 fails with "bridge 'vmbr2' does not exist".
3. Shut down VM102, add a third VirtIO network device on `vmbr2`, and start it. Confirm the lab still works (Kali reaches `10.20.10.1` and the internet).
4. OPNsense (from VM103): Interfaces → Assignments → add `vtnet2` with description `TARGETS`; then Interfaces → TARGETS: enable, static IPv4 `10.20.20.1/24`, IPv6 none; save and apply.
5. Services → Dnsmasq DNS & DHCP: make sure TARGETS is a listening interface (or the interface box is empty) and add a DHCP range `10.20.20.100`–`10.20.20.200`; save and apply so the service restarts.
6. Firewall → Rules → TARGETS: add the rules in the table below (all logged), then apply.
7. Firewall → Rules → LAN: add a block rule for source LAN net to destination `10.20.20.1`, then a pass rule "Lab to targets" for source LAN net to destination `10.20.20.0/24`. Both must be **above** "Block LAN net → PRIVATE_NETWORKS" (which would otherwise block `10.20.20.x`), and the block must sit above the pass.
8. Create target VM 105 (`target-01`): Ubuntu Server with the OpenSSH server option, 20 GiB disk, 2 GiB RAM, QEMU guest agent unchecked (the target has no internet), network device on `vmbr2` **only**. The installer cannot reach a mirror; continue without updating.
9. Snapshot VM102 as `targets-network-verified` after the tests pass (taken).

| # | Action | Protocol | Source | Destination | Purpose |
|---|---|---|---|---|---|
| 1 | Block | IPv6 | any | any | No IPv6 |
| 2 | Pass | UDP | any | any, port 67 | DHCP |
| 3 | Pass | TCP/UDP | TARGETS net | TARGETS address, port 53 | DNS |
| 4 | Pass | ICMP echo request | TARGETS net | TARGETS address | Ping gateway |
| 5 | Block | IPv4 | any | any | Everything else, visible in the log |

Targets therefore get an address and DNS but cannot reach the Kali network, the home network, or the internet. For updates, temporarily add a pass rule from TARGETS net to any and disable it afterwards. Reverse-shell exercises (target connecting back to Kali) are blocked by design; add a narrow, logged pass rule on a specific port only when needed.

### Anti-lockout rule finding

After the target network was built, `curl -kI --max-time 5 https://10.20.20.1` from Kali returned a response from the firewall instead of timing out. Live View showed Kali (`10.20.10.173`) reaching `10.20.20.1:443`, passed by the label **anti-lockout rule**. OPNsense creates that rule automatically and evaluates it before user rules, so the LAN block rule never matched; it also lets any LAN host reach the firewall's web interface.

The rule was disabled under System → Settings → Administration. Management access now depends on the explicit rule allowing VM103 (`10.20.10.157`) to port 443. **Confirmed:** VM103 can still log in to the OPNsense web interface after the change. Before changing this, snapshot VM102 and know the recovery path: the Proxmox console of VM102, then the shell option, `pfctl -d` (temporarily disables the firewall) and `pfctl -e` (re-enables it).

### DHCP troubleshooting

If the target installer shows "autoconfiguration failed": check that the VM's NIC is on `vmbr2` and not disconnected; that Interfaces → Overview shows TARGETS up with `10.20.20.1/24`; that Dnsmasq lists TARGETS and has a DHCP range for it (apply to restart the service); that the DHCP rule has destination port 67 and no source port; and watch Firewall → Log Files → Live View filtered to TARGETS while the target retries. A temporary manual address in the installer (for example `10.20.20.50/24`, gateway and DNS `10.20.20.1`) lets the install continue.

### CLI tests

```bash
# On target-01:
ip -br addr
ip route
ping -c 3 10.20.20.1
nslookup example.com 10.20.20.1
ping -c 3 1.1.1.1          # expected to fail
ping -c 3 10.20.10.1       # expected to fail
ping -c 3 192.168.50.1      # expected to fail

# On Kali:
ping -c 3 <TARGET_IP>
nmap -Pn -p 22 <TARGET_IP>
curl -kI --max-time 5 https://10.20.20.1    # expected to time out
curl -kI --max-time 5 https://10.20.10.1    # expected to time out
```

**Confirmed:** on the target, DHCP gave a `10.20.20.x` address, the gateway and DNS tests worked, and the three pings that should fail did fail. From Kali, the target was reachable. After the anti-lockout rule was disabled, both `curl` tests timed out from Kali.

### Lab isolation verification (confirmed)

Run after any change to OPNsense rules or Proxmox bridges. Nmap takes targets separated by **spaces**, not commas (a comma-separated list is treated as one hostname and scans nothing). Only scan your own systems.

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

In OPNsense Live View a blocked attempt from Kali should show as a block under the "Block LAN net → PRIVATE_NETWORKS" rule.

**Confirmed:** all of these tests produced the expected results with no exceptions. The Proxmox host firewall is not enabled; protection of the web interface relies on the OPNsense rules and on the lab bridges having no host address.

---

## 20. Planned improvements (not yet built or verified)

### Alerting

- Both alert channels (email and Discord) depend on the internet connection, so a full outage will not alert. Consider a local alert path if that matters.

### More targets

Add further target VMs (for example intentionally vulnerable machines) to `vmbr2` only, keep the TARGETS rules restrictive, and snapshot each before testing. Kali and VM103 still share `vmbr1`; moving VM103 to its own segment is optional.

### Proxmox host hardening

- Keep port 8006 reachable from the home LAN only; confirm it is not reachable from the lab network.
- Consider a small UPS to protect single-disk nodes from power loss.

### Backups

- Keep checking Datacenter → Tasks for missed runs, since the laptop must be awake and the drive connected. If that becomes a problem, consider a more always-on backup target.

---

## Official documentation

- Proxmox VE: https://pve.proxmox.com/pve-docs/
- Docker Engine on Ubuntu: https://docs.docker.com/engine/install/ubuntu/
- OPNsense: https://docs.opnsense.org/
- Kali Linux: https://www.kali.org/docs/
- Nmap: https://nmap.org/book/man.html
