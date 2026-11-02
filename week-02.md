# Week 2 – ICMP packet analysis in Wireshark

**Date:** 22/10/2026
**Capture:** `ping -4 google.com` on Wi-Fi, display filter: `icmp`
**Goal:** Break one ICMP Echo request and its reply into layers (Ethernet, IPv4, ICMP) and understand how each layer works.

> Note: MAC addresses and the interface GUID were hidden in the screenshots. The capture started in the middle of the ping, so the first request shown is Sequence 6.

---

## ICMP Echo request – Frame 100

Source: 192.168.1.123 → Destination: 142.250.75.46

| Layer | Wireshark display | Meaning |
|---|---|---|
| Frame 100 | 74 bytes on wire = 74 captured, Encapsulation: Ethernet, Protocols in frame: eth:ethertype:ip:icmp:data | Capture metadata (not an OSI layer). The full packet was captured, and the protocol chain shows the encapsulation order |
| Ethernet II (L2) | Src: my NIC, Dst: router | The next hop on the local network is the router |
| IPv4 (L3) | Src: 192.168.1.123, Dst: 142.250.75.46, TTL: 128, Header Length: 20 | My PC sends to Google |
| ICMP | Type 8 (Echo request), Sequence: 6 | A ping request, matched to its reply in frame 101 |

The frame is 74 bytes: 14 (Ethernet) + 20 (IP) + 8 (ICMP) + 32 (data).

### Screenshots
1. Frame – `images/frame-100.png`
2. Ethernet II – `images/frame-100-ethernet.png`
3. IPv4 – `images/frame-100-ipv4.png`
4. ICMP – `images/frame-100-icmp.png`

---

## ICMP Echo reply – Frame 101

Source: 142.250.75.46 → Destination: 192.168.1.123

| Layer | Wireshark display | Meaning |
|---|---|---|
| Frame 101 | 74 bytes on wire = 74 captured, Encapsulation: Ethernet, Protocols in frame: eth:ethertype:ip:icmp:data | Capture metadata (not an OSI layer) |
| Ethernet II (L2) | Src: router, Dst: my NIC | The reply arrives from the router, the last hop on the local network |
| IPv4 (L3) | Src: 142.250.75.46, Dst: 192.168.1.123, TTL: 114, Identification: 0x0000 | Google replies to my PC. The TTL is lower than in the request |
| ICMP | Type 0 (Echo reply), Sequence: 6, Request frame: 100, Response time: 10.349 ms | The reply to request 100, matched by the sequence number |

### Screenshots
1. Frame – `images/frame-101.png`
2. Ethernet II – `images/frame-101-ethernet.png`
3. IPv4 – `images/frame-101-ipv4.png`
4. ICMP – `images/frame-101-icmp.png`

---

## Observations

- **TTL:** The request was captured with TTL 128 before reaching the first router. The reply is a new packet created by Google, so its TTL started from Google's own value and dropped to 114 on the way back. This suggests about 14 hops, assuming an initial value of 128.
- **MAC vs. IP:** The destination MAC of the request is the router (default gateway), not Google, because MAC addresses only work inside the local network, while the destination IP stays the same all the way.
- **Identification:** This field is set by the sender of each packet (0x194a in the request, 0x0000 in the reply). The packet was not fragmented, as shown by Flags 0x0 and Fragment Offset 0.
- **Matching request and reply:** Wireshark matches the reply to the request using the ICMP Identifier (1) and Sequence Number (6), and calculates a response time of 10.349 ms.
- **Size:** The frame is 74 bytes: 14 (Ethernet) + 20 (IPv4) + 8 (ICMP) + 32 (data). The IP Total Length of 60 is 20 + 8 + 32, without the Ethernet header.
- **Surprise:** This was my first time using Wireshark. What surprised me most was how much detail it shows for a single ping: it breaks the packet into layers, matches the request to its reply, and even calculates the response time automatically.

---

*All captures were made in an isolated, self-owned environment for educational purposes.*
