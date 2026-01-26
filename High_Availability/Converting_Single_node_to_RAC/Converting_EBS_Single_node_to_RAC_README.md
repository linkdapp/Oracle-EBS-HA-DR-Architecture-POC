# Converting Single Instance Oracle E-Business Suite (EBS) Release 12.2 to a 2-Node Oracle RAC for High Availability

## Overview: Why High Availability (HA) with Oracle RAC?

High Availability ensures your EBS system remains operational despite hardware failures, maintenance, or spikes in load. Oracle RAC provides:

 - Redundancy and Failover: Multiple nodes share the database workload; if one node fails, others take over seamlessly.
 
 - Scalability: Load balancing across nodes improves performance for concurrent users and batch jobs.
 
 - Zero Downtime Maintenance: Patch one node while others handle traffic.
 
 - Business Continuity: Reduces unplanned downtime (e.g., from node crashes) and supports features like Parallel Concurrent Processing (PCP) in EBS.
 
 - Integration with Other Tools: Complements Data Guard (for DR) and GoldenGate (for replication), and monitoring via OEM.

Without HA, a single-instance setup risks extended outages. RAC achieves 99.99%+ uptime in production, but requires proper setup for interconnect (private network) and shared storage.

Estimated Time: 4-8 hours for setup, plus testing. Downtime during conversion: ~1-2 hours (can be minimized with careful planning).


## Prerequisites

### Note: 

Note: This process assumes your current setup (OEM Server, single-instance EBS App Server, single-instance DB Server) is on Linux x86-64 (as per your host and common EBS setups). RAC requires shared storage (e.g., via VirtualBox shared disks or NFS for testing). Test in a non-production environment first. Allocate VM resources: e.g., 8 CPUs and 16 GB RAM per node (total ~32 GB for 2 nodes + overhead).

Hardware/VM Setup:

Create 2 VirtualBox VMs: Node1 (oradbserv01). This is the current SINGLE NON-RAC DB server 
                         Node2 (oradbserv02). This is the new node to add (I made a clone of oradbserv01 without the database) 

OS: Oracle Linux 7 (matching your current setup).

