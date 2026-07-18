# Verifying the EBS 12.2 to 2-Node RAC Conversion

## Status

This is a working document for Phase 2 of the project (see [PROJECT_ROADMAP.md](../../PROJECT_ROADMAP.md)). Grid Infrastructure is installed on both nodes, `rconfig` has completed successfully, and SCAN/AutoConfig are done on the app tier per the [conversion walkthrough](single-instance-to-2node-rac-README.md). This doc is the checklist to run through in the lab to confirm the cluster is genuinely healthy end to end before building Data Guard on top of it — not just "the install finished without an error," but "every layer actually works."

Fill in actual output/screenshots as you run each section. Anything that doesn't come back clean is worth fixing now, before a standby DB and FSFO observer add more moving parts on top.

## Why this matters (plain language)

A RAC install can complete successfully and still have a node that isn't really pulling its weight — a service that only ever fails over to one instance, an ASM disk group that's mounted on one node but not the other, a SCAN listener that resolves but doesn't load-balance. None of that shows up as an "error" during setup. It shows up during an actual node failure, which is the worst possible time to discover it. This checklist exists to find those gaps on a Tuesday afternoon instead of during a real outage.

## 1. Clusterware health

```bash
# Run as grid user, from either node
crsctl check cluster -all
crsctl check crs
crsctl query crs softwareversion
crsctl stat res -t
```

What "good" looks like: `crsctl check cluster -all` reports the CRS stack online on both nodes; `crsctl stat res -t` shows every resource (ASM, listener, ONS, SCAN VIP, database, services) as `ONLINE` on its expected node(s), not just `OFFLINE` with no error — a resource that's supposed to be running and isn't is exactly what a failover would expose later.

_Paste output here:_

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
[grid@oradbserv01 ~]$
[grid@oradbserv01 ~]$
[grid@oradbserv01 ~]$ crsctl check crs
CRS-4638: Oracle High Availability Services is online
CRS-4537: Cluster Ready Services is online
CRS-4529: Cluster Synchronization Services is online
CRS-4533: Event Manager is online
[grid@oradbserv01 ~]$
[grid@oradbserv01 ~]$
[grid@oradbserv01 ~]$ crsctl query crs softwareversion
Oracle Clusterware version on node [oradbserv01] is [12.2.0.1.0]
[grid@oradbserv01 ~]$
[grid@oradbserv01 ~]$
[grid@oradbserv01 ~]$ crsctl stat res -t
--------------------------------------------------------------------------------
Name           Target  State        Server                   State details
--------------------------------------------------------------------------------
Local Resources
--------------------------------------------------------------------------------
ora.ASMNET1LSNR_ASM.lsnr
               ONLINE  ONLINE       oradbserv01              STABLE
               ONLINE  ONLINE       oradbserv02              STABLE
ora.DATA01.dg
               ONLINE  ONLINE       oradbserv01              STABLE
               ONLINE  ONLINE       oradbserv02              STABLE
ora.DATA02.dg
               ONLINE  ONLINE       oradbserv01              STABLE
               ONLINE  ONLINE       oradbserv02              STABLE
ora.FRA01.dg
               ONLINE  ONLINE       oradbserv01              STABLE
               ONLINE  ONLINE       oradbserv02              STABLE
ora.LISTENER.lsnr
               ONLINE  ONLINE       oradbserv01              STABLE
               ONLINE  ONLINE       oradbserv02              STABLE
ora.net1.network
               ONLINE  ONLINE       oradbserv01              STABLE
               ONLINE  ONLINE       oradbserv02              STABLE
ora.ons
               ONLINE  ONLINE       oradbserv01              STABLE
               ONLINE  ONLINE       oradbserv02              STABLE
--------------------------------------------------------------------------------
Cluster Resources
--------------------------------------------------------------------------------
ora.LISTENER_SCAN1.lsnr
      1        ONLINE  ONLINE       oradbserv02              STABLE
ora.LISTENER_SCAN2.lsnr
      1        ONLINE  ONLINE       oradbserv01              STABLE
ora.LISTENER_SCAN3.lsnr
      1        ONLINE  ONLINE       oradbserv01              STABLE
ora.MGMTLSNR
      1        OFFLINE OFFLINE                               STABLE
ora.asm
      1        ONLINE  ONLINE       oradbserv01              Started,STABLE
      2        ONLINE  ONLINE       oradbserv02              Started,STABLE
      3        OFFLINE OFFLINE                               STABLE
