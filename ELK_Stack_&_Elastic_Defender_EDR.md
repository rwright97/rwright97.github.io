# ELK Stack & Elastic Defender EDR

## Overview
This write-up documents my deployment of a multi node docker container architecture for Elastic Stack. My plan is to host the ELK stack through my docker server, and i will setup a fleet server container for easy agent management. The agent will be downloaded onto the Windows VM via Powershell, and i will test functionality by performing a HTML Smuggling attack on a seperate Kali Linux VM.

## Lab Goals
- Deploy a comprehensive SIEM solution for endpoints on my homelab network.
- Be able to methodically plan and execute an attack scenario. (HTML Smuggling using 


