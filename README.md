# Microsoft Fabric Git Integration & CI/CD Pipeline Reference Guide

## Power BI Reports Lifecycle Management

## Overview
This tutorial focuses specifically on implementing a complete CI/CD DevOps pipeline for **Power BI Reports** in Microsoft Fabric using Git integration and deployment pipelines. We'll cover the complete lifecycle management across three environments (DEV, UAT, PROD) with approval processes, including the essential Power BI Desktop configurations needed for seamless integration.

Development begins in dedicated feature workspaces, each connected to a corresponding feature branch in Git. After completing and testing changes in a feature workspace, you'll create a pull request to merge those changes into the main branch, which then triggers updates to the DEV environment and initiates the deployment pipeline.

## Architecture Overview - Option 3: Deploy using Fabric Deployment Pipelines

```
Power BI Desktop → Git Repository (Feature/Main Branches) → Fabric Deployment Pipelines

Development Flow:
1. Power BI Desktop ←→ Feature Workspaces (Pull/Push to Feature Branch)
2. Feature Branch → Main Branch (Merge via PR)
3. Main Branch → DEV Workspace (Sync/Update)
4. DEV → UAT → PROD (Deploy via Fabric Deployment Pipelines)
```

**Visual Architecture:**
<p align="center">
   <img src="images/powerbi_fabric_architecture.png" width="700"/>
</p>

<p align="center">
   <strong>Figure: Power BI Desktop &rarr; Feature Workspaces &rarr; Git Branches &rarr; DEV/UAT/PROD Workspaces via Fabric Deployment Pipelines</strong>
</p>

<details>
   <summary>Textual Architecture Flow</summary>
   <pre>
[Power BI Desktop] ←Pull/Push→ [Feature Workspaces] ←Sync→ [Feature Branch]
         ↓ (Merge)
[DEV Workspace] ←Sync/Update→ [Main Branch] ←Branch→ [Main Repository]
         ↓ (Deploy)
[UAT Workspace] 
         ↓ (Deploy with Approval)
[PROD Workspace]
   </pre>
</details>
</br>

**Key Components:**
- **Feature Workspaces:** Connected to feature branches for development
- **Main DEV Workspace:** Connected to main branch, source for deployment pipeline
- **Git Repository:** Central version control with feature and main branches
- **Fabric Deployment Pipelines:** Native Fabric tool for DEV→UAT→PROD deployments
- **Trigger Deployment:** Automated or manual deployment initiation

## Part 1: Power BI Desktop Configuration for Power BI Projects (.pbip)

### Step 1: Power BI Desktop Settings Configuration

Before connecting your workspace to Git, you need to configure Power BI Desktop to support the new Git integration workflow and ensure optimal compatibility with Fabric's version control system.

#### 1.1 Enable Preview Features
1. **Open Power BI Desktop**
2. **Go to File → Options and Settings → Options**
3. **Navigate to Preview Features section**
4. **Enable the following preview feature (Essential for Power BI Projects):**

   **Required for Power BI Projects:**
   
   - ✅ **"Power BI Project (.pbip) save option"**
     - *REQUIRED: This enables saving reports in the new .pbip format*
     - *Essential for version control and Fabric workspace integration*
     - *Allows reports to be stored as decomposed files for better collaboration*

   <p align="center">
      <img src="images/powerbi_desktop_pbip.png" width="600" alt="Power BI Desktop Configuration Settings"/>
   </p>
   <p align="center">
   <strong>Figure: Power BI Desktop &rarr; Preview Features &rarr; Power BI Project (.pbip) Save Option</strong>
</p>

   > **📝 Note:**  
   > The options **"Store semantic model using TMDL format"** and **"Store reports using enhanced metadata format (PBIR)"** are visible in Power BI Desktop but cannot be enabled (checked).  
   >  
   > **⚠️ Important:**  
   > These options should remain unchecked because enabling them may cause compatibility issues with Fabric deployment pipelines. Always use the .pbip format for seamless integration and deployment pipeline compatibility.

5. **Restart Power BI Desktop** after enabling the Power BI Project preview feature

#### 1.2 Configure Data Source Settings
The data source configuration is done in **Power Query Editor**, not in the main Options menu. Here's where to configure it:

