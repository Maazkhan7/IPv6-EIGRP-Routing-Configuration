# 🌐IPv6 EIGRP Routing Configuration

![Cisco](https://img.shields.io/badge/Cisco-Packet%20Tracer-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![IPv6](https://img.shields.io/badge/IPv6-EIGRP-2ea44f?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Completed%20%26%20Verified-success?style=for-the-badge)
![Routing](https://img.shields.io/badge/Routing-Dynamic-orange?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA-blueviolet?style=for-the-badge)

> A hands-on Cisco Packet Tracer lab demonstrating **IPv6 EIGRP (EIGRPv6)** routing between two routers, with full neighbor adjacency, topology table, and end-to-end IPv6 connectivity verification.

---

## 📌 Overview

This lab builds a dual-site IPv6 network where **R1** and **R2** exchange routes dynamically using **EIGRP for IPv6**. Each router connects a LAN segment (via a Layer 2 switch and 3 PCs) and peers with the other router over a shared IPv6 transit link. The goal was to configure, verify, and document a clean, fully-converged IPv6 EIGRP topology — while highlighting one of the most elegant differences between **IPv4 EIGRP** and **IPv6 EIGRP** route advertisement.

---

## 🗺️ Network Topology

```
      A:A:A:A::/64                                    B:B:B:B::/64
   ┌───────────────┐                                ┌───────────────┐
   │   Switch0      │                                │   Switch1      │
   └───────┬───────┘                                └───────┬───────┘
       ┌────┼────┐                                      ┌────┼────┐
       │    │    │                                      │    │    │
     PC1  PC2  PC3                                     PC4  PC5  PC6
   (::2)(::3)(::4)                                    (::2)(::3)(::4)
       │    │    │                                      │    │    │
       └────┴────┘                                      └────┴────┘
            │                                                 │
       Gig0/1 │ A:A:A:A::1/64                 B:B:B:B::1/64 │ Gig0/1
       ┌──────┴──────┐   2001::/64 (Transit)   ┌──────────────┴──┐
       │      R1       │◄───────────────────────►│       R2        │
       └──────┬──────┘   Gig0/0        Gig0/0    └──────────────┘
              │  2001::1/64            2001::2/64
              ▼
        IPv6 EIGRP AS 1
        (EIGRPv6 Adjacency)
```

| Device   | Interface | IPv6 Address        | Network        |
|----------|-----------|----------------------|----------------|
| R1       | Gi0/0     | `2001::1/64`         | Transit link   |
| R1       | Gi0/1     | `A:A:A:A::1/64`      | LAN A          |
| R2       | Gi0/0     | `2001::2/64`         | Transit link   |
| R2       | Gi0/1     | `B:B:B:B::1/64`      | LAN B          |
| PC1–PC3  | NIC       | `A:A:A:A::2` – `::4` | LAN A hosts    |
| PC4–PC6  | NIC       | `B:B:B:B::2` – `::4` | LAN B hosts    |

---

## ⚙️ Configuration

### 🔹 Router R1

```bash
Router(config)#hostname R1

R1(config)#interface gigabitEthernet 0/0
R1(config-if)#ipv6 address 2001::1/64
R1(config-if)#no shutdown

R1(config)#interface gigabitEthernet 0/1
R1(config-if)#ipv6 address A:A:A:A::1/64
R1(config-if)#no shutdown

R1(config)#ipv6 unicast-routing

R1(config)#ipv6 router eigrp 1
R1(config-rtr)#no shutdown
R1(config-rtr)#exit

R1(config)#interface gigabitEthernet 0/0
R1(config-if)#ipv6 eigrp 1

R1(config)#interface gigabitEthernet 0/1
R1(config-if)#ipv6 eigrp 1
```

### 🔹 Router R2

```bash
Router(config)#hostname R2

R2(config)#interface gigabitEthernet 0/0
R2(config-if)#ipv6 address 2001::2/64
R2(config-if)#no shutdown

R2(config)#interface gigabitEthernet 0/1
R2(config-if)#ipv6 address B:B:B:B::1/64
R2(config-if)#no shutdown

R2(config)#ipv6 unicast-routing

R2(config)#ipv6 router eigrp 1
R2(config-rtr)#no shutdown
R2(config-rtr)#exit

R2(config)#interface gigabitEthernet 0/0
R2(config-if)#ipv6 eigrp 1

R2(config)#interface gigabitEthernet 0/1
R2(config-if)#ipv6 eigrp 1
```

> ✅ No `network` statements. No wildcard masks. Just `ipv6 eigrp <AS>` directly under the interface. That's it — see **Key Concept** section below for why this matters.

---

## 🧠 Key Concept: IPv4 EIGRP vs IPv6 EIGRP Advertisement — The Real Beauty of EIGRPv6

This is the core learning takeaway of this lab, and honestly the most underrated difference between the two address families.

### In IPv4 EIGRP:
You advertise a **network**, not an interface:

```bash
router eigrp 1
 network 192.168.1.0 0.0.0.255
```

The catch: if that interface's IP address changes later (re-IP, DHCP renumber, subnet redesign, etc.), the old `network` statement no longer matches — routing breaks silently until you go back into the EIGRP process and add/edit the correct `network` line with the new address and wildcard mask. It's an indirect, address-dependent binding.

### In IPv6 EIGRP:
You don't advertise a network at all — you activate EIGRP directly **on the interface**:

```bash
interface gigabitEthernet 0/0
 ipv6 eigrp 1
```

Since the routing protocol is bound to the **interface itself**, not to whatever address happens to sit on it, any IPv6 address configured (or later changed) on that interface is automatically included in EIGRP — no re-touching the routing process required. Change the address, keep the same command, and EIGRPv6 just picks it up.

**That's the elegance of EIGRP for IPv6: interface-based participation instead of address-based matching.** It removes an entire class of "I changed the IP and forgot to update EIGRP" outages that IPv4 networks are prone to.

---

## ✅ Verification & Testing

### 1️⃣ IPv6 Routing Table on R1

```bash
R1#show ipv6 route
D   B:B:B:B::/64 [90/28416]
     via FE80::260:47FF:FE63:1401, GigabitEthernet0/0
C   2001::/64 [0/0]
     via GigabitEthernet0/0, directly connected
C   A:A:A:A::/64 [0/0]
     via GigabitEthernet0/1, directly connected
```
✔️ R1 has learned R2's LAN (`B:B:B:B::/64`) via EIGRP (D), route metric [90/28416].

### 2️⃣ EIGRP Neighbor Adjacency

```bash
R1#show ipv6 eigrp neighbors 1
H   Address                 Interface   Hold  Uptime    SRTT   RTO   Q   Seq
0   Link-local address:     Gig0/0      13    00:03:28  40     1000  0   3
    FE80::260:47FF:FE63:1401
```
✔️ Neighbor adjacency formed over the link-local address — EIGRPv6 always peers via link-local, not global unicast.

### 3️⃣ EIGRP Topology Table

```bash
R1#show ipv6 eigrp topology
P A:A:A:A::/64, 1 successors, FD is 28160
        via Connected, GigabitEthernet0/1
P B:B:B:B::/64, 1 successors, FD is 28416
        via FE80::260:47FF:FE63:1401 (28416/28160), GigabitEthernet0/0
P 2001::/64, 1 successors, FD is 2816
        via Connected, GigabitEthernet0/0
```
✔️ All three networks in **Passive (P)** state — fully converged, no active recomputation.

### 4️⃣ End-to-End Ping Test (from PC1 in LAN A)

```bash
C:\>ping a:a:a:a::1        → Reply, 0% loss  (default gateway)
C:\>ping 2001::1           → Reply, 0% loss  (R1 transit interface)
C:\>ping b:b:b:b::2        → Reply, 0% loss  (host across R2, TTL=126)
```
✔️ Full reachability confirmed across the EIGRP domain — the drop in TTL from 255 → 126 confirms the packet correctly traversed both routers.

### 5️⃣ Switch MAC Address Tables
Both `Switch0` and `Switch1` show dynamically learned MAC entries against the connected router and PC ports — confirming clean Layer 2 forwarding beneath the Layer 3 EIGRP domain.

---

## 🎯 Outcomes

- Established a fully converged **IPv6 EIGRP AS 1** domain between two routers
- Verified **EIGRPv6 neighbor adjacency** over link-local addressing
- Confirmed **route propagation** (D routes) in the IPv6 routing table
- Validated **topology table convergence** (all routes Passive)
- Achieved **100% end-to-end ping success** across both LANs and the transit link
- Demonstrated the **interface-based activation model** of IPv6 EIGRP vs the network-statement model of IPv4 EIGRP

---

## 🛠️ Tools Used

- Cisco Packet Tracer
- Cisco IOS (Router 1941)
- IPv6 EIGRP (EIGRP for IPv6, AS 1)

---

## 👤 Author

**Maaz Khan**
CCNA Certified | Network & NOC Engineer
📇 Cisco ID: CSCO15140445
🔗 [GitHub](https://github.com/maazkhanms) | [LinkedIn](https://www.linkedin.com/in/maazkhanms)

---

⭐ If you found this lab useful, consider giving the repo a star — more Cisco Packet Tracer projects (RIP, OSPF, EIGRP, BGP, VPN, VoIP, IPv6) are documented in my other repos.
