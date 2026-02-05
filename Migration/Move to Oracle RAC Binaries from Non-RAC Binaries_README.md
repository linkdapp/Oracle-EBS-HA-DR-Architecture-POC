
## Enabling the RAC Option for Oracle 12c database binaries.

# Overview

Advantages of Cloning Oracle Home in RAC Implementation (Without New Server Install)

 - Speed & Efficiency: 
 
   Quickly replicates existing Oracle software (binaries, patches) across RAC nodes without full OUI 
   re-installation on each.
      
 - Consistency: 
   Ensures identical configurations, versions, and patches on all nodes, reducing compatibility issues.
   
 - Error Reduction: Avoids manual setup mistakes; clone once-installed home to convert single-instance 
   to RAC or expand clusters.
   
 - Resource Savings: 
   Minimizes downtime and CPU/disk usage compared to fresh installs, ideal for existing multi-node setups.


# Assumption: 

 - Single RDBMS has been installed and an existing single Instance database has been created.

 - Grid Infrastructure is already installed, configured, and running successfully on all Nodes  in the cluster. If you are interested in seeing how I added a converted this single node into a RAC cluster node the refer to my article : <insert_article_link_here>
			

%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

/showing_that_rac_option_is_off_in_binaries1.png)


1. Find out if Oracle binary is RAC enabled: (Linux and Unix) Will not work on AIX.

```bash
	ar -t $ORACLE_HOME/rdbms/lib/libknlopt.a|grep kcsm.o
```

If above command does not return anything, RAC option is not linked in. 
A RAC enabled oracle binary should return "kcsm.o".

2. To check whether a running instance is a RAC instance :

```bash
	ps -ef| grep lmon | grep <ORACLE_SID>
```	
	
showing_database_status2.png

	Only RAC instances have lmon background process.
	
	
3. Check cluster_database parameter
	
```bash	
	sqlplus / as sysdba
	
	show parameter cluster_database
```
	
showing_that_rac_option3.png

	Output "true" means it's RAC instance but this is not reliable as a RAC instance may have cluster_database set to false during
	maintenance period.
	
	
	
	col comp_name for a40
	col status for a12
	col comp_id for a10
	col version for a10
	select substr(comp_name,1,40) comp_name, substr(comp_id,1,10)  
	comp_id,substr(version,1,12) version,status from dba_registry ;
	

*****************************************************************

BACKUP the SERVER: TAKE A SNAPSHOT.

*************************************************************************/


4. Execute the following on the ORACLE_HOME:

  - As ORACLE_HOME owner, stop all resources (database, listener, ASM etc) that's running from the 
    home. When stopping database, use NORMAL or IMMEDIATE option.

  - BACKUP ORACLE_HOME:

takiing_a_backup_of_oracle_home_b4_relink4a.png

```bash
	cd /u01/app/oracle/product/
	
	sudo tar -czvf 12.2.0.tar.gz 12.2.0/
```
	
 - As ORACLE_HOME owner, execute the following to relink:

executing_make_rac_on4b.png

```bash
	cd $ORACLE_HOME/rdbms/lib/

	make -f ins_rdbms.mk rac_on ioracle
```
	
- Verify successfully completed:

```bash
	make_rac_on_completed_binaries_RAC_enabled4b.png

	ar -t $ORACLE_HOME/rdbms/lib/libknlopt.a|grep kcsm.o
	kcsm.o    <--------- The kcsm.o file indicates ORACLE RDBMS is RAC enabled.
```
	

### As a side note:

	If interconnect is infiniband and RDS protocol is being used instead of UDP:
	9/28/2019 Document 284785.1
	https://support.oracle.com/epmos/faces/DocumentDisplay?_adf.ctrl-state=a287ekphm_4&id=284785.1 3/3
	cd $ORACLE_HOME/rdbms/lib
	make -f ins_rdbms.mk ipc_rds ioracle
	Caution: confirm infiniband interconnect and RDS protocol before executing it

	Note: If you are changing more than 1 home, repeat the make command for all homes.


  #                               Clonning the
  #                      ┌─┤ Oracle RAC enabled RDBMS HOME  ├──┐


5.  On the source server (Node1) Installation Node.

	Tar the Oracle RAC enabled ORACLE_HOME 
	
```bash
	cd /u01/app/oracle/product/
	
	sudo tar -czvf RAC12.2.0.tar.gz 12.2.0
```	
	
6. 	Copy the tar file to the target server (Node2) destination:/u01/app/oracle/product

```bash
	scp RAC12.2.0.tar.gz oracle@oradbserv02:/u01/app/oracle/product
```	
	
7. 	Log into the target server and untar the New Oracle RAC enabled home.

```bash
	cd /u01/app/oracle/product/
	
	sudo tar -xzvf RAC12.2.0.tar.gz
	
	ls -lrt
```	
	
8. 	Clone RAC home as ORACLE_HOME owner ( oracle user here ) on each node by specifying right 
    LOCAL_NODE for each node:

