# sscm_to_intune
# Change Management Document: Transition from SCCM to Intune-Only Mode Using SCCM-NAA as DEM Account

---

## 1. Change Request Overview

### 1.1 Change Title
Transition Windows 10 Devices from SCCM to Intune-Only Mode Using SCCM-NAA as DEM Account

### 1.2 Change Description
This change involves transitioning all Windows 10 devices currently managed by SCCM (System Center Configuration Manager) to Microsoft Intune-only mode. The process includes:
- Uninstalling the SCCM agent.
- Enrolling devices into Intune via Azure AD Join using the **SCCM-NAA** account as a Device Enrollment Manager (DEM).
- Automating the transition using Group Policy and PowerShell scripts.

### 1.3 Justification
The transition to Intune-only mode is driven by the following business needs:
- **Modern Device Management**: Intune provides advanced cloud-based management capabilities, including app deployment, policy enforcement, and conditional access.
- **Cost Optimization**: Moving away from SCCM reduces infrastructure and operational costs associated with on-premises management.
- **Scalability and Flexibility**: Intune is designed to scale seamlessly, making it ideal for organizations with remote or hybrid workforces.
- **Enhanced Security**: Intune integrates with Azure AD, enabling robust security features like Conditional Access, compliance policies, and threat protection.
- **Simplified Operations**: Centralized management through Intune eliminates the complexity of maintaining a hybrid SCCM/Intune environment.

### 1.4 Scope
The scope of this change includes:
- Uninstalling the SCCM agent from all Windows 10 devices.
- Enrolling devices into Intune using the SCCM-NAA DEM account.
- Automating the process through Group Policy and PowerShell scripts.

---

## 2. Risk Assessment

### 2.1 Potential Risks
1. **Device Downtime**: Temporary unavailability of devices during the uninstallation and enrollment process.
2. **Data Loss**: Misconfiguration could result in loss of SCCM-managed configurations or settings.
3. **Enrollment Failures**: Network issues, misconfigurations, or insufficient permissions may prevent devices from enrolling in Intune.
4. **User Disruption**: End users may experience interruptions if applications or settings are not properly migrated.
5. **Credential Exposure**: The SCCM-NAA DEM account credentials must be securely managed to prevent unauthorized access.

### 2.2 Mitigation Strategies
To address these risks, the following measures will be implemented:
- **Pilot Testing**: Conduct a small-scale test on a subset of devices before full deployment.
- **Backups**: Create backups of SCCM configurations and critical device data.
- **Rollback Plan**: Develop a rollback strategy to reinstall the SCCM agent if necessary.
- **Secure Credentials**: Store the SCCM-NAA account credentials in an encrypted format and restrict access to authorized personnel.
- **Communication**: Notify users in advance and provide clear instructions and support channels.

---

## 3. Implementation Plan

### 3.1 Pre-Implementation Tasks
Before initiating the transition, the following prerequisites must be completed:
1. **Azure AD Configuration**:
   - Ensure the Azure AD tenant is properly configured.
   - Assign Intune licenses to all users or devices.
2. **Permissions**:
   - Verify that the Global Admin or Intune Admin role has been assigned to the responsible team.
3. **DEM Account Setup**:
   - Configure the **SCCM-NAA** account as a Device Enrollment Manager (DEM) in Azure AD.
4. **Scripts Preparation**:
   - Develop and test the `Uninstall-SCCMAgent.ps1` script for removing the SCCM agent.
   - Develop and test the `Enroll-Intune.ps1` script for enrolling devices into Intune.
5. **Group Policy Configuration**:
   - Configure Group Policy Objects (GPOs) to deploy the scripts automatically.

### 3.2 Implementation Steps
#### Step 1: Uninstall SCCM Agent
- Deploy the `Uninstall-SCCMAgent.ps1` script via Group Policy Startup Script.
- Monitor logs to confirm successful uninstallation of the SCCM agent.

