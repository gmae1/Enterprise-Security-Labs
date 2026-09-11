# SEC-LAB Asset Inventory

## Overview

This document maintains an inventory of the systems deployed within the Enterprise Security Homelab.

## Network Information

| Property | Configuration |
|---|---|
| Internal Network | SEC-LAB |
| Network | 192.168.50.0/24 |
| Default Gateway | 192.168.50.1 |
| DHCP Server | pfSense |
| DHCP Pool | 192.168.50.100 - 192.168.50.199 |
| Active Directory Domain | corp.contoso.local |
| Active Directory DNS | 192.168.50.10 |

## Asset Inventory

| Asset | Operating System | Role | IP Address | Addressing |
|---|---|---|---|---|
| pfSense | pfSense 2.9 | Firewall / Router / DHCP / NAT | 192.168.50.1 | Static |
| DC-01 | Windows Server 2022 | Domain Controller / DNS | 192.168.50.10 | Static |
| Workstation1 | Windows 11 | Domain-Joined Endpoint | 192.168.50.101 | DHCP |
| LINUX-01 | Ubuntu Server 24.04 LTS | Linux Server / SSH | 192.168.50.102 | DHCP |
| WAZUH-01 | Wazuh Appliance | SIEM / Security Monitoring | 192.168.50.103 | DHCP |
| KALI-01 | Kali Linux | Security Testing Workstation | 192.168.50.104 | DHCP |

## Infrastructure Services

### pfSense

Provides the primary network infrastructure for SEC-LAB.

Services:
- Routing
- Firewalling
- DHCP
- NAT
- Default gateway

### DC-01

Provides centralized identity and name resolution services.

Services:
- Active Directory Domain Services
- DNS
- Domain authentication
- Global Catalog

Domain:

`corp.contoso.local`

### LINUX-01

Ubuntu Server used for Linux administration and future security monitoring exercises.

Services:
- OpenSSH

### WAZUH-01

Central security monitoring platform used for:

- Log collection
- Endpoint monitoring
- Security event analysis
- Detection and alerting

### KALI-01

Security testing workstation used for authorized testing against systems inside the isolated homelab.

### Workstation1

Windows 11 endpoint joined to:

`corp.contoso.local`

The workstation will be used for endpoint administration, Group Policy, authentication, logging, and security monitoring exercises.

## Addressing Strategy

Infrastructure systems that require predictable addresses use static IP addressing.

DC-01 uses `192.168.50.10` because domain clients depend on the server for DNS and Active Directory services.

Client systems use addresses from the pfSense DHCP pool:

`192.168.50.100 - 192.168.50.199`

Current DHCP addresses represent active lab assignments and may change as leases are renewed.
