# Project Roadmap: Oracle DBA Skills Showcase

This document is a working plan, not a finished deliverable. It covers what's already in the repo, what's wrong with it, and what to build next across the six focus areas: Disaster Recovery, High Availability (Data Guard FSFO + GoldenGate), Security, Automation, Performance Tuning, Monitoring (OEM), and Maintenance/Patching — plus an App DBA track woven throughout rather than run as a separate silo.

## 1. Audit of the current repo

**What's genuinely strong:** the existing runbooks (RAC conversion, OMR/OMS upgrades, OEM agent deployment, redo log reorg, AD/TXK patching) are detailed, screenshot-backed, MOS-doc-referenced, real hands-on work. That depth is the core asset — it just isn't packaged for the audience you now want (hiring managers skimming a portfolio), and it doesn't yet cover four of your six flagship topics.

**Structural problems found:**
- **Duplicate content in two places.** `Migration/Migrating_EBS_Single_node_to_RAC_README.md` and `High_Availability/RAC/Oracle_EBS_DB_Single_Node_to_RAC_README.md` are the same walkthrough. Same for `Migration/Move_to_Oracle_RAC_Binaries_from_Non-RAC_Binaries_README.md` and `High_Availability/RAC/Switch_Oracle_DB_Binary_to_RAC_enable_README.md`. Pick one home for RAC content (recommend `High_Availability/RAC/`) and delete the other.
- **Copy-paste title bug.** `OEM_13.5/OMS_upgrade_13.3_to_13.5.README.md` (an OMS upgrade) has the title "Upgrading OEM OMR Database from 12c to 19c" copied from the other doc. Needs its own title.
- **Broken links in `README.md`.** Every link under "TASK 1/2" points to a path that doesn't match the real file: wrong casing (`Monitoring/` vs `monitoring/`, `Database_Upgrade/` vs `database_upgrade/`), a folder that doesn't exist (`OEM_Upgrade/`), and filename mismatches (`online_redo_log_reorg_README.md` vs `Online_Redolog_Reorg_README.md`). None of these links currently resolve on GitHub.
- **Duplicate section numbering.** Two sections are both labeled "TASK 2" ("Upgrades Showcase" and "Maintenance Showcase").
- **Inconsistent naming conventions.** Folder casing is mixed (`High_Availability`, `Maintenance`, `Migration`, `OEM_13.5`, `database_upgrade`, `monitoring`) and file naming alternates between `_README.md` and `.README.md` and plain `.md`.
- **Missing coverage.** No content yet exists for Disaster Recovery / Data Guard FSFO, GoldenGate, Security, Automation, or Performance Tuning.
- **Nice find:** your own DNS notes already reserve `oradbserv04` / `192.168.56.130` as "Single instance dataguard db" — you'd already planned the Data Guard standby node before this conversation. The infra plan below builds on that instead of renaming it.

