# Secure Web Gateway Lab

## Overview

This project documents the buildout of a lab Secure Web Gateway using **OPNsense**, **Squid Proxy**, **C-ICAP**, **ClamAV**, and **Wazuh**. The goal of the lab was to simulate an enterprise web security control that forces client web traffic through a proxy, scans downloaded content for malware, blocks malicious files, and forwards relevant firewall/proxy/security events into a SIEM for centralized visibility.

## Lab Goals

- Route client web traffic through an OPNsense-based proxy.
- Use Squid as the Secure Web Gateway proxy service.
- Integrate Squid with C-ICAP for content inspection.
- Use ClamAV as the malware scanning engine.
- Validate malware blocking with the EICAR antivirus test file.
- Forward OPNsense, Squid, C-ICAP, and ClamAV logs to Wazuh using syslog.
- Document the architecture, rules, validation steps, and troubleshooting process.

---

## Infrastructure

### Network Layout

```text
Client Workstation
        |
        | Web Traffic
        v
OPNsense Firewall / Secure Web Gateway
        |
        | Squid Proxy
        v
C-ICAP Service
        |
        | Malware Scan Request
        v
ClamAV / clamd

OPNsense Logs
        |
        | Syslog
        v
Wazuh SIEM
```

### System Roles

| System | Role |
|---|---|
| OPNsense | Firewall, routing, proxy hosting, ICAP integration, and remote log forwarding |
| Squid Proxy | Forward proxy and web traffic enforcement point |
| C-ICAP | ICAP service used by Squid for content scanning |
| ClamAV | Malware scanning engine used by C-ICAP |
| Wazuh | SIEM used for centralized logging and security visibility |
| Windows Client | Test endpoint behind the gateway |

### Example IP Layout

| Component | Address / Port |
|---|---|
| OPNsense LAN | `10.10.10.1` |
| Client Workstation | `10.10.10.106` |
| Wazuh Server | `10.10.10.50` |
| Squid Proxy Port | `3128/TCP` |
| C-ICAP Port | `1344/TCP` |
| Syslog Port | `514/UDP` |

---

## Wazuh Deployment

Wazuh was deployed as a Docker-based SIEM server on a Linux VM.

### Install Docker

```bash
sudo apt update
sudo apt install -y git docker.io docker-compose-plugin
sudo systemctl enable --now docker
```

### Set Required Kernel Parameter

```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-wazuh.conf
sudo sysctl --system
```

### Pull the Wazuh Docker Repository

```bash
git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.5
cd wazuh-docker/single-node/
```

### Generate Certificates

```bash
sudo docker compose -f generate-indexer-certs.yml run --rm generator
```

### Start Wazuh

```bash
sudo docker compose up -d
```

### Verify Containers

```bash
sudo docker compose ps
```

The Wazuh dashboard was then accessed from a browser:

```text
https://<WAZUH-SERVER-IP>
```

---

## Wazuh Syslog Configuration

Wazuh was configured to receive syslog events from OPNsense.

The Wazuh manager configuration was edited:

```bash
sudo nano config/wazuh_cluster/wazuh_manager.conf
```

