# Infrastructure Monitoring with Zabbix

This repository documents the complete monitoring stack of my homelab, built using Zabbix Server, Zabbix Agent 2, and SNMP monitoring for network devices.  
It is designed as a clean, standalone project that shows the monitoring, observability, Linux administration, and network management configuration used in the homelab.

## 1. Project Overview

This monitoring setup provides:

- Observability of network devices, hypervisors, servers, and services  
- SNMP-based monitoring for MikroTik router and switch  
- Agent-based monitoring with Zabbix Agent 2 on Alpine Linux, DietPi, and Proxmox  
- Dashboards, graphs, triggers, alerts, and historical trends  
- A persistent, version-controlled documentation of the entire monitoring lifecycle  

The entire system runs on a dedicated DietPi Raspberry Pi 3 B+, acting as the monitoring node.

## 2. Infrastructure Summary

| Component        | Role                     | Technology                                      |
|------------------|--------------------------|-------------------------------------------------|
| Zabbix Server    | Monitoring backend/front | Zabbix 7.0 LTS on DietPi (Raspberry Pi 3 B+)    |
| Backup Server    | Agent-monitored node     | Zabbix Agent 2 (Alpine Linux)                   |
| Proxmox NUC      | Agent-monitored node     | Zabbix Agent 2 (Debian Bookworm / Proxmox VE)   |
| MikroTik hEX     | SNMP device              | RouterOS with SNMPv2 enabled                    |
| MikroTik RB260GS | SNMP device              | SwOS with SNMP enabled                          |
| LAN Subnet       | Managed infrastructure   | 172.16.0.0/24                                   |

Each host is monitored using the most appropriate method—SNMP or Zabbix Agent—providing metrics such as:

- CPU, RAM, and disk usage  
- Network throughput  
- Temperature and system health  
- Uptime, logs, storage, and performance indicators  

## 3. Repository Layout

Planned directory structure:

```text
infrastructure-monitoring-with-zabbix/
├── README.md                     # Overview of the monitoring project
├── Details/
│   └── README.md                 # Full end-to-end configuration steps
└── .gitignore
```

The `Details/README.md` (to be created) will include:

- Zabbix Server installation and configuration  
- MariaDB setup  
- Zabbix frontend initialization  
- Zabbix Agent 2 setup on:
  - Alpine Linux (Backup NUC)  
  - Proxmox VE (Intel NUC i5)  
  - DietPi (self-monitoring)  
- SNMP configuration for RouterOS and SwOS  
- Adding and grouping monitored hosts  
- Dashboards and graphs  

All images referenced in the documentation live in the main homelab repository:

- `homelab-core-rack/docs/`  
- https://github.com/Danar2714/homelab-core-rack/tree/main/docs

## 4. Relationship to the Core Homelab Project

The monitoring stack described here forms part of the larger homelab environment documented in:

- https://github.com/Danar2714/homelab-core-rack

That repository covers:

- Physical rack and cabling  
- Router and switch configuration  
- Hypervisors  
- Backup server  
- Raspberry Pi monitoring node  

This project focuses only on the monitoring layer.

## 5. Project Goals

- Deploy a monitoring solution on lightweight hardware  
- Demonstrate:
  - SNMP configuration  
  - Zabbix Agent   
  - Linux service administration  
  - Monitoring of network devices and servers  
  - Documentation and reproducibility  
- Provide a reference implementation that is easy to clone and extend  

## 6. License

This project is provided for educational purposes.  
Configuration examples should be reviewed and adapted before use in production environments.