```bash
	export ORACLE_HOME=/u01/app/oracle/product/12.2.0/db_1
	
	export PATH=$PATH:$ORACLE_HOME/bin:/usr/bin
	
	perl $ORACLE_HOME/clone/bin/clone.pl ORACLE_HOME=$ORACLE_HOME ORACLE_HOME_NAME=OraDB12Home1 \
	ORACLE_BASE=/u01/app/oracle OSDBA_GROUP=oinstall OSOPER_GROUP=oinstall \ 
	"CLUSTER_NODES={oradbserv01.usat.com,oradbserv02.usat.com}" "LOCAL_NODE=oradbserv02.usat.com"
```	
	
### Sample output

```bash   
oradbserv02.usat.com-oraebs-/u01/app/oracle/product
>
>perl $ORACLE_HOME/clone/bin/clone.pl ORACLE_HOME=$ORACLE_HOME ORACLE_HOME_NAME=OraDB12Home1 ORACLE_BASE=/u01/app/oracle OSDBA_GROUP=oinstall OSOPER_GROUP=oinstall "CLUSTER_NODES={oradbserv01.usat.com,oradbserv02.usat.com}" "LOCAL_NODE=oradbserv02.usat.com"
./runInstaller -clone -waitForCompletion  "ORACLE_HOME=/u01/app/oracle/product/12.2.0/db_1" "ORACLE_HOME_NAME=OraDB12Home1" "ORACLE_BASE=/u01/app/oracle" "oracle_install_OSDBA=oinstall" "oracle_install_OSOPER=oinstall" "CLUSTER_NODES={oradbserv01.usat.com,oradbserv02.usat.com}" "LOCAL_NODE=oradbserv02.usat.com" -silent -paramFile /u01/app/oracle/product/12.2.0/db_1/clone/clone_oraparam.ini
Starting Oracle Universal Installer...

Checking Temp space: must be greater than 500 MB.   Actual 16712 MB    Passed
Checking swap space: must be greater than 500 MB.   Actual 16382 MB    Passed
Preparing to launch Oracle Universal Installer from /tmp/OraInstall2019-10-29_10-00-51PM. Please wait ...You can find the log of this install session at:
 /u01/app/oraInventory/logs/cloneActions2019-10-29_10-00-51PM.log
..................................................   5% Done.
..................................................   10% Done.
..................................................   15% Done.
..................................................   20% Done.
..................................................   25% Done.
..................................................   30% Done.
..................................................   35% Done.
..................................................   40% Done.
..................................................   45% Done.
..................................................   50% Done.
..................................................   55% Done.
..................................................   60% Done.
..................................................   65% Done.
..................................................   70% Done.
..................................................   75% Done.
..................................................   80% Done.
..................................................   85% Done.
..........
Copy files in progress.

Copy files successful.

Link binaries in progress.

Link binaries successful.

Setup files in progress.

Setup files successful.

Setup Inventory in progress.

Setup Inventory successful.

Finish Setup successful.
The cloning of OraDB12Home1 was successful.
Please check '/u01/app/oraInventory/logs/cloneActions2019-10-29_10-00-51PM.log' for more details.

Setup Oracle Base in progress.

Setup Oracle Base successful.
..................................................   95% Done.

As a root user, execute the following script(s):
        
		1. /u01/app/oracle/product/12.2.0/db_1/root.sh

Execute /u01/app/oracle/product/12.2.0/db_1/root.sh on the following nodes:
[oradbserv02.usat.com]


..................................................   100% Done.
```


9. As a root user, execute the *root.sh* script

```bash
	/u01/app/oracle/product/12.2.0/db_1/root.sh
```
	

10. Verify the NEW Node2 and Node1 oraInventory contents and RAC is enabled.

enabling_rac_option_10.png
	
```bash
	ar -t $ORACLE_HOME/rdbms/lib/libknlopt.a|grep kcsm.o
	kcsm.o
	
	cat /u01/app/oraInventory/ContentsXML/inventory.xml
	
	<?xml version="1.0" standalone="yes" ?>
	<!-- Copyright (c) 1999, 2014, Oracle and/or its affiliates.
	All rights reserved. -->
	<!-- Do not modify the contents of this file by hand. -->
	<INVENTORY>
	<VERSION_INFO>
	<SAVED_WITH>12.2.0.0</SAVED_WITH>
	<MINIMUM_VER>2.1.0.6.0</MINIMUM_VER>
	</VERSION_INFO>
	<HOME_LIST>
	<HOME NAME="OraGI12Home1" LOC="/u01/app/12.2.0.1/grid" TYPE="O" IDX="1" CRS="true"/>
	<HOME NAME="OraDB12Home1" LOC="/u01/app/oracle/product/12.2.0/db_1" TYPE="O" IDX="2">
	<NODE_LIST>
		<NODE NAME="oradbserv02.usat.com"/>
		<NODE NAME="oradbserv01.usat.com"/>
	</NODE_LIST>
	</HOME>
	</HOME_LIST>
	<COMPOSITEHOME_LIST>
	</COMPOSITEHOME_LIST>
	</INVENTORY>
```	
	

### Notes:


How to Check Whether Oracle Binary/Instance is RAC Enabled and Relink Oracle Binary in RAC
(Doc ID 284785.1)

 
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!          SUCCESSFUL                !!!!!!!!!!!!!!!!!!!!!!!!
