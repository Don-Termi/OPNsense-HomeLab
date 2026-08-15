# OPNsense segmentation and Wazuh SIEM lab

A segmented enterprise-style network security lab built with **OPNsense** and **Wazuh**, implementing **network segmentation, least-privilege firewall policies, centralized logging, and automatic port scan detection**.

## Overview

This project simulates a small enterprise environment with three isolated security zones:

* **Management**: administrative workstations
* **DMZ**: public-facing services
* **Employees**: internal user network

The firewall enforces strict inter-zone access controls while forwarding security events to a centralized **Wazuh SIEM** for monitoring and alerting.

## Architecture

* **OPNsense 26.7.1**
* **Wazuh Manager**
* **Apache + ModSecurity WAF**
* **Linux Mint (Management)**
* **Windows (Employees)**

## Network topology

| Zone       | Subnet          | Purpose                   |
| ---------- | --------------- | ------------------------- |
| Management | 192.168.10.0/24 | Administrative access     |
| DMZ        | 192.168.20.0/24 | Wazuh and Apache services |
| Employees  | 192.168.30.0/24 | Internal user endpoints   |

## Security controls implemented

### Network segmentation

* Isolated Management, DMZ, and Employees networks
* Inter-zone communication controlled through explicit firewall policies

### Least-privilege firewall policy

* Employees cannot access Management
* Employees have restricted access to DMZ
* Management has controlled administrative access to DMZ services
* ICMP restrictions implemented per security zone

### Automatic port scan protection

A floating rule detects excessive TCP connection attempts using **PF state tracking**.

When the configured threshold is exceeded:

1. The source IP is added to the **PortScan** overload table.
2. Existing states are terminated.
3. Subsequent traffic from the offending host is blocked.

### Centralized logging

OPNsense forwards **filterlog** events via **Syslog** to the Wazuh server.

### SIEM integration

Wazuh ingests firewall logs and generates custom alerts for suspicious network activity.

## Attack simulation

The port scan detection mechanism was validated by generating a high rate of TCP connection attempts against the DMZ server.

Observed behavior:

* Detection of excessive connection attempts
* Automatic insertion into the PortScan table
* Firewall blocking of subsequent traffic
* Wazuh alert generation through a custom rule

## Project structure

See the `docs/` directory for detailed documentation.

## Skills demonstrated

* OPNsense firewall administration
* PF state tracking and overload tables
* Network segmentation
* Security policy design
* Syslog integration
* Wazuh SIEM configuration
* Detection engineering
* Security monitoring

## Future improvements

* Suricata IDS integration
* Threat intelligence feeds
* Automated active response
* Dashboard visualizations
* VPN-secured management access