1. **Open Power BI Desktop**
2. **Click "Transform Data" to open Power Query Editor**
3. **In Power Query Editor, go to Home → Manage Parameters**

**Configure for Multi-Environment Support:**

**Step 1: Create Environment Parameter**
1. Click **"New"** to create a new parameter
2. Configure the Environment parameter:

```
Name: Environment
Description: Current deployment environment (DEV, UAT, or PROD)
Required: ✅ (Check the Required checkbox)
Type: Text (select from dropdown)
Suggested Values: List of values (select from dropdown)
  - Add: DEV
  - Add: UAT  
  - Add: PROD
Current Value: DEV
```

3. Click **OK** to save the Environment parameter

<p align="center">
      <img src="images/Manage_parameters_pbi_env.png" width="600" alt="Power BI Desktop Configuration Settings"/>
   </p>
<p align="center">
   <strong>Figure: Power Query Editor &rarr; Manage Parameters &rarr; Environment Parameter</strong>
</p>

**Step 2: Create ServerName Parameter**
1. Click **"New"** again to create another parameter
2. Configure the ServerName parameter:

```
Name: SrvName
Description: Database server name based on environment
Required: ✅ (Check the Required checkbox)
Type: Text (select from dropdown)
Suggested Values: Any value (select from dropdown)
Current Value: dev-server-id.database.fabric.microsoft.com
```

3. Click **OK** to save the ServerName parameter

**Step 3: Create DatabaseName Parameter**
1. Click **"New"** for the third parameter
2. Configure the DatabaseName parameter:

```
Name: DbName
Description: Database name based on environment
Required: ✅ (Check the Required checkbox)
Type: Text (select from dropdown)
Suggested Values: Any value (select from dropdown)
Current Value: adventureworks-dev-workspace-id
```

3. Click **OK** to save the DatabaseName parameter

**How Environment Switching Actually Works:**

1. **Local Development:** Use DEV values in parameters
2. **Publish to DEV Workspace:** Parameters keep DEV values  
3. **Deploy DEV → UAT:** Fabric deployment pipeline automatically updates ServerName and DatabaseName parameters to UAT values
4. **Deploy UAT → PROD:** Fabric deployment pipeline automatically updates ServerName and DatabaseName parameters to PROD values

**Step 4: Configure Parameter Values for Each Environment**

When setting up your deployment pipeline, you'll configure parameter rules like this:

```
DEV Environment Parameters:
ServerName: dev-server-id.database.fabric.microsoft.com
DatabaseName: adventureworks-dev-workspace-id
Environment: DEV

UAT Environment Parameters:  
ServerName: uat-server-id.database.fabric.microsoft.com
DatabaseName: adventureworks-uat-workspace-id
Environment: UAT

PROD Environment Parameters:
ServerName: prod-server-id.database.fabric.microsoft.com  
DatabaseName: adventureworks-prod-workspace-id
Environment: PROD
```

**The deployment pipeline handles the parameter switching automatically - you don't need conditional logic in Power Query.**

**Step 5: Apply Parameters to Data Sources**

After creating simple parameters, connect them to your data source:

1. **Go to your data source query** (in Power Query Editor)
2. **Right-click on the Source step**
3. **Select "Edit Settings"** or click the gear icon
4. **Use your parameters in the connection:**
   - **Server:** Select `SrvName` parameter from dropdown
   - **Database:** Select `DbName` parameter from dropdown
   - **Authentication:** Choose **Microsoft account** or **Organizational account**

**Example Result:**
Your connection will look like:
```
Source = Sql.Database(SrvName, DbName)
```

<p align="center">
      <img src="images/Manage_parameters_pbi_setup_param.png" width="600" alt="Power BI Desktop Configuration Settings"/>
   </p>
<p align="center">
   <strong>Figure: Power Query Editor &rarr; Source step &rarr; Edit Settings</strong>
</p>  

**Step 6: Save the report as PBIP**

Use Consistent Naming Conventions:
   ```
   Report Files:
   - [ProjectName]_[ReportType]_[Version].pbip
   - Example: SalesAnalytics_Dashboard_v1.0.pbip
   ```

<p align="center">
      <img src="images/folder_structure_pbip.png" width="400" alt="Power BI Project"/>
   </p>
