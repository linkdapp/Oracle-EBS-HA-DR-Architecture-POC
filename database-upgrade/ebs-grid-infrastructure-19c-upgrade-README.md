# Upgrading Grid Infrastructure from 12.2.0.1 to 19c (2-Node RAC, Rolling, Silent)

## Status

**Complete.** This is the runbook for Phase 2 of the project (see [PROJECT_ROADMAP.md](../PROJECT_ROADMAP.md), item 3: "Database Upgrade — 19c + CDB conversion"). Grid Infrastructure was upgraded first — before the RDBMS home, before the CDB/PDB conversion — because GI must be at or above the version of any RDBMS home it serves. This hop: `12.2.0.1.220118` → `19.3.0.0.0` (patched to `19.29.0.0.251021` in the process), rolling across `oradbserv01`/`oradbserv02`, cluster never fully down.

## Why this matters

A RAC database can only be upgraded to a database version the cluster's Grid Infrastructure already supports — GI is the floor, not something that follows along afterward. Doing this rolling (one node at a time, cluster never fully down) is also the entire point of having two nodes: if this were a full-cluster-down upgrade, RAC would be buying nothing over a single instance during the maintenance window.

## Environment baseline (confirmed before starting)

| Component | Before | After |
|---|---|---|
| GI home | `/u01/app/grid/12.2.0`, `12.2.0.1.220118` (Jan 2022 RU) | `/u01/app/grid/19.3.0`, `19.29.0.0.251021` |
| Nodes | `oradbserv01` (1), `oradbserv02` (2) | unchanged |
| ASM disk groups | `DATA01`, `DATA02`, `FRA01`, `compatible.asm`/`compatible.rdbms` at `12.1.0.0.0` | unchanged — deliberately not raised yet; that bump happens after the RDBMS-tier upgrade is validated |
| RDBMS home | `/u01/app/oracle/product/12.2.0/db_1`, `12.1.0.2.0`, non-CDB `EBSAPPDB` | unchanged in this phase |
| GIMR | Not configured (`configureGIMR=false`) | unchanged, deliberate |

**Note found before starting this phase:** the standalone one-off Patch 28553832 (Dec 2018, scoped to GI `12.2.0.1.0` base) was not needed — it was already superseded by the previously-installed Patch 33678030 ("OCW JAN 2022 RELEASE UPDATE"), which lists bug 28553832 in its own fixed-bug set.

