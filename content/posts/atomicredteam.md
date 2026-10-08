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

The lab has three parts: the attack source, the monitored servers, and the SOC platform. Follow the arrows to see how activity and telemetry move through the environment.

<style>
.soc-architecture, .soc-architecture * { box-sizing: border-box; }
.soc-architecture { margin: 1.5rem 0; color: #e2e8f0; font: 15px/1.5 system-ui, sans-serif; }
.soc-architecture section { padding: 1.25rem; border: 1px solid; border-radius: 16px; }
.soc-architecture h3 { display: flex; align-items: center; gap: .65rem; margin: 0 0 1rem; color: #f8fafc; font-size: 1.05rem; }
.soc-architecture h3 span { display: inline-grid; width: 2rem; height: 2rem; place-items: center; border-radius: 50%; color: #0f172a; font-size: .9rem; }
.soc-attack { background: #211b30; border-color: #8b5cf6 !important; }
.soc-attack h3 span { background: #a78bfa; }
.soc-hosts { background: #172827; border-color: #14b8a6 !important; }
.soc-hosts h3 span { background: #5eead4; }
.soc-core { background: #1b2232; border-color: #60a5fa !important; }
.soc-core h3 span { background: #93c5fd; }
.soc-card { padding: 1rem; border: 1px solid #475569; border-radius: 12px; background: rgba(15, 23, 42, .55); }
.soc-card strong { display: block; margin-bottom: .2rem; color: #fff; font-size: 1rem; }
.soc-meta { color: #cbd5e1; font-size: .88rem; }
.soc-tools, .soc-services, .soc-output { display: grid; gap: .75rem; }
.soc-tools { grid-template-columns: repeat(3, minmax(0, 1fr)); margin-top: 1rem; }
.soc-tools span { padding: .45rem .6rem; border-radius: 999px; background: #34294b; color: #ddd6fe; text-align: center; font-size: .85rem; font-weight: 700; }
.soc-services { grid-template-columns: repeat(3, minmax(0, 1fr)); }
.soc-output { grid-template-columns: repeat(2, minmax(0, 1fr)); }
.soc-service-wazuh { border-color: #818cf8; }
.soc-service-suricata { border-color: #38bdf8; }
.soc-service-misp { border-color: #fbbf24; }
.soc-service-shuffle { max-width: 36rem; margin: 0 auto; border: 1px solid #c084fc; background: #2b2140; text-align: center; }
.soc-service-shuffle strong { color: #f3e8ff; }
.soc-service-hive { border-color: #fb7185; }
.soc-service-response { border-color: #f97316; }
.soc-connector { padding: .65rem .25rem; color: #94a3b8; text-align: center; font-size: .82rem; font-weight: 700; }
.soc-connector b { display: block; color: #38bdf8; font-size: 1.3rem; line-height: 1.2; }
.soc-flow { margin-top: 1rem; color: #cbd5e1; font-size: .9rem; }
@media (max-width: 600px) {
  .soc-architecture section { padding: 1rem; }
  .soc-services { grid-template-columns: 1fr; }
  .soc-tools span { font-size: .75rem; }
}
</style>

<div class="soc-architecture">
  <section class="soc-attack">
    <h3><span>1</span> Attacks</h3>
    <div class="soc-card">
      <strong>Kali Linux <span class="soc-meta">· Red Team</span></strong>
      <div class="soc-meta">192.168.120.128</div>
      <div class="soc-tools"><span>Nmap</span><span>Hydra</span><span>Metasploit</span></div>
      <div class="soc-meta" style="margin-top: .8rem;">Reconnaissance · Brute force · Exploitation</div>
    </div>
  </section>

  <div class="soc-connector">ATTACK ACTIVITY<b>↓</b></div>

  <section class="soc-hosts">
    <h3><span>2</span> Our Servers</h3>
    <div class="soc-output">
      <div class="soc-card">
        <strong>Ubuntu Victim</strong>
        <div class="soc-meta">Linux · 192.168.120.130</div>
        <hr style="border: 0; border-top: 1px solid #3d5b60; margin: .8rem 0;">
        <div>Wazuh Agent</div>
        <div>Auditd</div>
        <div>Suricata <span class="soc-meta">· NDR</span></div>
      </div>
      <div class="soc-card">
        <strong>Windows Server</strong>
        <div class="soc-meta">192.168.120.131</div>
        <hr style="border: 0; border-top: 1px solid #3d5b60; margin: .8rem 0;">
        <div>Wazuh Agent + Sysmon</div>
        <div>Windows Event Logging</div>
      </div>
    </div>
  </section>

  <div class="soc-connector">LOGS + TELEMETRY · TCP/UDP 1514<b>↓</b></div>

  <section class="soc-core">
    <h3><span>3</span> SOC Server</h3>
    <div class="soc-meta" style="margin: -.4rem 0 1rem 2.65rem;">192.168.120.129 · Docker Stack</div>
    <div class="soc-services">
      <div class="soc-card soc-service-wazuh">
        <strong>Wazuh</strong><div>SIEM + EDR</div><div class="soc-meta">Dashboard · Port 443</div>
      </div>
      <div class="soc-card soc-service-suricata">
        <strong>Suricata</strong><div>Network Detection · NDR</div><div class="soc-meta">eve.json ingestion</div>
      </div>
      <div class="soc-card soc-service-misp">
        <strong>MISP</strong><div>Threat Intelligence</div><div class="soc-meta">IOC enrichment · Port 8443</div>
      </div>
    </div>
    <div class="soc-connector">ALERTS + CONTEXT<b>↓</b></div>
    <div class="soc-card soc-service-shuffle">
      <strong>Shuffle · SOAR</strong>
      <div>Port 3001 · Alert → Enrich → Orchestrate</div>
      <div class="soc-meta">Automated workflow</div>
    </div>
    <div class="soc-connector"><b>↓</b></div>
    <div class="soc-output">
      <div class="soc-card soc-service-hive">
        <strong>TheHive</strong><div>Case Management</div><div class="soc-meta">Port 9001 · Incident tracking</div>
      </div>
      <div class="soc-card soc-service-response">
        <strong>Response Action</strong><div>For example: block or escalate</div>
      </div>
    </div>
  </section>
  <p class="soc-flow"><strong>Flow:</strong> Kali generates activity → endpoints produce telemetry → the SOC detects and enriches alerts → Shuffle coordinates a case and response.</p>
</div>
