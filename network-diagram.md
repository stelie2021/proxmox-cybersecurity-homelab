# Network diagram

Overview of the two-node Proxmox home lab. IP addresses are examples. Diagrams are written in Mermaid, which GitHub renders automatically.

## 1. Home network and where each VM runs

```mermaid
flowchart TB
  INET((Internet)) --- ROUTER["ISP router<br/>192.168.50.1"]
  ROUTER --- SWITCH["Managed switch<br/>192.168.50.2"]
  SWITCH --- HOST1["Proxmox node 1<br/>192.168.50.10"]
  SWITCH --- HOST2["Proxmox node 2<br/>192.168.50.11"]
  SWITCH --- BACKUP["Backup laptop<br/>with external drive"]

  HOST1 --- VM100["VM100 Ubuntu<br/>AdGuard primary + Uptime Kuma<br/>192.168.50.20"]
  HOST2 --- VM101["VM101 Ubuntu<br/>AdGuard replica + Uptime Kuma<br/>192.168.50.21"]
  HOST2 --- OPN["VM102 OPNsense firewall<br/>WAN 192.168.50.5"]
```

Node 1 runs production services. Node 2 runs the second DNS server and monitor plus the isolated security lab behind OPNsense (shown in diagram 2). The nodes are standalone (no cluster).

## 2. Isolated lab behind OPNsense

```mermaid
flowchart LR
  HOME["Home LAN<br/>192.168.50.0/24"] --- OPNW

  subgraph FW["VM102 OPNsense firewall"]
    OPNW["WAN<br/>192.168.50.5"]
    OPNL["LAN<br/>10.20.10.1"]
    OPNT["TARGETS<br/>10.20.20.1"]
  end

  OPNL --- B1["vmbr1 virtual bridge<br/>10.20.10.0/24<br/>no physical port"]
  B1 --- VM103["VM103 Ubuntu Desktop<br/>10.20.10.157<br/>firewall admin"]
  B1 --- KALI["VM104 Kali<br/>10.20.10.173"]

  OPNT --- B2["vmbr2 virtual bridge<br/>10.20.20.0/24<br/>no physical port"]
  B2 --- TGT["VM105 target-01<br/>10.20.20.x"]
```

Both lab bridges have no physical port and no host IP, so the only way between the lab and anything else is through OPNsense.

## 3. What can talk to what

| From | To | Result |
|---|---|---|
| Kali (`10.20.10.x`) | Target network (`10.20.20.0/24`) | Allowed, through OPNsense and logged |
| Kali, VM103 | Internet | Allowed, through OPNsense (NAT) |
| Kali, VM103 | Home LAN, Proxmox nodes, DNS servers | Blocked |
| Kali | OPNsense web interface | Blocked (only VM103 is allowed) |
| VM103 | OPNsense web interface (HTTPS) | Allowed |
| Targets (`10.20.20.x`) | Anything | Blocked, except DHCP, DNS and ping to the firewall itself |
| Home LAN | Lab networks | No route |
| Uptime Kuma hosts | OPNsense WAN address | Ping only |
| Kali and VM103 | Each other | Not filtered (same segment) |

## 4. Supporting flows

```mermaid
flowchart LR
  CLIENTS["Home devices"] --> RTR["ISP router<br/>forwards DNS"]
  RTR --> AG1["AdGuard primary<br/>VM100"]
  RTR --> AG2["AdGuard replica<br/>VM101"]
  AG1 -. "settings sync, one way" .-> AG2

  K1["Uptime Kuma<br/>VM100"] -- "watches" --> NODE2["Proxmox node 2"]
  K2["Uptime Kuma<br/>VM101"] -- "watches" --> NODE1["Proxmox node 1"]
  K1 --> ALERT["Email + Discord alerts"]
  K2 --> ALERT

  NODE1 -- "scheduled backups" --> SHARE["SMB share on laptop<br/>external drive"]
  NODE2 -- "scheduled backups" --> SHARE
```

- **DNS:** devices use the router, which forwards to both AdGuard servers. If one is down, the other answers.
- **Monitoring:** each Kuma instance watches the node the other one runs on, so a surviving node can raise the alert. Alerts go out by email and Discord.
- **Backups:** each node backs up its own VMs to a shared folder on a laptop-attached external drive. Snapshots are kept on the nodes for quick rollback but are not counted as backups.

## Legend and notes

- Solid lines are network links; dotted lines are logical relationships.
- `vmbr0` is the bridge attached to the home network, `vmbr1` and `vmbr2` are virtual-only bridges on node 2.
- SSH password login is disabled on the Ubuntu VMs (VM100, VM101, VM103); only key-based login works. Proxmox web logins use named accounts with two-factor authentication.