<p align="center">
   <strong>Figure: Power BI Project &rarr; Structure of Folders</strong>
</p>  

## Part 2: Microsoft Fabric Git Configuration for Power BI Reports

### Prerequisites
- Microsoft Fabric Premium or Trial capacity
- **Power BI Desktop (Latest Version)** with proper configuration
- GitHub repository / Azure Repository for version control
- Basic understanding of Git concepts (branches, commits, pull requests)
- Admin access to Fabric workspaces
- Understanding of Power BI report development lifecycle

### Step 1: Enable Git Integration in Fabric

<p align="center">
      <img src="images/Tenant_setting_git.png" width="400" alt="Power BI Project"/>
</p>
<p align="center">
   <strong>Figure: Tenant settings &rarr; Git integration</strong>
</p>  

<p align="center">
      <img src="images/create_items_fabric.png" width="400" alt="Power BI Project"/>
</p>
<p align="center">
   <strong>Figure: Tenant settings &rarr; Users can create Fabric Items</strong>
</p> 


#### 1.1 Tenant Settings Configuration
First, ensure Git integration is enabled at the tenant level. Git integration settings can be managed at two levels: **Tenant-wide** or **Capacity-specific** through delegated tenant settings.

1. Navigate to **Microsoft Fabric Admin Portal**
2. Go to **Capacity settings** (as shown in your screenshot)
3. Select your Fabric capacity
4. Navigate to **Delegated tenant settings** tab
5. Find **Git integration** section

**Understanding Delegated Tenant Settings:**

Delegated tenant settings allow capacity admins to **override** tenant-level settings for their specific capacity. This provides more granular control over Git integration.

**Git Integration Settings Configuration:**

**Core Git Integration Settings:**
- ✅ **"Users can synchronize workspace items with their Git repositories"**
  - *Can be enabled at tenant level OR capacity level*
  - Allows import and export of workspace items to Git repositories for collaboration and version control
  - **Key Setting:** "Override tenant admin selection" ✅ allows capacity-specific configuration

**Configuration Options (as shown in your screenshot):**
```
Enabled: ✅ (Toggle enabled for capacity)
Apply to: 
• All the users in capacity (recommended for organizational CI/CD)
○ Specific security groups (for limited rollout)
□ Except specific security groups (for exclusions)

Delegate setting to other admins:
✅ Workspace admins can enable/disable (allows workspace-level control)
```
<p align="center">
      <img src="images/Capacity_setting_git.png" width="400" alt="Power BI Project"/>
</p>
<p align="center">
   <strong>Figure: Capacity settings &rarr; Git integration</strong>
</p>  

**GitHub-Specific Settings:**
- ✅ **"Users can sync workspace items with GitHub repositories"**  
  - *Enabled for all users in capacity*
  - Users can select GitHub as their Git provider and sync items in their workspaces with GitHub repositories
  - Must be enabled along with the general Git synchronization setting

**Additional Git Integration Settings:**
- ✅ **"Users can export items to Git repositories in other geographical locations"**
  - *Enabled for all users in capacity*
  - Allows cross-region Git repository connections
  
- ✅ **"Users can export workspace items with applied sensitivity labels to Git repositories"**
  - *Enabled for all users in capacity*
  - Important for organizations using Microsoft Purview sensitivity labels

**General Fabric Settings:**
- ✅ **"Users can create Fabric items"**
  - *Enabled for all users in capacity*
  - Users can use production-ready features to create Fabric items

**Note: The following settings are only available at TENANT level, not capacity level:**

**Tenant-Only Settings (Configure in Admin Portal → Tenant Settings):**
- ✅ **"Create workspaces"** 
  - *Only configurable at tenant level*
  - Users can create app workspaces to collaborate on dashboards, reports, and other content

  <p align="center">
      <img src="images/ws_settings_tenant.png" width="400" alt="Power BI Project"/>
</p>
<p align="center">
   <strong>Figure: Tenant settings &rarr; Create workspaces</strong>
</p> 


**Best Practices for Delegated Tenant Settings:**

1. **Capacity-Level Control:**
   - Enable Git integration at **capacity level** for better control
   - Apply to **"All the users in capacity"** for organizational CI/CD
   - Enable **"Workspace admins can enable/disable"** for flexibility

