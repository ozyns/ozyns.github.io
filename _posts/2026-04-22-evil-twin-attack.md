---
title: "Evil Twin Attack with Fluxion"
date: 2026-04-22 00:00:00 +0000
categories: [Cybersecurity, Penetration Testing]
tags: [wifi, fluxion, evil-twin, kali-linux, wpa]
---

# Evil Twin Attack with Fluxion

> **⚠️ Disclaimer:** This project is for educational purposes only. Performing this attack without explicit permission from the network owner is illegal. Always conduct security tests in controlled environments with proper authorization.

---

## Introduction

**Fluxion** is a security auditing and penetration testing tool targeting wireless access points. It leverages social engineering to retrieve WPA/WPA2 authentication passwords from users. Fluxion supports two types of attacks:

- **Handshake Snooper** — Captures WPA/WPA2 authentication hashes from the 4-way handshake.
- **Captive Portal (Evil Twin)** — Creates a rogue network mimicking the target to phish credentials.

> **Note:** A wireless adapter capable of **monitor mode** is required.

---

## Installation

Fluxion is not included by default in Kali Linux. Clone it from the official GitHub repository:

```bash
git clone https://github.com/FluxionNetwork/fluxion.git
cd fluxion
./fluxion.sh -i
```

The `-i` flag triggers installation and dependency checking. If any dependencies are reported missing, install them manually before proceeding.

![Fluxion startup screen](_images/fluxion-startup.png)
*Fluxion ASCII logo on launch*

---

## Step 1 — Scanning for SSIDs

Select the **Handshake Snooper** attack and choose the wireless interface (e.g., `wlan0`). Fluxion will automatically enable monitor mode.

Then scan all available SSIDs on the 2.4 GHz band. A new `xterm` window will open listing nearby networks. Let it run until your target appears, then press `Ctrl+C`.

![SSID scan results](_images/ssid-scan.png)
*Scanning for available networks*

---

## Step 2 — Configuring the Handshake Capture

Select the target network from the list (e.g., `pen_test`). Then configure:

| Option | Choice | Reason |
|---|---|---|
| Tracking interface | Skip | Using same interface |
| Capture method | `mdk4` | Aggressive, fast deauth |
| Hash verifier | `cowpatty` | More reliable than aircrack-ng |
| Timing mode | Synchronous | Safer in VM environments |

---

## Step 3 — Handshake Snooper Attack

The attack begins — an `xterm` log viewer opens. All clients are deauthenticated from the target AP. When they attempt to reconnect, Fluxion captures the 4-way handshake.

![Handshake captured](_images/handshake-captured.png)
*Successful handshake capture in the log viewer*

Once a valid hash is captured, close the log window and move on to the Captive Portal attack.

---

## Step 4 — Configuring the Captive Portal

Select **Captive Portal** from the menu. Configure:

| Option | Choice |
|---|---|
| Deauth interface | `wlan0` |
| Deauth method | `mdk4` |
| Rogue AP | `hostapd` (faster than airbase-ng) |
| Hash verifier | `cowpatty` |
| Handshake source | Captured `.cap` file |
| SSL certificate | Generate new |
| Internet connectivity | Disconnected (lower failure rate) |
| Portal template | Generic (English) |

---

## Step 5 — The Attack in Action

Fluxion deauthenticates all clients from the real AP. Victims see two networks with the same SSID — the legitimate one and the rogue one.

![Victim view: two networks](_images/victim-wifi-view.png)
*The victim sees two networks with the same name*

When the victim connects to the rogue network, a captive portal page appears in their browser asking for the WPA key. If the entered password doesn't match the captured hash, the user is prompted to try again.

![Captive portal page](_images/captive-portal-page.png)
*The phishing portal displayed to the victim*

### Attacker's xterm Windows

| Window | Function |
|---|---|
| DHCP Service | Assigns an IP to the victim's device |
| hostapd | Captive portal log |
| AP Authenticator | Logs SSID, MAC addresses, failed attempts |
| DNS Service | Resolves all DNS queries to the rogue portal |
| Web Service | Hosts the phishing portal |
| Jammer | Handles continuous deauthentication |

![Attacker view: xterm windows](_images/attacker-xterm.png)
*Multiple xterm windows during the attack*

---

## Result

Once the victim enters the correct password, the captured hash is verified against the previously obtained one. On a match:

- The victim is redirected to the legitimate AP.
- The jammer stops — all users reconnect normally.
- The captured password is saved to:

```
/root/fluxion/attacks/Captive Portal/netlog/
```

---

## Conclusion

The **Evil Twin attack** creates a fraudulent access point mimicking a legitimate network to trick users into submitting credentials through a phishing portal. It typically combines client deauthentication with a convincing captive portal, and can lead to Wi-Fi password compromise, traffic interception, or malware installation.

### Defensive Measures

- Verify networks before connecting; disable auto-connect on public SSIDs.
- Use a trusted **VPN** on unknown networks.
- Keep devices and firmware up to date.
- Prefer **WPA3** and management frame protection **(802.11w)** where available.
- Network admins should monitor for duplicate SSIDs and abnormal signal levels, deploy strong infrastructure-side authentication, and educate users about rogue portal signs.

---

*Tools used: Kali Linux, Fluxion, mdk4, hostapd, cowpatty*
