# SCCM to Intune Migration - Implementation Plan

## Table of Contents
- [Project Overview](#project-overview)
- [Prerequisites](#prerequisites)
- [Implementation Phases](#implementation-phases)
- [Testing Procedures](#testing-procedures)
- [Rollback Plan](#rollback-plan)
- [Success Criteria](#success-criteria)

## Project Overview

### Objective
Automated migration of devices from SCCM to Intune using GPO and DEM account deployment.

### Scope
- Target Environment: [Specify OUs]
- Number of Devices: [Specify Count]
- Timeline: [Specify Duration]

### Success Metrics
- 100% devices enrolled in Intune
- SCCM agent successfully removed
- All policies applied correctly
- No user disruption

## Prerequisites

### Access Requirements
- Global Admin access to Microsoft 365
- Domain Admin rights
- SCCM Admin access
- Intune Admin access

### Technical Requirements
- Azure AD Connect configured
- Network connectivity to Intune endpoints
- GPO replication working
- SentinelOne exceptions (if required)

## Implementation Phases

### Phase 1: Pre-Implementation Setup

#### 1.1 Environment Preparation
```powershell
# Verify Azure AD Connect sync
Start-ADSyncSyncCycle -PolicyType Delta

# Test network connectivity
Test-NetConnection -ComputerName enterpriseregistration.windows.net -Port 443
Test-NetConnection -ComputerName login.microsoftonline.com -Port 443
```

#### 1.2 Documentation
- Export current SCCM device list
- Document existing policies
- Create system state backup
- Record current compliance state

### Phase 2: DEM Account Configuration

#### 2.1 Create Service Account
```powershell
# Create DEM account
New-ADUser -Name "SVC_IntuneEnroll" `
    -UserPrincipalName "SVC_IntuneEnroll@domain.com" `
    -AccountPassword $securePassword `
    -Enabled $true `
    -PasswordNeverExpires $true `
    -CannotChangePassword $true

# Add to required groups
Add-ADGroupMember -Identity "Domain Admins" -Members "SVC_IntuneEnroll"
```

#### 2.2 Intune Configuration
1. Navigate to: https://intune.microsoft.com/#view/Microsoft_Intune_DeviceSettings/DevicesMenu/~/enrollment
2. Add DEM account:
   - Devices > Enroll devices > Device enrollment managers
   - Add user > Enter DEM account
   - Set device limit

### Phase 3: GPO Creation

#### 3.1 Create Base GPO
```powershell
# Create new GPO
New-GPO -Name "Intune-Migration-Automation" -Comment "SCCM to Intune migration automation"

# Link to target OU
New-GPLink -Name "Intune-Migration-Automation" -Target "OU=Computers,DC=domain,DC=com"
```

#### 3.2 Configure Registry Settings
Settings to be deployed:
```registry
Path: HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM
Values:
- AutoEnrollMDM = 1 (DWORD)
- UseAADCredentialType = 1 (DWORD)
```

#### 3.3 Create Deployment Script
Save as: `Intune-Enrollment.ps1`
```powershell
# Log file setup
$LogPath = "C:\Windows\Temp\IntuneEnrollment.log"
$ErrorActionPreference = "Stop"

function Write-Log {
    param($Message)
    $LogMessage = "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss'): $Message"
    Add-Content -Path $LogPath -Value $LogMessage
    Write-Output $LogMessage
}

try {
    # Step 1: Uninstall SCCM Agent
    Write-Log "Starting SCCM uninstallation"
    if (Test-Path "c:\windows\ccmsetup\ccmsetup.exe") {
        Start-Process -FilePath "c:\windows\ccmsetup\ccmsetup.exe" -ArgumentList "/uninstall" -Wait
        Write-Log "SCCM uninstallation completed"
    }

    # Step 2: Leave AAD
    Write-Log "Leaving AAD"
    Start-Process -FilePath "dsregcmd.exe" -ArgumentList "/leave /debug" -Wait
    Start-Sleep -Seconds 30

    # Step 3: Configure Registry
    Write-Log "Setting MDM registry keys"
    $mdmPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM"
    if (!(Test-Path $mdmPath)) {
        New-Item -Path $mdmPath -Force
    }
    Set-ItemProperty -Path $mdmPath -Name "AutoEnrollMDM" -Value 1 -Type DWord -Force
    Set-ItemProperty -Path $mdmPath -Name "UseAADCredentialType" -Value 1 -Type DWord -Force

    # Step 4: Trigger Enrollment
    Write-Log "Starting auto-enrollment"
    Start-Process -FilePath "C:\Windows\system32\deviceenroller.exe" -ArgumentList "/c /AutoEnrollMDM" -Wait

    # Step 5: Schedule Reboot
    Write-Log "Scheduling reboot"
    Start-Process shutdown -ArgumentList "/r /t 120 /c `"System will restart for Intune enrollment`""

} catch {
    Write-Log "Error occurred: $_"
    exit 1
}
```

### Phase 4: Testing

#### 4.1 Pilot Deployment
1. Select test group (5-10 devices)
2. Create test OU
3. Move test devices
4. Apply GPO
5. Monitor for 24 hours

#### 4.2 Validation Tests
```powershell
# Check enrollment status
dsregcmd /status

# Verify MDM enrollment
Get-Item -Path "HKLM:\SOFTWARE\Microsoft\Enrollments\*" | Get-ItemProperty

# Check logs
Get-WinEvent -LogName "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin"
```

### Phase 5: Full Deployment

#### 5.1 Deployment Waves
1. Wave 1 (20% devices)
   - Monitor for 4 hours
   - Validate success
2. Wave 2 (30% devices)
   - Deploy after Wave 1 success
   - Monitor for issues
3. Wave 3 (Remaining devices)
   - Complete deployment
   - Full validation

#### 5.2 Post-Deployment Checks
- Verify device count in Intune
- Check policy application
- Test software deployment
- Validate security compliance

## Rollback Plan

### Trigger Conditions
- More than 10% enrollment failures
- Critical system accessibility issues
- Security policy failures

### Rollback Steps

#### 1. Stop Deployment
```powershell
# Disable GPO
Disable-GPO -Name "Intune-Migration-Automation"
```

#### 2. Restore SCCM
```powershell
# Reinstall SCCM agent
Start-Process -FilePath "\\<SCCM-Server>\Client\ccmsetup.exe" -Wait
```

#### 3. Remove Intune Enrollment
```powershell
# Leave AAD/Intune
dsregcmd.exe /leave
```

## Success Criteria

### Technical Validation
- All devices visible in Intune portal
- Policies applying correctly
- Applications deploying successfully
- Security compliance maintained

### Business Validation
- Users can access all required resources
- No disruption to business operations
- Help desk not overwhelmed with issues
- Security standards maintained

## Appendix

### Useful Commands

#### Enrollment Status
```powershell
# Check AAD join status
dsregcmd /status

# Get MDM enrollment status
Get-MDMEnrollmentStatus

# Check Intune sync
Start-Process -FilePath "$env:ProgramFiles\Microsoft Intune Management Extension\AgentExecutor.exe" -ArgumentList "syncapp"
```

#### Log Locations
- Intune Enrollment: %ProgramData%\Microsoft\IntuneManagementExtension\Logs
- SCCM Uninstall: C:\Windows\ccmsetup\Logs
- GPO: %SystemRoot%\Debug\UserMode\gpresult.html

### Common Issues and Resolution

#### Failed Enrollment
1. Check network connectivity
2. Verify service account permissions
3. Review event logs
4. Check SentinelOne blocking

#### Policy Application Issues
1. Force policy sync
2. Check device compliance
3. Review assignment filters
4. Verify scope tags
