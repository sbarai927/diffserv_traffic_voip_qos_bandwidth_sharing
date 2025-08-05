# # Exploring Traffic Management in DiffServ Domains
> QoS Scheduling · Bandwidth Sharing · Performance Metrics

## Table of Contents
- [Objective](#objective)
- [Quick Start](#quick-start)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Network Topology](#network-topology)
- [Setup Guide: Reproducing the Experiment](#setup-guide-reproducing-the-experiment)
  - [Environment](#environment)
  - [Device Configuration Import](#device-configuration-import)
  - [Asterisk & SIP User Setup](#asterisk--sip-user-setup)
  - [Traffic Generators](#traffic-generators)
  - [Enabling / Disabling QoS Policies](#enabling--disabling-qos-policies)
- [Running the Scenarios](#running-the-scenarios)
  - [Baseline VoIP Call](#baseline-voip-call)
  - [VoIP + iPerf (Without QoS)](#voip--iperf-without-qos)
  - [VoIP + iPerf (With QoS)](#voip--iperf-with-qos)
- [Data Collection & Analysis](#data-collection--analysis)
  - [Capturing PCAPs](#capturing-pcaps)
  - [Wireshark Graphing](#wireshark-graphing)
- [Results Snapshot](#results-snapshot)
- [Key Technologies & Protocols](#key-technologies--protocols)
- [Troubleshooting Tips](#troubleshooting-tips)

## Objective

This repository demonstrates, in a reproducible lab environment, how **Differentiated Services (DiffServ)** Quality of Service (QoS) techniques can protect delay-sensitive VoIP traffic from bandwidth-hungry data flows.

### Goals
- **Implement** a two-router DiffServ domain where a G.711/G.726 VoIP call (DSCP EF) traverses the same constrained serial link as a bulk **iPerf** data transfer (DSCP 0).  
- **Configure** Cisco MQC class-maps and policy-maps to classify, mark, and schedule traffic—giving voice strict-priority service while treating data as best-effort.  
- **Measure** key performance metrics—throughput, one-way delay, jitter, and packet loss—for both voice and data **with and without** QoS enabled.  
- **Visualize** the impact using annotated Wireshark I/O graphs and screenshots that compare scenarios.  
- **Provide** all router/SIP configs, PCAP captures, and step-by-step instructions so others can clone the repo, load the configs, and reproduce the exact results or extend the experiment (e.g., different codecs, link speeds, or scheduling policies).

This lab was carried out as a ULP completion requirement for the course **Advanced Multimedia Communication** (WS 2024) taught by **Prof. Dr. Andreas Grebe** at **Technische Hochschule Köln**.

## Quick Start

1. **Clone & enter project**  
   ```bash
   git clone https://github.com/<your-user>/diffserv_traffic_voip_qos_bandwidth_sharing.git
   cd diffserv_traffic_voip_qos_bandwidth_sharing
   ```
2. Launch topology & load configs
   - Open topology/diffserv-qos.gns3 or build the 2-router/2-switch diagram in images/network_topology.png
   - Copy configs from configs/network to R1, R2, S1, S2.
3. Start Asterisk PBX
   ```bash
   cd asterisk && docker compose up -d   # extensions 6001 / 6002 ready
   ```
4. Register soft-phones
   - PC-A → 6001@10.6.0.10 (pwd 6001)

   - PC-C → 6002@10.6.0.10 (pwd 6002)
  
5. Run scenarios
   | Step        | Command                                                           | Expectation                  |
| ----------- | ----------------------------------------------------------------- | ---------------------------- |
| Baseline    | Call 6001 → 6002                                                  | Steady \~64 kbps RTP         |
| No-QoS load | PC-C: `iperf -s` <br> PC-A: `iperf -c 10.6.2.2 -t 30`             | Voice breaks up              |
| QoS enabled | On both routers’ serial int: `service-policy output DIFFSERV-QOS` | Voice clear, iPerf throttled |

## Prerequisites

| Component | Tested Version | Purpose |
|-----------|----------------|---------|
| **GNS3** (or real Cisco routers) | ≥ 2.2 (PC) | Emulates the 2-router / 2-switch topology. |
| **Cisco IOS** image | 15.4(2)T (C2900) | R1 & R2 DiffServ configuration. |
| **Asterisk PBX** | 16 LTS or 18 | Registers SIP users 6001 / 6002. |
| **Docker + Docker Compose** (optional) | latest | Quick Asterisk deployment (`asterisk/docker-compose.yml`). |
| **Softphone** (Linphone, Zoiper, etc.) | any recent | Place the G.711/G.726 calls. |
| **iPerf** | 2.1.8 | Generates competing bulk data traffic. |
| **Wireshark** | ≥ 3.6 | Captures & graphs RTP / iPerf flows. |

## Repository Structure
```bash
.
├── configs/ # All configuration files
│ ├── extensions.conf # Asterisk dial-plan (optional)
│ ├── sip.conf # SIP user definitions (6001 / 6002)
│ └── network/ # Device configs (Cisco IOS & switches)
│ ├── config_r1.txt
│ ├── config_r2.txt
│ ├── config_s1.txt
│ └── config_s2.txt
├── docs/ # Course reports & lab task write-ups
│ ├── Lab_Task_1.pdf
│ ├── Lab_Task_2.pdf
│ ├── Lab_Task_3.pdf
│ └── report.pdf # Final report (complete analysis)
├── graphs/ # PDF throughput/jitter graphs exported from Wireshark
│ ├── capture_iPerf_traffic.pdf
│ ├── capture_voice_call_G726-32.pdf
│ ├── capture_voice_call_PCMA.pdf
│ ├── capture_voice_call_PCMA_with_iPerf_traffic_1mbps.pdf
│ ├── capture_voice_call_PCMU.pdf
│ ├── iperf_vs_voip_PCMA.pdf
│ └── iperf_vs_voip_PCMA_with_QoS.pdf
├── images/ # PNG screenshots & diagrams referenced in README/docs
│ ├── network_topology.png
│ ├── dce_r1.png / dte_r2.png # Serial controller details
│ ├── i_perf_receiver_test.png
│ ├── ping_* # Latency snapshots
│ └── phones_registered_in_asterisk.png
├── wireshark_captures/ # Raw .pcapng traces for every scenario
│ ├── capture_iPerf_traffic.pcapng
│ ├── capture_voice_call_G726-32.pcapng
│ ├── capture_voice_call_PCMA.pcapng
│ ├── capture_voice_call_PCMA_with_iPerf_traffic_1mbps.pcapng
│ ├── capture_voice_call_PCMU.pcapng
│ ├── capture_voip_vs_iperf.pcapng
│ └── capture_voip_vs_iperf_with_QoS.pcapng
└── README.md # Project overview & reproduction guide
```

## Network Topology

[Topology Diagram](images/network_topology.png)

| Segment | Devices | IP/Subnet | Notes |
|---------|---------|-----------|-------|
| **LAN-A** | Asterisk PBX, SIP phone 6001, iPerf client | 10.6.0.0/24 | Connected to R1 Gi0/1 via S1 |
| **Serial Link** | R1 S1/0 ↔ R2 S1/0 | 10.6.1.0/24 | 125 kbps, DiffServ QoS applied |
| **LAN-C** | SIP phone 6002, iPerf server | 10.6.2.0/24 | Connected to R2 Gi0/0 via S2 |

> Voice (RTP/SIP) and data (iPerf) share the constrained serial link.  
> DiffServ policies on R1/R2 mark RTP with **DSCP EF** and place it in a priority queue; all other traffic remains best-effort.

## Setup Guide: Reproducing the Experiment

### Environment
- **Emulator / HW:** GNS3 ≥ 2.2 —or two Cisco ISR routers & two L2 switches  
- **Router IOS:** 15.4(2)T (tested image: c2900-universalk9-mz.SPA.154-2.T.bin)  
- **Host OS:** Ubuntu 20.04 (Windows / macOS work with GNS3 + Docker)  
- **Aux tools:** Docker & Compose (for Asterisk), Wireshark ≥ 3.6, iPerf 2.1.8, any SIP soft-phone (Linphone, Zoiper)

---

### Device Configuration Import
```text
# R1
copy tftp://<host-ip>/configs/network/config_r1.txt running-config
write memory
# R2
copy tftp://<host-ip>/configs/network/config_r2.txt running-config
write memory
# Switches S1 / S2 (optional):
copy tftp://<host-ip>/configs/network/config_s1.txt running-config
```

### Asterisk & SIP User Setup
```text
cd asterisk
docker compose up -d        # spins up PBX with sip.conf already mounted
```

| Ext  | User-Pass | Registrar |
| ---- | --------- | --------- |
| 6001 | 6001      | 10.6.0.10 |
| 6002 | 6002      | 10.6.0.10 |

Register your soft-phones with the above credentials, codec preference G.711 A-law (PCMA).

### Traffic Generators

- Baseline call: Dial 6002 from phone 6001; keep the call active.
- Competing flow:
  
```bash
on LAN-C PC
iperf -s
on LAN-A PC
iperf -c 10.6.2.2 -t 30          # TCP default, saturates link
  ```
- To test UDP: ```bash iperf -c 10.6.2.2 -u -b 1M -t 30 ```

### Enabling / Disabling QoS Policies
```bash
! enable DiffServ priority for voice (default state in repo)
interface s1/0
 service-policy output DIFFSERV-QOS

! disable for comparison
interface s1/0
 no service-policy output DIFFSERV-QOS
```
Capture traffic via SPAN on S2 or router monitor, then view PCAPs with Wireshark filters:
```bash
ip.dsfield.dscp == 46   # EF-marked RTP (voice)
tcp.port == 5001        # iPerf stream
```
## Running the Scenarios

### Baseline VoIP Call
1. Place a call **6001 → 6002** and keep it active.  
2. Observe with Wireshark: RTP stream (~64 kbps, DSCP EF) should flow uninterrupted.  
3. Ping across the serial link (e.g., `ping 10.6.2.2`) → latency ≈ 14 ms, 0 % loss.  

---

### VoIP + iPerf (Without QoS)
```bash
# on LAN-C PC
iperf -s
# on LAN-A PC (while call is up)
iperf -c 10.6.2.2 -t 30
```
- Disable QoS first: no service-policy output DIFFSERV-QOS on both routers.

- Expectation: iPerf saturates the 125 kbps link; RTP packets drop, voice quality degrades (see graphs/iperf_vs_voip_PCMA.pdf).

### VoIP + iPerf (With QoS)

```bash
interface s1/0
 service-policy output DIFFSERV-QOS   ! re-enable priority queue
```
- Repeat the iPerf commands during the ongoing call.

- Expectation: RTP remains ~64 kbps with minimal loss/jitter; iPerf throughput is throttled (see graphs/iperf_vs_voip_PCMA_with_QoS.pdf).

## Data Collection & Analysis

### Capturing PCAPs
1. **Enable SPAN** on S2 to mirror traffic from the router-facing port to the capture PC.  
   ```text
   monitor session 1 source interface fa0/1
   monitor session 1 destination interface fa0/24
    ```
2. Start Wireshark on the capture PC and save each scenario trace to wireshark_captures/ using descriptive names (capture_voip_vs_iperf_with_QoS.pcapng, etc.).
3. Note the start/stop times so you can align graphs with the call and iPerf intervals.

### Wireshark Graphing

1. Filter by DSCP or Port

  - Voice: ip.dsfield.dscp == 46 or udp.port >= 10000 && udp.port <= 20000

  - iPerf: tcp.port == 5001 (or udp.port == 5001 if using UDP)

2. I/O Graphs

  - Navigate Statistics → I/O Graph → add two plots: one for voice filter, one for iPerf filter.

  - Set Y-Axis to Bits/Tick, tick interval = 1 s.

3. Export as PDF/PNG (File → Export) and store in graphs/.

4. Compare the throughput curves: without QoS voice collapses; with QoS voice remains constant while iPerf is capped.

## Results Snapshot

| Scenario | Voice Throughput | Voice Packet Loss | iPerf Throughput | Outcome |
|----------|-----------------|-------------------|------------------|---------|
| **Baseline Call** | ~64 kbps | 0 % | N/A | Clean audio |
| **Call + iPerf (No QoS)** | ≈ 0–10 kbps (starved) | 20 – 30 % | ~110 kbps (fills link) | Choppy / unusable voice |
| **Call + iPerf (With QoS)** | ~64 kbps (stable) | < 1 % | ~40 kbps (policed) | Clear voice, data constrained |

*(Exact numbers taken from `graphs/iperf_vs_voip_PCMA.pdf` and `graphs/iperf_vs_voip_PCMA_with_QoS.pdf`.)*

---

## Key Technologies & Protocols
- **DiffServ / DSCP (RFC 2474)** – packet marking for class-based QoS  
- **Cisco MQC** – class-map / policy-map configuration for priority queuing  
- **RTP & SIP (VoIP)** – G.711 A-law (PCMA) and G.726-32 codecs  
- **iPerf 2** – generates bulk TCP/UDP traffic to stress the link  
- **Wireshark** – capture, filter (`ip.dsfield.dscp==46`), and graph traffic flows  

---

## Troubleshooting Tips
- **Soft-phone won’t register:** check Asterisk is running (`docker ps`) and that `sip.conf` IP/port matches phone settings.  
- **No DSCP EF marking visible:** verify `class-map VOICE` and `service-policy DIFFSERV-QOS` are applied on both router serial interfaces.  
- **iPerf saturates even with QoS:** ensure the priority queue’s bandwidth isn’t accidentally set to 100 % (`police 64000 conform-action transmit`).  
- **High latency on serial link:** confirm clock rate is 2000000 on the DCE side (R1) and both ends show `show controller serial` OK.  
- **Capture empty / missing flows:** validate SPAN source/destination on S2; mirror the correct port and disable switchport filtering.
