# Uninstalling SCCM Agents from AD Joined Devices Automatically using Group Policy Object (GPO)

## Method 1: Uninstalling SCCM Agent using a Startup Script via Group Policy Object (GPO) - Step-by-Step Guide

This method uses Group Policy to execute a script during device startup, ensuring that when a device within the scope of the GPO restarts, the SCCM agent uninstallation process is initiated automatically.

**Step 1: Prepare the Uninstallation Script**

First, you need to create a script that will perform the SCCM agent uninstallation. We will use a batch script for this example.

1.  **Open Notepad:** Open a plain text editor like Notepad.

2.  **Enter the Script Content:** Copy and paste the following batch script code into Notepad:

    ```batch
    @echo off
    echo Uninstalling SCCM Agent...
    if exist "%windir%\ccmsetup\ccmsetup.exe" (
        "%windir%\ccmsetup\ccmsetup.exe" /uninstall
        echo Uninstall command executed.
    ) else (
        echo CCMSetup.exe not found. SCCM Agent may not be installed or already uninstalled.
    )
    echo Uninstall process completed.
    ```

3.  **Save the Script:**
    *   Click **"File"** > **"Save As"**.
    *   In the "Save As" dialog:
        *   **Save in:** Choose a location you can easily access temporarily (like your Desktop).
        *   **File name:** Type `UninstallSCCMAgent.bat`.
        *   **Save as type:** Select **"All Files (*.*)"**.
        *   **Encoding:** Ensure it is set to **"ANSI"**.
    *   Click **"Save"**.

    This script checks for the existence of `ccmsetup.exe` (the SCCM agent installer/uninstaller) and runs the uninstall command if found.

**Step 2: Create a Shared Folder for the Script**

The script needs to be placed in a network share that is accessible by all the computers from which you want to uninstall the SCCM agent.

1.  **Choose a Server:** Identify a server in your environment that is always available and can host this script. This could be a file server or a domain controller.

2.  **Create a New Folder:**
    *   On the chosen server, open **File Explorer**.
    *   Navigate to a suitable location where you want to create the shared folder (e.g., the root of a drive, or within an existing shares folder).
    *   Right-click in the folder pane, select **"New"** > **"Folder"**.
    *   Name the folder descriptively, for example, `SCCMUninstallScript`.

3.  **Share the Folder:**
    *   Right-click on the newly created folder (`SCCMUninstallScript`) and select **"Properties"**.
    *   Go to the **"Sharing"** tab.
    *   Click **"Advanced Sharing..."**.
    *   Check the box **"Share this folder"**.
    *   Enter a **"Share name:"**, for example, `SCCMUninstallScript$`. Adding a `$` at the end makes the share hidden from Browse but still accessible by path.
    *   Click **"Permissions"**.
    *   **Recommended Permissions:** Remove "Everyone" if it's listed. Click **"Add..."**, type `Authenticated Users`, and click **"Check Names"** then **"OK"**.
    *   Select "Authenticated Users" in the "Group or users names" list.
    *   Under "Permissions for Authenticated Users", ensure **"Read & Execute"** is allowed.  **"List folder contents"** and **"Read"** will also be automatically checked.
    *   Click **"OK"** in the "Permissions for SCCMUninstallScript$" window.
    *   Click **"OK"** in the "Advanced Sharing" window.
    *   Click **"Close"** in the "SCCMUninstallScript Properties" window.

4.  **Move the Script to the Shared Folder:**
    *   Locate the `UninstallSCCMAgent.bat` file you saved in **Step 1**.
    *   Move or copy this file into the newly created shared folder on the server (e.g., `\\YourServer\SCCMUninstallScript$`).

**Step 3: Create a Group Policy Object (GPO)**

Now, you will create a new Group Policy Object to deploy the startup script.

1.  **Open Group Policy Management:**
    *   On a Domain Controller or a computer with the Remote Server Administration Tools (RSAT) installed, press **Win + R**, type `gpmc.msc`, and press **Enter**. This opens the Group Policy Management Console.

2.  **Navigate to your Domain or OU:**
    *   In the Group Policy Management Console, expand your forest and domain.
    *   Locate the Organizational Unit (OU) that contains the computers from which you want to uninstall the SCCM agent. If you want to apply this to the entire domain, you can select your domain. **Be very careful when applying GPOs at the domain level as it affects all computers in the domain.** It's generally safer to target a specific OU.

3.  **Create a New GPO:**
    *   Right-click on your domain or the target OU.
    *   Select **"Create a GPO in this domain, and Link it here..."**.

4.  **Name the GPO:**
    *   In the "New GPO" dialog, enter a descriptive name for the GPO, such as `Uninstall SCCM Agent Startup Script`.
    *   Click **"OK"**.

**Step 4: Configure the GPO to Run the Startup Script**

Now you need to configure the GPO to run the script you placed in the shared folder every time the computers start.

1.  **Edit the GPO:**
    *   In the Group Policy Management Console, right-click the newly created GPO (`Uninstall SCCM Agent Startup Script`).
    *   Select **"Edit"**. This opens the Group Policy Management Editor.

