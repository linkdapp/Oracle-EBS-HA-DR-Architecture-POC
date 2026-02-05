# Apply latest patch only the GI home (Combo of OJVM + GI: p33559966_122010_Linux-x86-64.zip):


opatch version
OPatch Version: 12.2.0.1.6

OPatch succeeded.

mv /u01/app/12.2.0/grid/OPatch /u01/app/12.2.0/grid/OPatch.old

unzip /media/sf_eoracle/Patch/12c/p6880880_122010_Linux-x86-64v12.2.0.1.40.zip -d /u01/app/12.2.0/grid

opatch version
OPatch Version: 12.2.0.1.40

OPatch succeeded.


Apply Patch As Root On Both RAC nodes:

unzip /media/sf_eoracle/Patch/12c/p33559966_122010_Linux-x86-64.zip

opatchauto_apply_patch_nonrolling.png

	cd /u01/app/12.2.0/grid/OPatch
	
	
	./opatchauto report -type patches -logLevel ALL
	
	
	
	opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir /media/sf_eoracle/Patch/12c/33559966/33583921/33587128
	opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir /media/sf_eoracle/Patch/12c/33559966/33583921/33678030
	opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir /media/sf_eoracle/Patch/12c/33559966/33583921/33116894
	opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir /media/sf_eoracle/Patch/12c/33559966/33583921/26839277
	opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir /media/sf_eoracle/Patch/12c/33559966/33583921/33610989
	
	sudo ./opatchauto apply /media/sf_eoracle/Patch/12c/33559966/33583921 -oh /u01/app/12.2.0/grid
	
	OR
	
	opatch prereq CheckConflictAgainstOHWithDetail apply /media/sf_eoracle/Patch/12c/33559966/33583921/33587128  -oh /u01/app/12.2.0/grid
	opatch prereq CheckConflictAgainstOHWithDetail apply /media/sf_eoracle/Patch/12c/33559966/33583921/33678030  -oh /u01/app/12.2.0/grid
	opatch prereq CheckConflictAgainstOHWithDetail apply /media/sf_eoracle/Patch/12c/33559966/33583921/33116894  -oh /u01/app/12.2.0/grid
	opatch prereq CheckConflictAgainstOHWithDetail apply /media/sf_eoracle/Patch/12c/33559966/33583921/26839277  -oh /u01/app/12.2.0/grid
	opatch prereq CheckConflictAgainstOHWithDetail apply /media/sf_eoracle/Patch/12c/33559966/33583921/33610989  -oh /u01/app/12.2.0/grid
	
	
	opatch prereq CheckConflictAgainstOHWithDetail -phBaseDir /media/sf_eoracle/Patch/12c/33559966/33561275
	
	sudo ./opatchauto apply -nonrolling /media/sf_eoracle/Patch/12c/33559966/33561275 -oh /u01/app/12.2.0/grid




---- 
  ---------
         ---------- AS grid user Create the DATA01 disk group with compatibility of 12.1.0.1
					For EBS Datafiles. Database of lower version cannot mount diskgroup with
					higher compatibility. During the installation process gridSetup.sh doesn't 
					allow you to lower diskgroup compatibility.
						
	
>
Click **Specify Disk Group** > **Change Disk Discovery Path**
 

# Follow the instructions Enabling_RAC_Option_Oracle Binary_Single_node_to_RAC_README.md to clone the $ORACLE_HOME.

