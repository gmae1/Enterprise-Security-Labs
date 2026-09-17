# Lab 09 — TCP Port-Specific Firewall Rules

## Objective

The objective of this lab was to configure and test a port-specific pfSense firewall rule.

Rather than blocking an entire protocol such as ICMP, this lab demonstrated how firewall rules can selectively block a specific TCP service based on its destination port.

A custom rule was created to:

```text
Block TCP traffic originating from Win11Pro
when the destination port is TCP/443 (HTTPS).
```

The rule was tested against TCP/443 and compared with TCP/80 to verify that only the intended destination port was blocked.

---

## Lab Environment

| System | Role | IP Address |
|---|---|---|
| pfSense | Router / Firewall / Default Gateway | 192.168.50.1 |
| Win11Pro | Windows Client | 192.168.50.101 |
| DC-01 | Active Directory / DNS Server | 192.168.50.10 |
| Linux Server | Ubuntu Server | 192.168.50.102 |
| Kali Linux | Security Workstation | 192.168.50.104 |

**LAN Network:** `192.168.50.0/24`  
**Virtual Network:** `SEC-LAB`

---

## Initial TCP/443 Baseline

Before creating the firewall rule, TCP connectivity from Win11Pro to an external HTTPS service was tested.

The following PowerShell command was executed:

```powershell
Test-NetConnection example.com -Port 443
```

The result was:

```text
TcpTestSucceeded : True
```

This established that Win11Pro could successfully initiate TCP connections to destination port 443 before the firewall policy was applied.

---

## Understanding the Traffic Path

An important networking concept was reviewed before creating the firewall rule.

Win11Pro and DC-01 are both members of:

```text
192.168.50.0/24
```

Their addresses are:

```text
Win11Pro: 192.168.50.101
DC-01:    192.168.50.10
```

Because both systems are on the same subnet, Win11Pro does not need to send traffic destined for DC-01 through its default gateway.

Instead, Win11Pro can use ARP to determine DC-01's MAC address and communicate directly at Layer 2.

The ARP table was inspected using:

```powershell
arp -a
```

An entry for:

```text
192.168.50.10
```

was present.

The traffic path therefore resembles:

```text
Win11Pro
192.168.50.101
      |
      | Layer 2 communication
      | using DC-01 MAC address
      v
DC-01
192.168.50.10

pfSense
192.168.50.1
NOT IN PATH
```

This demonstrates an important firewall architecture principle:

> A firewall can only enforce policy on traffic that actually passes through it.

---

## Routed Traffic

Traffic destined for a network outside:

```text
192.168.50.0/24
```

requires Win11Pro to use its default gateway:

```text
192.168.50.1
```

The traffic path becomes:

```text
Win11Pro
192.168.50.101
      |
      v
pfSense
192.168.50.1
      |
      v
External Network
      |
      v
Internet
```

Because pfSense is now in the traffic path, its LAN firewall policy can inspect and control the traffic.

For this reason, an external TCP/443 connection was selected for the firewall test.

---

## Custom TCP/443 Firewall Rule

A custom rule was created under:

```text
Firewall → Rules → LAN
```

The rule was configured as:

```text
Action:            Block
Protocol:          TCP
Source:            192.168.50.101
Destination:       Any
Destination Port:  443 (HTTPS)
Logging:           Enabled
Description:       LAB09 - Block Win11 HTTPS
```

The rule was positioned above:

```text
Default allow LAN to any rule
```

This ensured that matching traffic would encounter the custom block rule before reaching the broader allow policy.

---

## Understanding the Rule

The rule does **not** mean:

```text
Block all TCP traffic from Win11Pro.
```

Instead, it means:

```text
IF:

Protocol = TCP
AND
Source = 192.168.50.101
AND
Destination Port = 443

THEN:

BLOCK
```

Conceptually:

```text
Win11Pro
192.168.50.101
      |
      | TCP
      | Destination Port 443
      v
pfSense
      |
      | LAB09 - Block Win11 HTTPS
      v
     MATCH
      |
      X
    BLOCK
```

---

## TCP/443 Block Test

After saving and applying the firewall rule, the original PowerShell test was repeated:

```powershell
Test-NetConnection example.com -Port 443
```

The result changed to:

```text
TcpTestSucceeded : False
```

This demonstrated that TCP/443 connectivity was no longer succeeding after the firewall policy was enabled.

However, the failed connection alone did not prove that pfSense caused the failure.

Firewall logs were therefore examined.

---

## Firewall Log Investigation

The pfSense firewall logs showed traffic similar to:

```text
192.168.50.101:51585 → 23.53.11.142:443
```

The important components were:

```text
192.168.50.101 = Win11Pro source IP
51585          = Ephemeral source port
23.53.11.142   = External destination IP observed during testing
443            = Destination TCP port
```

pfSense reported that this TCP traffic was blocked.

### Evidence
<img width="1024" height="768" alt="2 1-blocked-tcp-443-firewall-log" src="https://github.com/user-attachments/assets/24ddcb00-f79a-4836-8471-4d19545565a4" />


This provided direct evidence that the custom firewall rule was responsible for blocking TCP/443 traffic.

---

## Ephemeral Source Ports

The firewall logs showed Win11Pro using addresses similar to:

```text
192.168.50.101:51585
```

The value following the colon represents the TCP source port.

For example:

```text
192.168.50.101 : 51585
       IP          Port
```

When a client initiates a TCP connection, the operating system typically selects a temporary high-numbered source port.

The communication may therefore resemble:

```text
Client:
192.168.50.101:51585

        ↓

Server:
External-IP:443
```

The server's response can then be associated with the client's specific connection.

This reinforced the distinction between:

- Source IP
- Source port
- Destination IP
- Destination port

---

## Control Test

To verify that the firewall rule caused the TCP/443 failure, the custom rule was disabled.

The original test was repeated:

```powershell
Test-NetConnection example.com -Port 443
```

The result returned to:

```text
TcpTestSucceeded : True
```

The comparison was:

```text
Rule Enabled
TCP/443 → False

Rule Disabled
TCP/443 → True
```

Only the firewall rule state was intentionally changed between the tests.

This provided additional evidence that the custom rule was responsible for the failed TCP/443 connection.

---

## Testing TCP/80

The next objective was to determine whether the custom rule blocked all TCP traffic or only TCP/443.

With the TCP/443 firewall rule enabled, Win11Pro executed:

```powershell
Test-NetConnection example.com -Port 80
```

The result was:

```text
TcpTestSucceeded : True
```

### Evidence
<img width="1024" height="768" alt="2 2-tcp-80-allowed-control-test" src="https://github.com/user-attachments/assets/d7d72a60-1853-447d-9522-fd7dbfbf74ad" />



This demonstrated that TCP traffic was still permitted when the destination port did not match 443.

---

## Port-Specific Rule Processing

When Win11Pro attempts TCP/443:

```text
Win11Pro → TCP/443
        |
        v
LAB09 - Block Win11 HTTPS
        |
      MATCH
        |
        v
      BLOCK
```

When Win11Pro attempts TCP/80:

```text
Win11Pro → TCP/80
        |
        v
LAB09 - Block Win11 HTTPS
        |
   Destination = 443?
        |
       NO
        |
        v
Continue evaluating rules
        |
        v
Default allow LAN to any
        |
      MATCH
        |
        v
      ALLOW
```

The same concept would apply to another TCP destination port such as TCP/22.

Because TCP/22 does not match the custom TCP/443 rule, pfSense would continue evaluating subsequent firewall rules.

Whether the connection ultimately succeeds would then depend on later firewall policy, routing, and whether the destination service is available.

---

## TCP/443 vs TCP/80 Results

| Test | Custom Rule | Result |
|---|---|---|
| TCP/443 baseline | Disabled / not applied | Allowed |
| TCP/443 | Enabled | Blocked |
| TCP/443 control | Disabled | Allowed |
| TCP/80 | Enabled | Allowed |

The results demonstrate that the firewall policy was selective rather than blocking all TCP communication.

---

## Rule Order

The custom firewall rule was positioned above:

```text
Default allow LAN to any rule
```

pfSense evaluates applicable interface rules from top to bottom.

For TCP/443 traffic originating from Win11Pro:

```text
1. LAB09 - Block Win11 HTTPS
              |
            MATCH
              |
           BLOCK
              |
             STOP

2. Default allow LAN to any
       Not reached
```

For TCP/80 traffic:

```text
1. LAB09 - Block Win11 HTTPS
              |
          NO MATCH
              |
              v

2. Default allow LAN to any
              |
            MATCH
              |
            ALLOW
```

This demonstrates how rule specificity and rule order work together.

---

## Firewall Placement and Network Segmentation

This lab also demonstrated why firewall placement is important.

Systems on the same Layer 2 subnet can normally communicate directly without sending their traffic through the default gateway.

For example:

```text
192.168.50.101 → 192.168.50.10
```

can remain within:

```text
192.168.50.0/24
```

However, if systems were separated into different networks, such as:

```text
USER NETWORK
192.168.50.0/24

SERVER NETWORK
192.168.60.0/24
```

communication between the networks would require routing.

If pfSense performed that routing, firewall policy could be enforced between the networks:

```text
User Network
     |
     v
  pfSense
  Firewall
     |
     v
Server Network
```

This is one of the fundamental concepts behind network segmentation and inter-network firewall policy.

---

## Security Relevance

Port-specific firewall rules provide significantly more control than broad protocol-level blocking.

An administrator could create policies such as:

```text
ALLOW workstation → DNS server → TCP/UDP 53

ALLOW administrator → Linux server → TCP 22

ALLOW users → Internet → TCP 443

BLOCK unauthorized systems → management services
```

This follows the principle of restricting communication to the services that systems actually require.

Firewall logs can then provide evidence showing which connections were allowed or denied.

---

## Key Takeaways

1. Firewall rules can filter traffic based on destination port.
2. TCP/443 represents HTTPS when HTTPS is using TCP.
3. TCP/80 is commonly associated with HTTP.
4. TCP/22 is commonly associated with SSH.
5. Blocking TCP/443 does not automatically block all TCP traffic.
6. A firewall rule only applies when its configured conditions match the traffic.
7. Rule order determines which matching policy is applied first.
8. Firewall logs can confirm whether pfSense blocked a connection.
9. Client systems commonly use temporary ephemeral source ports.
10. Source ports and destination ports serve different purposes.
11. A failed connection is a symptom; firewall logs can help identify the cause.
12. Same-subnet traffic normally does not need to traverse the default gateway.
13. pfSense can only filter traffic that actually passes through it.
14. Network segmentation can place a firewall in the path between different security zones.
15. Control tests help validate whether a configuration change caused an observed network behavior.

---

## Lab Result

**PASS**

A port-specific pfSense firewall policy was successfully configured and tested.

The lab demonstrated:

- Blocking TCP/443 from a specific host
- Verifying blocked traffic using firewall logs
- Understanding ephemeral source ports
- Performing a control test
- Confirming TCP/80 remained permitted
- Applying first-match firewall rule processing
- Understanding why same-subnet traffic can bypass the router/firewall
- Connecting firewall policy with network segmentation concepts

The results confirmed that pfSense can selectively restrict access to specific TCP services while allowing other TCP communication to continue.
