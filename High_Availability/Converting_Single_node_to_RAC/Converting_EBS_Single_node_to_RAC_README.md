## Converting Single Instance Oracle E-Business Suite (EBS) Release 12.2 to a 2-Node Oracle RAC for High Availability

# Overview: Why High Availability (HA) with Oracle RAC?

High Availability ensures your EBS system remains operational despite hardware failures, maintenance, or spikes in load. Oracle RAC provides:

 - Redundancy and Failover: Multiple nodes share the database workload; if one node fails, others take over seamlessly.
 
 - Scalability: Load balancing across nodes improves performance for concurrent users and batch jobs.
 
 - Zero Downtime Maintenance: Patch one node while others handle traffic.
 
 - Business Continuity: Reduces unplanned downtime (e.g., from node crashes) and supports features like Parallel Concurrent Processing (PCP) in EBS.
 
 - Integration with Other Tools: Complements Data Guard (for DR) and GoldenGate (for replication), and monitoring via OEM.

Without HA, a single-instance setup risks extended outages. RAC achieves 99.99%+ uptime in production, but requires proper setup for interconnect (private network) and shared storage.

Estimated Time: 4-8 hours for setup, plus testing. Downtime during conversion: ~1-2 hours (can be minimized with careful planning).


# Prerequisites

# Note: 
Note: This process assumes your current setup (OEM Server, single-instance EBS App Server, single-instance DB Server) is on Linux x86-64 (as per your host and common EBS setups). RAC requires shared storage (e.g., via VirtualBox shared disks or NFS for testing). Test in a non-production environment first. Allocate VM resources: e.g., 8 CPUs and 16 GB RAM per node (total ~32 GB for 2 nodes + overhead).

Hardware/VM Setup:

Create 2 VirtualBox VMs: Node1 (oradbserv01). This is the current SINGLE NON-RAC DB server 
                         Node2 (oradbserv02). This is the new node to add (I made a clone of oradbserv01 without the database) 

OS: Oracle Linux 7 (matching your current setup).

