---
title: "Adversary Emulation Using Atomic Red Team"
date: 2026-04-10
description: "A practical exploration of MITRE ATT&CK techniques using Atomic Red Team...from adversary simulation to defensive validation."
tags:
  - linux
  - SIEM
image: /images/xsskernel/berserk.jpg
---

## Introduction


| The Problem | Limitation |
|---|---|
| SIEM alone | Passive — correlates and alerts but can't act. High false-positive rate. No real-time behavioral telemetry. |
| XDR alone | Limited long-term retention. Vendor lock-in. Poor log ingestion from legacy systems. |
| Isolated tools | Visibility silos. No cross-domain correlation. Manual response = slow MTTR. |

The usual fix is "buy more tooling." This project tests something else: a fully open-source SOC that matches commercial solutions on the metrics that matter, without the licensing cost.

```
Raw Attack  ->  Detection  ->  Correlation  ->  Enrichment  ->  Response
Kali            Wazuh          Wazuh            MISP            Wazuh
                Suricata       (SIEM+XDR)       Shuffle         Active Response
                                                 TheHive
```

| Layer | Tool(s) | Role |
|---|---|---|
| Raw Attack | Kali | Generates the attack |
| Detection | Wazuh, Suricata | Catches it |
| Correlation | Wazuh (SIEM+XDR) | Ties events together |
| Enrichment | MISP, Shuffle, TheHive | Adds context, automates triage |
| Response | Wazuh Active Response | Acts on it |

## Architecture

### Attacks

**Kali Linux — Red Team**

IP address: `192.168.120.128`

| Tools | Attack activity |
|---|---|
| Nmap, Hydra, Metasploit | Reconnaissance, brute force, exploitation |

Attack activity targets both monitored servers.

### Our Servers

| Server | Address | Monitoring and telemetry |
|---|---|---|
| Ubuntu Victim (Linux) | `192.168.120.130` | Wazuh Agent, Auditd, Suricata (NDR) |
| Windows Server 2019 | `192.168.120.131` | Wazuh Agent, Sysmon, Windows Event Logging |

Both servers forward logs and telemetry to the SOC Server over **TCP/UDP 1514**.

### SOC Server

**Docker Stack** · IP address: `192.168.120.129`

| Service | Role | Port / input |
|---|---|---|
| Wazuh | SIEM + EDR | Dashboard: `443` |
| Suricata | Network Detection (NDR) | `eve.json` ingestion |
| MISP | Threat intelligence and IOC enrichment | `8443` |
| Shuffle (SOAR) | Automates alert enrichment and response workflows | `3001` |
| TheHive | Case management | `9001` |

**Response workflow:** Wazuh alert → MISP enrichment → TheHive case → Telegram notification → block.
