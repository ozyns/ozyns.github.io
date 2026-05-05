---
title: "Red Team vs Blue Team — Part 1: Building the Lab"
date: 2026-02-21 00:00:00 +0000
categories: [Cybersecurity, Penetration Testing]
tags: [homelab, pfsense, active-directory, tailscale, vmware, kali-linux, network-segmentation]
---

> ⚠︎ **Disclaimer:** This project was conducted in a fully isolated virtual lab for academic purposes only. All techniques described are strictly educational and were performed in a controlled environment with no real systems involved.

---

## Series Overview

This is **Part 1 of a 4-part series** documenting a full Red Team vs Blue Team exercise built from scratch.

| Part | Topic | Status |
|------|-------|--------|
| **Part 1 — You are here** | Building the lab. network design, firewall, Active Directory, DMZ |
| Part 2 | Vulnerability Assessment — Nmap scanning, 12 findings |
| Part 3 | Exploitation & Pivoting — AD compromise, DMZ breach, lateral movement |
| Part 4 | Blue Team Response — Snort IDS deployment and detection |

The goal of the whole project was to simulate a real enterprise environment, attack it systematically from both outside and inside, then build a detection system to catch those attacks. This first post covers everything we built before a single attack was launched.

---

## What Are We Simulating?

Real corporate networks aren't flat. They're divided into **zones** with different trust levels, separated by firewalls and access controls:

- The **internet** — untrusted, anyone can be here
- The **DMZ** — semi-trusted, hosts public-facing services like web or FTP servers
- The **LAN** — trusted, internal workstations and servers

> **What is a DMZ?** A Demilitarized Zone is a network segment sitting between the internet and the internal network. It hosts services that need to be reachable from outside while keeping them isolated from sensitive internal systems. If an attacker compromises a DMZ server, they still can't directly reach the internal network in theory.

Our lab replicates exactly this architecture: external attackers, a firewall, a DMZ, an internal LAN, a Windows domain, and an insider threat, all running as virtual machines on a single physical machine.

---

## Chapter 1 — Project Objectives

We set out to accomplish five things:

1. **Build a realistic segmented network** with WAN, LAN, and DMZ zones connected through a firewall.
2. **Conduct a vulnerability assessment** using Nmap to identify weaknesses before attacking anything.
3. **Launch targeted attacks** from two perspectives — an external attacker and an insider threat already on the internal network.
4. **Pivot through the DMZ** — compromise a public-facing server and use it as a stepping stone to reach internal machines.
5. **Deploy a Snort IDS** to detect the attacks we launched, and prove the detection rules actually work.

### Methodology

We followed the standard penetration testing lifecycle:

```
Reconnaissance → Vulnerability Scanning → Exploitation → Pivoting → Detection & Defense
```

Each phase feeds into the next. What we find in scanning shapes what we exploit. What we exploit becomes the basis for our Snort detection rules in the defense phase.

---

## Chapter 2 — Building the Network

### 2.1 The Full Picture

Everything runs inside **VMware Workstation Pro**. Virtualization lets us create multiple isolated networks on a single machine and take snapshots before destructive tests, essential when you're intentionally breaking things.

The central piece holding everything together is **pfSense**, an open-source firewall/router that controls all traffic between zones.

<div id="topology-container" style="margin: 2rem 0;">
  <p id="topology-loading" style="text-align:center; color:#888;">Loading topology...</p>
</div>

