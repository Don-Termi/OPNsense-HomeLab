# DMZ Firewall Rules

## Overview

The DMZ (`192.168.20.0/24`) hosts the security and web infrastructure of the lab.

The main systems located in this network are:

| System               |     IP Address | Purpose                                  |
| -------------------- | -------------: | ---------------------------------------- |
| OPNsense DMZ Gateway | `192.168.20.1` | Default gateway and firewall             |
| Wazuh Manager        | `192.168.20.2` | SIEM and centralized security monitoring |
| Apache + ModSecurity | `192.168.20.3` | Web server protected by WAF              |

The DMZ is intentionally isolated from the Management and Employees networks. Access between zones is controlled through explicit OPNsense firewall rules.

The main objective is to apply a **least-privilege model** while still allowing the services required by the laboratory to operate correctly.

---

## Security Objectives

The DMZ policy follows these principles:

* Prevent unauthorized access from Employees to internal management services.
* Allow legitimate access to the web services hosted in the DMZ.
* Allow the Management workstation to administer the required DMZ systems.
* Restrict outbound traffic from DMZ systems where possible.
* Allow Wazuh agent communication where required.
* Forward OPNsense firewall events to the Wazuh Manager.
* Prevent the DMZ from becoming a trusted transit network between internal segments.

---

## DMZ Addressing

```text
Network:       192.168.20.0/24
Gateway:       192.168.20.1

Wazuh Manager: 192.168.20.2
Apache/WAF:     192.168.20.3
```

The DMZ gateway is provided by the OPNsense firewall.

---

## Main Traffic Policies

### Management → DMZ

Administrative traffic from the Management network is permitted only when required for administration and monitoring.

The Management workstation is:

```text
192.168.10.2
```

Access to management services is controlled using the `Management_Ports` alias where appropriate.

This allows the administrator to manage the security infrastructure without providing unrestricted access to the entire DMZ.

---

### Employees → DMZ

Employees are not considered a trusted administrative network.

Access from Employees to the DMZ is therefore restricted to explicitly required services.

For example, legitimate web access can be permitted to the Apache server:

```text
Employees
    |
    | HTTP / HTTPS
    v
192.168.20.3
Apache + ModSecurity
```

Direct access to management services such as SSH or the Wazuh administration interfaces is blocked.

This prevents an employee workstation from directly reaching infrastructure-management services.

---

### DMZ → Internal Networks

DMZ systems should not be able to freely initiate connections toward internal networks.

This reduces the impact of a potential compromise of a DMZ server.

For example, compromising the Apache server should not automatically provide network-level access to the Management network.

The firewall therefore treats the DMZ as a separate security zone rather than as an extension of the internal network.

---

### DMZ → Internet

Outbound Internet access from the DMZ is restricted according to the requirements of the individual services.

The web server may require outbound connectivity for legitimate operational purposes, while unnecessary protocols and destinations should remain blocked.

This reduces the potential for:

* Command-and-control communication.
* Unauthorized data exfiltration.
* Malware downloads.
* Unnecessary external exposure.

---

## Wazuh Communication

The Wazuh Manager is located inside the DMZ:

```text
Wazuh Manager
192.168.20.2
```

Wazuh agents communicate with the manager using the Wazuh agent communication ports.

The firewall therefore contains explicit rules where required for:

```text
TCP 1514
TCP 1515
```

This follows the same least-privilege principle used throughout the rest of the firewall configuration.

---

## OPNsense → Wazuh Syslog

OPNsense forwards firewall events to the Wazuh Manager using Syslog.

```text
OPNsense
192.168.20.1
      |
      | UDP 5514
      v
Wazuh Manager
192.168.20.2
```

The forwarded events include OPNsense `filterlog` entries.

These events provide visibility into firewall activity such as:

* Blocked connections.
* Allowed connections when logging is enabled.
* Source and destination addresses.
* Source and destination ports.
* Protocol information.
* Firewall rule activity.

The Wazuh Manager processes these logs and can generate security alerts using custom detection rules.

---

## Apache + ModSecurity

The Apache server is located at:

```text
192.168.20.3
```

It provides the web-service component of the DMZ.

ModSecurity is used as a Web Application Firewall (WAF), providing an additional security layer between network-level firewall controls and the web application.

The architecture therefore provides multiple defensive layers:

```text
Internet
   |
   v
OPNsense Firewall
   |
   v
DMZ
   |
   v
Apache
   |
   v
ModSecurity WAF
   |
   v
Web Application
```

This follows a defence-in-depth approach.

---

## Logging

Firewall rules related to security-sensitive traffic are configured with logging where appropriate.

The resulting `filterlog` events are forwarded to Wazuh through Syslog.

This provides centralized visibility instead of requiring an administrator to inspect OPNsense logs manually.

The resulting monitoring chain is:

```text
Network traffic
      |
      v
OPNsense PF
      |
      v
filterlog
      |
      v
Syslog UDP/5514
      |
      v
rsyslog
      |
      v
Wazuh Manager
      |
      v
Wazuh Analysis Engine
      |
      v
Security Alert
```

---

## Security Rationale

The DMZ configuration demonstrates several common network-security principles.

### Network Segmentation

The DMZ is isolated from both Management and Employees.

### Least Privilege

Only explicitly required traffic is allowed between security zones.

### Defence in Depth

The web server is protected by both network-level firewall controls and ModSecurity.

### Centralized Monitoring

Firewall events are forwarded to Wazuh for centralized analysis.

### Reduced Blast Radius

If a DMZ server is compromised, segmentation limits the attacker's ability to move directly into internal networks.

---

## Validation

The DMZ policy was validated by testing connectivity from the different security zones.

Tests included:

* Management access to required DMZ services.
* Employee access to permitted web services.
* Employee attempts to access restricted management services.
* Firewall logging of blocked traffic.
* Wazuh reception of OPNsense `filterlog` events.
* Wazuh agent communication with the Manager.

The validation results demonstrate that the DMZ behaves as an isolated security zone rather than a flat internal network.

---

## Summary

The DMZ firewall policy provides controlled access to the laboratory's security and web infrastructure while maintaining separation from the internal Management and Employees networks.

The combination of:

* OPNsense stateful filtering,
* explicit inter-zone policies,
* Apache + ModSecurity,
* Wazuh monitoring,
* Syslog forwarding,
* and centralized security detection

creates a layered network-security architecture suitable for demonstrating practical firewall and defensive-security concepts.
