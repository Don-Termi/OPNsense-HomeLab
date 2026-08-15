# Firewall Policy

## Overview

The OPNsense firewall is responsible for enforcing communication policies between the three security zones in the laboratory:

* Management
* Employees
* DMZ

The policy follows a **default-deny and least-privilege approach** wherever practical.

Traffic is explicitly allowed when required by the services or operational requirements of the environment. Unnecessary communication between security zones is blocked.

---

## Security Zones

| Zone       | Network           | Security Role                   |
| ---------- | ----------------- | ------------------------------- |
| Management | `192.168.10.0/24` | Administrative systems          |
| DMZ        | `192.168.20.0/24` | Security and web infrastructure |
| Employees  | `192.168.30.0/24` | User workstations               |

The zones are routed through OPNsense, making the firewall the central enforcement point for inter-zone communication.

---

# Policy Model

The high-level policy can be summarized as follows:

| Source     | Destination      | Policy                 | Purpose                             |
| ---------- | ---------------- | ---------------------- | ----------------------------------- |
| Management | Internet         | Allow                  | Administration and troubleshooting  |
| Management | DMZ              | Restricted Allow       | Administration of infrastructure    |
| Management | Employees        | Restricted             | Administrative access when required |
| Employees  | Internet         | Allow                  | Normal user connectivity            |
| Employees  | Management       | **Block**              | Prevent lateral movement            |
| Employees  | DMZ              | Restricted Allow       | Required application/web services   |
| DMZ        | Internet         | Restricted Allow       | Required server connectivity        |
| DMZ        | Management       | **Restricted / Block** | Protect internal administration     |
| DMZ        | Employees        | **Restricted / Block** | Prevent lateral movement            |
| Any        | PortScan sources | **Block**              | Automated abuse response            |

The exact rules are implemented per interface and floating rules in OPNsense.

---

# Management Policy

The Management network is the administrative zone.

The primary workstation is:

```text
192.168.10.2
```

Management access is intentionally more permissive than Employee access, but unrestricted access is avoided where possible.

Administrative access is controlled using aliases for infrastructure networks and required service ports.

The goal is:

```text
Administrator
     |
     v
Required management service
     |
     v
Infrastructure
```

rather than:

```text
Administrator
     |
     v
Any service
     |
     v
Any network
```

This reduces unnecessary exposure while retaining operational flexibility.

---

# Employees Policy

Employees are treated as an untrusted internal network.

The Employees network is:

```text
192.168.30.0/24
```

The primary security objective is to prevent a compromised employee workstation from reaching administrative infrastructure.

The most important restriction is:

```text
Employees
    |
    X
    |
Management
```

This prevents direct access to systems such as the administrative workstation and management interfaces.

---

## Internet Access

Employees require Internet access for normal activities such as:

* Web browsing.
* Software updates.
* Cloud services.
* External applications.

The firewall therefore permits required outbound traffic while applying additional restrictions to unnecessary protocols.

ICMP is treated separately from normal application traffic.

This demonstrates that blocking ICMP does not require blocking general Internet connectivity.

---

# DMZ Policy

The DMZ contains systems that provide security and application services.

```text
192.168.20.0/24
```

The primary systems are:

```text
Wazuh Manager
192.168.20.2

Apache + ModSecurity
192.168.20.3
```

The DMZ is considered less trusted than the Management network.

As a result, DMZ systems are not given unrestricted access to internal networks.

---

## Employees → DMZ

Employees can access services that are explicitly intended for users.

For example, web traffic to the Apache server can be permitted:

```text
Employees
    |
    | HTTP / HTTPS
    v
Apache + ModSecurity
192.168.20.3
```

Administrative services remain restricted.

This prevents normal user systems from directly interacting with infrastructure-management interfaces.

---

# Management → DMZ

Administrative access from Management to the DMZ is allowed where required.

This enables the administrator to:

* Manage Wazuh.
* Manage the web infrastructure.
* Perform troubleshooting.
* Validate firewall and security controls.

Access is controlled using specific aliases and service ports rather than relying on unrestricted access wherever practical.

---

# DMZ → Internal Networks

Connections initiated from the DMZ toward internal networks are restricted.

This is particularly important for the Apache server.

If the web server were compromised, the attacker should not automatically gain network-level access to the Management network.

The firewall therefore acts as a lateral-movement control.

```text
Compromised DMZ server
          |
          X
          |
   Management network
```

---

# ICMP Policy

ICMP is controlled independently according to the role of each network.

### Management

ICMP to the Internet is permitted.

This allows administrators to perform basic connectivity and troubleshooting tests.

### Employees

ICMP to the Internet is blocked or restricted.

Normal Internet applications remain available through their required protocols.

### DMZ

ICMP is allowed only where required for troubleshooting or operational purposes.

This provides a practical balance between security and diagnostic capability.

---

# DNS Policy

DNS is required for normal network operation.

The environment uses OPNsense's DNS infrastructure and DHCP configuration to provide name-resolution services to internal clients.

DNS traffic is therefore explicitly considered when defining outbound policies.

Blocking all Internet traffic without accounting for DNS would cause apparently unrelated applications such as browsers and cloud services to fail.

This was an important consideration during the firewall implementation.

---

# DHCP Policy