<script>
(() => {
  const data = {
    zones: [
      {
        name: "WAN (192.168.19.0/24)",
        color: "#f78166",
        machines: [
          { name: "pfSense (em0)", ip: "192.168.19.132", role: "Firewall / Router / VPN" }
        ]
      },
      {
        name: "Tailscale Overlay (100.64.0.0/10)",
        color: "#d2a8ff",
        machines: [
          { name: "External Kali #1", ip: "100.102.36.10", role: "External attacker" },
          { name: "External Kali #2", ip: "100.102.36.11", role: "External attacker" },
          { name: "pfSense (tailscale0)", ip: "100.102.36.58", role: "VPN gateway" }
        ]
      },
      {
        name: "LAN — VMNET2 (192.168.1.0/24)",
        color: "#58a6ff",
        machines: [
          { name: "Windows XP SP3", ip: "192.168.1.5", role: "Primary internal target" },
          { name: "Internal Kali", ip: "192.168.1.15", role: "Insider threat simulation" }
        ]
      },
      {
        name: "DMZ — VMNET4 (192.168.3.0/24)",
        color: "#ffa657",
        machines: [
          { name: "Metasploitable 2", ip: "192.168.3.16", role: "DMZ target / pivot point" }
        ]
      },
      {
        name: "AD Lab (192.168.2.0/24)",
        color: "#3fb950",
        machines: [
          { name: "Windows Server 2022", ip: "192.168.2.10", role: "Domain Controller (DC1)" },
          { name: "Windows 10 Client", ip: "192.168.2.100+", role: "Domain-joined workstation" }
        ]
      }
    ]
  };

  let html = '<div style="font-family:monospace; font-size:13px;">';
  data.zones.forEach(zone => {
    html += `<div style="border:1px solid ${zone.color}; border-radius:6px; margin-bottom:1rem; overflow:hidden;">`;
    html += `<div style="background:${zone.color}22; padding:8px 12px; border-bottom:1px solid ${zone.color}; color:${zone.color}; font-weight:bold;">${zone.name}</div>`;
    html += '<table style="width:100%; border-collapse:collapse;">';
    html += '<thead><tr style="border-bottom:1px solid #30363d;"><th style="padding:6px 12px; text-align:left; color:#8b949e;">Machine</th><th style="padding:6px 12px; text-align:left; color:#8b949e;">IP</th><th style="padding:6px 12px; text-align:left; color:#8b949e;">Role</th></tr></thead>';
    html += '<tbody>';
    zone.machines.forEach((m, i) => {
      const bg = i % 2 === 0 ? 'background:#0d1117;' : 'background:#161b22;';
      html += `<tr style="${bg}"><td style="padding:6px 12px; color:#c9d1d9;">${m.name}</td><td style="padding:6px 12px; color:#58a6ff;">${m.ip}</td><td style="padding:6px 12px; color:#8b949e;">${m.role}</td></tr>`;
    });
    html += '</tbody></table></div>';
  });
  html += '</div>';

  document.getElementById('topology-loading').style.display = 'none';
  document.getElementById('topology-container').innerHTML = html;
})();
</script>

---

### 2.2 WAN Segment — The Outside World

The WAN segment (`192.168.19.0/24`) represents the internet. pfSense's WAN interface (`em0`) connects here with IP `192.168.19.132`.

**No attacker machines sit directly on this network.** Instead, external attackers connect through **Tailscale**, simulating a remote attacker reaching internal systems via an encrypted VPN tunnel, which is a realistic scenario.

The only service directly exposed to the WAN is an FTP port forward pointing at the DMZ, which becomes the initial entry point in Part 3.

---

### 2.3 LAN Segment

The LAN (`192.168.1.0/24`) is the internal trusted network, what you'd imagine as a company's internal office network.

#### 2.3.1 pfSense Firewall

pfSense is the heart of the lab. It manages five network interfaces and controls all inter-zone traffic.

**Key firewall rules:**

During the attack phase, Tailscale rules were intentionally **disabled**, meaning external attackers couldn't reach the LAN or DMZ through the VPN directly. The only way in was through the WAN port forward pointing at the FTP service on Metasploitable.

> **What is NAT / port forwarding?** Network Address Translation lets a router redirect traffic arriving on a specific port to an internal machine. Here, any FTP connection to pfSense's WAN IP gets silently forwarded to Metasploitable's private IP, making it reachable from "outside" without exposing its real address.

This one port forward, FTP traffic on the WAN redirected to the DMZ host  becomes the entire initial attack surface in Part 3.

#### 2.3.2 Windows XP Target Machine

The primary internal target is a **Windows XP SP3** virtual machine at `192.168.1.5`. We chose XP specifically because it carries well-known unpatched vulnerabilities — most notably **MS08-067**, which we exploit in Part 3 during the pivoting phase.

