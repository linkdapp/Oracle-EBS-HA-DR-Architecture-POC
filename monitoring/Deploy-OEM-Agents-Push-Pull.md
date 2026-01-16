## Scenario Overview
In your Oracle E-Business Suite HA/DR proof-of-concept (running on VirtualBox VMs with 32 CPUs / 128 GB RAM host), I have:

- **OEM Server** (Oracle Enterprise Manager Cloud Control 13c or later)
- **DB Server** (single-instance Oracle 19c backend for EBS)
- **App Server** (single-instance EBS 12.2 application tier)

This guide demonstrates deploying **Oracle Management Agents (OMA)** to the DB and App servers for centralized monitoring (performance tuning, alerts, patching, Data Guard/GoldenGate visibility via OEM plugins).
Oracle recommends two main methods:

- **Agent Push** (preferred): OMS "pushes" the agent remotely from the console (Add Host Targets Wizard) — automated, secure, and scalable.
- **Agent Pull** (manual): Basic Management Agent install Target host "pulls" the agent software (using *'AgentPull.sh'*) — useful when push fails (firewall, SSH issues) or for air-gapped environments.
- **Agent Deploy** (manual): Ideal for a customized Management Agent install. Use EM CLI to download the Management Agent software onto the remote destination host before executing the script to install the Management Agent.  

Both methods are showcased here for completeness in this GitHub POC. (Reference: Oracle Enterprise Manager 13c Advanced Installation Guide and MOS Note 1596348.1)

## Prerequisites (Common to Both Methods)

  1. Configure *'/etc/hosts'* on OEM server to include IP addresses of target servers.
  2. OMS host can resolve target hostnames (or use IP).
  3. Test: ssh e.g user@target_hostname date 
  4. Targets have required OS packages (e.g., bc, libaio, net-tools on OL7/8).
  5. Firewall allows ports: 3872 (agent), 4900-4903 (upload), etc.
  6. In OEM create Named Credentials password for OS users such as Oracle. Grant it sudo privileges.


## Method 1: Agent Push (Recommended – from OEM Console)

1. Log in to OEM console as SYSMAN.

	![Step 1: Deploy Agent Screenshot](screenshots/step1_Log_in_to_OEM_console_as_SYSMAN.jpg)


2. Navigate: **Setup** → **Add Target** → **Add Targets Manually**

	![Step 2: Deploy Agent Screenshot](screenshots/Step2_setup_page_add_target.jpg)


3. Select **Install Agent on Host** → Click **Add**

	![Step 3: Deploy Agent Screenshot](screenshots/step3_install_agent_on_host.jpg)


4. Enter host details:
	- Hostname: dbserver_hostname (or IP)
	- Platform: Linux x86-64 (match your VM)
		
	![Step 4: Deploy Agent Screenshot](screenshots/step4_add_hosts_details.jpg)

	- Click **Next**
	
	
5. On Installation Details screen:
	- Installation Base Directory: *'/u01/app/oracle/agent'* (example – choose consistent path)
	- Instance Directory: auto-filled or *'/u01/app/oracle/agent/agent_inst'*
	- Port: 3872 (default agent listen port)
	- Registration Password: the one you set
	- Additional Parameters (optional): -ignorePrereqs if testing
	
	![Step 5: Deploy Agent Screenshot](screenshots/step5_host_installation_details.jpg)

	- Click **Next**
	
	
6. Review → **Deploy Agent**

	![Step 6: Deploy Agent Screenshot](screenshots/step6_review_and_deploy.jpg)

	- Click **Deploy Agent**
	
	
7. Monitor progress on **Add Host Status** page.

	![Step 7: Deploy Agent Screenshot](screenshots/step7_moniotor_progress.jpg)


8. After success: Run root.sh on each target (as root):

	```bash
	/u01/app/oracle/agent/root.sh
	```
   
	![Step 8: Deploy Agent Screenshot](screenshots/step8_successful_deployment.png)


9. Verify: 

	```bash
	emctl status agent
   ```
   
	![Step 9: Deploy Agent Screenshot](screenshots/step9_verify_agent_status.png)


Targets auto-discover (DB, listener, EBS apps) in OEM.





## Method 2: Agent Deploy (Manual – from Target Host)


1. On OMS set the Agent Registration Password. 
 
- Go to **Setup** → **Security** → **Registration Passwords**. 

	![Step 1: Deploy Agent Screenshot](screenshots/method2_step1a_create_agent_reg_passwd_in_oms.png)  

- Click **Create** > Enter a new password. Set **Expiry Date**  

	![Step 2: Deploy Agent Screenshot](screenshots/method2_step1b_create_agent_reg_passwd_in_oms.png)  


2. On OMS host and destination (target) host, create a staging Directory:  
	
	```bash
	mkdir -p /u01/app/oracle/staging/agentsoftware
	cd /u01/app/oracle/staging/agentsoftware
	```
	
3. Generate the Management Agent software and copy it to the destination server via scp":
	
	```bash
	emcli login -username=sysman
	emcli sync
	emcli get_supported_platforms # Identify your Platform mine is Linux x86-64
	emcli get_agentimage -destination=/u01/app/oracle/staging/agentsoftware -platform="Linux x86-64" -version=13.3.0.0.0
	scp *13.3.0.0.0_AgentCore_226.zip* oracle@dbserver_hostname:/u01/app/oracle/staging/agentsoftware
	```
	
	![Step 4: Deploy Agent Screenshot](screenshots/method2_step4_generate_management_agent_software.png)


4. On the target host (as oracle user) unzip 13.3.0.0.0_AgentCore_226.zip and edit the agent.rsp:

	```bash
	cd /u01/app/oracle/staging/agentsoftware
	unzip 13.3.0.0.0_AgentCore_226.zip
	vi agent.rsp # Typical entries below:
	OMS_HOST=oemserver01.usat.com
	EM_UPLOAD_PORT=4903
	AGENT_REGISTRATION_PASSWORD=agent12
	
	AGENT_BASE_DIR=/u01/app/oracle/agent
	AGENT_INSTANCE_HOME=/u01/app/oracle/agent/agent_inst
	AGENT_PORT=3872
	ORACLE_HOSTNAME=
	s_agentHomeName="agent13R3"
	```

	![Step 5: Deploy Agent Screenshot](screenshots/method2_step5_run_agentdeploy_com.png)
	

5. Run the *agentDeploy.sh* command (adjust paths/ports as needed):

	```bash
	cd /u01/app/oracle/staging/agentsoftware/
	./agentDeploy.sh \
	RESPONSE_FILE=/u01/app/oracle/staging/agentsoftware/agent.rsp \
	-ignorePrereqs
	```
 
6. Wait for install (may take 10-20 mins).


7. Run root.sh (as root):
	
	```bash
	/u01/app/oracle/agent/agent_13.3.0.0.0/root.sh
	```


8. Start and verify agent:

	```bash
	/u01/app/oracle/agent/agent_inst/bin/emctl start agent
	
	/u01/app/oracle/agent/agent_inst/bin/emctl status agent
	```
	
	![Step 8: Deploy Agent Screenshot](screenshots/method2_step9_agent_status_cmd.png)
	
	via OMS:
		
	![Step 8: Deploy Agent Screenshot](screenshots/method2_step9_agent_status_oms.png)

	