The following syslog listener was added inside the main `<ossec_config>` block:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.10.10.1/32</allowed-ips>
</remote>
```

The Wazuh stack was restarted:

```bash
sudo docker compose down
sudo docker compose up -d
```

Syslog listener validation:

```bash
sudo ss -lunp | grep ':514'
```

---

## OPNsense Plugin Configuration

The following OPNsense plugins were used:

- `os-squid`
- `os-clamav`
- `os-c-icap`

These plugins provided the proxy, malware scanning, and ICAP integration needed for the Secure Web Gateway.

---

## Squid Proxy Configuration

Squid was configured as the primary web proxy inside OPNsense.

**OPNsense location:**

```text
Services → Squid Web Proxy → Administration
```

### Proxy Listener

| Setting | Value |
|---|---|
| Proxy Port | `3128` |
| Listening Interface | `LAN` |
| Allowed Clients | Internal client subnet |

The Windows client was configured to use the OPNsense proxy:

| Setting | Value |
|---|---|
| Proxy Server | `10.10.10.1` |
| Proxy Port | `3128` |

Client proxy validation was performed by reviewing Squid access logs:

```bash
tail -f /var/log/squid/access.log
```

---

## ClamAV Configuration

ClamAV was enabled from the OPNsense web interface.

**OPNsense location:**

```text
Services → ClamAV
```

### Enabled Options

- Enable `clamd` service.
- Enable `freshclam` service.
- Enable TCP Port.

ClamAV signatures were downloaded and confirmed to be up to date:

- `daily.cvd`
- `main.cvd`
- `bytecode.cvd`

The ClamAV daemon was validated from the OPNsense shell:

```bash
sockstat -l | grep -Ei 'clamd|3310|clam'
```

Expected result:

```text
clamd listening on 127.0.0.1:3310
clamd socket available at /var/run/clamav/clamd.sock
```

---

## C-ICAP Configuration

C-ICAP was enabled from the OPNsense web interface.

**OPNsense location:**

```text
Services → C-ICAP
```

### Enabled Options

- Enable `c-icap` service.
- Enable ClamAV.
- Pass on error: Disabled.

C-ICAP was validated from the shell:

```bash
sockstat -l | grep -Ei 'c-icap|1344'
```

Expected result:

```text
c-icap listening on port 1344
```

---

## Stitching Squid, C-ICAP, and ClamAV Together

Squid was configured to send web traffic to C-ICAP for request and response scanning.

**OPNsense location:**

```text
Services → Squid Web Proxy → Administration → Forward Proxy → ICAP Settings
```

### Configured ICAP URLs

| ICAP Setting | Value |
|---|---|
| Request Modify URL | `icap://[::1]:1344/avscan` |
| Response Modify URL | `icap://[::1]:1344/avscan` |

The generated Squid configuration was verified from the OPNsense shell:

```bash
grep -iE 'icap|avscan|adaptation|bypass' /usr/local/etc/squid/squid.conf
```

Relevant configuration:

```text
icap_enable on
icap_service response_mod respmod_precache icap://[::1]:1344/avscan
icap_service request_mod reqmod_precache icap://[::1]:1344/avscan

adaptation_access response_mod allow localnet
adaptation_access request_mod allow localnet
adaptation_access response_mod deny all
adaptation_access request_mod deny all
```

This confirmed that Squid was configured to send both web requests and web responses to the ICAP antivirus scanning service.

---

## Firewall Rules Created

Firewall rules were created to enforce the Secure Web Gateway path.

### Rule Summary

| Rule | Action | Source | Destination | Port(s) | Purpose |
|---|---|---|---|---|---|
| Allow Client Access to Proxy | Pass | Client subnet | OPNsense LAN address | `TCP/3128` | Allow clients to use the Squid proxy |
| Block Direct Web Bypass | Block | Client subnet | Any | `TCP/80`, `TCP/443` | Prevent clients from bypassing the proxy |
| Allow DNS to Approved Resolver | Pass | Client subnet | OPNsense or approved DNS resolver | `TCP/UDP 53` | Prevent uncontrolled external DNS usage |
| Allow OPNsense Syslog to Wazuh | Pass | OPNsense | Wazuh server | `UDP/514` | Forward firewall and proxy logs to Wazuh |
| Allow Admin Access to Wazuh Dashboard | Pass | Admin workstation | Wazuh server | `TCP/443` | Access the Wazuh web dashboard |

---

## OPNsense Remote Logging

OPNsense was configured to forward logs to Wazuh.

**OPNsense location:**

```text
System → Settings → Logging → Targets
```

### Remote Syslog Target

| Setting | Value |
|---|---|
| Enabled | Yes |
| Transport | UDP |
| Destination | Wazuh Server IP |
| Port | `514` |
| Applications | firewall, squid, c-icap, clamav, system |

This allowed OPNsense firewall, proxy, and malware scanning events to be forwarded into Wazuh for centralized visibility.

---

## Malware Scanning Validation

A safe EICAR antivirus test file was created on OPNsense:

