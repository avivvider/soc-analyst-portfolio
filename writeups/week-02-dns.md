# Week 2 – DNS query and response analysis in Wireshark

**Date:** 10/11/2026
**Capture:** `nslookup google.com 8.8.8.8` on Wi-Fi, display filter: `dns`
**Goal:** Break one DNS query and its response into layers (Ethernet, IPv4, UDP, DNS) and understand how each layer points to the next one.

> Note: MAC addresses and the interface ID were removed from the screenshots, and the packet list shows only the two relevant packets. I queried 8.8.8.8 directly after a troubleshooting session showed that the default IPv6 DNS server did not respond.

---

## DNS query – Frame 547

Source: 192.168.1.123 → Destination: 8.8.8.8

| Layer | Wireshark display | Meaning |
|---|---|---|
| Frame 547 | 70 bytes on wire = 70 captured, Encapsulation: Ethernet, Protocols in frame: eth:ethertype:ip:udp:dns | Capture metadata (not an OSI layer). The protocol chain shows the encapsulation order |
| Ethernet II (L2) | Src: my NIC, Dst: router, Type: IPv4 (0x0800) | The next hop on the local network is the router. The EtherType tells the receiver that the payload is an IPv4 packet |
| IPv4 (L3) | Src: 192.168.1.123, Dst: 8.8.8.8, TTL: 128, Protocol: UDP (17), Total Length: 56, Identification: 0xeea3 | My PC sends the query to Google's public DNS server. The Protocol field tells the receiver that the payload is UDP |
| UDP (L4) | Src port: 51775, Dst port: 53, Length: 36, UDP payload: 28 bytes | Port 53 is DNS. The source port is a random high port, and the reply will come back to it |
| DNS | Transaction ID: 0x0002, Flags: 0x0100 (Standard query, recursion desired), Questions: 1, Answer RRs: 0, Response In: 548 | A standard query with no answers yet. The Transaction ID matches it to the response in frame 548 |

The frame is 70 bytes: 14 (Ethernet) + 20 (IPv4) + 8 (UDP) + 28 (DNS). The UDP Length of 36 is 8 + 28, and the IP Total Length of 56 is 20 + 36, without the Ethernet header.

### Screenshots
1. Frame – `images/dns/01_Frame_547_Request.png`
2. Ethernet II – `images/dns/02_Frame_547_Request_Ethernet.png`
3. IPv4 – `images/dns/03_Frame_547_Request_IPv4.png`
4. UDP – `images/dns/04_Frame_547_Request_UDP.png`
5. DNS – `images/dns/05_Frame_547_Request_DNS.png`

---

## DNS response – Frame 548

Source: 8.8.8.8 → Destination: 192.168.1.123

| Layer | Wireshark display | Meaning |
|---|---|---|
| Frame 548 | 86 bytes on wire = 86 captured, Encapsulation: Ethernet, Protocols in frame: eth:ethertype:ip:udp:dns | Capture metadata (not an OSI layer). Same protocol chain as the query |
| Ethernet II (L2) | Src: router, Dst: my NIC, Type: IPv4 (0x0800) | The reply arrives from the router, the last hop on the local network. The MAC addresses are swapped compared to the query |
| IPv4 (L3) | Src: 8.8.8.8, Dst: 192.168.1.123, TTL: 119, Protocol: UDP (17), Total Length: 72, Identification: 0x7281 | Google's DNS server replies to my PC. The TTL started from Google's own value and dropped to 119 on the way back |
| UDP (L4) | Src port: 53, Dst port: 51775, Length: 52, UDP payload: 44 bytes | The ports are swapped compared to the query, so the reply returns to the application that asked |
| DNS | Transaction ID: 0x0002, Flags: 0x8180 (Standard query response, No error), Questions: 1, Answer RRs: 1, Request In: 547, Time: 11.395 ms | The response to the query in frame 547, matched by the Transaction ID. It contains one answer: google.com, Type A, Class IN, Time to live: 210 seconds, Address: 142.250.75.110 |

The frame is 86 bytes: 14 (Ethernet) + 20 (IPv4) + 8 (UDP) + 44 (DNS). The DNS part is 12 bytes of header, 16 bytes of the repeated question, and 16 bytes of answer. This is why the response (44 bytes) is bigger than the query (28 bytes).

### Screenshots
1. Frame – `images/dns/06_Frame_548_Reply.png`
2. Ethernet II – `images/dns/07_Frame_548_Reply_Ethernet.png`
3. IPv4 – `images/dns/08_Frame_548_Reply_IPv4.png`
4. UDP – `images/dns/09_Frame_548_Reply_UDP.png`
5. DNS – `images/dns/10_Frame_548_Reply_DNS.png`

---

## Observations

- **Protocol and ports:** DNS uses UDP with destination port 53. The source port (51775) is a random high port chosen by my PC. UDP needs no handshake, so the query (28 bytes) and the response (44 bytes) are just two small packets, which is fast.
- **Matching:** The response is matched to the query by the Transaction ID (0x0002) and by the swapped ports (51775 → 53 in the query, 53 → 51775 in the response). Wireshark shows this as "Response In: 548" and "Request In: 547".
- **Two different TTLs:** The DNS TTL (210 seconds) tells how long the answer may be cached before asking again. The IP TTL (119) is a hop counter that every router decreases by one, which prevents packets from looping forever.
- **MAC vs. IP:** The destination MAC of the query is my router, not 8.8.8.8, because MAC addresses only work inside the local network. The destination IP stays 8.8.8.8 all the way, and the router forwards the packet.
- **Response time:** Wireshark calculated 11.395 ms between the query and the response.
- **Layer chaining:** Each layer points to the next one: the EtherType (0x0800) points to IPv4, the IPv4 Protocol field (17) points to UDP, and the UDP port (53) points to DNS.

---

*All captures were made in an isolated, self-owned environment for educational purposes.*
