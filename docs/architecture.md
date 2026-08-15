# Architecture

## Overview

This project implements a segmented network security laboratory built around OPNsense and Wazuh.

The environment was designed to reproduce several security concepts commonly found in enterprise networks:

* Network segmentation
* Stateful firewall enforcement
* Least-privilege access control
* DMZ isolation
* Centralized security monitoring
* Firewall event logging
* Automated detection and blocking
* Security event correlation

The environment is fully virtualized and consists of an OPNsense firewall, a Management workstation, an Employees workstation, a Wazuh SIEM, and a web server protected by ModSecurity.

---

## Network Topology

The environment is divided into three internal security zones:

| Zone       | Network           | Gateway        | Main Purpose                    |
| ---------- | ----------------- | -------------- | ------------------------------- |
| Management | `192.168.10.0/24` | `192.168.10.1` | Administrative access           |
| DMZ        | `192.168.20.0/24` | `192.168.20.1` | Security and web infrastructure |
| Employees  | `192.168.30.0/24` | `192.168.30.1` | User workstations               |

The OPNsense firewall provides routing, stateful filtering, DHCP services for the Employees network, and centralized firewall logging.

```text
                         INTERNET
                             |
                             |
                     +---------------+
                     |   OPNsense    |
                     |    Firewall   |
                     +-------+-------+
                             |
              +--------------+--------------+
              |              |              |
              |              |              |
              v              v              v
       MANAGEMENT          DMZ          EMPLOYEES
      192.168.10.0/24  192.168.20.0/24  192.168.30.0/24
              |              |              |
              |              |              |
        Linux Mint       Wazuh Manager   Windows 10
        192.168.10.2    192.168.20.2       DHCP
                             |
                             |
                       Apache + ModSecurity
                          192.168.20.3
```

The complete visual architecture is represented in the project's network architecture diagram.

---

## Security Zones

### Management

The Management network is reserved for administrative activities.

The primary workstation is:

```text
Linux Mint
192.168.10.2
```

This network is considered trusted for administration, but it is still subject to explicit firewall policies.

The Management network is not intended to provide unrestricted access to every service in the environment. Access is controlled according to the required administrative functions.

---

### DMZ

The DMZ contains infrastructure that needs to communicate across different security boundaries.

The main systems are:

```text
Wazuh Manager
192.168.20.2

Apache + ModSecurity
192.168.20.3
```

The DMZ is treated as a separate trust zone.

This prevents a compromise of a web-facing system from automatically providing unrestricted access to internal administrative resources.

---

### Employees

The Employees network represents normal user workstations.

```text
Network:
192.168.30.0/24
```

The Windows workstation receives its address dynamically from the OPNsense DHCP service.

Employees are intentionally subject to stricter network policies than the Management network.

In particular, direct access to Management resources is blocked.

---

## Addressing

The core addressing scheme is:

| Device / Network      |        Address |
| --------------------- | -------------: |
| OPNsense Management   | `192.168.10.1` |
| Linux Mint Management | `192.168.10.2` |
| OPNsense DMZ          | `192.168.20.1` |
| Wazuh Manager         | `192.168.20.2` |
| Apache + ModSecurity  | `192.168.20.3` |
| OPNsense Employees    | `192.168.30.1` |
| Windows Employees     |           DHCP |

Static addresses are used for infrastructure and administrative systems where predictable addressing is useful.

The Employees workstation uses DHCP to simulate a more realistic user environment.

---

## OPNsense Responsibilities

OPNsense acts as the central network-security enforcement point.

Its responsibilities include:

* Routing between security zones.
* Stateful firewall filtering.
* Network segmentation.
* DHCP for the Employees network.
* NAT for Internet connectivity.
* ICMP policy enforcement.
* Port-scan detection.
* Automatic source blocking.
* Firewall event logging.
* Syslog forwarding.

The firewall therefore provides both connectivity and security enforcement.

---

## Stateful Firewall Architecture

The firewall uses stateful packet filtering.

Rather than evaluating every packet independently, PF maintains connection state.

This allows the firewall to distinguish between:

* New connections.
* Existing connections.
* Return traffic belonging to an established session.
* Excessive connection creation.

This stateful behavior is also used by the PortScan protection mechanism described later in the documentation.

---

## Inter-Zone Security Model

The network follows a least-privilege model.

The intended trust relationship can be represented as:

```text
Management
    |
    | Administrative access
    v
   DMZ

Employees
    |
    | Restricted service access
    v
   DMZ

Employees
    |
    X
    |
Management
```

Management has greater administrative privileges than Employees, but access is still controlled by firewall rules.