Network: 3 NICs per server

 - Public NIC (NAT):                    enp0s3 (eth0): DHCP (Connect Automatically)
 - Private NIC (Host-Only-Adapter):     enp0s8 (eth1): IP=192.168.56.128, Subnet=255.255.255.0, Gateway=192.168.56.1, DNS=<blank>, Search=usat.com (Connect Automatically)
 - Virtual IP (VIP) (Internal Network:  enp0s9 (eth2): IP=192.168.2.128, Subnet=255.255.255.0, Gateway=<blank>, DNS=<blank>, Search=<blank> (Connect Automatically)

Shared Storage: Use VirtualBox shared disks (VDI files) for ASM (Automatic Storage Management). 
I am using ~450 GB shared disk for DB files.

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

	# On both oradbserv01 RAC Node1 and oradbserv02 RAC Node2
	
	# Install ASMlib as ASM Filter driver ASMFD has been deprecated.
	
	sudo yum install -y oracleasm
	sudo yum install -y oracleasm-support
	
	# Use fdisk command to format the ASM Disks. Don this On ONE server only. NODE1 (oradbserv01) 
	# For example
	
	sudo fdisk -l                    #  List disks 
	
	sudo fdisk /dev/sdd              #  Format disks.
	


Network DNS Configuration

 - Edit */etc/hosts* file with the new IP addresses
 
	Server names and IP addresses.
	
	# OEM Suite Server
	
	192.168.56.65     oemserver01.usat.com       oemserver01
	
	
	# E-Business APP Suite Server
	
	192.168.56.60  orappsserv01.usat.com       orappsserv01
	
	# ORACLE RAC Databases for EBS
	# Public
	192.168.56.126   oradbserv01.usat.com        oradbserv01
	192.168.56.127   oradbserv02.usat.com        oradbserv02
	
	# Private
	192.168.2.126   oradbserv01-priv.usat.com   oradbserv01-priv
	192.168.2.127   oradbserv02-priv.usat.com   oradbserv02-priv
	
	# Virtual
	192.168.56.131   oradbserv01-vip.usat.com    oradbserv01-vip
	192.168.56.132   oradbserv02-vip.usat.com    oradbserv02-vip
	
	# SCAN
	192.168.56.151   scan-oradbserv.usat.com  scan-oradbserv
	192.168.56.161   scan-oradbserv.usat.com  scan-oradbserv
	192.168.56.171   scan-oradbserv.usat.com  scan-oradbserv
	
	
	# Single instance dataguard db
	
	192.168.56.130 oradbserv04.usat.com   oradbserv04
	


=====> Disable SELinux. Open the config file and change the SELINUX variable from enforcing to disabled.


sudo vi /etc/selinux/config

======> Turn off and disable the firewall IPTables. If exists.


sudo chkconfig --list iptables



=====>  Make life easier, just turn off  firewalld.

sudo service firewalld stop


systemctl disable firewalld



=====> Verify that all the network interfaces are up.


sudo ip l


=====>  Install BIND on ALL RAC NODES in the cluster. if you do not have it

sudo yum install bind-libs bind bind-utils -y


=====> Enable BIND DNS to start at boot time.
 

sudo chkconfig named on


=====> Change named directory permissions

sudo ls -ltr /var/named

sudo touch /var/named/usat.com

sudo chmod 664 /var/named/usat.com

sudo chgrp named /var/named/usat.com

sudo chmod g+w /var/named




=====> Backup the BIND configuration file.


ls -ltr /etc/named.conf*



sudo cp /etc/named.conf /etc/named.conf.bak



=====> Change /etc/named.conf permissions.
       Otherwise, the original protection may cause trouble in the restarting named step with write-protection 
	   errors in /var/log/messages. 

sudo chmod 664 /etc/named.conf



=====> Replace or edit the /etc/named.conf file to change the named configuration manually
       RAC NODE1 (oradbserv01) will serve as the MASTER while RAC NODE2 (oradbserv02) the slave.

vi  /etc/named.conf file
//
// named.conf
//
// Provided by Red Hat bind package to configure the ISC BIND named(8) DNS
// server as a caching only nameserver (as a localhost DNS resolver only).
//
// See /usr/share/doc/bind*/sample/ for example named configuration files.
//
// See the BIND Administrator's Reference Manual (ARM) for details about the
// configuration located in /usr/share/doc/bind-{version}/Bv9ARM.html

options {
        listen-on port 53 { 192.168.56.126; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { 192.168.56.0/24; localhost; };
        allow-transfer  { 192.168.56.0/24; };

        /*
         - If you are building an AUTHORITATIVE DNS server, do NOT enable recursion.
         - If you are building a RECURSIVE (caching) DNS server, you need to enable
           recursion.
         - If your recursive DNS server has a public IP address, you MUST enable access
           control to limit queries to your legitimate users. Failing to do so will
           cause your server to become part of large scale DNS amplification
           attacks. Implementing BCP38 within your network would greatly
           reduce such attack surface
        */
        recursion yes;
        forward first;
        forwarders {
        10.0.2.3;
        };

        dnssec-enable yes;
        dnssec-validation yes;

        /* Path to ISC DLV key */
        bindkeys-file "/etc/named.root.key";

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";

zone "usat.com" {
  type master;
  file "usat.com";
};

zone "in-addr.arpa" {
  type master;
  file "in-addr.arpa";
};


=====> Create the zone file for the usat.com domain on oradbserv01 by running the 
       following command:


sudo vi /var/named/usat.com

(Paste this into the file)

$TTL 3H
@       IN SOA  oradbserv01        hostmaster      (
                                        101   ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
                NS      oradbserv01
                NS      oradbserv02
localhost       A       127.0.0.1
oradbserv01        A       192.168.56.126
oradbserv01-vip    A       192.168.56.131
oradbserv01-priv   A       192.168.2.126
oradbserv02        A       192.168.56.127
oradbserv02-vip    A       192.168.56.132
oradbserv02-priv   A       192.168.2.127
scan-oradbserv     A       192.168.56.151
scan-oradbserv     A       192.168.56.161
scan-oradbserv     A       192.168.56.171


sudo sudo cat /var/named/usat.com


=====> Create the reverse zone file oradbserv01.

sudo vi /var/named/in-addr.arpa
Copy and paste below command as root:

$TTL 3H
@       IN SOA  oradbserv01.usat.com.      hostmaster.usat.com. (
                                        101   ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
                NS      oradbserv01.usat.com.
                NS      oradbserv02.usat.com.

126.56.168.192  PTR     oradbserv01.usat.com.
131.56.168.192  PTR     oradbserv01-vip.usat.com.
126.2.168.192   PTR     oradbserv01-priv.usat.com.
127.56.168.192  PTR     oradbserv02.usat.com.
132.56.168.192  PTR     oradbserv02-vip.usat.com.
127.2.168.192   PTR     oradbserv02-priv.usat.com.
151.56.168.192  PTR     scan-oradbserv.usat.com.
161.56.168.192  PTR     scan-oradbserv.usat.com.
171.56.168.192  PTR     scan-oradbserv.usat.com.


sudo cat /var/named/in-addr.arpa



=====> Generate the rndc.key file.

sudo rndc-confgen -a -r /dev/urandom

sudo chgrp named /etc/rndc.key

sudo chmod g+r /etc/rndc.key

sudo ls -lrta /etc/rndc.key




=====> Check that the parameter PEERDNS is set to no in /etc/sysconfig/network-scripts/ifcfg-enp0s3 to prevent the resolv.conf 
       from being overwritten by the dhcp client.


sudo cp /etc/sysconfig/network-scripts/ifcfg-enp0s3	vi /etc/sysconfig/network-scripts/ifcfg-enp0s3.bak  
 

sudo vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
	   

sudo cat /etc/sysconfig/network-scripts/ifcfg-enp0s3

===> Before

TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="dhcp"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
UUID="608fff3b-9ee3-4661-8652-0ded1bfb7ef4"
DEVICE="enp0s3"
ONBOOT="yes"


===> After

TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="dhcp"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="yes"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="enp0s3"
UUID="608fff3b-9ee3-4661-8652-0ded1bfb7ef4"
DEVICE="enp0s3"
ONBOOT="yes"
PEERDNS="no"




=====> Restart the named service.

sudo service named restart




=====> Restart network service.

sudo  service network restart


=====> Change /etc/resolv.conf and check nslookup nodes and SCAN ips working fine.

============ BEFORE Change ========

	  
sudo cat  /etc/resolv.conf
# Generated by NetworkManager
search lan usat.com
nameserver 10.0.2.3

vi /etc/resolv.conf

============ AFTER Change ========
cat /etc/resolv.conf
# Generated by NetworkManager
search usat.com lan
nameserver 192.168.56.126
nameserver 192.168.56.127
nameserver 10.0.2.3


Make it immutable to prevent future overwrites:

chattr +i /etc/resolv.conf


To edit later if needed: 

chattr -i /etc/resolv.conf


=====> Restart the named service.

sudo service named restart




=====> Restart network service.

sudo  service network restart

nslookup oradbserv01.usat.com
Server:         ::1
Address:        ::1#53

Name:   oradbserv01.usat.com
Address: 192.168.56.126

[oracle@oradbserv01 ~]$
[oracle@oradbserv01 ~]$ nslookup oradbserv01
Server:         ::1
Address:        ::1#53

Name:   oradbserv01.usat.com
Address: 192.168.56.126

[oracle@oradbserv01 ~]$
[oracle@oradbserv01 ~]$ nslookup scan-oradbserv
Server:         ::1
Address:        ::1#53

Name:   scan-oradbserv.usat.com
Address: 192.168.56.171
Name:   scan-oradbserv.usat.com
Address: 192.168.56.161
Name:   scan-oradbserv.usat.com
Address: 192.168.56.151

[oracle@oradbserv01 ~]$
[oracle@oradbserv01 ~]$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=255 time=25.6 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=255 time=27.6 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=255 time=27.8 ms
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 25.649/27.065/27.878/1.023 ms
[oracle@oradbserv01 ~]$




=======> Configuration DNS RAC NODE2 (oradbserv02) as the SLAVE
	----	Stop the DNS service.

sudo service named stop


=====> As root, create the zone files for the usat.com domain on RAC NODE2 slave (oradbserv02)  
      by running the following command:

=======> 


		Modify the file /etc/named.conf by using the following command:

From oradbserv01 to oradbserv02

scp /etc/named.conf root@oradbserv02:/etc
 
sed -i -e 's/listen-on .*/listen-on port 53 { 192.168.56.127; };/' \
-e 's/type master;/type slave;\n masters {192.168.56.126; };/' \
/etc/named.conf


=======>  

		Start the named service.

[root@oradbserv02 named]# service named start




=======> 

		Check that both the master on oradbserv01 and slave on oradbserv02 DNS servers are working

[root@oradbserv02 ~]# dig @oradbserv01 oradbserv01.usat.com
[root@oradbserv02 ~]# dig @oradbserv01 oradbserv02.usat.com
[root@oradbserv02 ~]# dig @oradbserv01 oradbserv01-vip.usat.com
[root@oradbserv02 ~]# dig @oradbserv01 oradbserv02-vip.usat.com
[root@oradbserv02 ~]# dig @oradbserv01 oradbserv01-priv.usat.com
[root@oradbserv02 ~]# dig @oradbserv01 oradbserv02-priv.usat.com


[root@oradbserv01 ~]# dig @oradbserv02 oradbserv01.usat.com
[root@oradbserv01 ~]# dig @oradbserv02 oradbserv02.usat.com
[root@oradbserv01 ~]# dig @oradbserv02 oradbserv01-vip.usat.com
[root@oradbserv01 ~]# dig @oradbserv02 oradbserv02-vip.usat.com
[root@oradbserv01 ~]# dig @oradbserv02 oradbserv01-priv.usat.com
[root@oradbserv01 ~]# dig @oradbserv02 oradbserv02-priv.usat.com
[root@oradbserv01 ~]# dig @oradbserv02 scan-oradbserv


=======>




   ------------- Configuration for GRID user.


 groupadd -g 54327 asmdba
 groupadd -g 54328 asmadmin
 groupadd -g 54329 asmoper

[root@oradbserv01 ~]# useradd -u 54328 -g oinstall -G dba,asmadmin,asmdba,asmoper,racdba,vboxsf  grid
[root@oradbserv01 ~]# 
[root@oradbserv01 ~]# passwd grid
Changing password for user grid.
New password:



[root@oradbserv01 app]# id grid
uid=54328(grid) gid=54321(oinstall) groups=54321(oinstall),981(vboxsf),54328(asmadmin),54329(asmoper),54327(asmdba)


[root@oradbserv01 ~]# vi  /etc/security/limits.conf ------> ( If configuring manually)

# grid user
#
grid              soft    nproc   2047
grid              hard    nproc   16384
grid              soft    nofile  1024
grid              hard    nofile  65536
grid              soft    stack   10240
grid              soft    stack   32768


[root@oradbserv01 ~]# 
[root@oradbserv01 ~]# 
[root@oradbserv01 ~]# 
[root@oradbserv01 ~]# vi /etc/sudoers

## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL
oracle  ALL=(ALL)       ALL
grid    ALL=(ALL)       ALL






==========










    # ------------ As root. Configure Oracleasm on both RAC NODE1 (oradbserv01) and RAC NODE2 (oradbserv02)

df -ha |grep -i oracleasm


------oracleasm configure 

oracleasm configure -i


-------- Check oracleasm Status
 
oracleasm status

------- Load the oracleasm module 
	 
oracleasm init

--------- Check OracleAsm Status
 
oracleasm status 

------- Verify the oracleasm configuration
	 

df -ha |grep -i oracleasm


lsmod | grep oracleasm





# ------------ As root. create ASM disks. Do this on RAC Node1 only. (oradbserv01)

	   
oracleasm scandisks

oracleasm listdisks


oracleasm createdisk ASMDISK1 /dev/sdf1

oracleasm createdisk ASMDISK2 /dev/sdg1

oracleasm createdisk ASMDISK3 /dev/sdh1

oracleasm createdisk ASMDISK4 /dev/sdi1

oracleasm createdisk ASMDISK5 /dev/sdj1

oracleasm createdisk ASMDISK6 /dev/sdk1

oracleasm scandisks

oracleasm listdisks




 
Resources: 4 CPUs, 16 GB RAM per VM (host has plenty).


Ensure passwordless SSH between nodes as oracle user.