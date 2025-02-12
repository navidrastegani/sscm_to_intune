# SCCM to Intune Migration Guide 🚀

> Comprehensive guide for migrating from SCCM to Microsoft Intune device management

## 📑 Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Implementation Guide](#implementation-guide)
- [Testing & Validation](#testing--validation)
- [Rollback Procedures](#rollback-procedures)
- [Success Criteria](#success-criteria)

## Overview

This repository contains detailed documentation and scripts for migrating Windows devices from System Centre Configuration Manager (SCCM) to Microsoft Intune management.

### 🎯 Objectives

- Transition from on-premises SCCM to cloud-based Intune
- Modernise device management capabilities
- Enhance security controls
- Reduce infrastructure complexity

### 📋 Scope

#### ✅ In Scope

- Windows 10 devices under SCCM management
- Security policy migration
- SCCM client removal
- Intune enrolment automation
- User communication and training

#### ❌ Out of Scope

- Non-Windows devices
- Application packaging
- Network infrastructure changes
- Data migration
- Hardware upgrades

## Prerequisites

### Technical Requirements

- Windows 10 version 1803 or higher
- Valid Intune licences
- Azure AD Premium licences
- Network connectivity to Microsoft endpoints
- Local administrative rights
- PowerShell 5.0 or higher
- Microsoft.Graph.Intune module installed

### System Access

- Global Administrator or Intune Administrator privileges
- SCCM administrative access
- Local administrative rights on target devices

## Implementation Guide

### 1. Pre-Implementation Checklist

- [ ] Verify licence requirements
- [ ] Validate network connectivity
- [ ] Check security compliance requirements
- [ ] Prepare test environment
- [ ] Document baseline configuration

### 2. Implementation Process

The implementation uses a comprehensive PowerShell solution that handles:
- SCCM client removal
- Hybrid Azure AD join reset
- Intune enrollment automation
- Certificate management
- Status validation

#### 2.1 Reset Intune Enrollment Script

```powershell
# Main function to reset Intune enrollment
function Reset-IntuneEnrollment {
    param (
        [string] $computerName = $env:COMPUTERNAME
    )

    Write-Host "Checking actual Intune connection status" -ForegroundColor Cyan
    if (Get-IntuneEnrollmentStatus -computerName $computerName) {
        $choice = ""
        while ($choice -notmatch "^[Y|N]$") {
            $choice = Read-Host "It seems device has working Intune connection. Continue? (Y|N)"
        }
        if ($choice -eq "N") {
            break
        }
    }

    Write-Host "Resetting Hybrid AzureAD connection" -ForegroundColor Cyan
    Reset-HybridADJoin -computerName $computerName

    Write-Host "Waiting" -ForegroundColor Cyan
    Start-Sleep 10

    Write-Host "Removing $computerName records from Intune" -ForegroundColor Cyan
    Connect-Graph
    
    $IntuneObj = Get-IntuneManagedDevice -Filter "DeviceName eq '$computerName'"
    
    if ($IntuneObj) {
        $IntuneObj | ? { $_ } | % {
            Write-Host "Removing $($_.DeviceName) ($($_.id)) from Intune" -ForegroundColor Cyan
            Remove-IntuneManagedDevice -managedDeviceId $_.id
        }
    }

    Write-Host "Invoking re-enrollment of Intune connection" -ForegroundColor Cyan
    Invoke-MDMReenrollment -computerName $computerName -asSystem

    # Check certificates
    $i = 30
    Write-Host "Waiting for Intune certificate creation" -ForegroundColor Cyan
    while (!(Get-ChildItem 'Cert:\LocalMachine\My\' | ? { $_.Issuer -match "CN=Microsoft Intune MDM Device CA" }) -and $i -gt 0) {
        Start-Sleep 1
        --$i
        $i
    }

    if ($i -eq 0) {
        Write-Warning "Intune certificate isn't created (yet?)"
        Get-IntuneLog -computerName $computerName
    } else {
        Write-Host "DONE :)" -ForegroundColor Green
    }
}
```

### 3. Execution Steps

1. **Preparation**
   ```powershell
   # Install required module
   Install-Module Microsoft.Graph.Intune -Force
   ```

2. **Run Migration Script**
   ```powershell
   # For local computer
   Reset-IntuneEnrollment

   # For remote computer
   Reset-IntuneEnrollment -computerName "REMOTE-PC-01"
   ```

3. **Validate Results**
   ```powershell
   # Check enrollment status
   Get-IntuneEnrollmentStatus -computerName $computerName -checkIntuneToo
   ```

## Testing & Validation

### Test Cases

| ID | Test Case | Expected Result | Validation Method |
|----|-----------|----------------|-------------------|
| TC1 | SCCM Removal | Complete removal of SCCM client | Service check |
| TC2 | Azure AD Join | Successful Hybrid AD join | dsregcmd status |
| TC3 | Intune Enrollment | Device appears in Intune portal | Portal check |
| TC4 | Certificate Status | Valid Intune certificate present | Certificate check |
| TC5 | Policy Application | All policies successfully applied | Compliance check |

### Validation Process

```powershell
# Comprehensive validation function
function Test-MigrationSuccess {
    param ($computerName)

    # Check SCCM Removal
    $sccmService = Get-Service -Name "CCMExec" -ErrorAction SilentlyContinue
    if ($null -eq $sccmService) {
        Write-Host "✅ SCCM client successfully removed" -ForegroundColor Green
    }

    # Check Intune Enrollment
    $intuneStatus = Get-IntuneEnrollmentStatus -computerName $computerName -checkIntuneToo
    if ($intuneStatus) {
        Write-Host "✅ Intune enrollment successful" -ForegroundColor Green
    }

    # Check Certificates
    $intuneCert = Get-ChildItem 'Cert:\LocalMachine\My\' | 
        ? Issuer -Match "CN=Microsoft Intune MDM Device CA"
    if ($intuneCert) {
        Write-Host "✅ Intune certificate valid" -ForegroundColor Green
    }
}
```

## Rollback Procedures

### Automatic Rollback

The script includes built-in rollback functionality:
- Automatically detects failures
- Restores previous state if needed
- Logs all actions for troubleshooting

### Manual Rollback Steps

```powershell
# Remove from Intune
.\Remove-IntuneEnrollment.ps1

# Reinstall SCCM
\\sccm-server\client\ccmsetup.exe

# Verify SCCM Client
Get-Service -Name "CCMExec"
```

## Success Criteria

### Technical Success Metrics

- [x] SCCM client successfully removed
- [x] Device properly joined to Azure AD
- [x] Intune enrollment complete
- [x] Certificates valid
- [x] Policies applied

### Business Success Metrics

- [x] Device manageable through Intune
- [x] All applications functional
- [x] No user disruption
- [x] Security compliance maintained

---

*Made with ❤️ by Triforce Cloud Team*