**Lab state (as of this plan, confirmed with you directly, not yet in the repo docs):**
- RAC conversion is functionally complete: Grid Infrastructure installed, both nodes in the cluster, `rconfig` executed successfully, SCAN/AutoConfig done on the app tier. This just needs verification + doc cleanup, not new build work.
- An app-tier patch attempt (`R12.AD.C.Delta.14`, Patch 33600809) hit opatch conflicts and was abandoned without proper documentation. The app tier is currently stable, so this is not an outage — it's an unfinished patch to retry and document properly this time (the conflict-resolution story is better content than a clean first-try would have been).
- Long-term DB version target: 12.2.0.1 → 19c → Oracle AI Database 26ai. Both hops are currently Oracle-certified paths for EBS 12.2 on-premises (confirmed via Oracle's EBS technology blog, July 2026). The 19c hop also requires converting the EBS database from non-CDB to the multitenant architecture (single PDB) — a real, separate piece of work, not just an AutoUpgrade run.

## 2. Proposed repo structure

```
/
├── README.md                      (fixed links, one clean topic index)
├── PROJECT_ROADMAP.md             (this file)
├── disaster-recovery/             (NEW — Data Guard FSFO)
├── high-availability/
│   ├── dataguard/                 (NEW)
│   ├── goldengate/                (NEW)
│   └── rac/                       (existing RAC content, consolidated, dedup'd)
├── security/                      (NEW, includes EBS FND/responsibility security)
├── automation/                    (NEW, includes Rapid Clone, control-node scripts)
├── performance-tuning/            (NEW)
├── monitoring/                    (existing, keep)
├── maintenance/                   (existing, keep — merge patching + redo reorg + Delta 14 retry)
├── migration/                     (existing, keep as-is or fold into rac/ history)
└── database-upgrade/              (existing OMR upgrade doc, plus new EBS DB tier 19c/26ai upgrades)
```
All-lowercase-with-hyphens folder names, consistent `topic-name-README.md` file naming. No separate `app-dba/` folder — App DBA topics (Rapid Clone, AutoConfig, Concurrent Manager/PCP, WebLogic/OPMN, FND security) live inside whichever of the folders above they naturally belong to, so the repo reads as one coherent DBA skill set rather than two disconnected tracks.

## 3. Content model: pair every topic with two posts

Your existing docs are technical runbooks — correct for demonstrating depth, but not "explaining a complex topic to a general audience." Recommend a two-tier structure per topic so both audiences are served without diluting either:

- **Explainer post** (`README.md` at the top of each topic folder): plain-language, hiring-manager-facing. What's the problem, why does it matter to a business (uptime, revenue, compliance), what did you build, what would break without it. Light diagrams, no shell commands.
- **Technical runbook** (already your existing style, linked from the explainer): the actual commands, screenshots, MOS doc references. This is where you prove the depth.

This also means your existing runbooks don't need to be rewritten — they just need a plain-language front door written for each one.

## 4. The two-hop database upgrade path

Goal: `12.2.0.1 → 19c → 26ai`. Both hops are Oracle-certified for EBS 12.2 on-premises, but they land in different phases of this plan rather than back to back:

- **Hop 1 (12.2.0.1 → 19c), in Phase 2, before DR exists.** Includes the non-CDB → CDB/single-PDB conversion required for EBS 12.2 on 19c. Doing this while the topology is still just RAC (no standby yet) avoids upgrading a primary and a standby together on your very first attempt.
- **Hop 2 (19c → 26ai), in Phase 5, after DR is built.** Done as a rolling upgrade across the live Data Guard primary/standby pair. This is a genuinely advanced piece of content — "upgrading an HA/DR pair with near-zero downtime" — and only makes sense to write once the pair actually exists.

## 5. Series roadmap by focus area

| # | Topic | Phase | Explainer post | Technical runbook |
|---|---|---|---|---|
| 1 | RAC (verification) | 2 | Node failure walkthrough in plain terms | Verify + clean up existing RAC docs (dedup'd) |
| 2 | Maintenance — AD/TXK Delta 14 retry | 2 | Patching without downtime, and what happens when it goes wrong | Resolve/document the opatch conflicts on Patch 33600809 |
| 3 | Database Upgrade — 19c + CDB conversion | 2 | Why "just upgrade the database" is never just that | AutoUpgrade + non-CDB→CDB/PDB conversion on the RAC primary |
| 4 | Security | 2/3 | Defense in depth for a DBA: encryption, access, auditing | TDE, EBS system schema migration, FND_USER/responsibility security, unified auditing |
| 5 | Automation | 3 | Why DBAs script themselves out of 2am pages | Ansible/Python control node: patch rollout, failover test, backup verification, Rapid Clone |
| 6 | Disaster Recovery (standby build) | 4 | Why DR ≠ backup; RTO/RPO in plain terms | Build physical standby on `oradbserv04` (now on 19c) |
| 7 | HA — Data Guard FSFO (3-part mini-series) | 4 | Part 1: standby setup · Part 2: FSFO + observer config · Part 3: forced failover drill | Matches each explainer, one post/week |
| 8 | HA — GoldenGate | 4 | Real-time replication vs. Data Guard, when you need both | Extract/replicat setup on `oradbserv05`, lag monitoring |
| 9 | Database Upgrade — 26ai rolling upgrade | 5 | Upgrading a live HA/DR pair without an outage | Rolling upgrade across primary/standby using Data Guard |
| 10 | Performance Tuning | 6 | How a DBA finds "why is it slow" | AWR/ASH walkthrough, a real before/after case with real numbers |
| 11 | Monitoring (OEM) | ongoing | OEM as the nervous system of the estate | Existing agent deploy/golden image docs, extended to every new VM as it's built |

## 6. Infrastructure build-out plan

Starting point (existing): `oemserver01`, `orappsserv01`, `oradbserv01`+`oradbserv02` (2-node RAC primary, GI/rconfig/SCAN already complete).

Proposed additions, sized against your 32 vCPU / 128 GB host:

| VM | Role | vCPU | RAM | Notes |
|---|---|---|---|---|
| `oemserver01` (existing) | OEM 13c OMS/repo | 4 | 16 GB | no change |
| `orappsserv01` (existing) | EBS App tier | 6 | 20 GB | Delta 14 retry happens here |
| `oradbserv01`/`02` (existing) | Primary RAC DB | 8+8 | 32+32 GB | Upgraded in place: 12.2.0.1 → 19c/CDB (Phase 2), then → 26ai (Phase 5) |
| `oradbserv04` (new) | Data Guard physical standby | 8 | 32 GB | Built on 19c directly (post Hop 1); IP/hostname already reserved in your DNS notes |
| `dgobserver01` (new) | FSFO Observer | 2 | 2 GB | Lightweight; a real 3rd-site placement is the whole point of the DR story |
| `ctrlserv01` (new) | Automation/Ansible control node | 2 | 4 GB | Built in Phase 3; drives patching, failover drills, backup checks, VM start/stop against every other node |
| `oradbserv05` (new) | GoldenGate target/reporting DB | 4 | 12 GB | A Data Guard physical standby can't be open for writes, so GoldenGate needs a separate open database to replicate into |

Totals if every VM runs at once: ~42 vCPU / 150 GB — over both host limits if run simultaneously, which is expected and fine: keep OEM + App + Primary RAC running as the "always-on core," and bring up Standby/Observer/Control/GoldenGate-target only for the phase you're actively building. A VM start/stop-by-scenario script is itself good Automation-post material.

## 7. Build order and working agreements

**Phases, in order:**
1. **Repo hygiene** — dedupe RAC docs, fix README links, consistent naming, split the duplicate "TASK 2" section.
2. **Baseline solidification (no new VMs)** — verify/document the completed RAC conversion; retry and document the Delta 14 app-tier patch; upgrade the DB tier to 19c including the CDB/PDB conversion; baseline security and backup pass.
3. **Automation control node** — build `ctrlserv01` and use it to script the Phase 2 verification work, so later builds inherit it instead of automation being bolted on last.
4. **DR/HA build** — standby (`oradbserv04`), the Data Guard FSFO 3-part mini-series, then GoldenGate (`oradbserv05`).
5. **26ai rolling upgrade** across the now-live Data Guard pair.
6. **Performance tuning**, once there's a mature system with real history to measure.

**Cadence:** at least 1 post per week. Heavier topics (Data Guard FSFO) are split into short mini-series so weekly cadence holds without rushing the lab work; lighter topics (Security, most App DBA topics) ship as a single post.

**Lab time:** 2–3 hours/day is the working assumption, enough for steady progress given how much of Oracle infra work is unattended waiting (installs, patch downloads, verification runs) that can overlap with drafting.

**Shared responsibilities (Discover → Define → Develop → Deliver):** Develop (hands-on VM/lab work, the actual commands and errors) is entirely yours — I have no access to the VirtualBox lab. Discover (MOS doc research) and Deliver (drafting both post tiers from your notes/screenshots) are mine to drive. Define (scoping what a session or post covers) is shared — I propose, you confirm against what's realistic in the lab that day.

## 8. Next step

Recommended starting point given all of the above: begin Phase 1 (repo hygiene) and Phase 2's Delta 14 retry in parallel, since one is pure repo editing and the other is lab work you can start independently. The 19c/CDB upgrade is the next big Phase 2 item after that.
