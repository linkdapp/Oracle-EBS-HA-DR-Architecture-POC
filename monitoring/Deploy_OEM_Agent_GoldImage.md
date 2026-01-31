
# Creating a Golden Image in Oracle Enterprise Manager 13.5
# And Deploying an OMA using a Golden Image.

The process assumes:
	- OEM 13.5 is installed and configured.
	- You have the Enterprise Manager for Oracle Database plugin (or relevant plugins) installed.
	- The source Oracle Home is already discovered as a target in OEM.
	- You have admin privileges (e.g., EM_ADMIN role).
	- Name Credentials setup for the Oracle OS user.
	- Software Library Path exists: *mkdir -p /u01/app/oracle/Middleware/oms/13.5/swlib*
	- All plugins on the source agent either the same or supersede the plugins installed on the target agents. Applies only to upgrades!
	  (ERROR:Agent is not compatible with the given Gold Image. Few deployed plug-ins on the Management Ag the Gold Image: Systems Infrastructure:Plugin,Oracle Database:Plugin)

	  ![Step1: Deploy Gold Agent Screenshot](screenshots/plugin_check.png)
	  
	  
Reference: [Oracle Docs - Installing Oracle Management Agents](https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/embsc/installing-oracle-management-agents.html#GUID-E85E79E8-4A8D-4AE0-9E52-39A8920616D0) (for agent setup; for Gold Images, see the OEM Lifecycle Management Guide).

---

# Creating a Golden Image in Oracle Enterprise Manager 13.5

Oracle Enterprise Manager (OEM) 13.5 allows you to create "Gold Images" from existing Oracle Homes (e.g., database or middleware installations). These are standardized snapshots used for provisioning, patching, and compliance. This guide walks through the process step-by-step, including troubleshooting a common "stale configuration" error.

You need a standalone agent that means you cannot use the OMA on that is running on the OEM Server.

## Prerequisites

- OEM 13.5 Cloud Control console access.
- A discovered and monitored Oracle Home target (e.g., via agent push or discovery).
- Software Library configured in OEM (Setup > Provisioning and Patching > Software Library).
- Sufficient storage in the Software Library location for the image (e.g., /u01/app/em_swlib).

## Step-by-Step Guide

### Step 1: Log In to OEM Console

- Open your browser and navigate to the OEM URL (e.g., `https://<oem-host>:7803/em`).
- Log in with admin credentials.

### Step 2: Verify and Refresh the Oracle Home Target Configuration

- Gold Images require up-to-date configuration data from the source Oracle Home. If it's stale, you'll hit errors like: *"Agent Collection has became stale. Please refresh configuration for oracle home before creating a GoldImage version."*


- Navigate to **Targets > All Targets**.

	![Step1: Deploy Gold Agent Screenshot](screenshots/step1_agent_OH_refresh.png)
	
- Search for your Oracle Home target (e.g., filter by "Oracle Home" type).

	![Step2: Deploy Gold Agent Screenshot](screenshots/step2_agent_OH_refresh.png)
	
- Select the target > Right-click (or from the target page: **Oracle Home > Configuration > Refresh Configuration**).

	![Step3: Deploy Gold Agent Screenshot](screenshots/step3_agent_OH_refresh.png)

- Click **Refresh** (or **Refresh Collection**) to sync the latest config via the agent.
	
	![Step4: Deploy Gold Agent Screenshot](screenshots/step4_agent_OH_refresh.png)

- Wait for the job to complete (monitor under **Enterprise > Job > Activity**).
	
	![Step5: Deploy Gold Agent Screenshot](screenshots/step5_agent_OH_refresh.png)
	

**Tip:** If the refresh fails (e.g., agent issues), check agent status (**Targets > Agents > Select Agent > Status**) and restart if needed (`emctl stop agent; emctl start agent` on the agent host).

### Step 3: Navigate to Provisioning and Patching

- From the top menu:: **Setup > Provisioning and Patching > Software Library**.
  Another method **Enterprise > Provision and Patching > Software Library > Actions > Administration**

	![Step6: Deploy Gold Agent Screenshot](screenshots/step1_create_software_lib.png)
	
- Ensure a folder exists **Software Library: Administration** 
  Create one by clicking **Add > Add OMS Shared File System Location** 
  Fill out: 
  **Name** 
  **Location** 
  **Credential**
  **OK**
  
  	![Step7: Deploy Gold Agent Screenshot](screenshots/step2_create_software_lib.png)
	
### Step 4: Create the Gold Image

- From the Oracle Home target page (from Step 2): Select **Setup > Manage Cloud Control > Gold Agent Image**.

  ![Step8: Deploy Gold Agent Screenshot](screenshots/step1_agent_golden_image.png)

- Click on **Manage All Agents:** To manage your Gold Agent Images.

  ![Step9: Deploy Gold Agent Screenshot](screenshots/step2_agent_golden_images.png)

- On the Introduction page. Click on **Create** 
 
  ![Step10: Deploy Gold Agent Screenshot](screenshots/step3_agent_golden_images_manage.png)

  And enter details:

  **Image name**
  **Description**
  **Platform Name**

  Click **Submit** 

  ![Step10: Deploy Gold Agent Screenshot](screenshots/step4_agent_golden_images_create.png)

- In Gold Agent Images Name page: Section Gold Image Name you will see the new Image name that was just created. Click on it.

  ![Step12: Deploy Gold Agent Screenshot](screenshots/step5_agent_golden_image_name.png)

- Manage Image page: (**<Image_Name>**) Click on **Actions > Create**

  ![Step13: Deploy Gold Agent Screenshot](screenshots/step6_agent_golden_image_manage.png)

- Create Image Version page: Fill out:
  
  **Image Name**
  **Image Version Name**
  **Description**
  **Create image by** (Select the method: Selecting a source agent) Source agent cannot be the OEM Server Agent.
  **Source Agent**

  ![Step14: Deploy Gold Agent Screenshot](screenshots/step7_agent_golden_image_version1a.png)

  ![Step15: Deploy Gold Agent Screenshot](screenshots/step7_agent_golden_image_version1b.png)

- Review Create Image Summary page:

  ![Step16: Deploy Gold Agent Screenshot](screenshots/step8_agent_golden_image_summary.png)

  Click **Yes**

- Job will be submitted to create the Gold Agent Image.

  ![Step17: Deploy Gold Agent Screenshot](screenshots/step9_agent_golden_image_job_info1.png)

   click on **Show Activity details** to see more information about the job or you can just click on **OK** to close.
   You can also monitor job progress under **Enterprise > Job > Activity**


   ![Step18: Deploy Gold Agent Screenshot](screenshots/step9_agent_golden_image_job_info2.png)


### Step 5: Set the Agent Gold Image as the current version

- From the Oracle Home target page (from Step 2): Select **Setup > Manage Cloud Control > Gold Agent Image**
- In Gold Agent Images Name page: Section Gold Image Name you will see the new Image name that was just created. Click on it.
- Click **Manage Image Versions and Subscriptions**.

  ![Step19: Deploy Gold Agent Screenshot](screenshots/step10_agent_gold_subscript1.png)

- Select the **Versions and Drafts** tab. Select the gold image version that you want to set as the current version, then click **Set Current Version**.
  You can also monitor job progress under **Enterprise > Job > Activity**
  
  ![Step20: Deploy Gold Agent Screenshot](screenshots/step10_agent_gold_subscript2.png)
	
- Click on the **Gold Agent Image Activities** page, in the **Activities** tab to view the status of this job.
	
  ![Step21: Deploy Gold Agent Screenshot](screenshots/step10_agent_gold_subscript3.png)
	

### Step 6: Subscribing Management Agents to an Agent Gold Image Using Gold Agent Images

- From the Oracle Home target page (from Step 2): Select **Setup > Manage Cloud Control > Gold Agent Image**
- In Gold Agent Images Name page: Section Gold Image Name you will see the new Image name that was just created. Click on it.
- Click **Manage Image Versions and Subscriptions**.

- Select the **Subscriptions** tab. Click **Subscribe**.
  Search for and select the required Management Agents, then click **Select**.

  ![Step22: Deploy Gold Agent Screenshot](screenshots/step10_agent_gold_subscript4.png)
	
  ![Step23: Deploy Gold Agent Screenshot](screenshots/step10_agent_gold_subscript5.png)


### Step 7: Use the Agent Gold Image Version to Update the Standalone OMA 

- From the Oracle Home target page (from Step 2): Select **Setup > Manage Cloud Control > Gold Agent Image**
- In Gold Agent Images Name page: Section Gold Image Name you will see the new Image name that was just created. Click on it.
- Click **Manage Image Versions and Subscriptions**.
- Select the **Subscriptions** tab. Select the Management Agents that you want to update 
- select **Update**, then select **To Current Version**.

  ![Step23: Deploy Gold Agent Screenshot](screenshots/step11_agent_gold_update1.png)
  
  The Update Agents job page:
  
  ![Step23: Deploy Gold Agent Screenshot](screenshots/step11_agent_gold_update2.png)
  
  - Click **Next**
  
  Agent Update Notification page:
  
  ![Step23: Deploy Gold Agent Screenshot](screenshots/step11_agent_gold_update3.png)
  
  Click **OK**
  
  ![Step23: Deploy Gold Agent Screenshot](screenshots/step11_agent_gold_update4.png)
  
  
  Click **Update**
  
  Monitor

  ![Step23: Deploy Gold Agent Screenshot](screenshots/step12_agent_gold_jobmonitor1.png

  ![Step23: Deploy Gold Agent Screenshot](screenshots/step12_agent_gold_jobmonitor2.png
  
  Successful!
  
  ![Step23: Deploy Gold Agent Screenshot](screenshots/step13_agent_gold_update_verify.png




## Benefits
- Standardizes deployments across environments.
- Speeds up patching/provisioning (e.g., for cloud/OCI migrations).
- Integrates with OEM's compliance checks.


## Common Issues and Troubleshooting

- **Stale Configuration Error:** As mentioned, always refresh the target config first (Step 2). If persistent, re-collect inventory (`emctl upload agent` on the host) or rediscover the target.
- **Agent Communication Issues:** Ensure the agent is healthy (`emctl status agent` on the host) and ports are open (default: 3872).
- **Plugin compatibility Issues:** Ensure the Golden Image has all the necessary plugins as the target updating.


For more details, see the [OEM 13.5 Lifecycle Management Guide](https://docs.oracle.com/en/enterprise-manager/cloud-control/enterprise-manager-cloud-control/13.5/lcmgt/oracle-enterprise-manager-lifecycle-management.html).


