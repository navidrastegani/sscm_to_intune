# Mass Enrolling Hybrid Azure AD Joined Devices into Intune (After SCCM Agent Uninstallation) - Comprehensive Guide

## Introduction

This document provides a detailed, step-by-step guide to automatically mass enroll your hybrid Azure AD joined Windows devices into Microsoft Intune after successfully uninstalling the System Center Configuration Manager (SCCM) agent. This transition moves device management from SCCM to Intune, enabling modern, cloud-based management. This comprehensive guide is designed to be instructive and minimize potential issues during the migration process.

In addition to the automated enrollment script, this document now includes a PowerShell function, `Reset-IntuneEnrollment`, for advanced troubleshooting and manual re-enrollment of devices if needed. This function is not intended for automated mass deployment but is a valuable tool for administrators.

## Prerequisites - Detailed Verification Steps

Before initiating the mass enrollment, rigorously verify the following prerequisites are in place and correctly configured. Failure to do so can lead to enrollment failures and management issues.

1.  **Azure AD Connect Configured for Hybrid Azure AD Join - Detailed Verification:**
    *   **Active Directory Synchronization:** Ensure Azure AD Connect is actively synchronizing your on-premises Active Directory with Azure Active Directory.
        *   **Verification:** Open the Azure AD Connect console on your Azure AD Connect server. Check the "Synchronization Service Manager" to confirm that synchronization cycles are completing without errors and that the expected OUs and users are being synchronized.
    *   **Hybrid Azure AD Join Configuration:** Hybrid Azure AD Join must be properly configured within Azure AD Connect.
        *   **Verification:**
            *   Open the Azure AD Connect configuration wizard.
            *   Navigate to "Configure device options" and verify "Configure Hybrid Azure AD join" is enabled.
            *   Confirm you have selected the correct operating system environment (e.g., "Windows 10 and Later domain-joined devices").
            *   Ensure the correct authentication service is selected (typically "Azure Active Directory").
            *   Critically, verify that the **Domains** selected for Hybrid Azure AD Join accurately reflect the Active Directory domains where your target devices reside. Incorrect domain selection is a common cause of Hybrid Azure AD Join failures.
    *   **Organizational Unit (OU) Scope:** Confirm that the Organizational Units (OUs) containing the devices you intend to enroll in Intune are within the synchronization scope of Azure AD Connect.
        *   **Verification:** In the Azure AD Connect configuration, review the OU filtering settings to ensure the relevant OUs are included in the synchronization.
    *   **Service Connection Point (SCP) Configuration:** The Service Connection Point (SCP) in your on-premises Active Directory must be correctly configured to point to your Azure AD tenant. This is essential for devices to discover your Azure AD tenant for Hybrid Azure AD Join.
        *   **Verification:**
            *   Open **ADSI Edit** (`adsiedit.msc`) on a Domain Controller or a server with RSAT tools.
            *   Connect to the **Configuration** naming context.
            *   Navigate to: `Configuration [yourdomain.com] -> CN=Configuration,DC=yourdomain,DC=com -> CN=Services -> CN=Device Registration Configuration`.
            *   Right-click on `CN=Device Registration Configuration` and select "Properties".
            *   Examine the `keywords` attribute. It must contain entries similar to:
                *   `azureADId:<Your Azure AD Tenant ID>` (Replace `<Your Azure AD Tenant ID>` with your actual Azure AD Tenant ID - GUID format)
                *   `azureADName:AzureAD`
                *   `azureADServiceEndpointUri:https://enterpriseregistration.windows.net` (or appropriate endpoint for your cloud environment)
                *   `MicrosoftOnlineDirectoryServiceEndpointUri:https://msods.windows.net` (or appropriate endpoint)
            *   If SCP is missing or incorrectly configured, re-run the Azure AD Connect Hybrid Azure AD Join configuration wizard to rectify.

