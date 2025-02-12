# SCCM to Intune Migration Implementation Plan

## Project Overview

### Objective
Automated migration of devices from SCCM to Microsoft Intune using GPO deployment and Device Enrollment Manager (DEM) account.

### Scope
- Target environment: [Specify Target OUs]
- Estimated device count: [Number]
- Deployment timeline: [Timeline]

## Prerequisites

### Required Access
- Microsoft 365 Admin Portal access
- Intune Admin access
- Domain Admin rights
- SCCM Admin access

### Technical Requirements
- Azure AD Connect configured and syncing
- Devices are Azure AD joined
- Network connectivity to Intune endpoints
- GPO replication functional
- SentinelOne exceptions (if required)

## Implementation Steps

### Phase 1: DEM Account Setup

1. Create Service Account
```powershell
# Create new AD account
New-ADUser -Name "SVC_IntuneEnroll" `
    -UserPrincipalName "SVC_IntuneEnroll@domain.com" `
    -AccountPassword $securePassword `
    -Enabled $true `
    -PasswordNeverExpires $true `
    -CannotChangePassword $true

# Add to required groups
Add-ADGroupMember -Identity "Domain Admins" -Members "SVC_IntuneEnroll"
```

2. Configure Intune DEM
   - Navigate to: https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/DevicesMenu/~/enrollment
   - Add account as Device Enrollment Manager
   - Set device limit above target count

### Phase 2: GPO Configuration

1. Create Migration GPO
```powershell
# Create GPO
New-GPO -Name "Intune-Migration-Automation" -Comment "SCCM to Intune migration"
```

2. Configure Registry Settings
```registry
Path: HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM
Values:
- AutoEnrollMDM = 1 (DWORD)
- UseAADCredentialType = 1 (DWORD)
```

3. Create Migration Script
```powershell
# Save as: Intune-Migration.ps1
$LogPath = "C:\Windows\Temp\IntuneEnrollment.log"
$ErrorActionPreference = "Stop"

function Write-Log {
    param($Message)
    $LogMessage = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss'): $Message"
    Add-Content -Path $LogPath -Value $LogMessage
    Write-Output $LogMessage
}

try {
    # Verify AAD Join Status
    Write-Log "Checking AAD Join Status"
    $aadStatus = dsregcmd /status
    if ($aadStatus -notmatch "AzureAdJoined : YES") {
        Write-Log "Device not Azure AD joined - please check AAD join status"
        exit 1
    }

    # Uninstall SCCM Agent
    Write-Log "Starting SCCM uninstallation"
    if (Test-Path "c:\windows\ccmsetup\ccmsetup.exe") {
        Start-Process -FilePath "c:\windows\ccmsetup\ccmsetup.exe" -ArgumentList "/uninstall" -Wait
        Write-Log "SCCM uninstallation completed"
    }

    # Configure MDM Registry
    Write-Log "Setting MDM registry keys"
    $mdmPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM"
    if (!(Test-Path $mdmPath)) {
        New-Item -Path $mdmPath -Force
    }
    Set-ItemProperty -Path $mdmPath -Name "AutoEnrollMDM" -Value 1 -Type DWord -Force
    Set-ItemProperty -Path $mdmPath -Name "UseAADCredentialType" -Value 1 -Type DWord -Force

    # Trigger Enrollment
    Write-Log "Starting MDM enrollment"
    Start-Process -FilePath "C:\Windows\system32\deviceenroller.exe" -ArgumentList "/c /AutoEnrollMDM" -Wait

    # Schedule Reboot
    Write-Log "Scheduling reboot"
    Start-Process shutdown -ArgumentList "/r /t 120 /c `"System will restart to complete Intune enrollment`""

} catch {
    Write-Log "Error occurred: $_"
    exit 1
}
```

4. Configure GPO Settings
   - Computer Configuration > Preferences > Windows Settings > Registry
   - Add MDM registry keys
   - Computer Configuration > Preferences > Control Panel Settings > Scheduled Tasks
   - Create task to run migration script as SYSTEM

### Phase 3: Testing

1. Pre-Deployment Checks
   - Verify network connectivity
   ```powershell
   Test-NetConnection -ComputerName enterpriseregistration.windows.net -Port 443
   Test-NetConnection -ComputerName login.microsoftonline.com -Port 443
   ```
   - Check Azure AD sync status
   - Validate DEM account permissions

2. Pilot Deployment
   - Select 5-10 test devices
   - Create test OU
   - Link GPO to test OU
   - Monitor for 24 hours

3. Validation Tests
```powershell
# Check enrollment status
dsregcmd /status

# Verify MDM enrollment
Get-Item -Path "HKLM:\SOFTWARE\Microsoft\Enrollments\*"

# Review logs
Get-WinEvent -LogName "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin"
```

### Phase 4: Full Deployment

1. Deployment Waves
   - Wave 1: 20% of devices
   - Wave 2: 30% of devices
   - Wave 3: Remaining devices

2. Post-Migration Checks
   - Device presence in Intune portal
   - Policy application status
   - Application deployment verification
   - Security compliance check

## Rollback Plan

### Trigger Conditions
- Enrollment failure rate exceeds 10%
- Critical system access issues
- Security compliance failures

### Rollback Steps

1. Stop Deployment
```powershell
# Disable migration GPO
Disable-GPO -Name "Intune-Migration-Automation"
```

2. Restore SCCM
```powershell
# Reinstall SCCM client
Start-Process -FilePath "\\<SCCM-Server>\Client\ccmsetup.exe" -Wait
```

## Monitoring and Support

### Key Monitoring Points
- Intune device enrollment status
- Policy application success
- Application deployment status
- Help desk ticket volume

### Log Locations
- Intune Management Extension: %ProgramData%\Microsoft\IntuneManagementExtension\Logs
- SCCM Uninstall: C:\Windows\ccmsetup\Logs
- GPO Application: %SystemRoot%\Debug\UserMode\gpresult.html

### Troubleshooting Commands
```powershell
# Check MDM sync status
Start-Process -FilePath "$env:ProgramFiles\Microsoft Intune Management Extension\AgentExecutor.exe" -ArgumentList "syncapp"

# Get enrollment status
dsregcmd /status
Get-Item -Path "HKLM:\SOFTWARE\Microsoft\Enrollments\*"

# Force policy sync
gpupdate /force
```

## Success Criteria

### Technical Requirements
- All devices visible in Intune
- SCCM agent removed
- Policies applying correctly
- Applications deploying successfully

### Business Requirements
- Minimal user disruption
- All systems accessible
- Security compliance maintained
- Business applications functional


