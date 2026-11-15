# OpenFlow SDN Enterprise Router & Stateful Firewall

![Python](https://img.shields.io/badge/Python-2.7-3776AB?style=flat-square&logo=python&logoColor=white)
![OpenFlow](https://img.shields.io/badge/OpenFlow-1.0%20%2F%201.3-008080?style=flat-square)
![SDN](https://img.shields.io/badge/Controller-POX-4B8BBE?style=flat-square)
![Mininet](https://img.shields.io/badge/Emulation-Mininet-FF6C37?style=flat-square)

A Software-Defined Networking (SDN) control-plane implementation built with **POX** and **Mininet**. This project decouples network control logic from underlying OpenFlow virtual switches (`OVSSwitch`) to manage Layer-3 cross-subnet packet forwarding and enforce perimeter firewall policies across a multi-subnet enterprise network topology.

---

## Architecture & Network Topology

The emulated network models a multi-floor enterprise environment featuring segmented floor subnets, a dedicated data center server subnet, an external untrusted host, and a central core switch (`s4`) interconnecting all edge switches.

```text
   [h1: 10.1.1.10/24] ─── (p1) [s1] (p2) ───┐
                                            ├── (p1)
   [h2: 10.2.2.20/24] ─── (p1) [s2] (p2) ───┼── (p2) [s4: Core Switch] (p5) ─── (p2) [s5] (p1) ─── [h5: 10.5.5.50/24]
                                            ├── (p3)         │                                      (Internal Server)
   [h3: 10.3.3.30/24] ─── (p1) [s3] (p2) ───┘               (p4)
                                                             │
                                                   [h4: 123.45.67.89/24]
                                                     (Untrusted Host)
```

### Node & Subnet Specifications

| Device Role | Mininet Node | IPv4 Address / CIDR | MAC Address | Connected Switch (Port) |
| :--- | :--- | :--- | :--- | :--- |
| **Floor 1 Host** | `h1` | `10.1.1.10/24` | `00:00:00:00:00:01` | `s1` (Port 1) |
| **Floor 2 Host** | `h2` | `10.2.2.20/24` | `00:00:00:00:00:02` | `s2` (Port 1) |
| **Floor 3 Host** | `h3` | `10.3.3.30/24` | `00:00:00:00:00:03` | `s3` (Port 1) |
| **Untrusted Host** | `h4` | `123.45.67.89/24` | `00:00:00:00:00:04` | `s4` (Port 4) |
| **Data Center Server** | `h5` | `10.5.5.50/24` | `00:00:00:00:00:05` | `s5` (Port 1) |

---

## Core Engineering Features

* **Deterministic Layer-3 Packet Forwarding:** Eliminates broadcast storms on IP traffic by inspecting IPv4 headers (`srcip`, `dstip`) and switch Datapath IDs (`dpid`) upon `PacketIn` events, routing packets out explicit hardware ports rather than relying on `OFPP_FLOOD`.
* **Proactive Flow Table Caching (`ofp_flow_mod`):** Dynamically installs exact-match OpenFlow rules into switch flow tables with 30-second idle and hard timeouts, offloading subsequent packet forwarding from the control plane to the data plane for line-rate throughput.
* **Zero-Trust Perimeter Firewall:**
  * **Data Center Isolation:** Drops all IPv4 traffic originating from the external untrusted host (`123.45.67.89`) destined for the internal server (`10.5.5.50`) by installing empty-action flow entries at the core switch.
  * **ICMP Reconnaissance Prevention:** Inspects protocol headers to filter and drop all inbound ICMP traffic from the untrusted host toward any internal enterprise subnet (`10.1.1.0/24`, `10.2.2.0/24`, `10.3.3.0/24`, and `10.5.5.0/24`), preventing ping sweeps while preserving authorized TCP/UDP communication with floor hosts.
* **Non-IP Discovery Handling:** Permits Layer-2 address resolution (ARP) across subnets via controlled flooding (`OFPP_FLOOD`) while strictly governing Layer-3+ traffic.

---

## Repository Structure

```text
├── project2.py             # Mininet script defining the 5-switch, 5-host topology and port mappings
├── project2controller.py   # POX OpenFlow controller implementing L3 routing and firewall policies
└── README.md               # Project documentation
```

---

## Getting Started & Usage

### Prerequisites
* **OS:** Ubuntu Linux (native or VM)
* **Dependencies:** Mininet, Open vSwitch (`openvswitch-switch`), Python 2.7, and POX (`dart` branch)

### 1. Environment Setup
Clone the POX controller repository into your home directory and switch to the `dart` release branch:
```bash
cd ~
git clone [https://github.com/noxrepo/pox.git](https://github.com/noxrepo/pox.git)
cd ~/pox && git checkout dart
```

Copy or symlink the controller module into the POX `misc` directory and place the topology script in your home directory:
```bash
cp project2controller.py ~/pox/pox/misc/
cp project2.py ~/
```

### 2. Launch the SDN Controller & Topology
Open two separate terminal windows:

**Terminal 1 — Start the POX Controller:**
```bash
sudo mn -c
sudo python2 ~/pox/pox.py misc.project2controller
```

**Terminal 2 — Build and Start the Mininet Topology:**
```bash
sudo python2 ~/project
