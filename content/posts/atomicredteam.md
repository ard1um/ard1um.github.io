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

The lab is designed around a simple operational model: emulate real adversary behavior, collect telemetry from multiple hosts, and verify that the SOC pipeline can detect, correlate, enrich, and respond to it.

```text
┌────────────────────────────────────────────────────────────────────────────┐
│                      KALI LINUX (Red Team)                                 │
│              Nmap │ Hydra │ Metasploit │ Atomic Red Team                  │
│                          192.168.120.128                                   │
└───────────────────────────────┬─────────────────────────────────────────────┘
                                │ Attacks: recon, brute force, exploitation
                  ┌─────────────┴─────────────┐
                  │                           │
┌─────────────────▼──────────────┐  ┌──────────────▼────────────────────┐
│ UBUNTU VICTIM (Linux)         │  │ WINDOWS SERVER 2019            │
│ 192.168.120.130               │  │ 192.168.120.131               │
│                              │  │                               │
│ Wazuh Agent + Auditd          │  │ Wazuh Agent + Sysmon          │
│ Suricata (NDR)                │  │ Windows Event Logs            │
│                              │  │                               │
└──────────────┬───────────────┘  └──────────────┬────────────────┘
               │ TCP/UDP 1514 (logs + telemetry)           │
               └──────────────────────────────┬────────────┘
                                              │
                                              ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    SOC SERVER — 192.168.120.129 (Docker Stack)            │
│                                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐             │
│  │    WAZUH     │  │   SURICATA   │  │       MISP         │             │
│  │  SIEM + EDR  │  │    NDR       │  │ Threat Intelligence│             │
│  │  Port: 443   │  │  eve.json    │  │ Port: 8443         │             │
│  └──────┬───────┘  └──────┬───────┘  └────────┬──────────┘             │
│         │                  │                    │                          │
│         │ Webhook / alerts │ Ingestion          │ IOC enrichment           │
│         └──────────┬───────┴────────────────────┼──────────────────────────┘
│                    │                            │
│             ┌──────▼────────────────────────────▼──────┐
│             │               SHUFFLE                   │
│             │        SOAR / Playbooks                │
│             │          Port: 3001                    │
│             │  Alert → MISP → TheHive → Response     │
│             └──────────────┬──────────────────────────┘
│                            │
│                  ┌─────────▼──────────┐
│                  │      THEHIVE       │
│                  │ Case Management    │
│                  │ Port: 9001         │
│                  └─────────┬──────────┘
│                            │
│                     ┌──────▼───────┐
│                     │   GRAFANA    │
│                     │ SOC Dashboard│
│                     │ Port: 3000   │
│                     └──────────────┘
│                                                                          │
│ Supporting services: Graylog for log centralization, Prometheus for     │
│ metrics, and Docker orchestration for the full SOC stack.               │
└────────────────────────────────────────────────────────────────────────────┘
```

| Layer | Component | Purpose |
|---|---|---|
| Adversary | Kali Linux | Launches recon, credential attacks, exploitation, and persistence tests |
| Endpoint telemetry | Wazuh Agent + Auditd + Sysmon | Collects host-level logs and process activity |
| Network visibility | Suricata | Monitors suspicious traffic and protocol behavior |
| Correlation | Wazuh | Normalizes, correlates, and alerts on attacker activity |
| Intelligence | MISP | Enriches alerts with known indicators and threat context |
| Automation | Shuffle | Triggers playbooks and orchestrates response actions |
| Case handling | TheHive | Tracks incidents and analyst workflow |
| Visualization | Grafana | Presents dashboards and operational visibility |

### Attack Flow

```text
Kali Linux
  └─> Nmap / Hydra / Metasploit / Atomic Red Team
       └─> Ubuntu + Windows hosts
            └─> Wazuh + Suricata + Sysmon + Auditd
                 └─> SOC Server
                      └─> Wazuh alerting
                           └─> MISP enrichment
                                └─> Shuffle playbook
                                     └─> TheHive case creation
                                          └─> Response / escalation / dashboard visibility
```

### Why this architecture matters

- It mirrors a realistic enterprise environment rather than a single-tool demo.
- It validates both attack generation and defensive detection.
- It separates signal sources by platform: Linux telemetry, Windows telemetry, and network telemetry.
- It connects detection to response, which is where many red/blue labs stop short.

This is the core value of the lab: not just detecting malicious behavior, but proving that the full chain from telemetry to investigation to response is operational.

## Detection and Response Model

```text
Raw Attack  ->  Collection  ->  Correlation  ->  Enrichment  ->  Response
Kali         Wazuh Agent  Wazuh         MISP            Wazuh Active Response
             Suricata     Suricata      Shuffle         TheHive
             Sysmon       (SIEM+XDR)    TheHive         Grafana
             Auditd
```

| Stage | Tooling | Output |
|---|---|---|
| Collection | Wazuh, Suricata, Sysmon, Auditd | Logs, telemetry, and events |
| Correlation | Wazuh | High-confidence alerts based on combined signals |
| Enrichment | MISP, Shuffle | Threat context and automated triage |
| Response | Wazuh, TheHive, Grafana | Action, case creation, and operational visibility |

## Operational Goal

The goal is to validate that a malicious technique mapped to MITRE ATT&CK can be reproduced in the lab, detected by the SOC, correlated across multiple systems, and acted on without requiring commercial tooling.

In other words, the architecture is not just a stack of tools — it is a practical security control loop:

Attack -> Detection -> Correlation -> Investigation -> Response