ora.cvu
      1        ONLINE  ONLINE       oradbserv01              STABLE
ora.ebsappdb.db
      1        ONLINE  ONLINE       oradbserv01              Open,HOME=/u01/app/o
                                                             racle/product/12.2.0
                                                             /db_1,STABLE
      2        ONLINE  ONLINE       oradbserv02              Open,HOME=/u01/app/o
                                                             racle/product/12.2.0
                                                             /db_1,STABLE
ora.ebsappdb.ebsapp_serv.svc
      1        ONLINE  ONLINE       oradbserv01              STABLE
      2        ONLINE  ONLINE       oradbserv02              STABLE
ora.oradbserv01.vip
      1        ONLINE  ONLINE       oradbserv01              STABLE
ora.oradbserv02.vip
      1        ONLINE  ONLINE       oradbserv02              STABLE
ora.qosmserver
      1        ONLINE  ONLINE       oradbserv01              STABLE
ora.scan1.vip
      1        ONLINE  ONLINE       oradbserv02              STABLE
ora.scan2.vip
      1        ONLINE  ONLINE       oradbserv01              STABLE
ora.scan3.vip
      1        ONLINE  ONLINE       oradbserv01              STABLE
--------------------------------------------------------------------------------
[grid@oradbserv01 ~]$


## 2. ASM and shared storage

```bash
srvctl status asm -node oradbserv01 -detail -verbose
srvctl status asm -node oradbserv02 -detail -verbose
asmcmd lsdg
```

What "good" looks like: ASM instance up on both nodes, all disk groups (`DATA`, `FRA`) mounted on both nodes with matching `FREE_MB`/`TOTAL_MB` — a mismatch usually means a disk was only ever attached to one node (an easy miss when following the VirtualBox `storageattach` steps from the conversion doc).

_Paste output here:_


[oracle@oradbserv02 ~]$ srvctl status asm -node oradbserv01 -detail -verbose
ASM is running on oradbserv01
ASM is enabled on node oradbserv01.
Detailed state on node oradbserv01: Started
Detailed state on node oradbserv02: Started
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$ srvctl status asm -node oradbserv02 -detail -verbose
ASM is running on oradbserv02
ASM is enabled on node oradbserv02.
Detailed state on node oradbserv01: Started
Detailed state on node oradbserv02: Started
[oracle@oradbserv02 ~]$

[grid@oradbserv01 ~]$ asmcmd lsdg
State    Type    Rebal  Sector  Logical_Sector  Block        AU  Total_MB  Free_MB  Req_mir_free_MB  Usable_file_MB  Offline_disks  Voting_files  Name
MOUNTED  EXTERN  N         512             512   4096  67108864    309888   117568                0          117568              0             N  DATA01/
MOUNTED  EXTERN  N         512             512   4096  16777216    309984   309360                0          309360              0             Y  DATA02/
MOUNTED  EXTERN  N         512             512   4096  67108864    149888   131520                0          131520              0             N  FRA01/
[grid@oradbserv01 ~]$


## 3. Database and instances

```bash
srvctl status database -db ebsappdb -verbose
srvctl config database -db ebsappdb
sqlplus / as sysdba <<'EOF'
select instance_name, host_name, status, database_status from gv$instance;
EOF
```

What "good" looks like: both instances (`ebsappdb1`, `ebsappdb2`) report `OPEN` and `ACTIVE`, on the correct hosts.

_Paste output here:_

[oracle@oradbserv02 ~]$ srvctl status database -db ebsappdb -verbose
Instance ebsappdb1 is running on node oradbserv01 with online services ebsapp_serv. Instance status: Open,HOME=/u01/app/oracle/product/12.2.0/db_1.
Instance ebsappdb2 is running on node oradbserv02 with online services ebsapp_serv. Instance status: Open,HOME=/u01/app/oracle/product/12.2.0/db_1.
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$ srvctl config database -db ebsappdb
Database unique name: ebsappdb
Database name: ebsappdb
Oracle home: /u01/app/oracle/product/12.2.0/db_1
Oracle user: oracle
Spfile: +DATA01/spfileebsappdb.ora
Password file: +DATA01/orapwebsappdb
Domain:
Start options: open
Stop options: immediate
Database role: PRIMARY
Management policy: AUTOMATIC
Server pools:
Disk Groups: DATA01
Mount point paths:
Services: ebsapp_serv
Type: RAC
Start concurrency:
Stop concurrency:
OSDBA group: oinstall
OSOPER group: oinstall
Database instances: ebsappdb1,ebsappdb2
Configured nodes: oradbserv01,oradbserv02
Database is administrator managed
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$ sqlplus / as sysdba <<'EOF'
> select instance_name, host_name, status, database_status from gv$instance;
> EOF

