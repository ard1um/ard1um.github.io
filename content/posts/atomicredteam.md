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

```
                              +------------------------------+
                              |           ATTACKS            |
                              |   KALI LINUX - Red Team      |
                              |      192.168.120.128         |
                              |  Nmap | Hydra | Metasploit   |
                              | Recon | Brute force | Exploit|
                              +---------------+--------------+
                                              |
                                        Attack activity
                          +-------------------+-------------------+
                          |                                       |
                          v                                       v
       +--------------------------------+       +--------------------------------+
       |    Ubuntu Victim (Linux)       |       |     Windows Server 2019        |
       |       192.168.120.130          |       |       192.168.120.131          |
       |     Wazuh Agent | Auditd       |       |    Wazuh Agent | Sysmon        |
       |     Suricata (NDR)             |       |    Windows Event Logging       |
       +---------------+----------------+       +----------------+---------------+
                       |                                          |
                       +--------------------+---------------------+
                                            |
                              Logs + telemetry (TCP/UDP 1514)
                                            |
                                            v
       +-----------------------------------------------------------------------+
       |                    SOC SERVER - 192.168.120.129                       |
       |                           Docker Stack                                |
       |                                                                       |
       |  +------------------+  +------------------+  +---------------------+  |
       |  | Wazuh            |  | Suricata         |  | MISP                |  |
       |  | SIEM + EDR       |  | NDR              |  | Threat Intelligence |  |
       |  | Dashboard: 443   |  | eve.json         |  | IOC enrichment: 8443|  |
       |  +--------+---------+  +--------+---------+  +----------+----------+  |
       |           |                     |                       |             |
       |           +---------------------+-----------------------+             |
       |                                 |                                     |
       |                      Alerts + threat context                          |
       |                                 v                                     |
       |                    +------------------------+                         |
       |                    | Shuffle (SOAR)         |                         |
       |                    | Port 3001              |                         |
       |                    | Alert -> MISP ->       |                         |
       |                    | TheHive -> Telegram    |                         |
       |                    | -> Block               |                         |
       |                    +-----------+------------+                         |
       |                                |                                      |
       |                                v                                      |
       |                    +------------------------+                         |
       |                    | TheHive                |                         |
       |                    | Case Management        |                         |
       |                    | Port 9001              |                         |
       |                    +------------------------+                         |
       +-----------------------------------------------------------------------+
```
