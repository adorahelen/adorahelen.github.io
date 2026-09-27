---
title: "One Nationwide Network Behind Every Starbucks — Korea's KT-Managed Hotspot Architecture and Public-WiFi Risk"
date: 2026-09-27T10:00:00+09:00
tags:
  - security
  - network
  - wifi
  - evil-twin
  - cidr
  - privacy
summary: "Two Starbucks 30km apart handed out the same gateway IP, MAC, and address range. This unpacks the KT-operated nationwide managed WiFi (store AP → backhaul → central core), reads the /12 vs /24 through the netmask, and covers public-WiFi risks like evil twins and missing client isolation — with defenses."
---

> 🇰🇷 **[이 글의 한국어판 →](/ko/posts/network-recon-public-wifi/)**

> **Type**: Measurement-based network-architecture analysis + public-WiFi threat/defense notes.
> **In one line**: Two Starbucks branches 30km apart look like the same network because **KT operates the whole chain as one managed WiFi** — and that architecture carries the classic public-network risks (evil twin, missing client isolation) alongside its convenience.

> ⚠️ **Scope & ethics**: The concrete observations below come mainly from **connecting normally with my own device and then inspecting my own interface, gateway, and neighbor cache** (passive). Active scanning / interception techniques are described conceptually, assuming a **network you own or are authorized to test**. Third-party device identifiers (names, full MACs) are anonymized.

---

## 1. Two Locations 30km Apart, One Network

I connected at the Paju Geumchon branch (2026-09-26) and the Gwanghwamun branch (2026-09-27) and checked the interface and gateway.

| Field | Paju | Gwanghwamun | Difference |
| --- | --- | --- | --- |
| My IP | 172.30.96.14 | 172.30.59.196 | DHCP reassignment |
| Netmask | /12 | /12 | same |
| Gateway IP | 172.30.1.254 | 172.30.1.254 | same |
| Gateway MAC | 00:17:c3:XX:XX:XX | 00:17:c3:XX:XX:XX | **same (OUI match)** |

The IP changes per DHCP lease, but the **gateway MAC was identical at both branches**. A MAC is a device-unique value, so two stores 30km apart share the **same (or identically standardized) gateway**. That breaks the assumption of "an independent router per store."

## 2. The Netmask Explains It

An IP is 32 bits, and the netmask (`/N`) sets how many leading bits are the "network label."

- **`/12`** = first 12 bits are network → `172.16.0.0 – 172.31.255.255`, about **1,048,576** addresses.
- **`/24`** = first 24 bits are network → `172.30.59.0 – .255`, **256** addresses.

`172.30.96.14` (Paju) and `172.30.59.196` (Gwanghwamun) are the **same network under `/12`** (both inside 172.16–31) but **different segments under `/24`** (.96 vs .59). The chain uses one nationwide `/12` plan and hands out different `/24` slices per location/pool — a **two-tier structure**.

## 3. Architecture: KT-Operated Nationwide Managed WiFi

Starbucks Korea's in-store WiFi is **provided by KT**. The secure SSID is `KT_starbucks_Secure`, and the access portal sits on KT's (olleh) domain. So Starbucks does not place an independent router in each store — it **outsources to a carrier-managed WiFi**.

```
🌐 Internet
   ↑
HQ / IDC core (redundant, HA)
 [Wireless Controller WLC] [Central DHCP/AAA] [Gateway 172.30.1.254]
   ↑  Carrier backhaul · private-network tunnel (CAPWAP / GRE)
   ↑
🏬 Paju store AP     🏬 Gwanghwamun store AP     🏬 Hundreds of other stores
 172.30.96.0/24       172.30.59.0/24              each a different /24
```

- Store APs are **thin/edge devices** that tunnel client traffic up to the central core.
- The gateway, DHCP, and auth live **centrally as one**, so the whole country meets the same `172.30.1.254`.
- Physically hundreds of APs are scattered; **logically it is one network** — the same principle as a corporate branch VPN unifying every office into one private range.

**Why run it this way?** Outsourcing removes per-store IT staffing, enables nationwide central management, and lets a single name+email login work anywhere (capturing customer data). Notably, early Starbucks WiFi was open (unencrypted) and criticized for it; KT later introduced a secure-connection SSID to improve this.

## 4. Why It Is Risky

**① A target for evil twins.** My device treats Paju and Gwanghwamun as the "same saved WiFi" and auto-joins/auto-trusts. If an attacker stands up a fake AP cloning this SSID and gateway setup, devices can join automatically. And since the gateway MAC is uniform nationwide, **MAC alone cannot distinguish real from fake.** Evil-twin attack and detection has been studied extensively since 2005.

**② Client isolation and cleartext exposure.** If a managed hotspot does not enforce in-store client isolation, customers on the same segment can enumerate each other. Even with isolation on, devices broadcast **their names and shared-service info in cleartext via mDNS/Bonjour (UDP 5353)**. So isolation alone does not fully protect privacy.

## 5. Defenses — What a User Can Do

- **Use the secure SSID** — prefer a secure connection like `KT_starbucks_Secure` over the open one (encryption).
- **Always use a VPN** — tunneling the public-network segment neutralizes both evil twins and missing isolation.
- **Turn off auto-join** — disable "auto-join" for saved public WiFi to prevent auto-connecting to a fake AP.
- **Do sensitive work over cellular** — banking / internal access over LTE/5G when possible.
- **Check for HTTPS** — plaintext HTTP exposes content directly; HSTS blocks SSL stripping.

## 6. Methodology and Ethics

The concrete figures here come mainly from **connecting normally with my own device** and reading my own interface, gateway, and neighbor cache via `ifconfig`/`ip`/`arp` (passive). Active host enumeration (`nmap -sn`) and interception like ARP spoofing must only be done on **a network you own or are explicitly authorized to test**, and appear here only as concept. Third-party device identifiers and the gateway MAC's lower bytes are masked. Whether an active action is permitted is decided by **authorization, not capability**.

## Sources

- [Evil Twin Attack in Wi-Fi Networks: Evolution, Mutation Taxonomy, and Exposure Time Analysis (2005–2026), MDPI Electronics](https://doi.org/10.3390/electronics15153432)
- [DPETAs: Detection and Prevention of Evil Twin Attacks on Wi-Fi Networks, Springer](https://link.springer.com/chapter/10.1007/978-981-16-9012-9_45)
- [WPFD: Active User-Side Detection of Evil Twins, MDPI Applied Sciences](https://www.mdpi.com/2076-3417/12/16/8088)
- [Bypassing WiFi Client Isolation, Pulse Security](https://pulsesecurity.co.nz/articles/bypassing-wifi-client-isolation)
- [You're probably overestimating public Wi-Fi 'client isolation', dev.to](https://dev.to/lafine_systemsdesign/youre-probably-overestimating-public-wi-fi-client-isolation-2ao0)
- [Starbucks WiFi security strengthened with KT (Korean), Newsworks](https://www.newsworks.co.kr/news/articleView.html?idxno=774146)
- [Starbucks WiFi secure-connection guide (Korean), KT olleh](https://first.wifi.olleh.com/starbucks/secure_kor.html)

---

*This post is a study/defense-oriented write-up and does not encourage unauthorized scanning or interception of others' networks or devices. It is not a vulnerability report against a specific operator, but a user-side understanding of public-WiFi architecture.*
