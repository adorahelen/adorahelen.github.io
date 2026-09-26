---
title: "Network Recon on Public Wi-Fi — Passive Capture, Active Scanning, and the Line of Authorization"
date: 2026-09-26T19:00:00+09:00
tags:
  - security
  - network
  - wireshark
  - nmap
  - arp
  - ethics
summary: "Can packet capture and host discovery tell you who is connected and what they are doing? How switch/wireless isolation and encryption block eavesdropping, the intrusiveness gap between -sn and -sS, and why authorization — not capability — is the real boundary of the exercise."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/network-recon-public-wifi/)**

> **Type**: Lower-layer (L2–L4) network lab notes + ethics/legal analysis.
> **In one line**: The same command changes character with the **venue (authorization)**. ARP is the canonical way to enumerate assets, but seeing others' *activity* is doubly blocked by switch/wireless isolation and traffic encryption — and breaking that wall requires ARP spoofing (illegal interception).

> ⚠️ **Scope note**: All captured measurements below are **the author's own device traffic**. Other devices' names, MACs, and real IPs (personally identifiable) are anonymized (`Device A`, `XX:XX:...`), and active scanning / interception techniques are described conceptually, assuming a **network you own or are explicitly authorized to test**.

---

## 1. Three Questions

Connected to a public café Wi-Fi, can you determine:

1. How many devices are connected to this network?
2. What IP/MAC does each device have?
3. What is each device doing?

Up front: ①/② are partially answerable (in an authorized environment), but ③ is impossible by legitimate means because of a triple defense — **switch/wireless isolation + host firewalls + traffic encryption**. And a public café has **no administrator to authorize a scan**, which is the fundamental difference from a corporate lab.

## 2. Tools and Environment

- **tcpdump / tshark** — packet capture and analysis (`brew install --formula wireshark` installs tshark without the GUI)
- **nmap** — host discovery
- **arp** — local ARP cache inspection

The key variable in a public AP is **client isolation (AP isolation)**. Most café APs enable it so patrons on the same AP cannot see each other.

## 3. Step 1 — Passive Capture of My Own Traffic (Legitimate Anywhere)

The most legitimate starting point is to observe only **the traffic my own device sends and receives**. With capture running, I generated a DNS query and fresh HTTP/HTTPS connections to watch the textbook flow.

```bash
tshark -i en0 -w lab.pcap -q &
dig +short example.com
curl -s http://neverssl.com -o /dev/null   # plaintext HTTP
curl -s https://example.com -o /dev/null    # HTTPS (TLS)
```

Five observations:

1. **ARP** — MAC address resolution (`who-has` / `is-at`) between the gateway and my device.
2. **DNS** — the `example.com` query returned multiple IPs (CDN / load balancing).
3. **TCP 3-way** — `SYN → SYN,ACK → ACK`, with Seq/Ack numbers and MSS/window-scale negotiation.
4. **TLS ClientHello** — the **SNI (server name) is exposed in cleartext**; everything after is `Application Data` (ciphertext).
5. **Plaintext HTTP** — the `GET` method, host, path, and User-Agent are all visible.

**Lesson:** on public Wi-Fi, plaintext HTTP exposes its full content, while HTTPS leaks only the SNI. This is exactly why HTTPS/VPN are essential on public networks.

## 4. Step 2 — Host Discovery, and How Privilege Changed the Result

The same command `nmap -sn` produced **completely different results depending on privilege** (in an authorized test environment).

| Method | Probe technique | Result | Cause |
| --- | --- | --- | --- |
| non-root `nmap -sn` | ICMP + TCP SYN(80,443) | almost no responses | host firewalls silently drop |
| sudo `nmap -sn` | **ARP requests** | many found | ARP cannot be ignored |
| `arp -a` | local cache | neighbors left by the scan | — |

**Key point:** on the same L2 segment, the truth of host discovery is not ICMP/TCP but **ARP**. A device may ignore ICMP, but if it ignores ARP it cannot communicate at all, so it must respond. That is why local-segment discovery uses `sudo nmap -sn` or `arp-scan`.

Anonymized example result:

| Device | MAC (example) | Type |
| --- | --- | --- |
| Device A | XX:XX:XX:XX:XX:XX | smartphone (randomized MAC) |
| Device B | XX:XX:XX:XX:XX:XX | laptop |
| Gateway | XX:XX:XX:XX:XX:XX | router |

