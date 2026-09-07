# Enterprise DHCP Implementation & DORA Analysis — Cisco Packet Tracer Lab

A hands-on Cisco Packet Tracer lab implementing a dedicated DHCP server on a switched LAN, capturing the full DHCP DORA (Discover, Offer, Request, Acknowledge) exchange, breaking it down across the OSI model, and documenting the troubleshooting process from a broken network to a fully working one.

---

## 🖧 Network Topology

```
[ SRV1: DHCP Server ]
   192.168.1.100/24
        |
        | Fa0/1
   +----+----+
   |   SW1   |  (Cisco Catalyst 2960)
   +----+----+
    |        \
Gig0/1        \ Fa0/2
    |           +----------------+
    |                            |
+---+---+                  +-----+-----+
|  R1   |                  |    SW2    |  (Cisco Catalyst 2960)
|Gateway|                  +-----+-----+
+---+---+                        | Fa0/1
                                  |
                            +-----+-----+
                            |    PC1    |  (DHCP Client)
                            |192.168.1.10
                            +-----------+
```

### Addressing Architecture

| Device | Interface | IPv4 Address | Subnet Mask | Default Gateway | Role / Function |
|---|---|---|---|---|---|
| R1 | Gig0/0/0 | 192.168.1.1 | 255.255.255.0 | N/A | Default Gateway / L3 Boundary |
| SRV1 | Fa0 | 192.168.1.100 | 255.255.255.0 | 192.168.1.1 | Dedicated DHCP Server |
| SW1 | Fa0/1–Fa0/24 | Layer 2 Unmanaged | N/A | N/A | Distribution Switch |
| SW2 | Fa0/1–Fa0/24 | Layer 2 Unmanaged | N/A | N/A | Access Switch |
| PC1 | Fa0 | 192.168.1.10 (Dynamic) | 255.255.255.0 | 192.168.1.1 | End-user Client Station |

---

## ⚙️ DHCP Pool Configuration (`serverPool`)

| Parameter | Value |
|---|---|
| Service Status | Enabled |
| Binding Interface | FastEthernet0 |
| Default Gateway | 192.168.1.1 |
| DNS Server | 8.8.8.8 |
| Start IP Address | 192.168.1.10 |
| Subnet Mask | 255.255.255.0 (/24) |
| Max Number of Users | 50 |
| Usable Pool Range | 192.168.1.10 – 192.168.1.59 |
| Static Exclusions | 192.168.1.1–192.168.1.9 (infrastructure), 192.168.1.60–192.168.1.254 (server static subnet) |

---

## 🔬 Protocol Walkthrough: The DORA Exchange

```
PC1 (Client)                                   SRV1 (Server)          R1 (Gateway)
  |                                                 |                      |
  |---- 1. DHCP DISCOVER (Broadcast) -------------->|                      X  [Dropped by R1]
  |     UDP 68 -> 67 | 255.255.255.255              |
  |                                                  |
  |<--- 2. DHCP OFFER (Unicast/Broadcast) -----------|
  |     Offered IP: 192.168.1.10                     |
  |                                                  |
  |---- 3. DHCP REQUEST (Broadcast) ---------------->|
  |     Formal lease request                         |
  |                                                  |
  |<--- 4. DHCP ACK (Unicast/Broadcast) -------------|
  |     Lease committed, timers active               |
```

### OSI Layer Breakdown — DHCP Discover Packet

| OSI Layer | Field / Value | Function |
|---|---|---|
| L7 – Application | BOOTP/DHCP Discover, opcode `0x01`, Transaction ID (XID) | Formats the DHCP request |
| L6 – Presentation | Native data | No encryption/formatting applied |
| L5 – Session | Socket mapping | Tracks the active session state |
| L4 – Transport | UDP Src: 68, Dst: 67 | Connectionless transport; client listens on 68, server on 67 |
| L3 – Network | Src: 0.0.0.0, Dst: 255.255.255.255 | Source unassigned; destination is limited broadcast |
| L2 – Data Link | Src: 00E0.F9CD.8C9C, Dst: FFFF.FFFF.FFFF | Ethernet II frame, broadcast to all NICs on the segment |
| L1 – Physical | FastEthernet0 | Encoded as electrical signal over copper UTP |

---

## 🛠 Troubleshooting Log

### Incident 1 — STP Convergence Delay Blocking DHCP
- **Symptom:** `ipconfig /renew` on PC1 immediately returned `DHCP request failed`.
- **Root cause:** The switches run Spanning Tree Protocol (802.1D PVST+). On link-up, ports pass through Listening (15s) and Learning (15s) before Forwarding. The SW1–SW2 trunk was still in a blocking/transitioning state, so DHCP broadcast frames were being dropped.
- **Fix:** Let STP converge to the Forwarding state (verified by the port LED turning solid green), restoring Layer 2 connectivity between PC1 and SRV1.

### Incident 2 — DHCP Service Disabled on the Server
- **Symptom:** Link lights were all green, but the client still couldn't get a lease.
- **Root cause:** SRV1's static IP (192.168.1.100) was correctly configured, but the DHCP service itself was toggled **Off**.
- **Fix:** Enabled the DHCP service, confirmed it was bound to `FastEthernet0`, and saved the configuration. `ipconfig /renew` succeeded immediately after.

---

## 📊 Verification

**Lease acquisition:**
```
C:\> ipconfig /renew

IP Address.......: 192.168.1.10
Subnet Mask......: 255.255.255.0
Default Gateway..: 192.168.1.1
DNS Server.......: 8.8.8.8
```

**Reachability tests (0% packet loss):**
```
C:\> ping 192.168.1.1      → 4/4 received, avg 0ms
C:\> ping 192.168.1.100    → 4/4 received, avg <1ms
```

---

## 📁 Repository Structure

```
enterprise-dhcp-packet-tracer/
├── README.md                      # This documentation
├── docs/
│   ├── network_topology.png       # Converged physical/logical topology
│   ├── osi_encapsulation.png      # PDU header breakdown, Layers 7–1
│   ├── dora_simulation.png        # Event list showing DHCP broadcast drop at R1
│   └── terminal_verification.png  # ipconfig /renew and ping results
└── packet_tracer/
    └── enterprise_dhcp_lab.pkt    # Saved Packet Tracer source file
```

---

## 🚀 Key Takeaways

- **Broadcast domain boundaries:** Confirmed in simulation that a router terminates broadcast domains — a DHCP broadcast cannot cross a Layer 3 boundary without a relay agent (`ip helper-address`, RFC 1542).
- **Systematic troubleshooting:** Isolated root causes bottom-up — physical/STP state first, then the application-layer DHCP daemon — rather than guessing.
- **Encapsulation in practice:** Traced how the same exchange transforms from a pre-boot broadcast (`0.0.0.0` → `255.255.255.255`) to a committed unicast lease across all four DORA messages.

**Tools used:** Cisco Packet Tracer · DHCP · Spanning Tree Protocol (802.1D) · OSI Model Analysis
