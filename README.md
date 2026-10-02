# EIGRP Lab — Enhanced Interior Gateway Routing Protocol

A hands-on Cisco Packet Tracer lab demonstrating **EIGRP** configuration, neighbor adjacency, DUAL algorithm, Feasible Successor, and fast failover.

---

## 📌 Overview

This lab builds a 4-router topology running **EIGRP AS 100** and demonstrates:

- EIGRP neighbor discovery (Hello packets)
- Successor vs Feasible Successor
- Feasibility Condition (RD < FD)
- Instant failover using a pre-computed backup route
- Metric calculation (Bandwidth + Delay)

---

## 🗺️ Topology

```
                    10.0.0.0/30            10.0.1.0/30
             (10 Mbps)                      (10 Mbps)
        R1 -------------------- R2 -------------------- R4
         |                                              |
         | 10.0.2.0/30                    10.0.4.0/30  |
         | (1 Mbps)                        (10 Mbps)   |
         |                                              |
         R3 -------------------------------------------
                      10.0.4.0/30 (10 Mbps)

     LAN R1: 192.168.1.0/24          LAN R4: 192.168.4.0/24
```

---

## 🧮 IP Addressing Table

| Device | Interface   | IP Address     | Subnet Mask       | Clock Rate |
|--------|-------------|----------------|-------------------|------------|
| R1     | G0/0        | 192.168.1.1    | 255.255.255.0     | —          |
| R1     | S0/0/0      | 10.0.0.1       | 255.255.255.252   | 64000      |
| R1     | S0/0/1      | 10.0.2.1       | 255.255.255.252   | 64000      |
| R2     | S0/0/0      | 10.0.0.2       | 255.255.255.252   | DTE        |
| R2     | S0/0/1      | 10.0.1.1       | 255.255.255.252   | 64000      |
| R3     | S0/0/0      | 10.0.2.2       | 255.255.255.252   | DTE        |
| R3     | S0/0/1      | 10.0.4.1       | 255.255.255.252   | 64000      |
| R4     | G0/0        | 192.168.4.1    | 255.255.255.0     | —          |
| R4     | S0/0/0      | 10.0.1.2       | 255.255.255.252   | DTE        |
| R4     | S0/0/1      | 10.0.4.2       | 255.255.255.252   | DTE        |
| PC1    | Fa0         | 192.168.1.10   | 255.255.255.0     | —          |
| PC2    | Fa0         | 192.168.4.10   | 255.255.255.0     | —          |

> PC1 gateway: `192.168.1.1` | PC2 gateway: `192.168.4.1`

---

## 🧰 Devices Used

| Device | Model         | Quantity |
|--------|---------------|----------|
| Router | Cisco 2911    | 4        |
| Switch | Cisco 2960    | 2        |
| PC     | Generic       | 2        |
| Serial Cable (DCE/DTE) | —  | 4        |
| Straight-through Cable | —  | 4        |

---

## ⚙️ Configuration

### R1

```cisco
enable
configure terminal
hostname R1

interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.0
 no shutdown
 exit

interface Serial0/0/0
 ip address 10.0.0.1 255.255.255.252
 bandwidth 10000
 clock rate 64000
 no shutdown
 exit

interface Serial0/0/1
 ip address 10.0.2.1 255.255.255.252
 bandwidth 1000
 clock rate 64000
 no shutdown
 exit

router eigrp 100
 no auto-summary
 network 10.0.0.0 0.0.0.3
 network 10.0.2.0 0.0.0.3
 network 192.168.1.0 0.0.0.255
 passive-interface GigabitEthernet0/0
 exit
```

### R2

```cisco
enable
configure terminal
hostname R2

interface Serial0/0/0
 ip address 10.0.0.2 255.255.255.252
 bandwidth 10000
 no shutdown
 exit

interface Serial0/0/1
 ip address 10.0.1.1 255.255.255.252
 bandwidth 10000
 clock rate 64000
 no shutdown
 exit

router eigrp 100
 no auto-summary
 network 10.0.0.0 0.0.0.3
 network 10.0.1.0 0.0.0.3
 exit
```

### R3

```cisco
enable
configure terminal
hostname R3

interface Serial0/0/0
 ip address 10.0.2.2 255.255.255.252
 bandwidth 1000
 no shutdown
 exit

interface Serial0/0/1
 ip address 10.0.4.1 255.255.255.252
 bandwidth 10000
 clock rate 64000
 no shutdown
 exit

router eigrp 100
 no auto-summary
 network 10.0.2.0 0.0.0.3
 network 10.0.4.0 0.0.0.3
 exit
```

### R4

```cisco
enable
configure terminal
hostname R4

interface GigabitEthernet0/0
 ip address 192.168.4.1 255.255.255.0
 no shutdown
 exit

interface Serial0/0/0
 ip address 10.0.1.2 255.255.255.252
 bandwidth 10000
 no shutdown
 exit

interface Serial0/0/1
 ip address 10.0.4.2 255.255.255.252
 bandwidth 10000
 no shutdown
 exit

router eigrp 100
 no auto-summary
 network 10.0.1.0 0.0.0.3
 network 10.0.4.0 0.0.0.3
 network 192.168.4.0 0.0.0.255
 passive-interface GigabitEthernet0/0
 exit
```

