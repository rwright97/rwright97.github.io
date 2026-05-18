#  Secure Web Gateway Lab

##  Overview
This project documents the buildout of a lab **Secure Web Gateway** using OPNsense, Squid Proxy, C-ICAP, ClamAV, and Wazuh. The goal of the lab was to simulate an enterprise web security control that forces client traffic through a proxy, scans downloaded content for malware, blocks malicious files, and forwards security logs into a SIEM.

##  Lab Goals
- Route client web traffic through an OPNsense-based proxy.
- Use **Squid** as the Secure Web Gateway proxy service.
- Integrate Squid with **C-ICAP** for content inspection.
- Use **ClamAV** as the malware scanning engine.
- Validate malware blocking with the EICAR test file.
- Forward OPNsense/Squid security logs to **Wazuh** using syslog.
- Document the architecture, rules, validation, and troubleshooting.

---

##  Infrastructure

### Network Layout

<pre/><code>Client Workstation
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
Wazuh SIEM</code>

### System Roles

| System | Role |
| :--- | :--- |
| **OPNsense** | Firewall, routing, proxy, ICAP integration |
| **Squid Proxy** | Forward proxy and web traffic enforcement |
| **C-ICAP** | ICAP service used by Squid for content scanning |
| **ClamAV** | Malware scanning engine |
| **Wazuh** | SIEM for centralized logging and detection |
| **Windows Client** | Test endpoint behind the gateway |

### Example IP Layout

| Component | IP Address / Port |
| :--- | :--- |
| **OPNsense LAN** | `10.10.10.1` |
| **Client Workstation** | `10.10.10.106` |
| **Wazuh Server** | `10.10.10.50` |
| **Proxy Port** | `3128` |
| **ICAP Port** | `1344` |
| **Syslog Port** | `514/UDP` |

---

##  Wazuh Deployment
Wazuh was deployed as a Docker-based SIEM server on a Linux VM.

**1. Install Docker**
<pre/><code>sudo apt update
sudo apt install -y git docker.io docker-compose-plugin
sudo systemctl enable --now docker</code>

**2. Set Required Kernel Parameter**
<pre/><code>sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee /etc/sysctl.d/99-wazuh.conf
sudo sysctl --system</code>

**3. Pull the Wazuh Docker Repository**
<pre/><code>git clone https://github.com/wazuh/wazuh-docker.git -b v4.14.5
cd wazuh-docker/single-node/</code>

**4. Generate Certificates & Start Wazuh**
<pre/><code>sudo docker compose -f generate-indexer-certs.yml run --rm generator
sudo docker compose up -d</code>

**5. Verify Containers**
<pre/><code>sudo docker compose ps</code>
*The Wazuh dashboard was then accessed from a browser at `https://<WAZUH-SERVER-IP>`.*

---

##  Wazuh Syslog Configuration
Wazuh was configured to receive syslog events from OPNsense. The manager configuration was edited:

<pre/><code>sudo nano config/wazuh_cluster/wazuh_manager.conf</code>

The following syslog listener was added inside the main `<ossec_config>` block:
<pre/><code><remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>10.10.10.1/32</allowed-ips>
</remote></code>

The Wazuh stack was restarted to apply changes:
<pre/><code>sudo docker compose down
sudo docker compose up -d</code>

**Syslog listener validation:**
<pre/><code>sudo ss -lunp | grep ':514'</code>

---

##  OPNsense Configuration

### Plugin Configuration
The following OPNsense plugins were installed to provide the firewall, proxy, malware scanning, and ICAP integration:
- `os-squid`
- `os-clamav`
- `os-c-icap`

### Squid Proxy Configuration
Configured as the primary web proxy inside OPNsense.
- **Location:** `Services → Squid Web Proxy → Administration`
- **Proxy Port:** `3128`
- **Listening Interface:** LAN
- **Allowed Clients:** Internal client subnet

The Windows client was configured to use the OPNsense proxy (`10.10.10.1:3128`). Client proxy validation was performed by reviewing Squid access logs:
<pre/><code>tail -f /var/log/squid/access.log</code>

### ClamAV Configuration
- **Location:** `Services → ClamAV`
- **Enabled options:** Enable clamd service, Enable freshclam service, Enable TCP Port.

Signatures were downloaded (`daily.cvd`, `main.cvd`, `bytecode.cvd`). The daemon was validated from the shell:
<pre/><code>sockstat -l | grep -Ei 'clamd|3310|clam'</code>
*Expected result:*
<pre/><code>clamd listening on 127.0.0.1:3310
clamd socket available at /var/run/clamav/clamd.sock</code>

### C-ICAP Configuration
- **Location:** `Services → C-ICAP`
- **Enabled options:** Enable c-icap service, Enable ClamAV, Pass on error: Disabled.

C-ICAP was validated from the shell:
<pre/><code>sockstat -l | grep -Ei 'c-icap|1344'</code>
*Expected result:*
<pre/><code>c-icap listening on port 1344</code>

---

## 🔗 Stitching Squid, C-ICAP, and ClamAV Together
Squid was configured to send web traffic to C-ICAP for request and response scanning.
- **Location:** `Services → Squid Web Proxy → Administration → Forward Proxy → ICAP Settings`
- **Request Modify URL:** `icap://[::1]:1344/avscan`
- **Response Modify URL:** `icap://[::1]:1344/avscan`

