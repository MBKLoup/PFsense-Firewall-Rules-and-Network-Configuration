# Virtual Network & Firewall Configuration

## Overview

This project involved designing and deploying a fully segmented virtual network for a simulated small business environment, then hardening it with a custom firewall policy. I built everything from scratch using VirtualBox and pfSense — configuring static IPs, routing, firewall ACLs, and testing each security policy hands-on.

The goal was to simulate what a real IT/network security team would do when setting up a secure internal network: segment traffic by department, enforce least-privilege access, and verify that the firewall rules actually work.

---

## Network Architecture

```
[Admin Client]           [Corporate Client]
  7.7.7.7/28               8.8.8.8/28
  (adminNet)               (corpNet)
       |                        |
       +----------+-------------+
                  |
         [pfSense Firewall]
          WAN: 5.5.5.1/28
          LAN: 7.7.7.1/28
         OPT1: 8.8.8.1/28
                  |
            (servNet)
                  |
          [Ubuntu Server]
            5.5.5.5/28
    Apache | OpenSSH | vsftpd
```

The network is divided into three segments:
- **adminNet** — IT department traffic, privileged access
- **corpNet** — Regular corporate users, restricted access
- **servNet** — Server subnet, protected by firewall policies

---

## Technologies Used

| Technology | Role |
|---|---|
| VirtualBox | Virtualization platform |
| pfSense (FreeBSD) | Router and stateful firewall |
| Ubuntu Linux | Client and server OS |
| Apache2 | HTTP web server |
| OpenSSH | Secure shell server |
| vsftpd | FTP server |
| Remmina | RDP remote desktop client |

---

## Phase 1 — Network Build & Configuration

The first phase involved manually building the network from the ground up with no DHCP — every IP address was statically configured by hand.

### What I did

- Configured three Ubuntu VMs with static IPs using `/etc/network/interfaces`
- Set up default gateway routing using the `post-up` directive
- Configured pfSense with three network interfaces (WAN, LAN, OPT1)
- Set up hostname resolution via `/etc/hosts` on each client
- Deployed a custom Apache web server with a personal homepage
- Verified end-to-end connectivity using `ping` and `traceroute`

### Key Configuration Files

**Static IP configuration (Admin Client)**
```bash
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet static
        address 7.7.7.7/28
post-up ip route add default via 7.7.7.1
```

**Hostname resolution (both clients)**
```
5.5.5.5 web-server
```

**Custom web server homepage**
```html
<html>
  <body>
    <img src="MBKSelfie.png">
  </body>
</html>
```

---

## Phase 2 — Firewall Policy & Security Hardening

The second phase involved replacing a broad "allow all" firewall policy with specific, restrictive rules based on a corporate security policy. The principle of least privilege guided every rule — only the minimum necessary traffic was permitted.

### Security Policies Implemented

**Policy 1 — IT Department (adminNet) Access**
- Allow RDP (TCP 3389) from adminNet to server
- Allow SSH (TCP 22) from adminNet to server
- Block all HTTP (TCP 80) from adminNet to server
- Everything else denied by default

**Policy 2 — Corporate Users (corpNet) Access**
- Allow HTTP (TCP 80) from corpNet to server
- Allow FTP (TCP 21) from corpNet to server
- Everything else denied by default

**Policy 3 — ICMP Policy**
- Allow ping (echo request) from both subnets to server IP only
- Allow ping replies from server back to internal clients
- Block all other ICMP traffic outside source subnet

**Policy 4 — Default Deny & Log**
- All unmatched traffic on every interface is blocked and logged
- Provides audit trail for security analysis

### Firewall ACL Summary

| Interface | Protocol | Source | Destination | Port | Action |
|---|---|---|---|---|---|
| LAN | TCP | LAN subnets | 5.5.5.5 | 3389 | Pass |
| LAN | TCP | LAN subnets | 5.5.5.5 | 22 | Pass |
| LAN | ICMP | LAN subnets | 5.5.5.5 | - | Pass |
| LAN | Any | Any | Any | Any | Block+Log |
| OPT1 | TCP | OPT1 subnets | 5.5.5.5 | 80 | Pass |
| OPT1 | TCP | OPT1 subnets | 5.5.5.5 | 21 | Pass |
| OPT1 | ICMP | OPT1 subnets | 5.5.5.5 | - | Pass |
| OPT1 | Any | Any | Any | Any | Block+Log |
| WAN | TCP | LAN subnets | 5.5.5.5 | 80 | Block |
| WAN | ICMP | 5.5.5.5 | Any | - | Pass |
| WAN | Any | Any | Any | Any | Block+Log |