---

## ✅ Verification

### 1. Neighbor Adjacency

```cisco
show ip eigrp neighbors
```

Expected on R1: 2 neighbors (R2 via Se0/0/0, R3 via Se0/0/1).

### 2. Topology Table (Successor + Feasible Successor)

```cisco
show ip eigrp topology
```

Expected for `192.168.4.0/24` on R1:

```
P 192.168.4.0/24, 1 successors, FD is 1280256
     via 10.0.0.2 (1280256/768256), Serial0/0/0     <-- Successor
     via 10.0.2.2 (3584256/768256), Serial0/0/1     <-- Feasible Successor
```

**Feasibility Condition:** `RD (768256) < FD (1280256)` ✅

### 3. Routing Table

```cisco
show ip route eigrp
```

Expected: only the successor installed:

```
D  192.168.4.0/24 [90/1280256] via 10.0.0.2, Serial0/0/0
```

---

## 🔥 Failover Test (Feasible Successor)

Simulate link failure to R2 on R1:

```cisco
configure terminal
interface Serial0/0/0
shutdown
exit
```

Immediately check routing table:

```cisco
show ip route eigrp
```

Expected result — instant switch to R3:

```
D  192.168.4.0/24 [90/3584256] via 10.0.2.2, Serial0/0/1
```

**No Query, no Active state, sub-second convergence** — because R3 was a Feasible Successor.

Restore the link:

```cisco
configure terminal
interface Serial0/0/0
no shutdown
exit
```

R1 re-elects R2 as successor within ~15 seconds.

---

## 🧠 Key Concepts Learned

| Concept | Description |
|---------|-------------|
| **EIGRP** | Enhanced Interior Gateway Routing Protocol |
| **DUAL** | Diffusing Update Algorithm — loop-free path selection |
| **Successor** | Best path, installed in routing table |
| **Feasible Successor** | Backup path that passes Feasibility Condition |
| **FD** | Feasible Distance — total metric to destination |
| **RD** | Reported Distance — neighbor's metric to destination |
| **FC** | Feasibility Condition: RD < FD |
| **Metric** | Bandwidth + Delay (default K-values) |
| **Multicast** | 224.0.0.10 |
| **IP Protocol** | 88 (not TCP/UDP) |
| **AD** | 90 internal, 170 external |

---

## 📊 Show Commands Cheat Sheet

| Command | Purpose |
|---------|---------|
| `show ip eigrp neighbors` | Verify adjacencies |
| `show ip eigrp topology` | Successors and Feasible Successors |
| `show ip eigrp topology all-links` | Show all routes including non-FS |
| `show ip eigrp interfaces` | EIGRP-enabled interfaces |
| `show ip eigrp traffic` | EIGRP packet counters |
| `show ip route eigrp` | EIGRP routes in RIB |
| `show ip protocols` | EIGRP config summary |
| `show controllers serial 0/0/0` | Check DCE/DTE status |
| `debug eigrp packets` | Real-time packet trace (use carefully) |

---

## 🛠️ Troubleshooting

| Problem | Likely Cause | Fix |
|---------|--------------|-----|
| No neighbors | AS number mismatch | Ensure `router eigrp 100` on all |
| No neighbors | Subnet mismatch | Verify IP/mask on link |
| Interface down/down | Missing `no shutdown` | Enter interface, type `no shutdown` |
| Interface down/down | No clock rate on DCE | Add `clock rate 64000` on DCE side |
| `clock rate` rejected | Interface is DTE | Configure clock on the other side |
| Only 1 path | No Feasible Successor | Check `show ip eigrp topology all-links` |
| Slow convergence | No FS → diffusing | Add redundant path with lower RD |

---

## 🎯 Learning Outcomes

After completing this lab, you should be able to:

- [x] Configure EIGRP on multiple routers
- [x] Interpret EIGRP neighbor, topology, and routing tables
- [x] Identify Successor and Feasible Successor
- [x] Verify Feasibility Condition (RD < FD)
- [x] Demonstrate sub-second failover using DUAL
- [x] Understand EIGRP composite metric (Bandwidth + Delay)
- [x] Troubleshoot DCE/DTE serial link issues

---

## 📁 Repository Structure

```
eigrp-lab/
├── README.md
├── EIGRP-Lab.pkt              # Packet Tracer file
├── topology.png               # Network diagram
└── screenshots/
    ├── neighbors.png
    ├── topology-table.png
    ├── routing-table.png
    └── failover-test.png
```

---

## 📚 References

- [RFC 7868 — Cisco's EIGRP](https://datatracker.ietf.org/doc/html/rfc7868)
- Cisco Official Documentation — EIGRP
- Packet Tracer Version 8.x

---

## 👤 Author

**Your Name**
- GitHub: [@yourusername](https://github.com/yourusername)
- LinkedIn: [Your Profile](https://linkedin.com/in/yourprofile)

---

## 📜 License

This project is open-source and available under the [MIT License](LICENSE).

---

## ⭐ Support

If this lab helped you, please give the repo a ⭐ and share it with others learning networking!