**Discrepancy worth flagging:** the GI Release Update actually applied here is `19.29.0.0.251021` (Patch 38298204, via combo Patch 38273558). The RDBMS-tier plan (Doc 1594274.1, uploaded to this project) references DBRU `39034528` / `19.31.0.0.260421` for the database tier — a newer quarterly RU than what GI is now running. GI running a slightly older RU than the RDBMS tier will run is not a blocking misconfiguration (GI's version floor requirement is about the major/minor release — `19c` — not an exact RU match), but it's worth deciding deliberately: either apply the matching `19.31` GI RU before moving on, or explicitly accept `19.29` on the GI side and note why. Not resolved as part of this phase — flagged here for a decision before Phase 4/5.

## Pre-upgrade: OCR backup

```bash
# Check current OCR backup status
/u01/app/grid/12.2.0/bin/ocrconfig -showbackup

# Force a manual backup
sudo /u01/app/grid/12.2.0/bin/ocrconfig -manualbackup

# Verify it now exists
/u01/app/grid/12.2.0/bin/ocrconfig -showbackup

# Local registry integrity check
sudo /u01/app/grid/12.2.0/bin/ocrcheck -local
```

![OCR backup and manual backup verification](gi_upgrade/precheck_backup_manual_ocr.png)
![OLR integrity check](gi_upgrade/precheck_ocrcheck.png)

_OUTPUT:_
```
[grid@oradbserv01 ~]$ sudo /u01/app/grid/12.2.0/bin/ocrconfig -manualbackup
oradbserv01     2026/07/22 11:03:03     +DATA02:/oradbapp-clust/OCRBACKUP/backup_20260722_110303.ocr.261.1239274983     1870776542

[grid@oradbserv01 ~]$ sudo /u01/app/grid/12.2.0/bin/ocrcheck -local
Status of Oracle Local Registry is as follows :
         Version                  :          4
         Device/File integrity check succeeded
         Local registry integrity check succeeded
         Logical corruption check succeeded
```

## Step 1: Stage the 19c GI software (out-of-place)

```bash
mkdir -p /u01/app/grid/19.3.0
cd /u01/app/grid/19.3.0
unzip -q /path/to/LINUX.X64_193000_grid_home.zip
```

![Step 1: Unzipping GI binaries](gi_upgrade/Step1_Stage_GI_unziping_GI_binaries.png)

An early `cluvfy` pass was also run right after staging, as an extra sanity check before touching OPatch or applying anything:

```bash
export ORACLE_HOME=/u01/app/grid/19.3.0
export ORACLE_BASE=/u01/app/oracle
export PATH=$PATH:$ORACLE_HOME/bin:$ORACLE_HOME/OPatch

/u01/app/grid/19.3.0/runcluvfy.sh stage -pre crsinst -upgrade -rolling \
  -src_crshome /u01/app/grid/12.2.0 -dest_crshome /u01/app/grid/19.3.0 \
  -dest_version 19.0.0.0.0 -fixupnoexec -verbose -method root
```

![Step 1: Early cluvfy check, start](gi_upgrade/Step1_preinstall_check_1a_cluvfy_start.png)
![Step 1: Early cluvfy check, end](gi_upgrade/Step1_preinstall_check_1b_cluvfy_end.png)

## Step 2: Upgrade OPatch and apply the Release Update to the staged home

Patch-first, upgrade-second — the RU goes into the software the moment it's staged, never after the upgrade completes.

```bash
/u01/app/grid/19.3.0/OPatch/opatch version
# OPatch Version: 12.2.0.1.17

mv /u01/app/grid/19.3.0/OPatch /u01/app/grid/19.3.0/OPatch.bak_$(date +%Y-%m-%d_%H-%M-%S)
unzip -q /media/sf_eoracle/Patch/19c/p6880880_190000_v49_Linux-x86-64.zip -d /u01/app/grid/19.3.0

/u01/app/grid/19.3.0/OPatch/opatch version
# OPatch Version: 12.2.0.1.49
```

![Step 2: Upgrading OPatch](gi_upgrade/Step2_1a_upgrading_opatch.png)

Stage and apply the GI Release Update (combo OJVM + GI RU patch):

```bash
mkdir -p $ORACLE_BASE/staging/patch/gi
unzip -q /media/sf_eoracle/Patch/gi_db/COMBO_OF_OJVM_COMPONENT_19.29.0.0.251021+GI_RU_19.29.0.0_p38273558_190000_Linux-x86-64.zip \
  -d $ORACLE_BASE/staging/patch/gi
```

![Step 2: Unzipping the GI patch](gi_upgrade/Step3_1c_unzipping_GI_patch.png)

```bash
cd /u01/app/grid/19.3.0
./gridSetup.sh -applyRU /u01/app/oracle/staging/patch/gi/38273558/38298204
```

![Step 2: Launching gridSetup to apply the patch](gi_upgrade/Step3_1d_launching_gridsetup_apply_patch.png)

_OUTPUT:_
```
Preparing the home to patch...
Applying the patch /u01/app/oracle/staging/patch/gi/38273558/38298204...
Successfully applied the patch.
Launching Oracle Grid Infrastructure Setup Wizard...
```

## Step 3: Generate the response file (GUI wizard) and dry run

Applying the RU launches the Grid Infrastructure Setup Wizard automatically. Rather than hand-editing the `.rsp` template blind, the wizard's own "generate response file" path was used to produce a correct one for this exact staged home — the safer route, since it guarantees every variable name matches what this specific 19.3.0 build actually expects.

![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1e_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1f_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1g_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1h_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1i_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1j_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1k_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1l_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1m_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1n_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1o_gui_installer.png)
![Step 3: GUI installer walkthrough](gi_upgrade/Step3_1p_gui_installer.png)

Resulting response file, `/u01/app/grid/19.3.0/install/response/upgrade_response.rsp` (key sections):

```
# SECTION A - BASIC
INVENTORY_LOCATION=/u01/app/oraInventory
oracle.install.option=UPGRADE
ORACLE_BASE=/u01/app/oracle

# SECTION B - GROUPS
oracle.install.asm.OSDBA=asmdba
oracle.install.asm.OSOPER=asmoper
oracle.install.asm.OSASM=asmadmin

# SECTION C - SCAN
oracle.install.crs.config.scanType=LOCAL_SCAN
oracle.install.crs.config.gpnp.scanName=scan-oradbserv
oracle.install.crs.config.gpnp.scanPort=1521

# SECTION D - CLUSTER & GNS
oracle.install.crs.config.ClusterConfiguration=STANDALONE
oracle.install.crs.config.configureAsExtendedCluster=false
oracle.install.crs.config.clusterName=oradbapp-clust
oracle.install.crs.config.gpnp.configureGNS=false
oracle.install.crs.config.autoConfigureClusterNodeVIP=false
oracle.install.crs.config.clusterNodes=oradbserv01.usat.com,oradbserv02.usat.com
oracle.install.crs.configureGIMR=false
oracle.install.asm.configureGIMRDataDG=false

# SECTION H - UPGRADE
oracle.install.crs.config.ignoreDownNodes=false

# Root script execution configuration
oracle.install.crs.rootconfig.executeRootScript=false
```

Dry run (non-destructive, mandatory):

```bash
./gridSetup.sh -silent -dryRunForUpgrade -responseFile /u01/app/grid/19.3.0/install/response/upgrade_response.rsp
```

![Step 3: Dry run, creating the response file](gi_upgrade/Step3_1a_Dry%20run_creating_creating_responsefile.png)
![Step 3: Dry run, executing](gi_upgrade/Step3_1b_Dry%20run_executing_cmd.png)

## Step 4: Cluster verification utility

*(No screenshot for this step — command and outcome recorded from the terminal transcript.)*

```bash
./runcluvfy.sh stage -pre crsinst -upgrade -rolling \
  -src_crshome /u01/app/grid/12.2.0 -dest_crshome /u01/app/grid/19.3.0 \
  -dest_version 19.0.0.0.0 -fixup -verbose
```

## Step 5: Run the silent upgrade

```bash
/u01/app/grid/19.3.0/gridSetup.sh -silent -responseFile /u01/app/grid/19.3.0/install/response/upgrade_response.rsp -logLevel finest
```

![Step 5: Silent GI upgrade launch](gi_upgrade/Step5_Run_the_silent_upgrade.png)

_OUTPUT:_
```
Launching Oracle Grid Infrastructure Setup Wizard...

[WARNING] [INS-13014] Target environment does not meet some optional requirements.
   CAUSE: Some of the optional prerequisites are not met. See logs for details.
   ACTION: Identify the list of failed prerequisite checks from the log...

As a root user, execute the following script(s):
1. /u01/app/grid/19.3.0/rootupgrade.sh

Execute /u01/app/grid/19.3.0/rootupgrade.sh on the following nodes:
[oradbserv01, oradbserv02]

Run the script on the local node first. After successful completion, you can start
the script in parallel on all other nodes, except a node you designate as the last node.

Successfully Setup Software with warning(s).
```

The `INS-13014` warning is about optional (not mandatory) prerequisites and didn't block the upgrade — worth a look in the referenced log at some point, but it didn't need resolving before continuing.

## Step 6: Run `rootupgrade.sh` — rolling, one node at a time

*(No screenshot — captured from the terminal/log output directly.)*

**`oradbserv01` first, as root:**

```bash
/u01/app/grid/19.3.0/rootupgrade.sh
```

_OUTPUT (key milestones from the 18-step upgrade sequence):_
```
2026/07/22 11:12:30 CLSRSC-595: Executing upgrade step 1 of 18: 'UpgradeTFA'.
This version of AHF is older than 365 days. Please upgrade AHF using ahfctl upgrade
2026/07/22 11:15:31 CLSRSC-464: Starting retrieval of the cluster configuration data
PRCT-1470 : failed to reset the Rapid Home Provisioning (RHP) repository
PRCR-1172 : Failed to execute "srvmhelper" for -getmgmtdbnodename
CRS-4672: Successfully backed up the Voting File for Cluster Synchronization Service.
2026/07/22 11:15:49 CLSRSC-515: Starting OCR manual backup.
2026/07/22 11:16:02 CLSRSC-516: OCR manual backup successful.
2026/07/22 11:16:07 CLSRSC-486: At this stage of upgrade, the OCR has changed.
 Any attempt to downgrade the cluster after this point will require a complete cluster outage to restore the OCR.
2026/07/22 11:17:10 CLSRSC-468: Setting Oracle Clusterware and ASM to rolling migration mode
CRS-1131: The cluster was successfully set to rolling upgrade mode.
ASM configuration upgraded in local node successfully.
2026/07/22 11:21:29 CLSRSC-343: Successfully started Oracle Clusterware stack
2026/07/22 11:24:05 CLSRSC-325: Configure Oracle Grid Infrastructure for a Cluster ... succeeded
```

Two things worth noting from this log, not treated as blockers:
- `PRCT-1470`/`PRCR-1172` (failed to reset the RHP repository / get the management database node name) are expected here — this cluster deliberately has no GIMR/Management Database configured (`configureGIMR=false` in the response file above), so there's no MGMTDB node name to retrieve. Consistent with the environment, not a real failure.
- The AHF-staleness notice ("This version of AHF is older than 365 days") is a real, standing follow-up — `ahfctl upgrade` should be run at some point, just not blocking this upgrade.
- **`CLSRSC-486` is the important one to internalize:** once this point is reached, the OCR has changed, and downgrading the cluster from here requires a full cluster outage to restore it. This is the point of no easy return for this rolling upgrade.

Then `oradbserv02`, same command, same home path, only after node 1 completed successfully. (The transcript captured node 1's run explicitly; node 2's isn't shown in the pasted log, but the post-upgrade `activeversion` check below confirms it completed, since the cluster-wide active version only advances once every node has finished.)

## Step 7: Finish configuration tools

```bash
/u01/app/grid/19.3.0/gridSetup.sh -executeConfigTools -responseFile /u01/app/grid/19.3.0/install/response/upgrade_response.rsp -silent
```

![Step 7: executeConfigTools output](gi_upgrade/Step7_Finish_configuration_tools.png)

## Step 8: Verify

```bash
crsctl query crs activeversion
crsctl query crs softwareversion
crsctl check cluster -all
crsctl check crs
crsctl stat res -t
```

![Step 8: Post-upgrade verification](gi_upgrade/Step8_Verify.png)

_OUTPUT:_
```
[grid@oradbserv01 ~]$ crsctl query crs activeversion
Oracle Clusterware active version on the cluster is [19.0.0.0.0]
[grid@oradbserv01 ~]$ crsctl query crs softwareversion
Oracle Clusterware version on node [oradbserv01] is [19.0.0.0.0]
[grid@oradbserv01 ~]$ crsctl check cluster -all
**************************************************************
oradbserv01:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
oradbserv02:
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
**************************************************************
[grid@oradbserv01 ~]$ crsctl check crs
CRS-4638: Oracle High Availability Services is online
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
```

**Result: passed.** Both `activeversion` and `softwareversion` report `19.0.0.0.0`, and both nodes show every clusterware service online — the rolling upgrade held the cluster up throughout, exactly the point of doing it this way instead of a full-outage upgrade.

## Known issues and notes

- Patch 28553832 was **not required** — already superseded by 33678030, which was already installed on the old 12.2.0.1 home.
- `compatible.asm`/`compatible.rdbms` on `DATA01`/`DATA02`/`FRA01` intentionally stay at `12.1.0.0.0` — not raised in this phase. That bump is one-way and happens after the RDBMS-tier upgrade is validated.
- **GI RU vs. planned RDBMS RU version mismatch** (`19.29.0.0.251021` vs. the `19.31.0.0.260421` referenced for the database tier) — flagged above under Environment baseline, not yet resolved. Decide before Phase 4/5 whether to bring GI to `19.31` or proceed with the current `19.29` deliberately.
- AHF is more than 365 days out of date per the `rootupgrade.sh` log — `ahfctl upgrade` is a standing follow-up, not urgent.
- `PRCT-1470`/`PRCR-1172` RHP/MGMTDB warnings during `rootupgrade.sh` are expected given this cluster has no GIMR configured — not a real error.
- Once past `CLSRSC-486` in the `rootupgrade.sh` log, downgrading requires a full cluster outage to restore the OCR — this was the actual point of no easy return in this upgrade, not the start of the process.

## Rollback plan for this phase

Not needed — the upgrade completed cleanly. For reference, had a node failed partway: Oracle's rolling-upgrade downgrade path (called from the old GI home, before every node completes) is the supported recovery up until `CLSRSC-486` is logged; after that point, only a full cluster outage restores the OCR to a pre-upgrade state. VM snapshots taken before Step 1 cover the OS/binaries/config layer, but not the ASM-hosted OCR/voting files (shared/multi-attach disks aren't snapshotted in this lab).

## Still open

- [ ] Decide GI RU version alignment (`19.29` vs `19.31`) before starting the RDBMS-tier phase.
- [ ] Run `ahfctl upgrade` at some point (non-blocking).
- [ ] Review the `INS-13014` optional-prerequisites log for anything worth addressing before the next phase.
