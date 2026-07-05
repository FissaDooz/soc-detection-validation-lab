# Results

## Test 1: Brute force against Active Directory (Hydra)

| Item | Value |
|---|---|
| Target | Domain controller, LDAP authentication service |
| Tool | Hydra |
| Windows event generated | 4625 (An account failed to log on) |
| Failure reason | Unknown user name or bad password (`0xC000006D` / `0xC000006A`) |
| Logon type | 3 (network logon, SMB / NTLM) |
| Detection outcome | Event correctly forwarded to Splunk and flagged as anomalous authentication behavior |

An anonymized full sample of this event is available at [`../logs-samples/event-4625-failed-logon.log`](../logs-samples/event-4625-failed-logon.log).

**Reading**: the log confirms a repeated failed authentication pattern against the domain controller, sourced from a single host on the internal user network, using the NTLM authentication package. This is the expected signature for a brute-force / password-spray attempt and is exactly what the corresponding Splunk use case is meant to catch.

## Test 2: Reconnaissance scan against Active Directory (Nmap)

| Item | Value |
|---|---|
| Target | Domain controller |
| Tool | Nmap (horizontal and vertical scans) |
| Windows event generated | 5156 (Windows Filtering Platform has permitted a connection) |
| Port / service identified | 389/TCP (LDAP) |
| Detection outcome | Event correctly forwarded to Splunk and correlated with the expected alert |

An anonymized full sample of this event is available at [`../logs-samples/event-5156-network-scan.log`](../logs-samples/event-5156-network-scan.log).

**Reading**: a burst of event 5156 records from the same source host against the domain controller's LDAP port is a typical service-discovery signature, the reconnaissance step of an attack chain. Verifying that this reaches Splunk and correlates with the port-scan use case validates the detection pipeline end-to-end, not just the final exploitation step.

## Scope note

Both tests targeted **detection validation**, not exploitation. The goal was to confirm that expected Windows Security events are generated, forwarded, and correctly flagged by Splunk, not to compromise accounts or systems. No destructive or unauthorized testing was performed.

## What is not covered here

Quantitative detection metrics (mean time to detect, false positive/negative rates, or a detection-rate percentage across the full use case catalog) were not formally measured during this phase of the project and are intentionally not reported. Results above are qualitative pass/fail validations of two specific use cases (authentication brute force, network reconnaissance). Building UCT4S into the planned Forgejo CI/CD pipeline (see [methodology.md](methodology.md)) is what would make continuous, quantifiable measurement of the full use case catalog possible going forward.
