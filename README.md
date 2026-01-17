# Oracle-EBS-HA-DR-Architecture-POC
A VirtualBox-based POC showcasing Oracle EBS 12.2 with HA (Data Guard FSFO &amp; GoldenGate), DR, OEM monitoring, patching automation, performance tuning, security, and integrations with MongoDB/SQL Server

# Oracle EBS 12.2 High Availability and Disaster Recovery POC

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Oracle DBA Showcase](https://img.shields.io/badge/Expertise-Oracle%20DBA%20(15%2B%20Years)-blue)](https://github.com/yourusername/Oracle-EBS-HA-DR-POC)

## Overview
This repository is a proof-of-concept (POC) demonstrating a robust Oracle E-Business Suite (EBS) 12.2 architecture with a focus on **High Availability (HA)**, **Disaster Recovery (DR)**, **Security**, **Automation**, **Performance Tuning**, **OEM Monitoring**, **Maintenance Patching**, and **integrations with non-Oracle databases** like MongoDB and SQL Server. Built using VirtualBox VMs on a host with 32 CPUs and 128 GB RAM, it simulates a production-like environment starting from a single-instance EBS setup (App Tier + DB Tier) and evolving to a multi-site RAC-enabled system.

As an Oracle DBA with over 20 years of experience, this POC showcases real-world implementations based on Oracle My Oracle Support (MOS) documents, such as:
- Doc ID 2617787.1: Business Continuity with Data Guard on 19c.
- Doc ID 1626606.1: RAC with EBS 12.2.
- GoldenGate 12c labs for bidirectional replication.
- ETPAT-AT for patching (Doc ID 2749774.1).

Key Features:
- **HA/DR**: Data Guard FSFO for automatic failover; GoldenGate for zero-downtime migrations and active-active sync.
- **Database**: Oracle 19c RAC Multitenant (Single PDB) on ASM; convert single-instance to RAC.
- **App Tier**: EBS 12.2.9+ with FMW 11.1.1.9, WebLogic 10.3.6, Online Patching.
- **Monitoring**: OEM 13c with GoldenGate plugins for alerts, AWR reports, and performance (e.g., tuning mutex waits per Doc ID 1373500.1).
- **Tuning**: HugePages, AMM/ASMM (Doc ID 1134002.1); automation scripts for Linux kernel params.
- **Security**: EBS System Schema Migration (Doc ID 2755875.1), TDE encryption.
- **Patching**: AD/TXK Delta 15 (Doc ID 1617461.1), EURC-DT checks (Doc ID 2749775.1).
- **Integrations**: GoldenGate replicates EBS data to MongoDB (analytics) and SQL Server (legacy sync).
- **Automation**: Bash/Python scripts for setup, failover tests, and backups.

This setup runs on ~8 VirtualBox VMs (scalable to 4 for minimal POC), allocating ~64 GB RAM total across VMs.


## Architecture Diagram


<img width="1140" height="880" alt="image" src="https://github.com/user-attachments/assets/ac8fb60d-341d-4416-a057-009d940b3b49" />



Host Machine (32 CPUs, 128 GB RAM) - VirtualBox
├── Primary Site
│   ├── VM1: EBS App Tier → Connects to DB RAC
│   ├── VM2: DB RAC Node1 (19c, ASM) → Data Guard → GoldenGate
│   └── VM3: DB RAC Node2
├── DR Site
│   ├── VM4: Standby DB RAC Node1 (Active Data Guard)
│   ├── VM5: Standby DB RAC Node2
│   └── VM6: FSFO Observer
├── Monitoring
│   └── VM7: OEM Server → GoldenGate Monitor/Veridata
└── Integration
└── VM8: GoldenGate Hub → MongoDB/SQL Server

## Prerequisites
- Host: Oracle Linux 7/8 (or compatible).
- VirtualBox: Bridged networking for cluster interconnect.
- Software: Oracle EBS 12.2 StartCD (Patch 22066363), Grid Infrastructure 19c, GoldenGate 12c, OEM 13c.
- MOS Access: For docs like Doc ID 2674405.1 (XTTS Migration).

## Setup Instructions
1. **VM Creation**: Install OL7/8 on 8 VMs; configure shared disks for ASM.
2. **Grid/RAC Install**: Follow Doc ID 1453213.1 (adapt for 19c).
3. **EBS/DB Install**: Rapid Install for 12.2; migrate to RAC (Doc ID 1626606.1).
4. **HA/DR Config**: Data Guard (Doc ID 2617787.1); GoldenGate labs (included in /labs/).
5. **Integrations**: GoldenGate bidirectional setup to MongoDB/SQL Server.
6. **Tuning & Automation**: Run scripts/hugepages_tuning.sh (calculates params per Doc ID 1134002.1).
7. **Monitoring**: OEM setup with GoldenGate plugins (Lab Exercise 5).
8. **Test**: Simulate failover, load generation (GoldenGate Lab 7), patching (ETPAT-AT).

## Scripts and Tools
- `/scripts/patching.sh`: Automates AD/TXK RUPs and ETCC checks.
- `/scripts/tuning.sh`: Linux kernel tuning for Oracle (shared memory, HugePages).
- `/scripts/failover_test.sh`: Automates Data Guard FSFO tests.
- `/labs/`: GoldenGate exercises (e.g., downstream capture, Veridata).

## Contributing
Fork and PR your enhancements! Focus on DBA best practices.

## License
MIT License. Credits: Oracle MOS docs, usatdba@gmail.com.

## Contact
For questions, reach out via GitHub Issues or X: usatdba@gmail.com.


## TASK 1: OEM Agent Deployment Showcase
Demonstrating both **Agent Push** (console) and **Agent Pull** (`AgentPull.sh`) methods for monitoring DB/App servers in EBS HA/DR POC.

Detailed step-by-step guide:  
[Deploying OEM Agents - Push vs Pull](monitoring/Deploy-OEM-Agents-Push-Pull.md)

## TASK 2: Upgrades Showcase
- [OMR Database Upgrade (12c to 19c)](database_upgrade/Upgrading_OEM_OMR_Database.README.md) – Using AutoUpgrade for best practices.  
- [OMS Upgrade (13.3 to 13.5)](oem_upgrade/README.md) – Security-focused upgrade.