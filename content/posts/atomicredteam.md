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

<div class="architecture-card">
  <strong>Kali Linux <span class="architecture-muted">— Red Team</span></strong>
  <p><code>192.168.120.128</code></p>
  <p><strong>Tools:</strong> Nmap · Hydra · Metasploit</p>
  <p><strong>Activity:</strong> Reconnaissance · Brute force · Exploitation</p>
</div>

<p class="architecture-flow">Attack activity targets the monitored servers below.</p>

### Our Servers

<div class="architecture-hosts">
  <div class="architecture-card">
    <strong>Ubuntu Victim <span class="architecture-muted">— Linux</span></strong>
    <p><code>192.168.120.130</code></p>
    <ul>
      <li>Wazuh Agent</li>
      <li>Auditd</li>
      <li>Suricata (NDR)</li>
    </ul>
  </div>
  <div class="architecture-card">
    <strong>Windows Server 2019</strong>
    <p><code>192.168.120.131</code></p>
    <ul>
      <li>Wazuh Agent + Sysmon</li>
      <li>Windows Event Logging</li>
    </ul>
  </div>
</div>

<p class="architecture-flow">Logs and telemetry → TCP/UDP 1514 → SOC Server</p>

### SOC Server

<div class="architecture-card">
  <strong>SOC Server <span class="architecture-muted">— Docker Stack</span></strong>
  <p><code>192.168.120.129</code></p>
</div>

<div class="architecture-services">
  <div class="architecture-card">
    <strong>Wazuh</strong>
    <p>SIEM + EDR</p>
    <p class="architecture-muted">Dashboard · Port 443</p>
  </div>
  <div class="architecture-card">
    <strong>Suricata</strong>
    <p>Network Detection (NDR)</p>
    <p class="architecture-muted">eve.json ingestion</p>
  </div>
  <div class="architecture-card">
    <strong>MISP</strong>
    <p>Threat Intelligence</p>
    <p class="architecture-muted">IOC enrichment · Port 8443</p>
  </div>
</div>

<p class="architecture-flow">Wazuh alerts + threat intelligence → Shuffle SOAR · Port 3001</p>

<div class="architecture-card">
  <strong>Shuffle <span class="architecture-muted">— SOAR</span></strong>
  <p>Automated workflow: alert → enrich → orchestrate</p>
  <p class="architecture-muted">Alert → MISP → TheHive → Telegram → Block</p>
</div>

<p class="architecture-flow">Incident case management</p>

<div class="architecture-card">
  <strong>TheHive</strong>
  <p>Case Management · Port 9001</p>
</div>

<style>
.architecture-card {
  padding: 1rem;
  border: 1px solid var(--default_stroke);
  border-radius: 10px;
  background: var(--default_hl_bg);
}
.architecture-card p {
  margin: .35rem 0 0;
}
.architecture-card ul {
  margin-bottom: 0;
}
.architecture-muted {
  color: var(--default_dim_fg);
  font-size: .9em;
}
.architecture-hosts,
.architecture-services {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(min(100%, 220px), 1fr));
  gap: .75rem;
}
.architecture-flow {
  margin: .75rem 0;
  color: var(--default_accent);
  text-align: center;
  font-weight: 600;
}
</style>
