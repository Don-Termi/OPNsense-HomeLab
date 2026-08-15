# Employees firewall rules

The Employees network (`192.168.30.0/24`) represents internal user workstations and client devices.

This security zone is intentionally **highly restricted** to prevent lateral movement, unauthorized access to administrative systems, and unnecessary exposure of DMZ services.

## Security objectives

The Employees network was designed with the following goals:

* provide controlled internet access,
* allow access only to authorized DMZ services,
* completely isolate the Management network,
* restrict ICMP usage,
* and limit opportunities for internal reconnaissance.

## Security model

Employees are treated as **unprivileged internal users**.

Unlike the Management network, employee systems do not have administrative privileges and are prevented from accessing infrastructure services unless explicitly required.

## Rule summary

| Priority | Action | Source    | Destination | Ports   | Purpose                       |
| -------- | ------ | --------- | ----------- | ------- | ----------------------------- |
| 1        | Pass   | Employees | Internet    | 80, 443 | Web access                    |
| 2        | Pass   | Employees | DMZ         | 80, 443 | Access to web services        |
| 3        | Block  | Employees | Management  | Any     | Prevent administrative access |
| 4        | Block  | Employees | Internet    | ICMP    | Restrict ICMP                 |
| 5        | Block  | Employees | DMZ         | Any     | Block unauthorized DMZ access |

## Internet access

Employees are allowed outbound web access using:

* HTTP (80),
* HTTPS (443).

Restricting outbound traffic to web services reduces unnecessary exposure and limits the ability of compromised hosts to communicate freely.

## DMZ access

Employees may access only the **public web services** hosted in the DMZ.

This allows normal application access while preventing direct communication with infrastructure services such as:

* Wazuh,
* SSH,
* and administrative interfaces.

## Management isolation

One of the most important controls is the complete isolation of the Management network.

Employees are explicitly blocked from accessing:

* administrative workstations,
* firewall management,
* infrastructure services,
* and management interfaces.

This significantly reduces lateral movement opportunities.

## ICMP restrictions

ICMP is intentionally restricted for Employees.

This limits basic network reconnaissance techniques such as:

* ping sweeps,
* host discovery,
* and simple connectivity mapping.

Management retains ICMP permissions for troubleshooting purposes.

## Default deny behavior

The final blocking rule prevents unrestricted communication with the DMZ.

Only explicitly authorized web services remain accessible.

This ensures that new DMZ services are **not automatically exposed** to internal users.

## Security rationale

This policy implements **least privilege** by allowing employees only the minimum network access required for normal operation.

If an employee workstation becomes compromised, the attacker is prevented from:

* reaching administrative systems,
* accessing SIEM infrastructure,
* communicating with unauthorized DMZ services,
* and performing unrestricted network reconnaissance.

## Validation

The following scenarios were validated during testing:

* web access to the internet,
* access to authorized DMZ web services,
* blocked access to the Management network,
* blocked ICMP traffic,
* and logging of firewall policy violations through OPNsense filterlog.

## Monitoring

Blocked connections from the Employees network are logged by OPNsense and forwarded to Wazuh through Syslog integration.

This provides visibility into:

* unauthorized management access attempts,
* blocked ICMP activity,
* and suspicious internal network behavior.

The Employees rule set forms the primary internal security boundary of the laboratory and represents the main enforcement point for lateral movement prevention.
