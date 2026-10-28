Project 2: Firewall Configuration & System Hardening

Overview

This project focuses on configuring a firewall and applying basic system-hardening practices in a Kali Linux virtual machine.

Objectives

- Configure UFW (Uncomplicated Firewall).
- Apply a default-deny incoming traffic policy.
- Allow selected traffic using firewall rules.
- Test network connectivity from a Windows host.
- Review running services and listening ports.
- Verify the final firewall configuration.

Lab Environment

- Operating System: Kali Linux
- Virtualization: Oracle VirtualBox
- Testing Machine: Windows 11
- Firewall: UFW
- Tools: Netcat, PowerShell, ss, and systemctl

Methodology

1. Verify network connectivity.
2. Check the initial firewall status and listening ports.
3. Configure UFW firewall rules.
4. Test TCP port accessibility from Windows.
5. Compare results with the firewall enabled and disabled.
6. Review running services and listening ports.
7. Verify the final firewall configuration.

Key Findings

The lab demonstrates how firewall policies affect incoming TCP connections and how explicit allow rules change port accessibility.

Evidence

Screenshots, test results, configuration outputs, and the project report will be maintained in the corresponding project folders.

Security Recommendations

- Deny unnecessary incoming traffic.
- Allow only required services and trusted sources.
- Disable unnecessary services after reviewing their dependencies.
- Keep the operating system updated.
- Enable firewall logging and review relevant events.

Scope

Testing was performed in a controlled VirtualBox lab using systems under my control.