2.  **Navigate to Startup Scripts:**
    *   In the Group Policy Management Editor, in the left pane, expand **"Computer Configuration"**.
    *   Expand **"Policies"**.
    *   Expand **"Windows Settings"**.
    *   Click on **"Scripts (Startup/Shutdown)"**.

3.  **Configure the Startup Script:**
    *   In the right pane, double-click **"Startup"**. This opens the "Startup Properties" window.
    *   Click the **"Add..."** button. This opens the "Add a Script" dialog.

4.  **Specify the Script Path:**
    *   In the "Add a Script" dialog:
        *   **Script Name:** Click **"Browse..."**.
        *   In the "Open" dialog, in the "File name" field, type the UNC path to your script in the shared folder. For example, `\\YourServer\SCCMUninstallScript$\UninstallSCCMAgent.bat`. Then press **Enter** or click **"Open"**.  *(It's important to type the UNC path directly and press Enter or Open, rather than Browse the network, to ensure the path is correctly resolved in the GPO).*
        *   **Script Parameters:** Leave this field blank.
        *   Click **"OK"** in the "Add a Script" dialog.

5.  **Close GPO Editor:**
    *   Click **"OK"** in the "Startup Properties" window.
    *   Close the Group Policy Management Editor window.

**Step 5: Link and Enforce the GPO (If not already linked)**

Ensure the GPO is linked to the correct OU or domain and, if needed, enforce it.

1.  **Verify Linking:**
    *   In the Group Policy Management Console, navigate to the OU or Domain where you created the GPO.
    *   In the right pane, under **"Linked Group Policy Objects"**, you should see your newly created GPO (`Uninstall SCCM Agent Startup Script`).

2.  **Enforce the GPO (Use with Caution):**
    *   If you need to ensure that this GPO setting is applied and not overridden by any GPOs linked at a lower level in the OU hierarchy, you can enforce it.
    *   **Caution:** Enforcing GPOs should be done carefully as it can prevent lower-level OUs from having conflicting policies. Only enforce if you are sure this is necessary.
    *   To enforce, right-click on the GPO (`Uninstall SCCM Agent Startup Script`) and select **"Enforced"**.  A little "!" icon will appear on the GPO icon indicating it is enforced. If you don't need to enforce it, skip this step.

**Step 6: Test and Verify**

It's crucial to test the GPO on a test device before rolling it out widely.

1.  **Apply Group Policy Update on Test Device:**
    *   On a test computer that is within the OU where you linked the GPO, open Command Prompt as Administrator.
    *   Run the command `gpupdate /force` and press **Enter**. This forces the device to immediately update its group policy settings.

2.  **Restart the Test Device:**
    *   Restart the test computer.

3.  **Verify Uninstallation:**
    *   After the device restarts and you log in, check if the SCCM agent has been uninstalled. You can verify this in several ways:
        *   **Control Panel:** Check if the **"Configuration Manager"** Control Panel applet is no longer present.
        *   **Services:** Open Services (`services.msc`) and check if the **"SMS Agent Host"** service is removed.
        *   **Logs:** Check the log file at `%windir%\ccmsetup\logs\CCMSetup.log`. Open this file with Notepad and look for entries indicating a successful uninstallation. You can search for "Uninstall completed successfully".

4.  **Troubleshooting (If Uninstallation Fails):**
    *   **Check Event Logs:** On the test computer, check the Application and System event logs for any errors related to Group Policy or script execution.
    *   **Verify Script Path in GPO:** Double-check the UNC path to the script in the GPO settings to ensure it is correct and accessible.
    *   **Permissions on Shared Folder:** Ensure "Authenticated Users" have "Read & Execute" permissions on the shared folder and the script file.
    *   **GPO Application:** Use `gpresult /r` or `gpresult /h <filename>.html` in Command Prompt on the test device to verify that the GPO is being applied and that the Startup script is listed under "Computer Settings\Scripts\Startup".

5.  **Proceed with Wider Deployment:** Once you have verified successful uninstallation on the test device, you can allow the GPO to apply to all devices in the targeted OU. This will happen automatically as devices restart or as Group Policy refreshes.

**Step 7: Remove the GPO (Optional, after successful uninstallation)**

After you have confirmed that the SCCM agent has been uninstalled from all desired devices, you might want to remove the GPO to prevent the script from running again on future restarts.

1.  **Unlink the GPO (Recommended if you might need to re-enable it later):**
    *   In the Group Policy Management Console, navigate to the OU or Domain where the GPO is linked.
    *   Right-click on the GPO (`Uninstall SCCM Agent Startup Script`).
    *   Select **"Link Enabled"** to uncheck it. The GPO link is now disabled, but the GPO object still exists and can be re-enabled later by checking "Link Enabled" again.

2.  **Delete the GPO (If you are sure you no longer need it):**
    *   In the Group Policy Management Console, right-click on the GPO (`Uninstall SCCM Agent Startup Script`).
    *   Select **"Delete"**.
    *   Click **"OK"** to confirm the deletion. The GPO object is now permanently removed.

