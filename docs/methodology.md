# Methodology

## SIEM deployment

Splunk Enterprise 9.3.1 was deployed on a Rocky Linux 9.6 virtual machine, matching the version used in a target production environment. The goal was a working SIEM able to collect and analyze logs from Windows hosts, Linux hosts, and network devices.

## Log collection: Universal Forwarders

Splunk Universal Forwarders (UF) were used to collect and transfer logs and events from source systems. Key characteristics that drove the choice:

- Real-time collection of log files, syslog streams, and system/application events
- Secure transmission to a Splunk indexer over TCP or SSL, with optional encryption and authentication
- Lightweight, low system impact, suited to large-scale deployments
- Modular architecture allowing data filtering/transformation before transfer
- Compatible with Splunk add-ons
- Support for major Linux distributions (Rocky Linux, CentOS, Red Hat, Ubuntu), plus Windows and macOS
- Can run under a dedicated low-privilege user for tighter security
- Centralized management support via Deployment Server
- Installable via RPM, DEB, or tarball packages, compatible with automation tools such as Ansible, Puppet, or Chef
- Performs log pre-parsing to simplify log format before ingestion

**Ansible** was used to automate the installation and configuration of the Splunk Universal Forwarder across the Rocky Linux servers.

## Secure transport

A site-to-site IPsec VPN (IKEv2, VTI) was set up to carry logs securely between the tested infrastructure and the SIEM segment, guaranteeing confidentiality and integrity of log traffic in transit.

## Evaluating advanced collection architectures

Before committing to a Universal-Forwarder-based architecture, more advanced log collection and normalization solutions were investigated: **Cribl** and **Splunk Connect for Syslog (SC4S)**. These tools provide broader processing capabilities (advanced filtering, upstream data normalization, and finer control over ingested volumes) and are common in mature SOC architectures where flow management and cost optimization matter.

However, adopting them adds configuration and maintenance complexity, plus a non-negligible learning curve. Given the project's time constraints and training objectives, a simpler, more direct approach was prioritized. Universal Forwarders were chosen as the pragmatic path to a functional SIEM quickly. This does not rule out migrating to Cribl or SC4S later.

## Evaluating `contentctl`

As requested during the project, the official Splunk tool `contentctl` was investigated.

For the actual problem at hand, it did not turn out to be the most relevant fit. While technically possible to approach the goal with it, the implementation would have been heavy and disproportionate: it requires full detection modeling, building dedicated datasets, and a dedicated environment.

`contentctl` fundamentally works in the opposite direction of the actual need: it is built to test detections **before** they reach Splunk, not to diagnose the behavior of detections that are already deployed. Using it solely to answer "is this already-deployed detection still working?" would mean a technically demanding, high-cost implementation for a low return.

Its real strength is enabling development, validation, and automated testing of use cases **before** deployment: version control, lifecycle management, and quality gates for detections before they reach production. `contentctl` becomes legitimate and relevant for a different problem: building a structured process to create, evolve, and maintain use cases over time, with systematic quality control, an as-code approach, CI/CD integration, and automated pre-production testing. It relies on an integrated toolset to simulate realistic offensive behavior and run detections in an isolated but controlled Splunk environment, and can be configured to closely mirror a production Splunk instance (apps, sources, indexes, dependencies), which is powerful but proportionally costly to set up.

## UCT4S: a purpose-built alternative

To answer the actual need (validating detections that are already live), a custom tool was designed: **UCT4S (Use Case Tester for Splunk)**.

UCT4S is a platform able to:
- Automatically inject synthetic (but realistic) logs generated from real captured logs
- Run the associated SPL searches
- Continuously validate Splunk detection use cases
- Verify that expected detections actually trigger

The end goal is to integrate UCT4S into a **Forgejo CI/CD pipeline**, so that every use case modification or addition automatically triggers its validation, and so that a full regression test of all use cases can be triggered regularly and automatically.

## Traffic design: management vs. log traffic

**Problem**: when designing the SOC/SIEM connectivity, the intent was to separate management traffic (administration, supervision) from log-collection traffic (events sent to Splunk), while part of the traffic already transited through a VPN.

**Difficulty**: maintaining that full separation proved complex at the routing/rule level, and carried a real risk of losing logs if they were routed outside the VPN tunnel that had been set up.

**Decision**: all SOC-bound traffic (including logs) was routed through the VPN. This gave simpler routing via VTI, more reliable log delivery, and a reduced attack surface, at the cost of some traffic-separation granularity.

## Validation tests

### Brute force

**Goal**: assess the robustness of Active Directory's authentication mechanism against brute-force attacks, and Splunk's ability to detect and correlate the resulting failed-logon events.

**Scope**: the Windows Server domain controller, the LDAP/AD authentication service, associated security policies, and Splunk's detection/correlation of the events. No destructive or unauthorized testing was performed.

**Environment**: an Active Directory domain controller (DC), a Parrot Security 6.4 Linux attack workstation, and a log collection pipeline feeding Splunk.

**Tool**: [Hydra](https://github.com/vanhauser-thc/thc-hydra), an open-source auditing tool for testing authentication mechanisms via repeated login attempts across multiple protocols. It is commonly used to evaluate password complexity, lockout policies, and detection mechanisms.

**Method**: a controlled brute-force simulation using a custom username list and a common-pattern password list. The objective was not to compromise accounts, but to observe AD's behavior (lockout, delays) and the SIEM's detection effectiveness.

**Observation**: LDAP generated Windows event 4625 (failed logon attempts); these logs were transmitted to Splunk, which correctly flagged the anomalous authentication behavior.

### Reconnaissance scan

**Goal**: identify open ports, running services, and active hosts on the target network.

**Tool**: [Nmap](https://nmap.org/), open-source network scanner.

**Method**: horizontal and vertical scans against the domain controller. The scan confirmed port 389/TCP (LDAP) was open, identified the target hostname (DC1) and LDAP version, and provided reconnaissance data ahead of the brute-force test. Logs were checked both in Active Directory and in Splunk, in correlation with the generated alert.

**Observation**: the scan generated Windows event 5156 (Windows Filtering Platform connection permitted), correctly forwarded to and correlated in Splunk.

## Operational troubleshooting

**`Invalid key: recursive=true`**: when configuring a `monitor://` stanza on the Universal Forwarder, Splunk rejected the `recursive=true` attribute. Cause: this attribute is not supported on UF `monitor://` entries. Splunk automatically watches a directory and its files but does not recurse into subdirectories the way some assume.

**Missing `User=splunkfwd` in `systemctl status SplunkForwarder`**: initially looked like a misconfiguration. Explanation: `systemd` does not always display every internal directive in `systemctl status` output. The service was running correctly; the missing line did not indicate a malfunction.