---

## Testing & Verification

Every rule was tested and verified to work correctly. Here is a summary of results:

| Test | Client | Expected Result | Outcome |
|---|---|---|---|
| Ping server | Admin | Success | Pass |
| Ping server | Corporate | Success | Pass |
| Traceroute to server | Admin | Blocked | Pass |
| Traceroute to server | Corporate | Blocked | Pass |
| RDP to server | Admin | Success | Pass |
| SSH to server | Admin | Success | Pass |
| HTTP to server | Admin | Blocked | Pass |
| FTP to server | Admin | Blocked | Pass |
| HTTP to server | Corporate | Success | Pass |
| FTP to server | Corporate | Success | Pass |
| RDP to server | Corporate | Blocked | Pass |
| Policy 4 logging | Both | Logged | Pass |

---

## Challenges & How I Solved Them

**MAC address conflicts after VM cloning**
When I cloned the Ubuntu VMs, all three inherited the same MAC address. This caused VirtualBox's internal network to get confused about which machine was which, breaking routing entirely. I solved this by regenerating unique MAC addresses for each VM's network adapter.

**VirtualBox internal network name case sensitivity**
After renaming my internal networks to fix the MAC conflict issue, connectivity broke again. After extensive troubleshooting I discovered that VirtualBox's internal network names are case sensitive — `servNet` and `servnet` are treated as two completely different networks. Standardizing all names to lowercase fixed it immediately.

**pfSense stateful firewall retaining old connection states**
After adding block rules for FTP and HTTP, the traffic was still getting through. This happened because pfSense's stateful packet inspection remembered previously approved connections and continued allowing them even after the rules changed. Resetting the firewall state table via Diagnostics → States forced all connections to be re-evaluated against the new rules.

**Anti-lockout rule interfering with HTTP blocking**
pfSense has a built-in anti-lockout rule that permanently allows HTTP traffic on the LAN interface so administrators can never lock themselves out of the web interface. This meant any HTTP block rule on LAN was being bypassed. The solution was to place the HTTP block rule on the WAN interface instead, targeting traffic from adminNet specifically to the server's IP.

**Guest Additions and shared folder access**
Transferring files into VMs without internet access proved more difficult than expected. Shared folders required VirtualBox Guest Additions which failed to install due to a missing `bzip2` dependency. The practical workaround was temporarily switching the VM's network adapter from Internal Network to NAT, downloading the file, then switching back.

**Duplicate routing entries**
After configuring both the `/etc/network/interfaces` file and NetworkManager with static IP settings, the routing table ended up with duplicate default gateway entries. This caused unpredictable routing behavior. I resolved it by manually removing the duplicate routes using `ip route del`.

---

## Practical Applications

The concepts demonstrated in this project map directly to real-world enterprise networking:

- **Network segmentation** is standard practice in corporate environments to limit the blast radius of a security breach. If an attacker compromises a corporate workstation, segmentation prevents them from freely accessing the server or IT systems.

- **Principle of least privilege** applied to firewall rules means users only have access to exactly what they need — nothing more. This is a foundational concept in both network security and access control.

- **Stateful packet inspection** is how modern firewalls work. Understanding how connection states affect rule processing is critical for troubleshooting firewall behavior in production environments.

- **Default deny with logging** is considered best practice for any production firewall. All unmatched traffic is blocked and logged, giving security teams an audit trail to detect and investigate suspicious activity.

- **Remote access protocols** — RDP and SSH are the two most common ways administrators remotely manage servers. Knowing how to configure, test, and restrict access to these protocols is a core sysadmin skill.

---

## What I Learned

- How to manually configure static IP addresses and routing on Linux
- How pfSense processes firewall rules and how rule order affects traffic
- The difference between stateless and stateful packet inspection
- How to read and interpret firewall logs for security analysis
- How VirtualBox internal networking works and its quirks
- Practical troubleshooting methodology — isolating problems layer by layer
- Why network segmentation matters from a security perspective
- How the Anti-Lockout rule in pfSense affects firewall design decisions

---

## Author

Kalala | Network Security & Systems Administration
