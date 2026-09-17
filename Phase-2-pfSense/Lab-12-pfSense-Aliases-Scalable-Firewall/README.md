# Lab 12 — pfSense Aliases & Scalable Firewall Management

## Objective

This lab focused on using **pfSense aliases** to create more readable, reusable, and scalable firewall policies.

Instead of hard-coding individual IP addresses and ports directly into firewall rules, aliases were created to represent:

- A specific security workstation
- A group of related network service ports

The existing firewall policy was then modified to use these aliases and tested to verify that the policy continued to function correctly.

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| pfSense | Router / Firewall | `192.168.50.1` |
| Windows 11 Pro | Client Workstation | `192.168.50.101` |
| Kali Linux | Security Workstation | `192.168.50.104` |
| SEC-LAB | Internal Network | `192.168.50.0/24` |

The external destination used for connectivity testing was:

```text
104.20.23.154
```

---

## Why Use Firewall Aliases?

Firewall policies can become difficult to manage when IP addresses, networks, and port numbers are manually entered into many different rules.

For example, a firewall might contain several rules referencing:

```text
192.168.50.104
```

If that host's address changed, every rule containing the old IP would need to be located and modified individually.

Aliases solve this problem by creating reusable objects.

Instead of writing:

```text
Source: 192.168.50.104
```

a firewall rule can reference:

```text
Source: KALI_HOST
```

The alias then stores the actual IP address.

This improves:

- Readability
- Scalability
- Maintainability
- Consistency
- Administrative efficiency

---

## Creating the Kali Host Alias

The first alias was created under:

```text
Firewall → Aliases → IP
```

The following configuration was used:

| Setting | Value |
|---|---|
| Name | `KALI_HOST` |
| Description | `Kali Linux Security Workstation` |
| Type | Host(s) |
| IP Address | `192.168.50.104` |

Conceptually:

```text
KALI_HOST
    ↓
192.168.50.104
```

Firewall rules can now reference `KALI_HOST` instead of manually entering Kali's IP address.

---

## Creating the Web Port Alias

A second alias was created under:

```text
Firewall → Aliases → Ports
```

The alias was named:

```text
WEB_PORTS
```

with the description:

```text
Standard HTTP and HTTPS Ports
```

Two ports were added:

| Port | Service |
|---:|---|
| 80 | HTTP |
| 443 | HTTPS |

Conceptually:

```text
WEB_PORTS
├── TCP/80  → HTTP
└── TCP/443 → HTTPS
```

This allows a single firewall rule to reference both ports.

---

## Updating the Firewall Rule

The firewall rule from the previous lab originally contained hard-coded values:

```text
Action:       Block
Protocol:     TCP
Source:       192.168.50.104
Destination:  Any
Port:         80
```

The rule was modified to use the new aliases:

```text
Action:       Block
Protocol:     TCP
Source:       KALI_HOST
Destination:  Any
Port:         WEB_PORTS
```

The description was changed to:

```text
LAB12 - Block Kali Web Traffic Using Aliases
```

The resulting policy can be represented as:

```text
IF
    Source = KALI_HOST
AND Protocol = TCP
AND Destination Port = WEB_PORTS
THEN
    BLOCK
```

Because:

```text
KALI_HOST = 192.168.50.104
```

and:

```text
WEB_PORTS = 80, 443
```

the firewall effectively evaluates:

```text
192.168.50.104 → TCP/80  → BLOCK
192.168.50.104 → TCP/443 → BLOCK
```

---

## Testing HTTP From Kali

The first test attempted a TCP connection from Kali to port 80:

```bash
nc -nvz -w 5 104.20.23.154 80
```

The connection timed out.

This demonstrated that TCP/80 was successfully matched through the `WEB_PORTS` alias and blocked by the firewall.

---

## Testing HTTPS From Kali

A second test attempted a TCP connection to port 443:

```bash
nc -nvz -w 5 104.20.23.154 443
```

This connection also timed out.

This was significant because the previous firewall rule only blocked TCP/80.

After replacing the single port with:

```text
WEB_PORTS
```

the same firewall rule could match both:

```text
80
443
```

without requiring a second firewall rule.

---

## Firewall Log Verification

The pfSense firewall logs were inspected after generating the test traffic.

The logs identified:

```text
LAB12 - Block Kali Web Traffic Using Aliases
```

as the firewall rule responsible for handling the blocked Kali traffic.

Entries were observed for both destination ports:

```text
TCP/80
TCP/443
```

This provided firewall-side evidence that the alias-based rule was functioning correctly.

### Evidence

<img width="1024" height="768" alt="5 1-alias-web-ports-blocked" src="https://github.com/user-attachments/assets/8382c69e-7fd1-443a-ac2e-338891b9202c" />

---

## Windows 11 Control Test

The firewall rule remained enabled while Windows 11 attempted to connect to the same external destination over TCP/80.

Command:

```powershell
Test-NetConnection 104.20.23.154 -Port 80
```

Result:

```text
TcpTestSucceeded : True
```

### Evidence
<img width="1024" height="768" alt="5 2-win11-http-allowed-controled" src="https://github.com/user-attachments/assets/e6954dc6-7458-40bd-8853-77ec9a90a4fb" />

  
Windows 11 remained allowed because its source address was:

```text
192.168.50.101
```

while:

```text
KALI_HOST = 192.168.50.104
```

Therefore Windows 11 did not satisfy the source condition of the Lab 12 rule.

pfSense continued evaluating subsequent firewall rules until the traffic matched the existing default allow policy.

---

## Alias-Based Rule Logic

The firewall rule can be viewed as several conditions working together:

```text
Source = KALI_HOST
AND
Protocol = TCP
AND
Destination Port = WEB_PORTS
```

All required conditions must match for this particular block rule to apply.

Examples:

| Traffic | Matches LAB12? | Result |
|---|---|---|
| Kali `.104` → TCP/80 | Yes | Blocked |
| Kali `.104` → TCP/443 | Yes | Blocked |
| Windows `.101` → TCP/80 | No | Continues to later rules |
| Windows `.101` → TCP/443 | No | Continues to later rules |

Aliases do not remove firewall rule specificity.

They provide a more manageable way to define the values used by that policy.

---

## Centralized Management

One of the primary advantages of aliases is centralized configuration.

Assume Kali's IP address changed from:

```text
192.168.50.104
```

to:

```text
192.168.50.150
```

Without an alias, every firewall rule containing the old address might need to be changed.

With an alias, only this object needs to be updated:

```text
KALI_HOST
192.168.50.104
        ↓
192.168.50.150
```

Every firewall rule referencing:

```text
KALI_HOST
```

can continue using the same logical object.

This becomes increasingly useful as the number of firewall rules grows.

---

## Scaling Port Policies

Port aliases provide the same management benefit.

Currently:

```text
WEB_PORTS = 80, 443
```

Suppose TCP/8080 needed to become part of the same policy.

The alias could be updated to:

```text
WEB_PORTS = 80, 443, 8080
```

Because the existing firewall rule already references `WEB_PORTS`, a separate firewall rule would not necessarily be required just to include TCP/8080 in that same policy.

Conceptually:

```text
LAB12 Firewall Rule
        │
        └── WEB_PORTS
             ├── 80
             ├── 443
             └── 8080
```

This separates the firewall policy from the individual values used to define groups of systems or services.

---

## Hard-Coded Rules vs Alias-Based Rules

### Hard-Coded Policy

```text
BLOCK TCP
Source: 192.168.50.104
Destination Port: 80
```

The administrator must understand what the IP and port represent.

### Alias-Based Policy

```text
BLOCK TCP
Source: KALI_HOST
Destination Port: WEB_PORTS
```

The purpose of the rule becomes much easier to identify.

Instead of seeing only numbers, the administrator sees logical objects representing systems and services.

---

## Security and Administrative Relevance

Aliases are particularly useful in larger firewall environments.

Examples could include:

```text
DOMAIN_CONTROLLERS
WEB_SERVERS
DATABASE_SERVERS
ADMIN_WORKSTATIONS
SECURITY_TOOLS
DNS_SERVERS
WEB_PORTS
MANAGEMENT_PORTS
```

A policy could then be expressed logically as:

```text
ADMIN_WORKSTATIONS
        ↓
MANAGEMENT_PORTS
        ↓
SERVERS
```

rather than requiring administrators to repeatedly enter individual addresses and ports.

This can make firewall policies easier to review, troubleshoot, and maintain.

---

## Key Concepts Demonstrated

### Host Aliases

Host aliases allow meaningful names to represent individual IP addresses or groups of hosts.

Example:

```text
KALI_HOST = 192.168.50.104
```

### Port Aliases

Port aliases allow multiple service ports to be grouped under one logical name.

Example:

```text
WEB_PORTS = 80, 443
```

### Reusability

The same alias can be referenced by multiple firewall rules.

### Centralized Administration

Updating an alias can update the underlying object referenced by multiple policies without manually rewriting each rule.

### Readability

A rule containing:

```text
KALI_HOST → WEB_PORTS
```

communicates its purpose more clearly than:

```text
192.168.50.104 → 80,443
```

### Rule Specificity

Aliases do not change how firewall conditions are evaluated.

The source, protocol, destination, and port conditions must still match the configured policy.

### Log-Based Validation

Firewall logs were used to verify that the expected rule handled the blocked traffic rather than relying solely on endpoint connection failures.

---

## Troubleshooting Lessons

This lab reinforced an important troubleshooting principle:

> A failed connection is evidence of a symptom, while firewall logs can provide evidence of the enforcement point responsible for that failure.

The workflow used was:

```text
1. Establish connectivity
        ↓
2. Apply firewall policy
        ↓
3. Generate test traffic
        ↓
4. Observe endpoint behavior
        ↓
5. Inspect firewall logs
        ↓
6. Compare against a control host
```

This provides stronger evidence than assuming a firewall caused a connection failure simply because a timeout occurred.

---

## Lab Result

**PASS**

The lab successfully demonstrated:

- Creation of a pfSense host alias
- Creation of a pfSense port alias
- Grouping TCP/80 and TCP/443 under `WEB_PORTS`
- Replacing a hard-coded IP with `KALI_HOST`
- Replacing a hard-coded port with `WEB_PORTS`
- Blocking Kali HTTP traffic
- Blocking Kali HTTPS traffic
- Allowing Windows 11 as a control host
- Verifying firewall enforcement through pfSense logs
- Understanding centralized alias management
- Understanding how aliases improve firewall scalability and readability

---

## Conclusion

Lab 12 demonstrated how pfSense aliases can make firewall policies more scalable and maintainable.

The host alias:

```text
KALI_HOST
```

abstracted Kali's IP address, while:

```text
WEB_PORTS
```

grouped HTTP and HTTPS under a reusable service object.

The firewall rule could therefore be expressed logically as:

```text
BLOCK KALI_HOST → WEB_PORTS
```

rather than relying entirely on hard-coded IP addresses and port numbers.

As firewall environments grow, this approach reduces repetitive configuration and makes policies easier to understand, update, troubleshoot, and audit.
