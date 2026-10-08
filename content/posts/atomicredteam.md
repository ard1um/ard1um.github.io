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

The architecture is shown in three visual stages: the red-team attack source, the monitored servers, and the SOC stack that receives and handles their telemetry.

### Attacks

```text
┌─────────────────────────────────────────────────────────────┐
│                 KALI LINUX (Red Team)                       │
│                                                             │
│              Nmap │ Hydra │ Metasploit                      │
│                    192.168.120.128                          │
└─────────────────────────────────────────────────────────────┘
```

```text
      Reconnaissance │ Brute force │ Exploitation
                         │
                         ▼
```

### Our Servers

```text
┌─────────────────────────────────────────────────────────────┐
│                    MONITORED HOSTS                          │
│                                                             │
│  ┌────────────────────────┐    ┌────────────────────────┐   │
│  │ UBUNTU VICTIM (Linux)  │    │ WINDOWS SERVER         │   │
│  │ 192.168.120.130        │    │ 192.168.120.131        │   │
│  │                        │    │                        │   │
│  │ Wazuh Agent            │    │ Wazuh Agent + Sysmon   │   │
│  │ Auditd                 │    │ Windows Event Logging  │   │
│  │ Suricata (NDR)         │    │                        │   │
│  └────────────┬───────────┘    └────────────┬───────────┘   │
│               └──────────────┬───────────────┘               │
└─────────────────────────────────────────────────────────────┘
```

```text
            Logs + telemetry (TCP/UDP 1514)
                         │
                         ▼
```

### SOC Server

```text
┌─────────────────────────────────────────────────────────────┐
│                    SOC SERVER                               │
│             192.168.120.129 — Docker Stack                  │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │    WAZUH     │  │  SURICATA    │  │    MISP      │      │
│  │   SIEM/EDR   │  │  eve.json    │  │ Threat Intel │      │
│  │   Port 443   │  │  Ingestion   │  │  Port 8443   │      │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘      │
│         └──────────────┬───┴───────────────┘                │
│                        │ alerts + IOC enrichment            │
│                        ▼                                    │
│              ┌──────────────────────┐                       │
│              │   SHUFFLE (SOAR)     │                       │
│              │      Port 3001       │                       │
│              │ Alert → MISP →        │                       │
│              │ TheHive → Block       │                       │
│              └──────────┬───────────┘                       │
│                         │                                   │
│                         ▼                                   │
│              ┌──────────────────────┐                       │
│              │      THEHIVE         │                       │
│              │  Case Management     │                       │
│              │      Port 9001       │                       │
│              └──────────────────────┘                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Attack-to-response path

```text
Kali attack  →  Ubuntu / Windows telemetry  →  Wazuh + Suricata
             →  Shuffle workflow  →  MISP enrichment  →  TheHive case
```

The diagram focuses on the roles and direction of data: Kali generates activity, the two servers provide endpoint telemetry, and the SOC stack processes alerts into cases and response actions.
