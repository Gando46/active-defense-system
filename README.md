# 🛡️ Active Defense System (ADS)
### Automated Intrusion Prevention System

![Python](https://img.shields.io/badge/Python-3.10+-3572A5?style=shields.io&logoColor=white&logo=python)
![Suricata](https://img.shields.io/badge/Suricata-8.0+-FF6600?style=shields.io&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-Ubuntu%2022.04+-FCC624?style=shields.io&logoColor=white&logo=linux)
![SQLite](https://img.shields.io/badge/SQLite-3-003087?style=shields.io&logoColor=white)


> An automated **Intrusion Prevention System (IPS)** that combines real-time network threat detection with automatic firewall response and forensic logging. Built as a lightweight SIEM/SOAR pipeline: **Suricata → Python → iptables + SQLite**.

---

## 📸 Demo

![Active Blocking](docs/screenshots/ips-blocking-demo.jpg)
*Figure 1: Orchestrator detecting a ping flood and executing an automated Iptables DROP rule. Note the "Request timeout" on the attacker's terminal.*

### 2. Deep Packet Inspection & Log Correlation
![Log Analysis](docs/screenshots/suricata-logs-analysis.jpg)
*Figure 2: Analysis of Suricata JSON alerts. The system identifies an SSH Brute Force signature and extracts the source IP for blocking.*

### 3. Service Stability
![Service Status](docs/screenshots/service-status.jpg)
*Figure 3: Verification of the Suricata engine running as a system service, ensuring high availability of the detection layer.*

### 4. Incident Database (SQL)
![SQL Audit](docs/screenshots/database-integration.jpg)
*Figure 4: Persistent logging of blocked incidents in SQLite, facilitating post-mortem analysis and security reporting.*

---

## 🏗️ Architecture

```
┌──────────────┐     Network      ┌─────────────────────┐
│              │  ─────────────►  │                     │
│   Attacker   │  ICMP / TCP      │   Linux Server      │
│  (Mac Host)  │  Packets         │   (Ubuntu 22.04)    │
│              │                  │                     │
└──────────────┘                  │  ┌─────────────┐    │
                                  │  │  enp0s1     │    │
                                  │  │  (NIC)      │    │
                                  │  └──────┬──────┘    │
                                  │         │ raw pkts  │
                                  │         ▼           │
                                  │  ┌─────────────┐    │
                                  │  │  Suricata   │    │
                                  │  │  IDS Engine │    │
                                  │  │  (rules)    │    │
                                  │  └──────┬──────┘    │
                                  │         │           │
                                  │         │ alert     │
                                  │         ▼ (JSON)    │
                                  │  ┌─────────────┐    │
                                  │  │  eve.json   │    │
                                  │  └──────┬──────┘    │
                                  │         │           │
                                  │         │ tail -f   │
                                  │         ▼           │
                                  │  ┌─────────────┐    │
                                  │  │  Python     │    │
                                  │  │  SOAR       │    │
                                  │  │  (ADS)      │    │
                                  │  └──────┬──────┘    │
                                  │         │           │
                                  │    ┌────┴────┐      │
                                  │    ▼         ▼      │
                                  │ ┌──────┐ ┌──────┐   │
                                  │ │iptab-│ │SQLite│   │
                                  │ │les   │ │  DB  │   │
                                  │ │DROP  │ │logs  │   │
                                  │ └──────┘ └──────┘   │
                                  └─────────────────────┘
```

### Data Flow

1. **Attacker** sends malicious traffic (Ping Flood, Port Scan) toward the server.
2. **Suricata** captures packets on `enp0s1`, matches them against custom rules in `local.rules`.
3. A matching rule generates an **alert** written to `/var/log/suricata/eve.json` in real time.
4. **active_defense.py** continuously tails `eve.json`, parses each JSON line, and checks if the alert's Signature ID (SID) is in the critical list.
5. If critical: the script calls **iptables** to DROP all inbound traffic from the attacker's IP, and inserts an incident record into **SQLite** with timestamp, IP, signature, and action taken.

---

## 🛠️ Tech Stack

| Tool | Role |
|------|------|
| **Suricata 8.0** | Network IDS — captures packets, matches signatures, writes alerts |
| **Python 3.10+** | SOAR orchestrator — reads logs, parses JSON, orchestrates response |
| **iptables** | Linux firewall — enforces DROP rules to block attacker IPs |
| **SQLite3** | Forensic database — persists every incident for auditing |
| **EVE JSON** | Structured logging format — Suricata's native alert output |
| **Ubuntu 22.04** | Host OS (ARM64, running in UTM on Apple Silicon) |

---

## 📁 Project Structure

```
active-defense-system/
│
├── active_defense.py       # Main SOAR script (monitor → parse → block → log)
├── install.sh              # Automated environment setup
├── cleanup.sh              # Resets firewall and temp files
│
├── config/
│   └── local.rules         # Custom Suricata detection rules (4 signatures)
│
├── docs/
│   └── screenshots/        # Demo screenshots for README
│
├── .gitignore              # Excludes .db, .log, blocked_ips.txt
└── README.md               # This file
```

---

## 🚀 Quick Start

### Prerequisites

- Ubuntu Server 22.04+ (tested on ARM64 via UTM)
- Python 3.10+
- Root / sudo access

### 1. Install dependencies

```bash
sudo bash install.sh
```

### 2. Configure your network interface

```bash
# Find your active interface
ip a
# Example output: enp0s1 → inet 192.168.64.7/24

# Edit Suricata config
sudo nano /etc/suricata/suricata.yaml

# Set these two values:
#   af-packet:
#     - interface: enp0s1          ← your interface
#   vars:
#     address-groups:
#       HOME_NET: "[192.168.64.0/24]"  ← your subnet
```

### 3. Restart Suricata

```bash
sudo systemctl restart suricata
sudo systemctl status suricata
```

### 4. Run the Active Defense System

```bash
python3 active_defense.py
```

### 5. Trigger an alert (from another machine on the same network)

```bash
# Simple ping (triggers SID 1000003)
ping -c 5 <SERVER_IP>

# Ping Flood (triggers SID 1000001)
ping -i 0.05 -c 50 <SERVER_IP>
```

### 6. Verify

```bash
# See blocked IPs
sudo iptables -L INPUT -v -n --line-numbers

# Query incident database
sqlite3 defense_log.db "SELECT * FROM incidents;"
```

---

## 📋 Detection Rules

Defined in `config/local.rules` and deployed to `/var/lib/suricata/rules/local.rules`:

| SID | Signature | Trigger Condition | Severity |
|-----|-----------|-------------------|----------|
| 1000001 | ICMP Ping Flood | >10 ICMP echo requests / second from same source | Critical |
| 1000002 | TCP Port Scan | >20 SYN packets in 5 seconds from same source | High |
| 1000003 | ICMP Ping (Test) | Any single ICMP echo request | Medium |
| 1000004 | SSH Brute Force | >5 TCP connections to port 22 in 60 seconds | Critical |

---

## 🔍 Key Design Decisions

**Why `tail -f` in Python instead of a syslog listener?**
Simplicity and portability. `eve.json` is a line-delimited JSON file. A custom `tail_follow()` generator reads new lines with minimal CPU overhead and zero external dependencies.

**Why iptables instead of nftables?**
`iptables` is still the most widely documented firewall tool on Ubuntu and the expected answer in most cybersecurity curricula. The `subprocess` call is identical in structure to what a SOC analyst would run manually — making the automation layer transparent and auditable.

**Why SQLite instead of a remote DB?**
This is a single-node defensive system. SQLite gives us zero-config persistence, ACID transactions, and instant queryability — all with no external service. Scaling to PostgreSQL/MySQL would be a natural next step for a multi-node deployment.

---

## 🧪 Running Tests

```bash
# Pre-flight check (verifies Suricata, rules, and database)
./test_system.sh

# Full reset (flushes iptables, removes temp files, keeps DB)
./cleanup.sh
```

---

## 🎓 Skills Demonstrated

- **Network Security**: TCP/IP, ICMP protocol analysis, signature-based detection
- **SIEM/SOAR**: Event correlation, automated incident response pipeline
- **Python**: JSON parsing, SQLite3, subprocess orchestration, file I/O, error handling
- **Linux Administration**: systemd services, iptables firewall management, file permissions
- **DevSecOps**: Infrastructure-as-code (install.sh), test automation, forensic logging
- **Suricata**: Custom rule authoring (threshold, classtype, itype), EVE JSON logging
