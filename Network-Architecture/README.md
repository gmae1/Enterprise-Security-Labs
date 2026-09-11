## Network Architecture

The lab uses a centralized pfSense firewall/router to control connectivity between the internal security lab and external networks.

### Network

- **Internal Network:** `SEC-LAB`
- **Subnet:** `192.168.50.0/24`
- **Default Gateway:** `192.168.50.1`
- **DHCP Server:** pfSense
- **DHCP Pool:** `192.168.50.100 - 192.168.50.199`
- **Active Directory DNS:** `192.168.50.10`
- **AD Domain:** `corp.contoso.local`

### Systems

| System | Role | IP Address |
|---|---|---|
| pfSense | Firewall / Router / DHCP / NAT | `192.168.50.1` |
| DC-01 | Active Directory / DNS | `192.168.50.10` |
| Workstation1 | Windows 11 Domain Endpoint | `192.168.50.101` |
| LINUX-01 | Ubuntu Server / SSH | `192.168.50.102` |
| WAZUH-01 | SIEM / Security Monitoring | `192.168.50.103` |
| Kali Linux | Security Testing Workstation | `192.168.50.104` |

### Traffic Flow

All internal systems use a single `SEC-LAB` network interface and rely on pfSense for routing outside the internal network.

`Endpoint → SEC-LAB → pfSense → External Network`

This design prevents internal lab machines from bypassing the firewall through separate VirtualBox NAT adapters.