2. **Security Considerations:**
   - Use **"Specific security groups"** for phased rollouts
   - Consider **"Except specific security groups"** for sensitive workspaces
   - Review sensitivity label integration if using Microsoft Purview

3. **Administrative Delegation:**
   - ✅ **"Override tenant admin selection"** gives capacity admins control
   - ✅ **"Workspace admins can enable/disable"** provides workspace-level flexibility
   - Allows decentralized management while maintaining governance

**Verification Steps:**
1. **Check capacity assignment:** Ensure your workspaces are assigned to this capacity
2. **Verify settings cascade:** Capacity settings override tenant settings
3. **Test workspace access:** Confirm users can see Git integration options
4. **Validate permissions:** Ensure workspace admins have appropriate delegation rights

> **⚠️ Important:** When using delegated tenant settings at the capacity level, these settings **override** the tenant-wide settings. This means you have more granular control but need to ensure all necessary capacities have the correct configuration for your CI/CD pipeline to work across DEV, UAT, and PROD environments.

#### 2.2 Workspace Structure for Architecture
For the Fabric deployment pipelines approach, you'll need multiple workspaces to support the complete CI/CD lifecycle. Here's an example of a well-structured workspace organization:

**Example Workspace Structure:**

<p align="center">
      <img src="images/ws_config_fabric.png" width="400" alt="Power BI Project"/>
</p>
<p align="center">
   <strong>Figure: Microsoft Fabric &rarr; Workspaces</strong>
</p> 

1. **Development Environment:**
   ```
   Workspace Name: FABRIC-CATALYST-GH-DEV
   Purpose: Main development workspace connected to Git main branch
   Role: Source workspace for deployment pipeline
   Git Connection: Connected to main branch
   ```

2. **Feature Development Workspaces:**
   ```
   Workspace Name: FABRIC-CATALYST-GH-FEATURE
   Purpose: Individual feature development and testing
   Role: Connected to feature branches for isolated development
   Git Connection: Connected to feature branches (feature/dashboard, feature/reports, etc.)
   ```

3. **User Acceptance Testing Environment:**
   ```
   Workspace Name: FABRIC-CATALYST-GH-STG (Staging/UAT)
   Purpose: User acceptance testing and validation
   Role: Target for DEV deployments, source for PROD deployments
   Git Connection: Managed through deployment pipeline (no direct Git connection)
   ```

4. **Production Environment:**
   ```
   Workspace Name: FABRIC-CATALYST-GH-PROD
   Purpose: Live production environment
   Role: Final deployment target for validated reports
   Git Connection: Managed through deployment pipeline (no direct Git connection)
   ```

**Workspace Naming Convention Benefits:**
✅ **Consistent Prefix:** `FABRIC-CATALYST-GH` identifies the project
✅ **Clear Environment Identification:** DEV, FEATURE, STG, PROD
✅ **Git Integration Indicator:** `GH` indicates GitHub integration
✅ **Professional Structure:** Enterprise-ready naming convention

**Workspace Assignment Requirements:**
- **All workspaces must be assigned to Premium capacity** (shown by capacity icons in screenshot)
- **Required for Git integration:** Only Premium capacity workspaces support Git
- **Required for deployment pipelines:** Both Git and deployment pipelines need Premium

**Workspace Role Mapping:**
```
Git Integration Workflow:
Power BI Desktop → FABRIC-CATALYST-GH-FEATURE → Feature Branch (Git)
Feature Branch → Main Branch (Pull Request/Merge)
Main Branch → FABRIC-CATALYST-GH-DEV (Sync/Update)

Deployment Pipeline Workflow:
FABRIC-CATALYST-GH-DEV → FABRIC-CATALYST-GH-STG → FABRIC-CATALYST-GH-PROD
(Deploy via Fabric deployment pipelines with approval gates)
```

**Next Steps for Your Configuration:**
1. ✅ **Workspaces created** with excellent naming convention
2. ✅ **Premium capacity assigned** (visible in screenshot)
3. **Connect FABRIC-CATALYST-GH-DEV to Git main branch**
4. **Connect FABRIC-CATALYST-GH-FEATURE to feature branches**
5. **Create deployment pipeline** linking DEV → STG → PROD
6. **Configure approval gates** for STG and PROD deployments