Generated configuration was verified:
<pre/><code>grep -iE 'icap|avscan|adaptation|bypass' /usr/local/etc/squid/squid.conf</code>
*Relevant configuration confirmed:*
<pre/><code>icap_enable on
icap_service response_mod respmod_precache icap://[::1]:1344/avscan
icap_service request_mod reqmod_precache icap://[::1]:1344/avscan

adaptation_access response_mod allow localnet
adaptation_access request_mod allow localnet
adaptation_access response_mod deny all
adaptation_access request_mod deny all</code>

---

##  Firewall Rules Created
Firewall rules were created to enforce the Secure Web Gateway path.

| Rule Name | Action | Source | Destination | Port | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Allow Client Access to Proxy** | Pass | Client subnet | OPNsense LAN | TCP/3128 | Allow clients to use Squid proxy |
| **Block Direct Web Bypass** | Block | Client subnet | Any | TCP/80, 443 | Prevent bypassing the proxy |
| **Allow DNS to Resolver** | Pass | Client subnet | Approved DNS | TCP/UDP 53 | Prevent uncontrolled external DNS |
| **Allow Syslog to Wazuh** | Pass | OPNsense | Wazuh server | UDP/514 | Forward logs to Wazuh |
| **Allow Admin to Wazuh** | Pass | Admin PC | Wazuh server | TCP/443 | Access Wazuh web dashboard |

---

##  OPNsense Remote Logging
OPNsense was configured to forward events to Wazuh for centralized visibility.
- **Location:** `System → Settings → Logging → Targets`
- **Enabled:** Yes
- **Transport:** UDP
- **Destination:** Wazuh Server IP
- **Port:** 514
- **Applications:** firewall, squid, c-icap, clamav, system

---

## Validation & Testing

### 1. Malware Scanning Validation (EICAR)
A safe EICAR antivirus test file was created on OPNsense:
<pre/><code>echo 'WDVPIVAlQEFQWzRcUFpYNTQoUF4pN0NDKTd9JEVJQ0FSLVNUQU5EQVJELUFOVElWSVJVUy1URVNULUZJTEUhJEgrSCo=' | base64 -d > /tmp/eicar.txt</code>
Local ClamAV scan:
<pre/><code>clamscan /tmp/eicar.txt
clamdscan /tmp/eicar.txt</code>
*Successful detection:* `/tmp/eicar.txt: Eicar-Test-Signature FOUND`

### 2. C-ICAP Validation
The C-ICAP `avscan` service was tested directly:
<pre/><code>c-icap-client -i ::1 -p 1344 -s avscan -f /tmp/eicar.txt -v</code>
*Successful result:*
<pre/><code>MALWARE FOUND
Eicar-Test-Signature
HTTP/1.0 403 Forbidden
X-Infection-Found: Type=0; Resolution=2; Threat=Eicar-Test-Signature;</code>

### 3. End-to-End Client Validation
The client was tested through the Squid proxy using the EICAR file:
<pre/><code>curl.exe -v -x http://10.10.10.1:3128 http://pkg.opnsense.org/test/eicar.com.txt -o eicar-test.txt</code>
*Expected result:*
<pre/><code>HTTP/1.0 403 Forbidden
MALWARE FOUND
Eicar-Test-Signature</code>

### 4. Wazuh Log Validation
On the Wazuh server, incoming syslog traffic was validated to ensure logs were being received:
<pre/><code>sudo tcpdump -ni any udp port 514</code>
*(Useful Wazuh search terms used: `opnsense`, `squid`, `c-icap`, `clamav`, `eicar`, `blocked`, `deny`, `firewall`)*

---

##  Troubleshooting Notes
During the build, layers were independently validated:
1. Confirmed client traffic was reaching Squid.
2. Confirmed Squid had ICAP enabled.
3. Confirmed C-ICAP was listening on port `1344`.
4. Confirmed ClamAV was listening on `127.0.0.1:3310`.
5. Confirmed ClamAV detected EICAR locally.
6. Confirmed C-ICAP blocked EICAR directly.
7. Confirmed Squid blocked EICAR through the proxy.
8. Confirmed OPNsense logs were forwarded to Wazuh.

**Key Troubleshooting Commands:**
<pre/><code>tail -f /var/log/squid/access.log
tail -f /var/log/squid/cache.log
sockstat -l | grep -Ei 'c-icap|1344'
sockstat -l | grep -Ei 'clamd|3310|clam'
grep -iE 'icap|avscan|adaptation|bypass' /usr/local/etc/squid/squid.conf
clamscan /tmp/eicar.txt
c-icap-client -i ::1 -p 1344 -s avscan -f /tmp/eicar.txt -v</code>

---

##  Final Result
The lab successfully implemented a Secure Web Gateway with malware scanning and SIEM visibility. 

**Validated Traffic Flow:**
> **Client Workstation** ➔ **Squid Proxy** *(OPNsense)* ➔ **C-ICAP** *(Antivirus Service)* ➔ **ClamAV** *(Malware Scan)* ➔ **Malicious File Blocked** ➔ **Logs Forwarded to Wazuh**
