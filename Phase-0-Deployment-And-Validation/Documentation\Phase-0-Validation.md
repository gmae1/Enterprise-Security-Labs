# Phase 0 — Infrastructure Validation Report

## Objective

The objective of Phase 0 was to deploy and validate the infrastructure required for future networking, system administration, and cybersecurity exercises.

The environment was tested to confirm that routing, DHCP, DNS, Active Directory, internet connectivity, SSH, SIEM access, and internal network communication were functioning correctly.

## Validation Results

| Test | Source | Destination / Service | Result |
|---|---|---|---|
| Gateway Connectivity | Windows 11 | pfSense `192.168.50.1` | PASS |
| Internet Connectivity | Windows 11 | Internet | PASS |
| DHCP Assignment | Windows 11 | pfSense DHCP | PASS |
| Linux Gateway Connectivity | LINUX-01 | pfSense | PASS |
| Linux Internet Connectivity | LINUX-01 | Internet | PASS |
| SSH Connectivity | Windows 11 | LINUX-01 | PASS |
| Wazuh Network Connectivity | WAZUH-01 | pfSense | PASS |
| Wazuh Internet Connectivity | WAZUH-01 | Internet | PASS |
| Wazuh Dashboard Access | Windows 11 | WAZUH-01 | PASS |
| Kali Gateway Connectivity | KALI-01 | pfSense | PASS |
| Kali Internet Connectivity | KALI-01 | Internet | PASS |
| DC Gateway Connectivity | DC-01 | pfSense | PASS |
| DC Internet Connectivity | DC-01 | Internet | PASS |
| AD DNS Resolution | DC-01 | `corp.contoso.local` | PASS |
| Client AD DNS Resolution | Windows 11 | `corp.contoso.local` | PASS |
| Domain Join | Windows 11 | `corp.contoso.local` | PASS |

## DHCP Validation

pfSense successfully provided dynamic IPv4 addressing to client systems from the configured DHCP pool:

`192.168.50.100 - 192.168.50.199`

Client systems successfully received:

- IPv4 addresses
- Subnet configuration
- Default gateway information

## Routing Validation

Internal systems successfully reached the pfSense LAN interface at:

`192.168.50.1`

External connectivity was also successfully tested, confirming that internal traffic could traverse:

`Endpoint → SEC-LAB → pfSense → Internet`

## DNS and Active Directory Validation

DC-01 was configured with the static address:

`192.168.50.10`

The Windows 11 endpoint was configured to use DC-01 as its DNS server.

DNS successfully resolved:

`corp.contoso.local → 192.168.50.10`

The Windows 11 workstation successfully discovered and joined the `corp.contoso.local` Active Directory domain.

## Linux and SSH Validation

LINUX-01 successfully obtained network connectivity through SEC-LAB.

OpenSSH was enabled and tested successfully from the Windows workstation.

This validated internal TCP/IP connectivity and remote administration capability between Windows and Linux systems.

## Wazuh Validation

The Wazuh appliance successfully connected to SEC-LAB and reached the pfSense gateway and external networks.

The Wazuh Dashboard was successfully accessed from the Windows workstation.

This confirms that the SIEM infrastructure is available for future endpoint monitoring and detection exercises.

## Kali Linux Validation

KALI-01 successfully:

- Received network configuration
- Reached the pfSense gateway
- Resolved DNS
- Accessed external networks
- Completed system updates

Kali will be used only for authorized security testing inside the isolated homelab.

## Phase 0 Result

**PASS — Core Infrastructure Operational**

The SEC-LAB environment is ready to support future networking and cybersecurity projects.
