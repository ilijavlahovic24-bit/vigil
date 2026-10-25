# Vigil

> A high-performance Network Intrusion Detection & Response System (NIDS/SOC-in-a-box) written in Rust.

![Build](https://img.shields.io/badge/build-passing-brightgreen)
![License](https://img.shields.io/badge/license-MIT-blue)
![Language](https://img.shields.io/badge/language-Rust-orange)
![Status](https://img.shields.io/badge/status-planned-lightgrey)

Vigil is a full-stack network security monitoring platform built from scratch in Rust. It captures and decodes network traffic, detects intrusions via signature-based and anomaly-based engines, correlates security events across flows, and triggers automated responses — all within a single, self-contained system.

---

## Features

- **Packet Capture** — pcap replay (CIC-IDS2017, CTU-13) and live capture via AF_PACKET
- **Zero-Copy Protocol Decoding** — L2–L4 parser stack (Ethernet, IPv4/IPv6, TCP, UDP, ICMP)
- **Stateful Flow Tracking** — per-flow state machine with TCP reassembly and timeout eviction
- **Signature-Based Detection** — Aho-Corasick multi-pattern matching engine with a custom rule DSL
- **Anomaly-Based Detection** — statistical profiling of flow features (packet rate, byte distribution, IAT)
- **Event Correlation** — temporal correlation of alerts across flows with rule-driven escalation
- **Automated Response** — nftables integration for dynamic IP/port blocking, webhook alerting
- **Dashboard** — real-time terminal UI (TUI) showing active flows, alert feed, and detection metrics

---

## Architecture

Vigil is structured as a multi-stage pipeline. Traffic enters through the capture layer and flows through decoding, flow tracking, and parallel detection engines before reaching the response and correlation layers.

```
┌─────────────┐     ┌─────────────┐     ┌──────────────────┐
│   Capture   │────▶│   Decode    │────▶│  Flow Tracker    │
│ pcap/AF_PKT │     │  L2–L4      │     │ (sharded table,  │
│             │     │  zero-copy  │     │  TCP reassembly) │
└─────────────┘     └─────────────┘     └────────┬─────────┘
                                                  │
                              ┌───────────────────┼───────────────────┐
                              ▼                   ▼                   ▼
                     ┌───────────────┐   ┌────────────────┐   ┌──────────────┐
                     │  Signature    │   │    Anomaly     │   │  (future)    │
                     │  Engine       │   │    Engine      │   │  ML Engine   │
                     │ Aho-Corasick  │   │  Statistical   │   │              │
                     └───────┬───────┘   └───────┬────────┘   └──────┬───────┘
                             │                   │                   │
                             └───────────────────▼───────────────────┘
                                        ┌─────────────────┐
                                        │   Correlator    │
                                        │ (event graph,   │
                                        │  rule engine)   │
                                        └────────┬────────┘
                                                 │
                              ┌──────────────────┼──────────────────┐
                              ▼                  ▼                  ▼
                     ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
                     │   Responder  │   │    Logger    │   │  Dashboard   │
                     │  nftables /  │   │  JSON/NDJSON │   │     TUI      │
                     │  webhooks    │   │              │   │   (ratatui)  │
                     └──────────────┘   └──────────────┘   └──────────────┘
```

### Workspace Layout

```
vigil/
├── capture/        # Packet sources: pcap replay, AF_PACKET live capture
├── decode/         # Zero-copy L2–L4 protocol parsers
├── flow/           # Flow table, TCP reassembly, timeout eviction
├── detect/
│   ├── signature/  # Rule DSL, Aho-Corasick engine
│   └── anomaly/    # Statistical flow profiling
├── correlate/      # Alert correlation, event graph, escalation rules
├── respond/        # nftables control, webhook/alerting integrations
├── dashboard/      # ratatui TUI
└── cli/            # Entry point, config, CLI flags
```

---

## Detection Engines

### Signature Engine
Rules are defined in a lightweight DSL inspired by Snort/Suricata syntax, compiled at startup into an Aho-Corasick automaton for O(n) multi-pattern matching over reassembled payload bytes.

```
alert tcp any any -> any 80 (msg:"SQL Injection attempt"; content:"' OR '1'='1"; sid:1001;)
alert tcp any any -> any 22 (msg:"SSH brute force"; threshold: count 5, seconds 10; sid:1002;)
```

### Anomaly Engine
Per-flow features (packet rate, byte volume, inter-arrival time distribution, flag ratios) are compared against learned baseline profiles. Deviations beyond configurable thresholds generate anomaly events.

---

## Datasets

Vigil is evaluated against standard labeled datasets:

| Dataset | Traffic Type | Attacks |
|---|---|---|
| CIC-IDS2017 | Mixed | DoS, PortScan, Brute Force, Web Attacks |
| CTU-13 | Botnet | C&C, DDoS, Spam |
| MAWI | Backbone | Anomalies, Scans |

---

## Performance

Target throughput on commodity hardware (pcap replay mode):

| Mode | Target |
|---|---|
| Pcap replay | ≥ 1M packets/sec |
| Live capture (AF_PACKET) | ≥ 500K packets/sec |
| Detection latency (p99) | < 1ms per packet |

---

## Getting Started

### Prerequisites

- Rust 1.78+ (stable)
- libpcap (`sudo apt install libpcap-dev`)
- Linux (AF_PACKET and nftables response require Linux kernel ≥ 5.10)

### Build

```bash
git clone https://github.com/ilijavlahovic24-bit/vigil
cd vigil
cargo build --release
```

### Run (pcap replay)

```bash
cargo run --release -- \
  --mode replay \
  --input data/CIC-IDS2017/Monday.pcap \
  --rules rules/default.vrl
```

### Run (live capture)

```bash
sudo cargo run --release -- \
  --mode live \
  --interface eth0 \
  --rules rules/default.vrl
```

---

## Roadmap

- [x] Project design & architecture
- [ ] `capture`: pcap replay source
- [ ] `decode`: L2–L4 zero-copy parsers
- [ ] `flow`: sharded flow table + TCP reassembly
- [ ] `detect/signature`: rule DSL + Aho-Corasick engine
- [ ] `detect/anomaly`: statistical profiling
- [ ] `correlate`: event correlation engine
- [ ] `respond`: nftables + webhook integration
- [ ] `dashboard`: ratatui TUI
- [ ] Live capture (AF_PACKET)
- [ ] Evaluation on CIC-IDS2017 / CTU-13
- [ ] AF_XDP/eBPF fast path (stretch goal)
- [ ] ML-based detection engine (stretch goal)

---

## License

MIT

---

## Author

Ilija Vlahović — [GitHub](https://github.com/ilijavlahovic24-bit) · [LinkedIn](https://linkedin.com/in/ilija-vlahovic-ba9946201)
