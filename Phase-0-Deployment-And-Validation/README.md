## Phase 0 — Infrastructure Deployment

Phase 0 focused on building and validating the core infrastructure required for future networking and cybersecurity labs.

### Completed Infrastructure

- Deployed pfSense as the lab firewall, router, DHCP server, and NAT gateway
- Created the isolated `SEC-LAB` network (`192.168.50.0/24`)
- Configured pfSense DHCP scope (`192.168.50.100 - 192.168.50.199`)
- Deployed Windows 11 workstation
- Deployed Ubuntu Server 24.04 LTS
- Installed and validated OpenSSH on Ubuntu Server
- Deployed Wazuh SIEM
- Deployed Kali Linux security workstation
- Deployed Windows Server 2022
- Configured DC-01 with static IP `192.168.50.10`
- Installed Active Directory Domain Services and DNS
- Created the `corp.contoso.local` Active Directory domain
- Configured Windows 11 to use DC-01 for DNS
- Successfully joined the Windows 11 workstation to the domain

### Validation

Connectivity and services were tested throughout the environment.

- Internal hosts successfully received DHCP leases from pfSense
- Internal systems successfully reached the pfSense gateway
- Internet connectivity through pfSense was verified
- DNS resolution was verified
- `corp.contoso.local` successfully resolved through DC-01
- SSH connectivity to Ubuntu Server was verified
- Wazuh Dashboard access was verified
- Kali Linux network connectivity was verified
- Windows 11 successfully discovered and joined the Active Directory domain

### Key Concepts Practiced

`DHCP` `DNS` `IPv4` `Subnetting` `Default Gateways` `NAT` `Firewalling` `Routing` `Active Directory` `Domain Controllers` `SSH` `SIEM` `Virtual Networking`