SQL*Plus: Release 12.1.0.2.0 Production on Wed Jul 15 20:05:22 2026

Copyright (c) 1982, 2014, Oracle.  All rights reserved.


Connected to:
Oracle Database 12c Enterprise Edition Release 12.1.0.2.0 - 64bit Production
With the Partitioning, Real Application Clusters, Automatic Storage Management, OLAP,
Advanced Analytics and Real Application Testing options

SQL>
INSTANCE_NAME
----------------
HOST_NAME                                                        STATUS
---------------------------------------------------------------- ------------
DATABASE_STATUS
-----------------
ebsappdb2
oradbserv02.usat.com                                             OPEN
ACTIVE

ebsappdb1
oradbserv01.usat.com                                             OPEN
ACTIVE

INSTANCE_NAME
----------------
HOST_NAME                                                        STATUS
---------------------------------------------------------------- ------------
DATABASE_STATUS
-----------------


SQL> Disconnected

## 4. Services (this is the one most EBS RAC setups get wrong)

```bash
srvctl status service -db ebsappdb -service ebsapp_serv -verbose
srvctl config service -db ebsappdb -service ebsapp_serv
```

What "good" looks like: the `ebsapp_serv` service shows `preferred` instances on **both** nodes, not just one — check this against the `srvctl add service` command in the conversion doc (`-preferred ebsappdb1,ebsappdb2`). A service configured with only one preferred instance will not fail over the app tier's connections if that instance goes down; it just fails, since the "preferred" instance isn't there to take over. This is the single most common RAC-for-EBS misconfiguration and won't surface until node 1 actually goes down.

_Paste output here:_

[oracle@oradbserv02 ~]$ srvctl status service -db ebsappdb -service ebsapp_serv -verbose
Service ebsapp_serv is running on instance(s) ebsappdb1,ebsappdb2
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$ srvctl config service -db ebsappdb -service ebsapp_serv
Service name: ebsapp_serv
Server pool:
Cardinality: 2
Disconnect: false
Service role: PRIMARY
Management policy: AUTOMATIC
DTP transaction: false
AQ HA notifications: false
Global: false
Commit Outcome: false
Failover type: SELECT
Failover method: BASIC
TAF failover retries:
TAF failover delay:
Connection Load Balancing Goal: LONG
Runtime Load Balancing Goal: NONE
TAF policy specification: BASIC
Edition:
Pluggable database name:
Maximum lag time: ANY
SQL Translation Profile:
Retention: 86400 seconds
Replay Initiation Time: 300 seconds
Session State Consistency:
GSM Flags: 0
Service is enabled
Preferred instances: ebsappdb1,ebsappdb2
Available instances:
[oracle@oradbserv02 ~]$


## 5. SCAN and listener

```bash
srvctl status scan
srvctl status scan_listener
lsnrctl status
nslookup scan-oradbserv
```

What "good" looks like: SCAN resolves to all 3 configured IPs (round-robin), and `srvctl status scan_listener` shows all SCAN listeners running, ideally distributed across nodes rather than stacked on one.

_Paste output here:_

[oracle@oradbserv02 ~]$ srvctl status scan
SCAN VIP scan1 is enabled
SCAN VIP scan1 is running on node oradbserv02
SCAN VIP scan2 is enabled
SCAN VIP scan2 is running on node oradbserv01
SCAN VIP scan3 is enabled
SCAN VIP scan3 is running on node oradbserv01
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$ srvctl status scan_listener
SCAN Listener LISTENER_SCAN1 is enabled
SCAN listener LISTENER_SCAN1 is running on node oradbserv02
SCAN Listener LISTENER_SCAN2 is enabled
SCAN listener LISTENER_SCAN2 is running on node oradbserv01
SCAN Listener LISTENER_SCAN3 is enabled
SCAN listener LISTENER_SCAN3 is running on node oradbserv01
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$ lsnrctl status

LSNRCTL for Linux: Version 12.1.0.2.0 - Production on 15-JUL-2026 20:07:00