The Employees network uses DHCP provided by OPNsense.

The Windows workstation receives its address dynamically from the configured DHCP pool.

This separates client addressing from infrastructure addressing.

Infrastructure systems such as:

```text
Linux Mint       192.168.10.2
Wazuh Manager    192.168.20.2
Apache/WAF        192.168.20.3
```

use predictable addresses, while the Employee workstation receives an address dynamically.

---

# Floating Rules

Floating rules are used for security controls that should operate independently of a single interface policy.

The most important example is the PortScan protection mechanism.

The architecture uses two complementary rules:

```text
PortScan Detection
       |
       v
PortScan Table
       |
       v
Floating Block Rule
```

This allows the firewall to automatically respond to excessive connection creation.

The detailed implementation is documented in:

`docs/portscan-detection.md`

---

# Logging Policy

Security-relevant firewall rules have logging enabled where appropriate.

OPNsense generates `filterlog` events for matching firewall activity.

These events are forwarded to the Wazuh Manager:

```text
OPNsense
    |
    | Syslog UDP/5514
    v
Wazuh Manager
```

This allows blocked traffic to be monitored centrally.

Logging is particularly important for:

* Blocked inter-zone traffic.
* Port-scan activity.
* Suspicious connection attempts.
* Security-policy violations.

---

# PortScan Blocking

The firewall implements automated blocking using a PF overload table.

The table is:

```text
PortScan
```

When the connection-rate threshold is exceeded, the source IP can be added to the table.

A separate floating rule then blocks traffic originating from addresses in that table.

Conceptually:

```text
Excessive connections
        |
        v
Connection threshold
        |
        v
PortScan table
        |
        v
Automatic block
        |
        v
filterlog
        |
        v
Wazuh
```

This provides a simple automated response mechanism against connection-based reconnaissance.

---

# Rule Ordering

Rule ordering is important in OPNsense because traffic can match different rules.

The configuration therefore uses explicit ordering and the `Quick` option where appropriate.

The PortScan blocking rule uses `Quick` so that a source already present in the block table cannot continue through subsequent pass rules.

Conceptually:

```text
Packet
  |
  v
Is source in PortScan?
  |
 YES ---------> BLOCK
  |
 NO
  |
  v
Continue firewall evaluation
```

This prevents a later permissive rule from unintentionally allowing traffic from an already blocked source.

---

# Logging and SIEM Integration

The firewall policy is not considered complete simply because traffic is allowed or blocked.

Security events must also be observable.

The project therefore integrates OPNsense with Wazuh.

The complete chain is:

```text
Firewall Decision
       |
       v
filterlog
       |
       v
Syslog
UDP 5514
       |
       v
rsyslog
       |
       v
Wazuh Manager
       |
       v
Detection Rule
```

The custom Wazuh detection rule created for the project uses ID:

```text
100100
```

This demonstrates how firewall enforcement can be combined with centralized security monitoring.

---

# Least-Privilege Principles

The firewall policy follows several least-privilege principles.

### Explicit Access

Services are allowed because they are required, not simply because the destination is trusted.

### Network Separation

Management, Employees, and DMZ systems have different security policies.

### Restricted Administration

Administrative services are not exposed to normal employee workstations.

### Limited DMZ Trust

DMZ systems are not automatically trusted by internal networks.

### Controlled Internet Access

Different zones have different outbound requirements.

### Centralized Logging

Security-relevant activity is logged and monitored.

---

# Security Objectives

The policy is designed to achieve the following objectives:

* Prevent lateral movement from Employees to Management.
* Isolate DMZ infrastructure.
* Protect administrative services.
* Maintain required Internet connectivity.
* Provide controlled access to web services.
* Detect excessive connection attempts.
* Automatically block abusive sources.
* Centralize firewall security events in Wazuh.
* Provide sufficient logging for investigation.

---

# Policy Validation

The policy was validated through practical connectivity tests.

Examples include:

```text
Management → Internet
Employees → Internet
Management → DMZ
Employees → Management
Employees → DMZ
DMZ → Internet
```

Blocked connections were verified through OPNsense firewall logs.

The PortScan mechanism was additionally validated by generating a high rate of TCP connection attempts and confirming that the source IP was added to the `PortScan` table.

---

# Security Trade-offs

A firewall policy always involves a balance between security and usability.

During development, several examples demonstrated this directly.

For example, excessively restrictive rules initially prevented legitimate Internet access from internal clients.

Similarly, extremely low PortScan thresholds caused legitimate browser traffic to trigger the overload table.

These issues were resolved by tuning the policies according to the actual requirements of each network.

This highlights an important security-engineering principle:

> A secure firewall is not simply the firewall that blocks the most traffic. It is the firewall that enforces the intended security policy while preserving legitimate functionality.

---

# Summary

The OPNsense policy implements a segmented, least-privilege network model.

The most important controls are:

* Management isolation.
* Employee-to-Management blocking.
* DMZ isolation.
* Restricted inter-zone communication.
* Zone-specific ICMP policies.
* Stateful firewall filtering.
* Automated PortScan blocking.
* Security event logging.
* Wazuh SIEM integration.

Together, these controls provide a layered network-security architecture suitable for a practical cybersecurity laboratory and portfolio project.