```bash
echo 'WDVPIVAlQEFQWzRcUFpYNTQoUF4pN0NDKTd9JEVJQ0FSLVNUQU5EQVJELUFOVElWSVJVUy1URVNULUZJTEUhJEgrSCo=' | base64 -d > /tmp/eicar.txt
```

File size was verified:

```bash
wc -c /tmp/eicar.txt
```

Expected result:

```text
68 /tmp/eicar.txt
```

The file was scanned locally with ClamAV:

```bash
clamscan /tmp/eicar.txt
```

Successful detection:

```text
/tmp/eicar.txt: Eicar-Test-Signature FOUND
```

The file was also scanned through the ClamAV daemon:

```bash
clamdscan /tmp/eicar.txt
```

Successful detection:

```text
/tmp/eicar.txt: Eicar-Test-Signature FOUND
```

---

## C-ICAP Validation

The C-ICAP `avscan` service was tested directly:

```bash
c-icap-client -i ::1 -p 1344 -s avscan -f /tmp/eicar.txt -v
```

Successful result:

```text
MALWARE FOUND
Eicar-Test-Signature
HTTP/1.0 403 Forbidden
X-Infection-Found: Type=0; Resolution=2; Threat=Eicar-Test-Signature;
```

This confirmed that C-ICAP was successfully handing files to ClamAV and returning a malware block response.

---

## End-to-End Client Validation

The client was tested through the Squid proxy using the EICAR test file.

Example forced proxy test:

```powershell
curl.exe -v -x http://10.10.10.1:3128 http://pkg.opnsense.org/test/eicar.com.txt -o eicar-test.txt
```

Expected result:

```text
HTTP/1.0 403 Forbidden
MALWARE FOUND
Eicar-Test-Signature
```

This confirmed the full Secure Web Gateway malware blocking path:

```text
Client → Squid Proxy → C-ICAP → ClamAV → Block Page
```

---

## Wazuh Log Validation

After syslog forwarding was configured, Wazuh was checked for incoming OPNsense events.

Useful Wazuh search terms:

- `opnsense`
- `squid`
- `c-icap`
- `clamav`
- `eicar`
- `blocked`
- `deny`
- `firewall`

On the Wazuh server, incoming syslog traffic was validated with:

```bash
sudo tcpdump -ni any udp port 514
```

This confirmed that OPNsense was forwarding events to the Wazuh SIEM.

---

## Troubleshooting Notes

Several layers were validated independently during the build:

1. Confirmed client traffic was reaching Squid.
2. Confirmed Squid had ICAP enabled.
3. Confirmed C-ICAP was listening on port `1344`.
4. Confirmed ClamAV was listening on `127.0.0.1:3310`.
5. Confirmed ClamAV detected EICAR locally.
6. Confirmed C-ICAP blocked EICAR directly.
7. Confirmed Squid blocked EICAR through the proxy.
8. Confirmed OPNsense logs were forwarded to Wazuh.

### Useful Troubleshooting Commands

```bash
tail -f /var/log/squid/access.log
tail -f /var/log/squid/cache.log

sockstat -l | grep -Ei 'c-icap|1344'
sockstat -l | grep -Ei 'clamd|3310|clam'

grep -iE 'icap|avscan|adaptation|bypass' /usr/local/etc/squid/squid.conf

clamscan /tmp/eicar.txt
clamdscan /tmp/eicar.txt

c-icap-client -i ::1 -p 1344 -s avscan -f /tmp/eicar.txt -v
```

---

## Final Result

The lab successfully implemented a Secure Web Gateway with malware scanning and SIEM visibility.

Final validated traffic flow:

```text
Client Workstation
        ↓
Squid Proxy on OPNsense
        ↓
C-ICAP Antivirus Service
        ↓
ClamAV Malware Scan
        ↓
Malicious File Blocked
        ↓
Logs Forwarded to Wazuh
```

## Skills Demonstrated

- Secure Web Gateway architecture
- OPNsense firewall and service configuration
- Squid forward proxy configuration
- ICAP-based content inspection
- ClamAV malware scanning integration
- Firewall rule design for proxy enforcement
- Remote syslog forwarding
- Wazuh SIEM ingestion validation
- Layered troubleshooting and service validation