> **What is MS08-067?** A critical remote code execution vulnerability in Windows XP's Server service. Exploiting it gives an attacker a SYSTEM-level shell without needing any credentials. It's the same vulnerability used by the Conficker worm in 2008.

#### 2.3.3 Internal Kali Linux — Insider Threat

A second Kali Linux machine sits directly on the LAN at `192.168.1.15`, representing a **compromised workstation or malicious insider** — someone who already has internal access.

> **Why simulate an insider threat?** Firewalls protect against attacks coming from outside. But an insider is already inside — they can scan, attack, and pivot without ever touching the perimeter firewall. This is one of the most dangerous and underestimated threat vectors in real organizations.

---

### 2.4 Tailscale Overlay Network

**Tailscale** connects our external attacker machines to the lab. It creates a private, encrypted mesh network using **WireGuard**, each machine gets a `100.x.x.x` IP and can communicate securely regardless of physical location.

> **What is Tailscale?** A zero-config VPN built on WireGuard. Instead of a traditional VPN server, Tailscale uses a coordination server to automatically establish encrypted peer-to-peer tunnels between devices. It's fast, easy, and increasingly used in real enterprise environments.

> **What is WireGuard?** A modern VPN protocol — faster and simpler than older protocols like OpenVPN or IPSec. Uses state-of-the-art cryptography and runs inside the Linux kernel.

**Tailscale node assignments:**

| Node | Tailscale IP |
|------|-------------|
| pfSense | 100.102.x.x |
| External Kali #1 | 100.102.x.x |
| External Kali #2 | 100.102.x.x |

pfSense joined the Tailscale network and got a new interface (`tailscale0`, configured as OPT5). Firewall rules allowed Tailscale traffic to reach the DMZ and LAN — giving our remote Kali machines a tunnel into the lab.

**Why this matters for security:** This illustrates why VPN entry points need to be monitored just as carefully as public-facing services. An attacker who compromises a VPN-connected machine effectively bypasses the perimeter firewall entirely. In Part 4, we configure Snort specifically on the Tailscale interface because of this.

---

### 2.5 Active Directory Lab — Windows Server 2022

To simulate a realistic enterprise environment, we added a full **Active Directory** domain. This is what enables the most sophisticated attacks in Part 3 — ZeroLogon, LDAP Anonymous Binding, AS-REP Roasting.

> **What is Active Directory?** Microsoft's directory service used in virtually every corporate Windows environment. It manages users, computers, groups, and policies across an organization. A **Domain Controller (DC)** is the server that runs AD — it's the most critical server in any Windows network. If an attacker controls the DC, they control everything.

#### 2.5.1 Network Design

The AD lab runs on its own isolated subnet (`192.168.2.0/24`) connected to pfSense, with:

- **Deny all inbound** from other internal networks to the AD segment
- **Allow outbound** from AD_LAB to the DMZ and LAN (to enable attack scenarios)
- **DHCP disabled** on pfSense — the Domain Controller handles DHCP itself

#### 2.5.2 Setting Up Windows Server 2022

The setup followed these steps:

**Step 1 — Install Windows Server 2022**
Installed from a Microsoft evaluation ISO onto a VM with static IP `192.168.2.10`.

**Step 2 — Install Active Directory Domain Services**
Added via Server Manager → Add Roles and Features → Active Directory Domain Services + DNS Server.

**Step 3 — Create a New Forest**
Promoted the server to Domain Controller and created a new forest with root domain **`ad.lab`**.

> **Why `ad.lab` and not `.local`?** The `.local` suffix can interfere with mDNS traffic on local networks. Using `.lab` avoids this. Also avoid real TLDs like `.com` or `.org` for lab domains, they could accidentally resolve to real internet addresses.

**Step 4 — Configure DNS Forwarders**
Pointed at a public DNS server so the DC can resolve both internal and external names.

**Step 5 — Set Up DHCP Server**
Configured to lease addresses in the range `192.168.2.100–192.168.2.200` to domain-joined clients.

**Step 6 — Create Domain User Accounts**
Two standard domain user accounts created for testing: `IT_User01` and `IT_User02`. One is deliberately configured with weak settings that enable attacks in Part 3.