Copyright (c) 1991, 2014, Oracle.  All rights reserved.

Connecting to (ADDRESS=(PROTOCOL=tcp)(HOST=)(PORT=1521))
STATUS of the LISTENER
------------------------
Alias                     LISTENER
Version                   TNSLSNR for Linux: Version 12.2.0.1.0 - Production
Start Date                15-JUL-2026 19:23:31
Uptime                    0 days 0 hr. 43 min. 28 sec
Trace Level               off
Security                  ON: Local OS Authentication
SNMP                      OFF
Listener Parameter File   /u01/app/grid/12.2.0/network/admin/listener.ora
Listener Log File         /u01/app/oracle/diag/tnslsnr/oradbserv02/listener/alert/log.xml
Listening Endpoints Summary...
  (DESCRIPTION=(ADDRESS=(PROTOCOL=ipc)(KEY=LISTENER)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=192.168.56.127)(PORT=1521)))
  (DESCRIPTION=(ADDRESS=(PROTOCOL=tcp)(HOST=192.168.56.132)(PORT=1521)))
Services Summary...
Service "+ASM" has 1 instance(s).
  Instance "+ASM2", status READY, has 1 handler(s) for this service...
Service "+ASM_DATA01" has 1 instance(s).
  Instance "+ASM2", status READY, has 1 handler(s) for this service...
Service "+ASM_DATA02" has 1 instance(s).
  Instance "+ASM2", status READY, has 1 handler(s) for this service...
Service "+ASM_FRA01" has 1 instance(s).
  Instance "+ASM2", status READY, has 1 handler(s) for this service...
Service "ebs_patch" has 1 instance(s).
  Instance "ebsappdb2", status READY, has 1 handler(s) for this service...
Service "ebsapp_serv" has 1 instance(s).
  Instance "ebsappdb2", status READY, has 1 handler(s) for this service...
Service "ebsappdb" has 1 instance(s).
  Instance "ebsappdb2", status READY, has 1 handler(s) for this service...