Network: 3 NICs per server

 - Public NIC (NAT):                    enp0s3 (eth0): DHCP (Connect Automatically)
 - Private NIC (Host-Only-Adapter):     enp0s8 (eth1): IP=192.168.56.128, Subnet=255.255.255.0, Gateway=192.168.56.1, DNS=<blank>, Search=usat.com (Connect Automatically)
 - Virtual IP (VIP) (Internal Network:  enp0s9 (eth2): IP=192.168.2.128, Subnet=255.255.255.0, Gateway=<blank>, DNS=<blank>, Search=<blank> (Connect Automatically)

Shared Storage: Use VirtualBox shared disks (VDI files) for ASM (Automatic Storage Management). Allocate ~300 GB shared disk for DB files.

```bash
	# Clone Golden Image Linux 7 VDI Disks
	
	"c:\Program Files\Oracle\VirtualBox\VBoxManage" clonehd "G:\virtualbox_vm\BackupLoc\Linux7server\bk_swapdisk01.vdi" "E:\virtualbox_vm\oradbserv02\Disks\oradbserv02_SWAPDISK.vdi"
	"c:\Program Files\Oracle\VirtualBox\VBoxManage" clonehd "G:\virtualbox_vm\BackupLoc\Linux7server\bk_rootdisk01.vdi" "E:\virtualbox_vm\oradbserv02\Disks\oradbserv02_DISK1.vdi"
	
	# Attach Disks to the VM RAC Node2.
	
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv02 --storagectl "SATA" --port 0 --device 0 --type hdd  --medium "E:\virtualbox_vm\oradbserv02\Disks\oradbserv02_SWAPDISK.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv02 --storagectl "SATA" --port 1 --device 0 --type hdd  --medium "E:\virtualbox_vm\oradbserv02\Disks\oradbserv02_DISK1.vdi"

	# Create the disks and associate them with VirtualBox as virtual media.

    # Rcongif will need extra 300 GB to convert the database to RAC in scenario
			
	# Remove the ASM DISKS Extra DISK GROUPS after RConfig is successful.


	"C:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename "F:\virtualbox_vm\ASMDISKS\DATA1_ASM01.vdi" --size 155000 --format VDI --variant Fixed   
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename "F:\virtualbox_vm\ASMDISKS\DATA2_ASM02.vdi" --size 155000 --format VDI --variant Fixed
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename "G:\virtualbox_vm\ASMDISKS\DATA3_ASM03.vdi" --size 155000 --format VDI --variant Fixed
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename "G:\virtualbox_vm\ASMDISKS\DATA4_ASM04.vdi" --size 155000 --format VDI --variant Fixed
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename "G:\virtualbox_vm\ASMDISKS\FRA1_ASM01.vdi" --size  75000 --format VDI --variant Fixed
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" createhd --filename "G:\virtualbox_vm\ASMDISKS\FRA2_ASM02.vdi" --size  75000 --format VDI --variant Fixed
	
	# Make HDs shareable.
	
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd "F:\virtualbox_vm\ASMDISKS\DATA1_ASM01.vdi" --type shareable
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd "F:\virtualbox_vm\ASMDISKS\DATA2_ASM02.vdi" --type shareable
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd "G:\virtualbox_vm\ASMDISKS\DATA3_ASM03.vdi" --type shareable
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd "G:\virtualbox_vm\ASMDISKS\DATA4_ASM04.vdi" --type shareable
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd "G:\virtualbox_vm\ASMDISKS\FRA1_ASM01.vdi" --type shareable
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" modifyhd "G:\virtualbox_vm\ASMDISKS\FRA2_ASM02.vdi" --type shareable
	
	# Attach ASMDISKS to ONE SERVER RAC Node1 (oradbserv01) only for now.
	
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv01 --storagectl "SATA" --port 5 --device 0 --type hdd  --medium "F:\virtualbox_vm\ASMDISKS\DATA1_ASM01.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv01 --storagectl "SATA" --port 6 --device 0 --type hdd  --medium "F:\virtualbox_vm\ASMDISKS\DATA2_ASM02.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv01 --storagectl "SATA" --port 7 --device 0 --type hdd  --medium "G:\virtualbox_vm\ASMDISKS\DATA3_ASM03.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv01 --storagectl "SATA" --port 8 --device 0 --type hdd  --medium "G:\virtualbox_vm\ASMDISKS\DATA4_ASM04.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv01 --storagectl "SATA" --port 9 --device 0 --type hdd  --medium "G:\virtualbox_vm\ASMDISKS\FRA1_ASM01.vdi" 
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv01 --storagectl "SATA" --port 10 --device 0 --type hdd  --medium "G:\virtualbox_vm\ASMDISKS\FRA2_ASM02.vdi" 

	# RAC Node2 (oradbserv02) LATER!
	
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv02 --storagectl "SATA" --port 0 --device 0 --type hdd  --medium "F:\virtualbox_vm\ASMDISKS\DATA1_ASM01.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv02 --storagectl "SATA" --port 1 --device 0 --type hdd  --medium "F:\virtualbox_vm\ASMDISKS\DATA2_ASM02.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv02 --storagectl "SATA" --port 0 --device 0 --type hdd  --medium "G:\virtualbox_vm\ASMDISKS\DATA3_ASM03.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv02 --storagectl "SATA" --port 1 --device 0 --type hdd  --medium "G:\virtualbox_vm\ASMDISKS\DATA4_ASM04.vdi"
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv02 --storagectl "SATA" --port 0 --device 0 --type hdd  --medium "G:\virtualbox_vm\ASMDISKS\FRA1_ASM01.vdi" 
	"C:\Program Files\Oracle\VirtualBox\VBoxManage" storageattach oradbserv02 --storagectl "SATA" --port 1 --device 0 --type hdd  --medium "G:\virtualbox_vm\ASMDISKS\FRA2_ASM02.vdi" 

```

```bash
	# Log into the oradbserv01 RAC Node1. Also do this on oradbserv02 RAC Node2
	
	# Install ASMlib as ASM Filter driver ASMFD has been deprecated.
	
	sudo yum install -y oracleasm
	sudo yum install -y oracleasm-support
	
	# Use fdisk command to format the ASM Disks. Don this On ONE server only. NODE1 (oradbserv01) 
	# For example
	
	sudo fdisk -l                    #  List disks 
	
	sudo fdisk /dev/sdd              #  Format disks.
	
```
	
Resources: 4 CPUs, 16 GB RAM per VM (host has plenty).


Ensure passwordless SSH between nodes as oracle user.