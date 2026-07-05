# Architecture Detail

All addressing below is anonymized: real network prefixes were replaced with generic ranges (`10.x.x.x`) and internal domains were replaced with `*.example.lab`. Host suffixes, VLAN IDs, and the overall segmentation logic are kept identical to the original design, since that structure, not the specific IPs, is what matters technically.

> **MVP scope note (kept from the original documentation):** this architecture was intentionally limited to its functional perimeter. Splunk Connect for Syslog (SC4S), Cribl, and secured syslog over TCP 6514 (TLS) were identified as possible improvements but not implemented, due to time constraints. The architecture was designed to allow that evolution later.

## Remote access: Client-to-Site VPN and Bastion

```mermaid
flowchart TB
    CLIENT["Remote client<br/>VPN client"] -->|IPsec IKEv2 tunnel over Internet| FW1VPN["FW1 - Fortigate<br/>VPN gateway"]
    FW1VPN --> BASTION["Bastion - Guacamole<br/>web-based SSH/RDP termination"]
    BASTION -->|SSH| NETGEAR["Network gear<br/>switches, routers, MLS, firewalls"]
    BASTION -->|SSH / RDP| DC["Datacenter (PVE1 / PVE2)<br/>pfSense, Active Directory"]
```

The bastion is the single entry point for administrative access: no network device or server port is exposed directly. A remote administrator connects over an IPsec tunnel to the Fortigate VPN gateway, then reaches the Guacamole web portal, which brokers SSH/RDP sessions to the target systems without giving the client direct network-layer access to them.

## Per-site datacenter layout (Proxmox)

Both datacenter sites follow the same pattern: a pfSense virtual firewall fronts two segments: an Active Directory segment and a Linux services segment. Site 2 mirrors site 1 for redundancy, except for the log collector and the phishing-simulation host, which only exist on site 1.

```mermaid
flowchart TB
    subgraph PVE1["PVE1 - uplink 10.20.5.0/24"]
        PFS1[pfSense]
        DC1["DC1 - 10.20.20.1"]
        FS1["FS1 - 10.20.20.2"]
        DHCP1["DHCP1 - 10.20.21.1"]
        DNS1["DNS1 - 10.20.21.2"]
        TFTP1["TFTP1 - 10.20.21.3"]
        CA1["CA1 - 10.20.21.4"]
        LOGC["LogCollector - 10.20.21.14"]
        GOPHISH["Gophish - 10.20.21.89"]
    end
    PFS1 --- DC1
    PFS1 --- FS1
    PFS1 --- DHCP1
    PFS1 --- DNS1
    PFS1 --- TFTP1
    PFS1 --- CA1
    PFS1 --- LOGC
    PFS1 --- GOPHISH

    subgraph PVE2["PVE2 - uplink 10.20.8.0/24"]
        PFS2[pfSense]
        DC2["DC2 - 10.20.22.1"]
        FS2["FS2 - 10.20.22.2"]
        DHCP2["DHCP2 - 10.20.23.1"]
        DNS2["DNS2 - 10.20.23.2"]
        TFTP2["TFTP2 - 10.20.23.3"]
        CA2["CA2 - 10.20.23.4"]
    end
    PFS2 --- DC2
    PFS2 --- FS2
    PFS2 --- DHCP2
    PFS2 --- DNS2
    PFS2 --- TFTP2
    PFS2 --- CA2
```

- Active Directory domain: `ad.example.lab` (DC1/DC2 are domain controllers, FS1/FS2 are file servers)
- Certificate hierarchy: CA1 (site 1) is the offline root CA; CA2 (site 2) is the intermediate CA that signs certificates for network hosts
- `Gophish` is deployed and functional but is not used in this phase of the project (reserved for future phishing-awareness scenarios)

## Log flow

```mermaid
flowchart LR
    NETGEAR["Network devices<br/>(syslog)"] -->|syslog| LOGC["Log Collector - PVE1<br/>rsyslog + Splunk UF"]
    SRV1["Servers - Datacenter 1<br/>Linux UF / Windows UF"] -->|Splunk UF| SPLUNK
    SRV2["Servers - Datacenter 2<br/>Linux UF / Windows UF"] -->|Splunk UF| SPLUNK
    LOGC -->|forwarded via core switch link| PVE2FWD["PVE2 - forwarding"]
    PVE2FWD -->|IPsec IKEv2 site-to-site tunnel| SPLUNK["Splunk Enterprise - PVE3<br/>SOC segment"]
```

Two collection paths converge on Splunk:
1. **Network device logs** (routers, switches, firewalls) are sent via syslog to a dedicated Log Collector on site 1, which also runs a Splunk Universal Forwarder. From there, logs are relayed through site 2 and cross the IPsec site-to-site tunnel into the SOC segment.
2. **Server logs** (Windows and Linux hosts on both datacenter sites) are sent directly to Splunk through their own Universal Forwarders.

This keeps the SOC segment's only inbound path a single, encrypted, monitored tunnel, consistent with the isolation principle described in the main [README](../README.md).
