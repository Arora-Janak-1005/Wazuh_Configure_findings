# Wazuh SOC Home Lab

A hands-on cybersecurity lab for learning Security Operations Center
(SOC) concepts using Wazuh, Linux endpoints, and simulated security
events.

The goal of this repository is not just to deploy Wazuh, but to learn
how to investigate security events from raw endpoint telemetry through
detection, correlation, investigation, and documentation.

## Current Environment

-   **SIEM / Security Monitoring:** Wazuh
-   **Endpoint:** Kali Linux
-   **Remote connectivity:** Tailscale VPN
-   **Log source:** `/var/log/auth.log`
-   **Primary investigation:** SSH authentication activity
-   **Wazuh Agent:** Kali (Agent ID `001`)

## Investigation Workflow

The lab follows a simple SOC investigation process:

``` text
Security Event
     ↓
Endpoint Log
     ↓
Wazuh Agent
     ↓
Wazuh Manager
     ↓
Decoder
     ↓
Detection Rule
     ↓
Alert
     ↓
Investigation
     ↓
Timeline / Evidence
     ↓
Assessment
     ↓
Documentation
```

The focus is on understanding **why** an alert was generated and what
evidence supports the investigation, rather than simply viewing alerts
in the Wazuh dashboard.

## Current Investigations

### Incident 001 --- SSH Authentication Investigation

A controlled lab simulation involving repeated failed SSH password
attempts followed by a successful authentication.

The investigation covered:

-   Identifying the source and destination
-   Understanding Wazuh agent and source IP fields
-   Inspecting Wazuh rule `5760`
-   Understanding Wazuh rule levels
-   Identifying the `sshd` decoder
-   Correlating Wazuh alerts with `/var/log/auth.log`
-   Establishing the number of actual failed authentication attempts
-   Reconstructing an SSH session timeline
-   Identifying a successful authentication after failed attempts
-   Confirming session creation and termination
-   Identifying limitations in available endpoint telemetry

See
[`incidents/incident-001-ssh-authentication.md`](incidents/incident-001-ssh-authentication.md).

## Learning Principles

This lab follows a few important investigation principles:

1.  **Separate facts from assumptions.**
2.  **Do not label an event malicious without supporting evidence.**
3.  **Use raw endpoint logs to validate SIEM alerts.**
4.  **Correlate multiple events into a timeline.**
5.  **Distinguish authentication from actual post-authentication
    activity.**
6.  **Document missing telemetry instead of guessing what happened.**

## Planned Improvements

Future investigations will expand the lab to cover:

-   Post-authentication activity
-   Process execution telemetry
-   Privilege escalation
-   File integrity monitoring
-   Additional Linux security events
-   Wazuh rules and decoders
-   Detection engineering
-   Incident response workflows
-   Network security telemetry

Additional tools such as Suricata will be introduced only after the core
Wazuh/SOC workflow is understood.

## Disclaimer

All security events in this repository are generated in an authorized
personal lab environment for learning and defensive security purposes.
