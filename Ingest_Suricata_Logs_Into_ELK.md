# Ingesting Suricata Logs Into Elasticsearch

## Overview
This lab demonstrates the configuration and deployment of a custom log forwarding pipeline to ingest Suricata IDS logs from an OPNsense firewall into the Elastic Stack. By leveraging a custom syslog-ng configuration, Suricata's eve.json logs are successfully forwarded over UDP to a containerized Elastic Agent. This project highlights network security monitoring setup, log routing, and endpoint agent configuration. By placing IDS/IPS log data into the SIEM we can enrich our attack stories with captured web traffic from the network.

## Lab Goals
-Configure OPNsense Log Forwarding: Utilize the OPNsense Syslog-ng daemon to read raw Suricata EVE JSON logs and forward them to a remote destination.

-Deploy Elastic Agent: Stand up a Dockerized Elastic Agent with custom port bindings to receive incoming UDP log streams.

-Configure Elastic Fleet Integrations: Create and apply a "Custom UDP Logs" integration policy within Elastic Fleet to parse and ingest the incoming JSON telemetry.

-Verify Network Traffic: Utilize packet sniffing tools (tcpdump) on both the source (OPNsense) and destination (Elastic Agent host) to validate successful log transmission and reception.

## Technical Setup

1. OPNsense & Syslog-ng Configuration
The first step required enabling the local syslog daemon on the firewall, as shown running in the OPNsense dashboard in Syslog-NG-Daemon.jpg. A custom configuration file was created to dictate the flow of the logs. As seen below, syslog-ng is instructed to read the /var/log/suricata/eve.json file without parsing it (flags(no-parse)). The destination is set to the Elastic Agent's IP address (192.168.3.5) over UDP port 5144, formatting the payload strictly as the original JSON message.

<img width="3822" height="2312" alt="Syslog-NG-Daemon" src="https://github.com/user-attachments/assets/1c43f783-f249-4554-8572-a79fceceee89" />


2. Elastic Agent Deployment
To receive the logs, an Elastic Agent was deployed as a Docker container on the destination host. The deployment script, explicitly maps port 5144:5144/udp to ensure the container can listen for the incoming Syslog traffic from the firewall.

<img width="3822" height="2312" alt="Create_Elastic_Agent" src="https://github.com/user-attachments/assets/88bbd6e0-77f3-43f0-8b17-2557a1e54a0d" />


4. Elastic Fleet Integration
To process the incoming data stream, a specific integration policy was created in Elastic Fleet.

    As shown here, the "Custom UDP Logs" integration was added to the agent's policy.

<img width="3822" height="2312" alt="Installed_Integrations" src="https://github.com/user-attachments/assets/b7a08023-8557-4818-882e-a94498b69f17" />


   this  details the setup, configuring the integration to listen on 0.0.0.0 at port 5144 and writing the data to the suricata.eve dataset.
  
   <img width="3822" height="2312" alt="UDP Integration Pic1" src="https://github.com/user-attachments/assets/22ed8c73-bb12-426d-888e-a95b9b08a0ba" />


  To ensure the raw JSON payload is handled correctly upon ingestion, a decode_json_fields processor was added to the advanced configuration, targeting the message field, as captured in below.

  <img width="3822" height="2312" alt="UDP_Integration_Pic2" src="https://github.com/user-attachments/assets/6ec3d2dd-5314-45d3-b8cb-831f67e45d96" />


5. Traffic Verification
To guarantee the pipeline was operating correctly, network traffic was captured at both ends of the connection:

    This confirms that syslog-ng successfully restarted and that UDP packets were actively leaving the OPNsense interface (192.168.3.1) destined for port 5144.

<img width="3822" height="2312" alt="Test_Custome_SysLog-NG" src="https://github.com/user-attachments/assets/8fdfe889-db27-4236-93ba-af832950982d" />



  Similarly, this picture below shows a tcpdump on the destination Docker host verifying that the UDP packets were arriving and traversing the virtual container interfaces successfully.


<img width="3840" height="2332" alt="image" src="https://github.com/user-attachments/assets/c71b9155-1eb2-4be0-8dd4-ec29248d102d" />


## Conclusion

This lab successfully established a robust, custom log ingestion pipeline. By manually configuring syslog-ng on OPNsense and pairing it with a Dockerized Elastic Agent running a custom UDP integration, Suricata EVE logs are now actively streaming into the Elastic environment. This setup provides a scalable foundation for building advanced threat hunting dashboards and automated alerting rules based on Suricata IDS signatures.
