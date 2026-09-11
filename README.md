# Enterprise-Security-Labs
Enterprise security homelab featuring pfSense, Active Directory, Linux, Wazuh SIEM, Kali Linux, and Windows 11.

# Enterprise Security Homelab

## Overview

This project documents the design and deployment of an enterprise-style cybersecurity homelab built using Oracle VirtualBox.

The environment was designed to provide hands-on experience with networking, firewalls, Windows and Linux administration, Active Directory, SIEM monitoring, security testing, and SOC-style investigations.

The lab uses pfSense as the central router, firewall, DHCP server, and NAT gateway. All internal systems are connected to an isolated VirtualBox network named `SEC-LAB` and route external traffic through pfSense.

## Technologies

- pfSense
- Windows 11
- Windows Server 2022
- Active Directory Domain Services
- DNS
- Ubuntu Server 24.04 LTS
- Wazuh SIEM
- Kali Linux
- Oracle VirtualBox
- SSH

## Current Status

**Phase 0 — Infrastructure Deployment: Complete ✅**

The core virtual enterprise environment has been deployed and network connectivity between systems has been validated.


### Network Topology

<img width="693" height="850" alt="Screenshot 2026-09-11 115134" src="https://github.com/user-attachments/assets/ad281958-a90d-4ee5-a42f-030772739c07" />