The command completed successfully
[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$ nslookup scan-oradbserv
Server:         192.168.56.126
Address:        192.168.56.126#53

Name:   scan-oradbserv.usat.com
Address: 192.168.56.161
Name:   scan-oradbserv.usat.com
Address: 192.168.56.171
Name:   scan-oradbserv.usat.com
Address: 192.168.56.151

[oracle@oradbserv02 ~]$
[oracle@oradbserv02 ~]$


## 6. App tier connectivity through SCAN

From the app tier (`orappsserv01`), confirm the JDBC connect string actually uses the SCAN name and service name (not a direct host:port to one instance) — this is the part that makes the whole exercise worthwhile:

```bash
grep -i "s_apps_jdbc_connect_descriptor" $CONTEXT_FILE
tnsping ebsappdb
```

_Paste output here:_

[applmgr@orappsserv01 ~]$ grep -i "s_apps_jdbc_connect_descriptor" $CONTEXT_FILE
         <jdbc_url oa_var="s_apps_jdbc_connect_descriptor">jdbc:oracle:thin:@(DESCRIPTION=(ADDRESS_LIST=(LOAD_BALANCE=YES)(FAILOVER=YES)(ADDRESS=(PROTOCOL=tcp)(HOST=oradbserv01.usat.com)(PORT=1521))(ADDRESS=(PROTOCOL=tcp)(HOST=oradbserv02.usat.com)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=ebsapp_serv)))</jdbc_url>
[applmgr@orappsserv01 ~]$
[applmgr@orappsserv01 ~]$ tnsping ebsapp_serv

TNS Ping Utility for Linux: Version 10.1.0.5.0 - Production on 15-JUL-2026 20:11:24

Copyright (c) 1997, 2003, Oracle.  All rights reserved.

Used parameter files:

TNS-03505: Failed to resolve name
[applmgr@orappsserv01 ~]$

### Root cause and fix

Editing `s_apps_jdbc_connect_descriptor` directly and rerunning `adconfig.sh` did not fix this — the descriptor reverted to the old two-host address list every time, no matter which tool was used to set it: a direct file edit + `adconfig.sh`, or a more targeted `UpdateContext` call followed by `txkManageDBConnectionPool.pl -options=updateDSOffline` and `adgentns.pl` (which did successfully push the fix into the WebLogic DataSource, but the context file itself still lost the change on the next AutoConfig pass).

Two explanations were tested before finding the real one.

The first theory was that the database itself was the source of the stale value. EBS context files carry a serial number (`s_contextserial`) that's mirrored in a database table, `FND_OAM_CONTEXT_FILES`, and if that stored copy is treated as authoritative, AutoConfig can pull the old value back from the database instead of the file system. To test this, the table was backed up (`CREATE TABLE applsys.fnd_oam_context_files_bak AS SELECT * FROM applsys.fnd_oam_context_files`), the row for this app tier's context file was deleted, and `adconfig.sh` was rerun. This ruled the theory out rather than confirming it: the row simply reappeared afterward with the *same* broken descriptor, which means AutoConfig regenerated the bad value itself and re-uploaded it — the database wasn't the one overwriting the file, AutoConfig was overwriting both.

That pointed at the real cause: the context variable `s_jdbc_connect_descriptor_generation` was `TRUE` (its default). With this flag on, AutoConfig doesn't treat `s_apps_jdbc_connect_descriptor` as a fixed value at all — it reconstructs it on every run, using RAC instance/node information it resolves elsewhere in the environment (the EBS node registry, `FND_NODES`). That registry still had the pre-SCAN node entries for `oradbserv01` and `oradbserv02` as individual hosts, so every AutoConfig run rebuilt the exact same old load-balanced, two-host descriptor regardless of what the context file said going in.

The fix:

```bash
# Stop AutoConfig from regenerating the JDBC URL on every run
java -classpath <ad classpath> oracle.apps.ad.context.UpdateContext $CONTEXT_FILE s_jdbc_connect_descriptor_generation FALSE

# Re-apply the SCAN-based descriptor
java -classpath <ad classpath> oracle.apps.ad.context.UpdateContext $CONTEXT_FILE s_apps_jdbc_connect_descriptor \
  "jdbc:oracle:thin:@(DESCRIPTION=(LOAD_BALANCE=YES)(FAILOVER=YES)(ADDRESS_LIST=(ADDRESS=(PROTOCOL=TCP)(HOST=scan-oradbserv)(PORT=1521)))(CONNECT_DATA=(SERVICE_NAME=ebsapp_serv)))"

# Confirm it survives a full AutoConfig run
adconfig.sh contextfile=$CONTEXT_FILE
grep -i "s_apps_jdbc_connect_descriptor" $CONTEXT_FILE
```

This held through a subsequent `adconfig.sh` run — the first time the value survived an AutoConfig pass since the RAC conversion. **Confirmed:** the WebLogic `EBSDataSource` connection pool now reflects the SCAN-based descriptor.

### A second instance of the same gap: tnsnames.ora

The JDBC fix above only covers WebLogic — its connect descriptor is embedded directly in the context variable and never touches `tnsnames.ora`. Everything else that connects via a TNS alias (Concurrent Manager, `sqlplus`, and any other OCI-based client) goes through `TWO_TASK`, which on this app tier is set to `ebsappdb`:

```bash
env | grep -i two_task
# TWO_TASK=ebsappdb
```

`ebsappdb` is generated by `adgentns.pl` into `tnsnames.ora`, and it had the identical problem: single host (`oradbserv01`), single instance, no `ADDRESS_LIST`, no SCAN — meaning Concurrent Manager had no RAC failover at all. This was masked because the app "worked fine" with both nodes up; it would only have surfaced the next time `oradbserv01` went down.

`tnsnames.ora` isn't governed by `s_jdbc_connect_descriptor_generation` — that flag only controls the JDBC context variable, so this required a separate fix. Editing the generated file directly wasn't an option (AutoConfig overwrites it every run), and correcting the root cause properly means fixing the stale RAC node data in `FND_NODES` — a bigger, still-pending change. Instead, this used EBS's supported customization path: `tnsnames.ora` already references an `IFILE` at the bottom, and Oracle Net gives precedence to whichever definition of a duplicate service name it reads *last* — so a corrected `ebsappdb` entry placed in the ifile overrides the broken auto-generated one, and survives every future `adgentns.pl` run since AutoConfig never touches the ifile.

```bash
cat > /u01/app/oracle/ebsR122/fs1/inst/apps/ebsappdb_orappsserv01/ora/10.1.2/network/admin/ebsappdb_orappsserv01_ifile.ora <<'EOF'
ebsappdb=
    (DESCRIPTION=
        (ADDRESS_LIST=
            (LOAD_BALANCE=YES)
            (FAILOVER=YES)
            (ADDRESS=(PROTOCOL=TCP)(HOST=scan-oradbserv)(PORT=1521))
        )
        (CONNECT_DATA=
            (SERVICE_NAME=ebsapp_serv)
        )
    )

ebsappdb_patch=
    (DESCRIPTION=
        (ADDRESS_LIST=
            (LOAD_BALANCE=YES)
            (FAILOVER=YES)
            (ADDRESS=(PROTOCOL=TCP)(HOST=scan-oradbserv)(PORT=1521))
        )
        (CONNECT_DATA=
            (SERVICE_NAME=ebs_patch)
        )
    )
EOF
```

**Confirmed:** `tnsping ebsappdb` resolves through SCAN, and a live `sqlplus` login from the app tier through `TWO_TASK=ebsappdb` succeeded.

### Dual file system: fs1, fs2, and a near-miss with FND_OAM_CONTEXT_FILES

EBS 12.2's dual-filesystem model means `fs1` and `fs2` are peer editions, each with a complete, independent tree — `EBSapps`, `inst`, `FMW_Home` — not shared. Every fix above was applied only to fs1, the current run edition. `fs2`, currently the idle patch edition, has an identical but separate context file, env files, and `tnsnames.ora`, deliberately left untouched during this investigation. (`fs_ne`, the non-editioned file system, is unrelated to any of this — it holds only shared transactional/log data, not app-tier configuration.)

This matters because fs1/fs2 roles swap on every completed patching cycle: whatever is on fs2 today is what goes live after the Delta 14 apply and cutover. `adop phase=prepare` syncs the patch file system from the run file system and reruns AutoConfig as part of that, so some of this may carry over automatically — but that isn't confirmed, particularly for hand-created customizations like the ifile. **Action for the Delta 14 retry: re-check fs2's context descriptor, generation flag, and tnsnames ifile immediately after the next `phase=prepare`, before proceeding further.**

One concrete consequence already surfaced: the `FND_OAM_CONTEXT_FILES` test above deleted the database rows for *both* fs1 and fs2's context files, not just fs1's. fs1's row was automatically re-created the next time `adconfig.sh` ran there; fs2's was not, since fs2 was intentionally left alone. A missing patch-edition row in `FND_OAM_CONTEXT_FILES` can cause `adop phase=prepare` to fail outright, so this was caught and fixed proactively using the supported `CtxSynchronizer` utility to re-upload fs2's (unmodified) context file into the database:

```bash
# run from the fs1 (run edition) environment
$ADJVAPRG oracle.apps.ad.autoconfig.oam.CtxSynchronizer action=upload \
  contextfile='/u01/app/oracle/ebsR122/fs2/inst/apps/ebsappdb_orappsserv01/appl/admin/ebsappdb_orappsserv01.xml' \
  logfile=/tmp/patchctxupload.log
```

**Confirmed:** both fs1 and fs2 rows present again in `FND_OAM_CONTEXT_FILES`.

### Still open

- `FND_NODES` still has stale per-node RAC registrations — the actual structural root cause behind both the JDBC and tnsnames symptoms. `s_jdbc_connect_descriptor_generation=FALSE` and the tnsnames ifile are durable workarounds, not fixes to that source data.
- Confirm `s_jdbc_connect_descriptor_generation` is currently `FALSE` — it was briefly flipped to `TRUE` mid-troubleshooting to test an unrelated theory and needs to be reset before this is considered closed.
- `ebsappdb_patch` (the ifile entry for the patch-edition alias) hasn't been individually `tnsping`-tested yet — needed before the Delta 14 apply.
- fs2 hasn't been checked against any of the fixes above — re-verify after the next `adop phase=prepare` (see above).



## 7. The test that actually matters: kill a node

Nothing above proves failover works — it only proves the configuration looks correct. The real test:

```bash
# From node2, while an app-tier session is active against node1's instance
srvctl stop instance -db ebsappdb -instance ebsappdb1 -o abort
```

Then confirm from the app tier that active sessions reconnect to `ebsappdb2` (via the service, not by hand) and that OEM raises an alert. Document what actually happened here — including anything that _didn't_ fail over cleanly. That gap, if there is one, is more useful content than a clean pass.

**Result: passed.** Failover completed in approximately 2 minutes.

_Screenshot/command output still to be added._