#### Step 2: Enroll Devices in Intune
- Deploy the `Enroll-Intune.ps1` script via Group Policy Startup Script.
- Use the **SCCM-NAA** DEM account to perform Azure AD Join.
- Monitor the Intune portal to verify device enrollment.

#### Step 3: Validate Transition
- Confirm that all devices appear in the Intune portal.
- Check compliance with Intune policies to ensure proper configuration.

---

## 4. Rollback Plan

### 4.1 Rollback Trigger
A rollback will be initiated under the following conditions:
- More than 10% of devices fail to enroll in Intune.
- Critical business applications or services are disrupted.

### 4.2 Rollback Steps
1. **Reinstall SCCM Agent**:
   - Deploy the SCCM client installation script via Group Policy using the following command:
     ```powershell
     Start-Process -FilePath "\\SCCMServer\ccmsetup\ccmsetup.exe" -ArgumentList "/mp:SCCMServer FSP=SCCMServer" -Wait
     ```
2. **Disable Intune Enrollment**:
   - Remove the `Enroll-Intune.ps1` script from Group Policy.
3. **Verify SCCM Management**:
   - Confirm that devices reappear in the SCCM console and are functioning as expected.

---

## 5. Validation and Testing

### 5.1 Pilot Testing
- Select a small group of 5-10 devices for testing.
- Perform the following checks:
  - The SCCM agent is successfully uninstalled.
  - Devices are enrolled in Intune using the SCCM-NAA DEM account.
  - Devices comply with Intune policies.

### 5.2 Full Deployment Validation
- Monitor key metrics during the full deployment:
  - Number of devices successfully enrolled in Intune.
  - Compliance rate with Intune policies.
  - User-reported issues or disruptions.

### 5.3 Post-Deployment Testing
- Verify that critical applications and services are functioning correctly.
- Conduct user acceptance testing (UAT) to ensure a smooth transition for end users.

---

## 6. Communication Plan

### 6.1 Stakeholders
The following stakeholders will be involved in the change:
- IT Operations Team
- End Users
- Business Unit Leaders

### 6.2 Communication Channels
- Notify users via email about the upcoming change and its benefits.
- Publish an internal knowledge base article with FAQs and troubleshooting steps.
- Provide helpdesk support to address user concerns and issues.

---

## 7. Scripts

### 7.1 SCCM Agent Uninstallation Script
```powershell
# Uninstall SCCM Agent Script
$SCCMService = "ccmsetup"
$SCCMProcess = "ccmsetup.exe"

# Stop SCCM Service if running
if (Get-Service -Name $SCCMService -ErrorAction SilentlyContinue) {
    Stop-Service -Name $SCCMService -Force
}

# Uninstall SCCM Agent
if (Test-Path "$env:WinDir\ccmsetup\ccmsetup.exe") {
    Start-Process -FilePath "$env:WinDir\ccmsetup\ccmsetup.exe" -ArgumentList "/uninstall" -Wait
}

# Remove SCCM directories
Remove-Item -Path "$env:WinDir\ccmsetup" -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item -Path "$env:WinDir\CCM" -Recurse -Force -ErrorAction SilentlyContinue

Write-Host "SCCM Agent has been uninstalled successfully."

7.2 Intune Enrollment Script

# Intune Enrollment Script
param (
    [Parameter(Mandatory=$true)]
    [string]$AADUsername,
    [Parameter(Mandatory=$true)]
    [string]$AADPassword
)

# Convert password to secure string
$SecurePassword = ConvertTo-SecureString -String $AADPassword -AsPlainText -Force
$Credential = New-Object -TypeName System.Management.Automation.PSCredential -ArgumentList $AADUsername, $SecurePassword

# Perform Azure AD Join using SCCM-NAA DEM account
Add-Computer -DomainName "yourdomain.onmicrosoft.com" -Credential $Credential -Restart -Force

Write-Host "Device is being enrolled into Intune via Azure AD Join using SCCM-NAA DEM account."