**Step 7 — Join a Windows 10 Client**
A Windows 10 Enterprise VM was joined to the `ad.lab` domain, making it a realistic domain-joined workstation.

---

### 2.6 DMZ — Metasploitable 2

The final piece is **Metasploitable 2**, placed in the DMZ subnet (`192.168.3.0/24`) at IP `192.168.3.16`.

> **What is a DMZ in practice?** Think of a bank, the lobby is accessible to the public, but the vault requires extra credentials. The DMZ is the lobby: exposed enough to serve external users, isolated from the real sensitive assets inside.

Metasploitable serves as our "lobby server" intentionally vulnerable, exposed to the outside via a port-forwarded FTP service, and the launching point for the pivoting attack in Part 3.

**pfSense rules for the DMZ:**

- **Inbound:** Only FTP traffic (via WAN port forward) reaches Metasploitable
- **Outbound:** DMZ hosts can freely reach the LAN, this is the **deliberate misconfiguration** that makes pivoting possible

This single misconfiguration allowing outbound DMZ traffic to reach the LAN, is what turns a "contained" compromise into a full internal network takeover. We exploit it in detail in Part 3.

---

## Lab Summary

<div id="summary-container" style="margin: 2rem 0;">
  <p id="summary-loading" style="text-align:center; color:#888;">Loading summary...</p>
</div>

<script>
(() => {
  const machines = [
    { name: "pfSense", os: "FreeBSD", role: "Firewall / Router / VPN", ip: "192.168.19.132 (WAN)" },
    { name: "Windows Server 2022", os: "Windows Server", role: "Domain Controller (DC1)", ip: "192.168.2.10" },
    { name: "Windows 10", os: "Windows 10", role: "Domain-joined workstation", ip: "192.168.2.100+" },
    { name: "Windows XP SP3", os: "Windows XP", role: "Internal attack target", ip: "192.168.1.5" },
    { name: "Metasploitable 2", os: "Ubuntu (vulnerable)", role: "DMZ target / pivot point", ip: "192.168.3.16" },
    { name: "Internal Kali", os: "Kali Linux", role: "Insider threat simulation", ip: "192.168.1.15" },
    { name: "External Kali #1", os: "Kali Linux", role: "External attacker", ip: "100.102.x.x (Tailscale)" },
    { name: "External Kali #2", os: "Kali Linux", role: "External attacker", ip: "100.102.x.x (Tailscale)" }
  ];

  let html = '<table style="width:100%; border-collapse:collapse; font-size:13px;">';
  html += '<thead><tr style="border-bottom:2px solid #30363d;">';
  ['Machine','OS','Role','IP'].forEach(h => {
    html += `<th style="padding:8px 12px; text-align:left; color:#8b949e;">${h}</th>`;
  });
  html += '</tr></thead><tbody>';
  machines.forEach((m, i) => {
    const bg = i % 2 === 0 ? '#0d1117' : '#161b22';
    html += `<tr style="background:${bg}; border-bottom:1px solid #21262d;">
      <td style="padding:8px 12px; color:#c9d1d9; font-weight:bold;">${m.name}</td>
      <td style="padding:8px 12px; color:#8b949e;">${m.os}</td>
      <td style="padding:8px 12px; color:#8b949e;">${m.role}</td>
      <td style="padding:8px 12px; color:#58a6ff; font-family:monospace;">${m.ip}</td>
    </tr>`;
  });
  html += '</tbody></table>';

  document.getElementById('summary-loading').style.display = 'none';
  document.getElementById('summary-container').innerHTML = html;
})();
</script>

---

## What's Next

With the lab fully built and every machine configured, we're ready to start finding weaknesses.

In **Part 2**, we conduct the full vulnerability assessment:
- Nmap scanning methodology (ping sweep, service enumeration, SYN scan, NSE scripts)
- NetBIOS name resolution with nbtscan
- LDAP Root DSE enumeration confirming anonymous bind
- Full catalogue of all 12 vulnerabilities found, with severity ratings and CVE references

---

*Tools used in this part: VMware Workstation Pro · pfSense · Tailscale · Windows Server 2022 · Kali Linux · Metasploitable 2*

*→ Continue to [Part 2: Vulnerability Assessment](#) — coming soon*