> **What if it were a public AP?** With client isolation on, even an ARP scan finds nothing but the gateway, because the switch/AP will not forward L2 frames between patrons. So ①/② are effectively unanswerable — and that is normal.

**MAC randomization:** modern phones randomize their MAC for privacy (second nibble `2/6/A/E`), defeating vendor (OUI) tracking. Only fixed infrastructure (IP cameras, wired terminals) exposes a real MAC and can be tracked long-term.

## 5. The Intrusiveness Scale — How Far Is Allowed

Network actions form a scale of intrusiveness, and the venue (authorization) determines what is permitted. **This table is the core of this post.**

| Action | Character | My own network | Public AP (unauthorized) |
| --- | --- | --- | --- |
| Capture my own traffic | passive · mine | OK | OK |
| Observe broadcast/mDNS | passive · receive only | OK | mostly OK |
| `nmap -sn` (host discovery) | active · low | OK | **gray area** |
| `nmap -sS` (port scan) | active · high | OK | **over the line** |
| ARP spoofing (MITM) | interception attack | can experiment | illegal |

The dividing line is **"presence check" vs "internal recon."**

- **`-sn`** only asks "is there a device at that address?" Like knocking to see if someone is home — it does not look inside. Even unauthorized it is a gray area.
- **`-sS`** asks "what ports/services does that device open?" Opening every door and window one by one — the start of vulnerability recon. Unauthorized port scanning of others' devices crosses into intrusion-attempt territory from here.

So **the line not to cross on a public AP starts at `-sS` port scanning**; `-sn` is a gray area best avoided without authorization. Active scanning should only be done on **(a) a network you own, or (b) one you are explicitly authorized to test**.

## 6. Why Seeing "Activity" Requires an Attack (Concept)

The key misconception: it is not that "you would see it, just encrypted" — **other people's packets never even reach your NIC** before encryption is a factor.

Old hubs copied every frame to all ports, so promiscuous mode saw everyone's traffic. Modern switches forward only to the destination port via a MAC address table (CAM), and wireless public APs add **client isolation** for double isolation. So passive capture only sees broadcast/multicast and your own traffic.

**ARP spoofing (concept).** ARP has no authentication, so a forged reply is accepted and updates the cache. If an attacker floods both the victim and gateway with "the other's MAC is my MAC," each side's frames route through the attacker (MITM). Stealthy interception forwards received packets on to the real destination (IP forwarding) so the victim notices nothing; without forwarding it becomes denial of service, not eavesdropping.

**Yet the content is mostly still ciphertext.** Even intercepted, HTTPS/TLS remains encrypted; what is visible is metadata — SNI, volume, peer IP. Today the practical yield of MITM is closer to "plaintext-protocol exposure" and "metadata collection."

## 7. Defenses and Self-Protection

**Network (AP) side**

| Technique | Principle |
| --- | --- |
| Client Isolation | blocks L2 traffic between same-AP users |
| DAI (Dynamic ARP Inspection) | discards forged ARP by comparing to DHCP bindings |
| DHCP Snooping | blocks bindings off untrusted ports; DAI's basis |
| 802.1X | authentication at connect blocks unauthorized use |

**User (my) side**

- **Use a VPN** — encrypts the public-network segment in a tunnel; even under MITM, content and SNI are hidden.
- **HTTPS only** — avoid plaintext HTTP; HSTS blocks SSL stripping.
- **Defer sensitive work** — avoid banking / internal systems on public networks, or use a VPN first.
- **Turn off sharing** — disable unnecessary discovery/sharing (AirDrop, etc.).

## 8. Conclusion — What the Venue Changes

The same technical action changes character with the venue (authorization). Asset enumeration is canonically ARP-based, but others' *activity* is doubly blocked by isolation and encryption.

- **How many?** — In an isolated environment, usually only the gateway is visible. The exact count cannot be known without authorization.
- **Each IP/MAC?** — Even if observed, it is third-party PII and cannot be enumerated or published.
- **Each activity?** — Impossible by legitimate means. Interception requires illegal MITM, which is not performed.

**In one line:** whether an active action is permitted is determined by **authorization, not capability**. On public Wi-Fi, the safe line is **your own device, your own traffic, and observing network settings**. If you need to practice active scanning or interception, do it in an authorized environment such as **your own home lab**.

---

*This post is a study/defense-oriented summary and does not encourage unauthorized scanning or interception of others' networks or devices.*