Employees cannot directly access the Management network.

The DMZ is isolated from internal networks and only exposes explicitly required services.

---

## Internet Access

Internet connectivity is controlled independently for each security zone.

The project implements different ICMP policies depending on the network's role.

### Management → Internet

ICMP is permitted to support administrative troubleshooting and connectivity testing.

### Employees → Internet

ICMP is restricted or blocked according to the firewall policy.

Normal application traffic such as web browsing can still be permitted independently.

### DMZ → Internet

ICMP is permitted only when required for troubleshooting or operational purposes.

This demonstrates that security policies are based on the role of each network rather than applying a single global rule.

---

## Wazuh Integration

Wazuh is located in the DMZ:

```text
192.168.20.2
```

The Wazuh Manager receives security telemetry from the laboratory.

The OPNsense firewall uses Syslog to forward `filterlog` events to Wazuh:

```text
OPNsense
192.168.20.1
      |
      | UDP 5514
      v
Wazuh Manager
192.168.20.2
```

Other systems in the environment use Wazuh agents where appropriate.

This creates a centralized monitoring architecture.

---

## Logging Architecture

The logging pipeline is:

```text
                    OPNsense
                       |
                       | filterlog
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
                Wazuh Analysis
                       |
                       v
                Security Alert
```

This separation allows OPNsense to focus on enforcement while Wazuh provides centralized analysis and monitoring.

---

## Detection and Response Architecture

One of the main security features of the project is automated port-scan detection.

The architecture is:

```text
TCP Connection Burst
        |
        v
OPNsense PF State Tracking
        |
        v
Connection Threshold Exceeded
        |
        v
PortScan PF Table
        |
        v
Floating Block Rule
        |
        v
Traffic Blocked
        |
        v
filterlog
        |
        v
Wazuh
```

This provides a basic automated detection-and-response workflow at the network perimeter.

---

## Defence in Depth

The project does not rely on a single security mechanism.

Instead, several layers are combined:

```text
                    Internet
                       |
                       v
              OPNsense Firewall
                       |
          +------------+------------+
          |            |            |
          v            v            v
    Segmentation    Stateful     Filtering
                   Inspection
          |
          v
          DMZ
          |
     +----+----+
     |         |
     v         v
   Wazuh    ModSecurity
     |
     v
 Centralized
 Monitoring
```

The main defensive layers are:

1. Network segmentation.
2. Stateful firewall rules.
3. Least-privilege access control.
4. DMZ isolation.
5. Web application protection through ModSecurity.
6. Firewall event logging.
7. Centralized Wazuh monitoring.
8. Automated port-scan blocking.

---

## Design Decisions

Several design decisions were made during implementation.

### Separate Security Zones

Management, Employees, and DMZ traffic is separated to reduce lateral movement.

### Dedicated DMZ

Security and web infrastructure is isolated from normal user systems.

### Centralized Monitoring

Firewall events are forwarded to Wazuh instead of relying exclusively on local firewall logs.

### Native PF Protection

Port-scan detection uses PF's state tracking and overload table functionality rather than depending exclusively on an external IDS.

### Tunable Detection

Connection thresholds are configurable because excessively aggressive values can generate false positives.

---

## Limitations

This laboratory is intentionally designed as a practical demonstration environment rather than a production enterprise deployment.

The main limitations include:

* Small number of endpoints.
* Virtualized infrastructure.
* Simplified network topology.
* Limited traffic volume.
* Basic connection-rate based port-scan detection.
* No redundant firewall.
* No dedicated IDS/IPS deployment.

These limitations provide opportunities for future development.

---

## Future Improvements

Potential extensions include:

* Deploying Suricata as an IDS/IPS.
* Adding additional Windows and Linux endpoints.
* Creating more advanced Wazuh correlation rules.
* Integrating threat-intelligence feeds.
* Implementing automated incident response.
* Adding VPN infrastructure.
* Introducing high-availability OPNsense nodes.
* Expanding network segmentation.
* Creating dedicated server and database zones.
* Adding vulnerability-scanning workflows.

---

## Summary

The architecture combines network segmentation, stateful firewall enforcement, DMZ isolation, centralized logging, SIEM monitoring, and automated firewall response.

The resulting environment provides a practical demonstration of how network security controls can work together:

```text
Segmentation
     +
Least Privilege
     +
Stateful Firewall
     +
Automated Blocking
     +
Centralized SIEM
     =
Layered Network Security
```

This architecture forms the foundation for the firewall policies, Wazuh integration, PortScan detection, and validation procedures documented in the remaining project documentation.