2.  **Active Microsoft Intune Subscription:** You must have an active and valid Microsoft Intune subscription associated with your Azure AD tenant.
    *   **Verification:** Log in to the [Microsoft Intune admin center](https://endpoint.microsoft.com/) ([https://endpoint.microsoft.com/](https://www.google.com/url?sa=E&source=gmail&q=https://endpoint.microsoft.com/)) using your administrative credentials. If you can access the Intune portal without licensing errors, your subscription is likely active. Double-check your Microsoft 365 subscription status in the Microsoft 365 admin center to confirm Intune is included and active.

3.  **Microsoft Intune Licenses Assigned to Users:** Each user who will be the primary user of a device being enrolled in Intune must be assigned a Microsoft Intune license.
    *   **Verification:** In the Microsoft 365 admin center ([https://admin.microsoft.com/](https://www.google.com/url?sa=E&source=gmail&q=https://admin.microsoft.com/)), navigate to "Users" > "Active users". Select a test user who will be logging into a device you plan to enroll. Check their "Licenses and apps" to ensure a Microsoft Intune license (e.g., Microsoft 365 E3, Microsoft 365 E5, Microsoft Intune Plan 1, etc.) is assigned. Plan for licensing all users who will be primary users of managed devices.

4.  **Administrative Privileges - Ensure Correct Account Usage:**
    *   **Domain Administrator Privileges:** You must be logged in with an account that has Domain Administrator privileges in your on-premises Active Directory to create and link Group Policy Objects.
    *   **Azure AD Global Administrator or Intune Administrator Privileges:** You need to use an account with either Azure AD Global Administrator or Intune Administrator roles to manage settings within the Microsoft Intune admin center and Azure AD portal.  Use separate accounts as needed for on-premises AD and Azure AD administration, ensuring you are using the correct credentials for each environment.

5.  **Network Connectivity - Validate Device Internet Access:** Devices must have reliable internet connectivity to communicate with Azure AD and Intune services throughout the enrollment process. Firewalls or proxy servers must not block necessary endpoints.
    *   **Verification:** On a test device, open a web browser and attempt to access:
        *   `https://login.microsoftonline.com` (Azure AD login endpoint)
        *   `https://device.login.microsoftonline.com` (Device registration endpoint)
        *   `https://enrollment.manage.microsoft.com` (Intune enrollment endpoint)
        *   `https://portal.azure.com` (Azure portal)
        *   `https://endpoint.microsoft.com` (Intune admin center)
        *   Ensure these websites load without certificate errors or connectivity issues. If you use a proxy server, ensure it is correctly configured to allow traffic to these Microsoft online services.

6.  **SCCM Agent Uninstallation Completed - Confirm Agent Removal:** Before proceeding with Intune enrollment, rigorously verify that the SCCM agent uninstallation process, preferably using the Group Policy method described in the previous document, has been successfully applied to your target devices.
    *   **Verification (Repeat for multiple test devices):**
        *   **Control Panel:** On a test device, check the Control Panel. The "Configuration Manager" Control Panel applet **must not be present**. If it is still there, the SCCM agent is not fully uninstalled.
        *   **Services:** Open the Services application (`services.msc`). Verify that the **"SMS Agent Host"** service is **not listed**. If it is still running or present, the agent uninstallation was not successful.
        *   **Logs:** Examine the log file at `%windir%\ccmsetup\logs\CCMSetup.log`. Open this file using Notepad or a text editor. Search for the phrase **"Uninstall completed successfully"**. The presence of this entry is a strong indicator of successful uninstallation. If you cannot find this, or see error messages, the uninstallation may have failed and needs to be re-addressed.

## Step 1: Re-verify SCCM Agent Uninstallation (Crucial Step)

Before moving forward, it is critical to re-verify that the SCCM agent uninstallation has been fully and successfully applied to your target devices.  Incomplete SCCM agent removal can interfere with Intune enrollment or cause conflicts.

1.  **Systematic Check Across Test Devices:** Perform the verification steps from prerequisite #6 on a representative sample of devices from your target OUs. Do not proceed to Intune enrollment until you are absolutely certain the SCCM agent is uninstalled on these devices.

2.  **Address Devices with Failed Uninstallation:** If you identify any devices where the SCCM agent is still present, revisit your SCCM agent uninstallation GPO. Ensure it is correctly configured, linked to the right OUs, and is being applied. You may need to:
    *   Force a Group Policy update on the affected devices (`gpupdate /force`).
    *   Restart the devices again to ensure the startup script runs.
    *   Review event logs on the devices for any errors related to GPO processing or script execution.
    *   Consult the `CCMSetup.log` file on devices that failed uninstallation for specific error messages that can guide troubleshooting.

3.  **Document Findings:** Keep a record of devices checked and their SCCM agent uninstallation status. Only proceed to the next steps once you have high confidence in the successful SCCM agent removal across your target device population.

## Step 2: Configure Hybrid Azure AD Join - Step-by-Step Configuration and Verification

If you are certain that Hybrid Azure AD Join is already configured, you should still perform the verification steps outlined in the Prerequisites section to confirm its correct setup. If you are setting up Hybrid Azure AD Join for the first time, follow these detailed steps:

1.  **Configure Hybrid Azure AD Join in Azure AD Connect:**
    *   **Open Azure AD Connect:** On your Azure AD Connect server, launch the Azure AD Connect application.
    *   **Click "Configure":** Select "Configure" from the main menu.
    *   **Choose "Configure device options":** Select the task "Configure device options" and click "Next".
    *   **Enter Azure AD Credentials:** Provide credentials for an Azure AD Global Administrator account.
    *   **Select "Configure Hybrid Azure AD join":** On the "Device options" page, ensure the box next to "Configure Hybrid Azure AD join" is checked. Click "Next".
    *   **Choose Operating System Environment:** Select the option that corresponds to your environment (e.g., "Windows 10 and Later domain-joined devices"). Click "Next".
    *   **Authentication Service:** Select "Azure Active Directory" as the authentication service. Click "Next".
    *   **Domain Selection:** **Carefully select the Active Directory domains** that contain the computers you want to Hybrid Azure AD join. **Incorrect domain selection is a common error.** Ensure you select the parent domains of the OUs where your target devices reside. Click "Next".
    *   **SCP Configuration Review:** Azure AD Connect will configure the Service Connection Point (SCP) in the selected domains. Review the summary of actions.
    *   **Configure and Exit:** Click "Configure" to apply the Hybrid Azure AD Join settings. Wait for the configuration to complete. Once done, click "Exit".

2.  **Verify Service Connection Point (SCP) in Active Directory - Detailed Procedure:**
    *   **Open ADSI Edit:** Launch ADSI Edit (`adsiedit.msc`) on a Domain Controller or a server with RSAT tools installed.
    *   **Connect to Configuration Naming Context:** Right-click "ADSI Edit" in the left pane and select "Connect to...". In the "Connection Settings" dialog, under "Select a well-known Naming Context:", choose "Configuration" and click "OK".
    *   **Navigate to Device Registration Configuration Container:** Expand the tree in the left pane to navigate to: `Configuration [yourdomain.com] -> CN=Configuration,DC=yourdomain,DC=com -> CN=Services -> CN=Device Registration Configuration`.
    *   **Access Properties of CN=Device Registration Configuration:** Right-click on `CN=Device Registration Configuration` and select "Properties".
    *   **Examine the `keywords` Attribute:** In the Properties window, locate and select the `keywords` attribute. Click "Edit".
    *   **Verify `keywords` Values:** In the String Attribute Editor, carefully verify that the following values are present and correctly formatted. **Pay close attention to the Azure AD Tenant ID (GUID) and ensure it matches your Azure AD tenant ID.**
        *   `azureADId:<Your Azure AD Tenant ID>` (Replace `<Your Azure AD Tenant ID>` with your actual Azure AD Tenant ID - GUID format)
        *   `azureADName:AzureAD`
        *   `azureADServiceEndpointUri:https://enterpriseregistration.windows.net` (or the correct endpoint for your Azure cloud - e.g., for US Government cloud, it would be different)
        *   `MicrosoftOnlineDirectoryServiceEndpointUri:https://msods.windows.net` (or the correct endpoint for your cloud)
    *   **Correct SCP if Necessary:** If any of these values are missing or incorrect, it indicates an issue with the Hybrid Azure AD Join configuration. Re-run the Azure AD Connect Hybrid Azure AD Join configuration wizard to correct the SCP settings. Manually editing the SCP in ADSI Edit is **not recommended** unless you are highly experienced with Active Directory schema and Azure AD Connect configurations.

3.  **Group Policy for Hybrid Azure AD Join - For Devices Not Automatically Hybrid Joined (and Verification):**
    *   **Open Group Policy Management:** Launch Group Policy Management Console (`gpmc.msc`).
    *   **Locate or Create GPO:** Find an existing GPO that is applied to the OUs containing your target devices, or create a new GPO specifically for Hybrid Azure AD Join and Intune enrollment. It is best practice to create a dedicated GPO for device enrollment settings. Link this GPO to the OUs containing the devices you want to manage in Intune.
    *   **Edit the GPO:** Right-click on the GPO and select "Edit".
    *   **Navigate to Device Registration Policy:** In the Group Policy Management Editor, navigate to: `Computer Configuration -> Policies -> Administrative Templates -> Windows Components -> Device Registration`.
    *   **Configure "Register domain-joined computers as devices":** Double-click on the policy setting "Register domain-joined computers as devices".
    *   **Enable the Policy and Select Hybrid Azure AD Join:** In the policy setting dialog:
        *   Set the policy to **"Enabled"**.
        *   From the dropdown menu under "Options", select **"Hybrid Azure AD joined"**.
        *   Click "Apply" and then "OK".
    *   **Link the GPO to OUs:** Ensure the GPO is linked to the Organizational Units that contain the devices you want to Hybrid Azure AD join and enroll in Intune. Verify the GPO link is enabled.
    *   **Force Group Policy Update on a Test Device:** On a test device within the target OU, open Command Prompt as administrator and run the command `gpupdate /force`. Press Enter and wait for the policy update to complete.
    *   **Restart the Test Device:** Restart the test device to ensure the Group Policy settings are fully applied, especially device-level settings.
    *   **Verify Hybrid Azure AD Join Status on Test Device - Detailed Verification:** After the device restarts and you log in:
        *   Open Command Prompt as administrator.
        *   Run the command `dsregcmd /status` and press Enter.
        *   Examine the output carefully. Look for the following key indicators under "Device State":
            *   **`AzureAdJoined` : YES** - This confirms the device is successfully Azure AD joined.
            *   **`DomainJoined` : YES** - This confirms the device is still domain-joined to your on-premises Active Directory.
            *   **`AzureAdPrt` : YES** - (Ideally) This indicates the device has a Primary Refresh Token from Azure AD, which is often seen in successful Hybrid Azure AD Join scenarios.
        *   If `AzureAdJoined` is **NO**, Hybrid Azure AD Join has failed. Review the `dsregcmd /status` output for error messages under "Diagnostic Data" or "User Context". Common issues include SCP misconfiguration, network connectivity problems, or issues with Azure AD Connect synchronization. Troubleshoot and resolve Hybrid Azure AD Join issues before proceeding to Intune enrollment.

## Step 3: Configure Intune Auto-Enrollment via Group Policy - Detailed Configuration

Once you have confirmed successful SCCM agent uninstallation and Hybrid Azure AD Join is correctly configured and working for your devices, proceed with configuring Intune auto-enrollment via Group Policy.

1.  **Open Group Policy Management:** Launch the Group Policy Management Console (`gpmc.msc`).

2.  **Locate the GPO:** Find the same GPO you used for Hybrid Azure AD Join in Step 2 (or the new GPO you created for device enrollment). Edit this GPO by right-clicking and selecting "Edit".

3.  **Navigate to MDM Enrollment Policy:** In the Group Policy Management Editor, navigate to: `Computer Configuration -> Policies -> Administrative Templates -> Windows Components -> MDM Enrollment`.

4.  **Enable Automatic MDM Enrollment Policy - Detailed Settings:** Double-click on the policy setting "Enable automatic MDM enrollment using default Azure AD credentials".

    *   **Set to "Enabled":** In the policy setting dialog, set the radio button to **"Enabled"**.
    *   **Credential Type:** Under "Options", ensure that the dropdown menu "Select Credential type to use" is set to **"User Credential"**. This is the default and recommended setting for user-driven Intune enrollment in Hybrid Azure AD Join scenarios.  **Do not change this to "Device Credential" unless you have a very specific and advanced scenario requiring device-based enrollment without user context, which is not typical for standard user devices.**
    *   **Apply and OK:** Click "Apply" and then "OK" to save the policy setting.

5.  **Link and Enforce GPO - Ensure Policy Application:** Verify that the GPO containing the Intune auto-enrollment policy is correctly linked to the Organizational Units that contain your target devices. Ensure the GPO link is enabled.  Enforce the GPO link if you need to ensure that these settings are prioritized and not overridden by other GPOs (use enforcement cautiously and only if necessary).

6.  **Apply Group Policy Update on Test Device:** On a test device within the target OU, open Command Prompt as administrator and run `gpupdate /force`. Press Enter and wait for the policy update to complete. Restart the test device.

## Step 4: Verify Intune Enrollment - Comprehensive Verification Steps

After configuring the GPO for Intune auto-enrollment, applying it, and restarting devices, it is essential to thoroughly verify that devices are successfully enrolling in Intune.

1.  **Check Enrollment Status on Device - Detailed Verification:**
    *   **Restart and User Login:** Restart the test device and have a licensed user log in with their Azure AD credentials (the same credentials they use for domain login, as part of Hybrid Azure AD Join).
    *   **Wait for Enrollment:** After login, allow sufficient time for the device to enroll in Intune. Enrollment processes run in the background and may take several minutes to complete, especially on the first enrollment. Wait for at least 15-30 minutes after user login before proceeding with verification.
    *   **Access "Access work or school" Settings:** Go to the Windows "Settings" app. Click on "Accounts", and then select "Access work or school".
    *   **Verify Azure AD Connection:** You should see a connection listed here, indicating the device is connected to your Azure AD tenant. The connection name will typically be the name of your organization or Azure AD tenant. If you do not see a connection listed, enrollment has likely not started or has failed.
    *   **Check Intune Management Status - Detailed Information:** Click on the connected account (e.g., your organization's name) in "Access work or school", and then click on the "Info" button.
        *   **"Managed by: Microsoft Intune":** Under the "Device management" section, you **must** see the text "Managed by: Microsoft Intune". This is the definitive confirmation that the device is successfully enrolled in Intune management. If it says something else or is blank, the device is not Intune managed.
        *   **"Connection info":** Click on "Connection info" to view technical details about the MDM connection, including the management server address. Verify that the "Management Server Address" points to the Intune service endpoint (typically `https://dm.microsoft.com`).
    *   **Check Registry (Advanced Verification - Optional):** For more technical verification, you can check the registry:
        *   Open Registry Editor (`regedit.exe`).
        *   Navigate to `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Enrollments`.
        *   Under this key, you should see subkeys representing enrollment sessions. Look for a subkey that corresponds to the user who logged in and initiated enrollment.
        *   Within this subkey, you should find values indicating successful MDM enrollment, such as `EnrollmentStatus` being set to `1` (Success).

2.  **Verify in Intune Admin Center - Confirm Device Listing and Management:**
    *   **Open Intune Admin Center:** Access the Microsoft Intune admin center in a web browser ([https://endpoint.microsoft.com/](https://endpoint.microsoft.com/)).
    *   **Navigate to "Devices" -> "Windows":** In the Intune portal, go to "Devices" in the left-hand navigation menu, and then select "Windows".
    *   **Search for the Device:** Use the "Search" function (or browse the device list) to find the name of the test device you are verifying.
    *   **Confirm "Managed By" Column:** In the device list, locate the "Managed by" column for your test device. It should clearly indicate **"Intune"**. If it shows "ConfigMgr" or "Co-management", or is blank, the device is not solely Intune managed as expected.
    *   **Device Details and Enrollment Status:** Click on the device name to open its details page. Review the "Overview" and "Properties" tabs.
        *   **Enrollment Status:** Check the "Enrollment status" section. It should show "Enrolled in Microsoft Intune".
        *   **Compliance Status:** Initially, compliance state might be "Pending" or "Not evaluated". After Intune policies are applied and evaluated, it should reflect the actual compliance status.
        *   **Device Properties:** Review other device properties like OS version, Azure AD Join type (should be "Hybrid Azure AD joined"), and management details.

3.  **Troubleshooting Enrollment Issues - Detailed Troubleshooting Steps:** If a device fails to enroll in Intune:
    *   **Review Event Logs - Focus on MDM Enrollment Logs:** On the affected device, examine the **"DeviceManagement-Enterprise-Diagnostics-Provider"** event logs located under `Applications and Services Logs -> Microsoft -> Windows -> DeviceManagement-Enterprise-Diagnostics-Provider` in Event Viewer (`eventvwr.msc`).
        *   **Look for Errors and Warnings:** Filter for "Error" and "Warning" events within the last few hours, specifically related to MDM enrollment, Auto-enrollment, or Device Registration.
        *   **Analyze Event Details:** Carefully read the details of any error or warning events. Event IDs and descriptions can provide clues about the cause of enrollment failure (e.g., network connectivity issues, authentication problems, licensing issues, GPO application problems, etc.). Note down any specific error codes or messages.
    *   **Verify GPO Application - Use `gpresult` in Detail:** On the device, run `gpresult /h <filename>.html` in Command Prompt as administrator to generate a detailed Group Policy Results report in HTML format. Open the HTML report in a web browser.
        *   **Check Applied GPOs:** In the report, navigate to "Computer Settings" -> "Policies" -> "Windows Settings" -> "Scripts (Startup/Shutdown)" and confirm your Intune enrollment startup script is listed and applied without errors.
        *   **Verify MDM Enrollment Policy:** Also, in "Computer Settings" -> "Policies" -> "Administrative Templates" -> "Windows Components" -> "MDM Enrollment", verify that the "Enable automatic MDM enrollment using default Azure AD credentials" policy is listed and set to "Enabled".
        *   **Look for Policy Conflicts or Errors:** Review the entire `gpresult` report for any policy conflicts, errors in GPO processing, or indications that the GPOs are not being applied correctly to the device.
    *   **Network Connectivity Tests - Advanced Checks:** Ensure devices can reliably reach all necessary Azure AD and Intune endpoints.
        *   **Endpoint Verification:** Use `Test-NetConnection` PowerShell cmdlet to test connectivity to key Microsoft endpoints on ports 80 and 443 (e.g., `Test-NetConnection login.microsoftonline.com -Port 443`, `Test-NetConnection device.login.microsoftonline.com -Port 443`, `Test-NetConnection enrollment.manage.microsoft.com -Port 443`, `Test-NetConnection dm.microsoft.com -Port 443`, `Test-NetConnection portal.azure.com -Port 443`, `Test-NetConnection endpoint.microsoft.com -Port 443`).
        *   **Proxy and Firewall Review:** If you use proxy servers or firewalls, meticulously review their configurations to ensure that they are not blocking traffic to the required Microsoft online services. You may need to add exceptions or allow rules for these endpoints. Consult your network and security teams to verify firewall and proxy settings.
    *   **Licensing Re-Verification:** Double-check that the user account used for testing Intune enrollment has a valid and properly assigned Microsoft Intune license. Verify license assignment in the Microsoft 365 admin center. License assignment delays can sometimes occur; allow some time after assigning licenses before troubleshooting enrollment failures.
    *   **Time Synchronization:** Ensure the device's system clock is correctly synchronized with a reliable time source. Time synchronization issues can sometimes cause authentication problems with Azure AD.
    *   **Restart Device Again:** In some cases, a second restart of the device after applying the GPO settings can resolve transient issues and allow enrollment to complete successfully.

## Step 5: Post-Enrollment Tasks and Considerations - Expanding Management

After you have successfully mass enrolled your hybrid Azure AD joined devices into Intune, you can proceed with the following post-enrollment tasks to fully manage and secure these devices using Intune.

1.  **Deploy Intune Configuration Policies:** Begin creating and deploying Intune configuration policies to enforce desired settings on your managed devices.
    *   **Examples of Configuration Policies:**
        *   **Device Configuration Profiles:** Configure Wi-Fi profiles, VPN settings, email profiles, certificates, device restrictions (e.g., camera, Bluetooth, cloud storage access), browser settings, and more.
        *   **Endpoint Security Policies:** Implement Antivirus, Firewall, Endpoint Detection and Response (EDR), and Account protection policies to enhance device security.
        *   **Administrative Templates:** Leverage Administrative Templates (ADMX-backed policies) to manage a wide range of Windows settings using a familiar Group Policy-like interface within Intune.
    *   **Policy Deployment Strategy:** Plan your policy deployment strategy. Start with baseline security and configuration policies. Test policies on pilot groups before wider deployment. Use Intune policy assignment filters to target policies to specific device groups or user groups as needed.

2.  **Deploy Intune Compliance Policies:** Define compliance policies to establish health and security baselines for your managed devices.
    *   **Examples of Compliance Settings:**
        *   Password complexity and length requirements.
        *   BitLocker encryption requirement.
        *   Antivirus status and real-time protection.
        *   Firewall enabled status.
        *   Operating system version requirements.
        *   Device health attestation.
    *   **Compliance Actions:** Configure actions for non-compliant devices, such as sending email notifications to users, blocking access to corporate resources (Conditional Access integration), or remotely retiring non-compliant devices.

3.  **Deploy Applications via Intune:** Start deploying applications to your managed devices using Intune's application management capabilities.
    *   **Application Types:** Intune supports deploying various application types, including:
        *   Microsoft Store apps
        *   Win32 apps (traditional desktop applications - MSI, EXE)
        *   Microsoft 365 Apps
        *   Web apps
        *   Line-of-business (LOB) apps
    *   **Application Deployment Methods:** Choose appropriate deployment methods (Required, Available for enrolled devices, Uninstall). Utilize Intune's app assignment features to target apps to users or devices based on groups. Leverage Intune's app update management features to keep applications current.

4.  **Monitoring and Reporting in Intune:** Utilize Intune's built-in monitoring and reporting features to track device enrollment status, policy compliance, application deployment status, and device health.
    *   **Intune Dashboards:** Regularly review Intune dashboards in the admin center for an overview of your managed environment.
    *   **Reports:** Generate and schedule reports to monitor device compliance, enrollment trends, policy effectiveness, and application installation status. Use Intune's reporting data to identify and address any issues or trends in your managed device environment.
    *   **Alerts and Notifications:** Configure Intune alerts to be notified of critical events, such as compliance violations, enrollment failures, or security threats.

## Step 6: Advanced Troubleshooting - Reset-IntuneEnrollment PowerShell Function

The following PowerShell function, `Reset-IntuneEnrollment`, is provided as an **advanced troubleshooting tool**. It is designed to be run **manually on a device** by an administrator if a device is experiencing persistent Intune enrollment issues. **Do not deploy this function via Group Policy startup script for mass execution.**  It performs a series of actions to reset the Intune management connection on a device, including un-joining from Azure AD, removing Intune certificates and registry data, and attempting to re-enroll the device.

**Important Security Warning:** This script contains functions (`Invoke-AsSystem`, `Connect-Graph`) that require careful handling and may require specific permissions or module installations (Microsoft.Graph.Intune). Use this script with caution and only when necessary for troubleshooting.  Understand what each part of the script does before execution.

```powershell
function Reset-IntuneEnrollment {
    <#
    .SYNOPSIS
    Function for resetting device Intune management connection.

    .DESCRIPTION
    Function for resetting device Intune management connection.

    It will:
     - check actual Intune status on device
     - reset Hybrid AzureAD join
     - remove device records from Intune
     - remove Intune connection data and invoke re-enrollment

    .PARAMETER computerName
    (optional) Name of the computer.

    .EXAMPLE
    Reset-IntuneEnrollment

    .NOTES
    # How MDM (Intune) enrollment works [https://techcommunity.microsoft.com/t5/intune-customer-success/support-tip-understanding-auto-enrollment-in-a-co-managed/ba-p/834780](https://techcommunity.microsoft.com/t5/intune-customer-success/support-tip-understanding-auto-enrollment-in-a-co-managed/ba-p/834780)
    #>

    [CmdletBinding()]
    param (
        [string] $computerName = $env:COMPUTERNAME
    )

    $ErrorActionPreference = "Stop"

    #region helper functions
    function Connect-Graph {
        <#
        .SYNOPSIS
        Function for connecting to Microsoft Graph.

        .DESCRIPTION
        Function for connecting to Microsoft Graph.
        Support interactive authentication or application authentication
        Without specifying any parameters, interactive auth. will be used.

        .PARAMETER TenantId
        ID of your tenant.

        Default is "e4fb6bec-b1f4-46dc-9ab8-c67549adc56d"

        .PARAMETER AppId
        Azure AD app ID (GUID) for the application that will be used to authenticate

        .PARAMETER AppSecret
        Specifies the Azure AD app secret corresponding to the app ID that will be used to authenticate.
        Can be generated in Azure > 'App Registrations' > SomeApp > 'Certificates & secrets > 'Client secrets'.

        .PARAMETER Beta
        Set schema to beta.

        .EXAMPLE
        Connect-Graph

        .NOTES
        Requires module Microsoft.Graph.Intune
        #>

        [CmdletBinding()]
        [Alias("Connect-MSGraph2", "Connect-MSGraphApp2")]
        param (
            [string] $TenantId = "e4fb6bec-b1f4-46dc-9ab8-c67549adc56d"
            ,
            [string] $AppId
            ,
            [string] $AppSecret
            ,
            [switch] $beta
        )

        if (!(Get-Command Connect-MSGraph, Connect-MSGraphApp -ea silent)) {
            throw "Module Microsoft.Graph.Intune is missing"
        }

        if ($beta) {
            if ((Get-MSGraphEnvironment).SchemaVersion -ne "beta") {
                $null = Update-MSGraphEnvironment -SchemaVersion beta
            }
        }

        if ($TenantId -and $AppId -and $AppSecret) {
            $graph = Connect-MSGraphApp -Tenant $TenantId -AppId $AppId -AppSecret $AppSecret -ea Stop
            Write-Verbose "Connected to Intune tenant $TenantId using app-based authentication (Azure AD authentication not supported)"
        } else {
            $graph = Connect-MSGraph -ea Stop
            Write-Verbose "Connected to Intune tenant $($graph.TenantId)"
        }
    }

    function Invoke-MDMReenrollment {
        <#
        .SYNOPSIS
        Function for resetting device Intune management connection.

        .DESCRIPTION
        Force re-enrollment of Intune managed devices.

        It will:
        - remove Intune certificates
        - remove Intune scheduled tasks & registry keys
        - force re-enrollment via DeviceEnroller.exe

        .PARAMETER computerName
        (optional) Name of the remote computer, which you want to re-enroll.

        .PARAMETER asSystem
        Switch for invoking re-enroll as a SYSTEM instead of logged user.

        .EXAMPLE
        Invoke-MDMReenrollment

        Invoking re-enroll to Intune on local computer under logged user.

        .EXAMPLE
        Invoke-MDMReenrollment -computerName PC-01 -asSystem

        Invoking re-enroll to Intune on computer PC-01 under SYSTEM account.

        .NOTES
        [https://www.maximerastello.com/manually-re-enroll-a-co-managed-or-hybrid-azure-ad-join-windows-10-pc-to-microsoft-intune-without-loosing-current-configuration/](https://www.maximerastello.com/manually-re-enroll-a-co-managed-or-hybrid-azure-ad-join-windows-10-pc-to-microsoft-intune-without-loosing-current-configuration/)

        Based on work of MauriceDaly.
        #>

        [Alias("Invoke-IntuneReenrollment")]
        [CmdletBinding()]
        param (
            [string] $computerName,

            [switch] $asSystem
        )

        if ($computerName -and $computerName -notin "localhost", $env:COMPUTERNAME) {
            if (! ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
                throw "You don't have administrator rights"
            }
        }

        $allFunctionDefs = "function Invoke-AsSystem { ${function:Invoke-AsSystem} }"

        $scriptBlock = {
            param ($allFunctionDefs, $asSystem)

            try {
                foreach ($functionDef in $allFunctionDefs) {
                    . ([ScriptBlock]::Create($functionDef))
                }

                Write-Host "Checking for MDM certificate in computer certificate store"

                # Check&Delete MDM device certificate
                Get-ChildItem 'Cert:\LocalMachine\My\' | ? Issuer -EQ "CN=Microsoft Intune MDM Device CA" | % {
                    Write-Host " - Removing Intune certificate $($_.DnsNameList.Unicode)"
                    Remove-Item $_.PSPath
                }

                # Obtain current management GUID from Task Scheduler
                $EnrollmentGUID = Get-ScheduledTask | Where-Object { $_.TaskPath -like "*Microsoft*Windows*EnterpriseMgmt\*" } | Select-Object -ExpandProperty TaskPath -Unique | Where-Object { $_ -like "*-*-*" } | Split-Path -Leaf

                # Start cleanup process
                if (![string]::IsNullOrEmpty($EnrollmentGUID)) {
                    Write-Host "Current enrollment GUID detected as $([string]$EnrollmentGUID)"

                    # Stop Intune Management Exention Agent and CCM Agent services
                    Write-Host "Stopping MDM services"
                    if (Get-Service -Name IntuneManagementExtension -ErrorAction SilentlyContinue) {
                        Write-Host " - Stopping IntuneManagementExtension service..."
                        Stop-Service -Name IntuneManagementExtension
                    }
                    if (Get-Service -Name CCMExec -ErrorAction SilentlyContinue) {
                        Write-Host " - Stopping CCMExec service..."
                        Stop-Service -Name CCMExec
                    }

                    # Remove task scheduler entries
                    Write-Host "Removing task scheduler Enterprise Management entries for GUID - $([string]$EnrollmentGUID)"
                    Get-ScheduledTask | Where-Object { $_.Taskpath -match $EnrollmentGUID } | Unregister-ScheduledTask -Confirm:$false
                    # delete also parent folder
                    Remove-Item -Path "$env:WINDIR\System32\Tasks\Microsoft\Windows\EnterpriseMgmt\$EnrollmentGUID" -Force

                    $RegistryKeys = "HKLM:\SOFTWARE\Microsoft\Enrollments", "HKLM:\SOFTWARE\Microsoft\Enrollments\Status", "HKLM:\SOFTWARE\Microsoft\EnterpriseResourceManager\Tracked", "HKLM:\SOFTWARE\Microsoft\PolicyManager\AdmxInstalled", "HKLM:\SOFTWARE\Microsoft\PolicyManager\Providers", "HKLM:\SOFTWARE\Microsoft\Provisioning\OMADM\Accounts", "HKLM:\SOFTWARE\Microsoft\Provisioning\OMADM\Logger", "HKLM:\SOFTWARE\Microsoft\Provisioning\OMADM\Sessions"
                    foreach ($Key in $RegistryKeys) {
                        Write-Host "Processing registry key $Key"
                        # Remove registry entries
                        if (Test-Path -Path $Key) {
                            # Search for and remove keys with matching GUID
                            Write-Host " - GUID entry found in $Key. Removing..."
                            Get-ChildItem -Path $Key | Where-Object { $_.Name -match $EnrollmentGUID } | Remove-Item -Recurse -Force -Confirm:$false -ErrorAction SilentlyContinue
                        }
                    }

                    # Start Intune Management Extension Agent service
                    Write-Host "Starting MDM services"
                    if (Get-Service -Name IntuneManagementExtension -ErrorAction SilentlyContinue) {
                        Write-Host " - Starting IntuneManagementExtension service..."
                        Start-Service -Name IntuneManagementExtension
                    }
                    if (Get-Service -Name CCMExec -ErrorAction SilentlyContinue) {
                        Write-Host " - Starting CCMExec service..."
                        Start-Service -Name CCMExec
                    }

                    # Sleep
                    Write-Host "Waiting for 30 seconds prior to running DeviceEnroller"
                    Start-Sleep -Seconds 30

                    # Start re-enrollment process
                    Write-Host "Calling: DeviceEnroller.exe /C /AutoenrollMDM"
                    if ($asSystem) {
                        Invoke-AsSystem -runAs SYSTEM -scriptBlock { Start-Process -FilePath "$env:WINDIR\System32\DeviceEnroller.exe" -ArgumentList "/C /AutoenrollMDM" -NoNewWindow -Wait -PassThru }
                    } else {
                        Start-Process -FilePath "$env:WINDIR\System32\DeviceEnroller.exe" -ArgumentList "/C /AutoenrollMDM" -NoNewWindow -Wait -PassThru
                    }

            } else {
                # Apparently Intune has never been configured, so just start the enrollment task
                if($tasks = @(Get-ScheduledTask | Where-Object { $_.TaskPath -like "*Microsoft*Windows*EnterpriseMgmt\*" })) {
                    # There should be only one task here
                    if($tasks.Count -eq 1) {
                        $tasks | Start-ScheduledTask
                    } else {
                        $tasks | ft
                        throw "Unsure about enrollment task to be run. This might be a bug in the script"
                    }
                } else {
                    throw "Unable to obtain enrollment task from task scheduler. Aborting"
                }
            }

            # check certificates
            $i = 30
            Write-Host "Waiting for Intune certificate creation"  -ForegroundColor Cyan
            Write-Verbose "two certificates should be created in Computer Personal cert. store (issuer: MS-Organization-Access, MS-Organization-P2P-Access [$(Get-Date -Format])]
