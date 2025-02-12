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

### 2. SCCM Client Removal

Use the following PowerShell script to remove the SCCM client:

```powershell
# Stop SCCM Service
Stop-Service -Name "CCMExec" -Force

# Uninstall SCCM Client
c:\windows\ccmsetup\ccmsetup.exe /uninstall

# Verify Removal
Get-Service -Name "CCMExec" -ErrorAction SilentlyContinue
```

### 3. Intune Enrolment

Execute this script to configure and initiate Intune enrolment:

```powershell
# Set Registry Keys
$registryPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM"
New-Item -Path $registryPath -Force
Set-ItemProperty -Path $registryPath -Name "AutoEnrollMDM" -Value 1 -Type DWord
Set-ItemProperty -Path $registryPath -Name "UseAADCredentialType" -Value 1 -Type DWord

# Remove from AAD
dsregcmd.exe /leave /debug

# Trigger Enrolment
C:\Windows\system32\deviceenroller.exe /c /AutoEnrollMDM

# Wait and Restart
Start-Sleep -Seconds 120
Restart-Computer -Force
```

## Testing & Validation

### Test Cases

| ID | Test Case | Expected Result | Validation Method |
|----|-----------|----------------|-------------------|
| TC1 | SCCM Removal | Complete removal of SCCM client | Service check |
| TC2 | Registry Config | MDM keys correctly set | Registry verification |
| TC3 | Intune Enrolment | Device appears in Intune portal | Portal check |
| TC4 | Policy Application | All policies successfully applied | Compliance check |
| TC5 | App Access | Applications functioning normally | Functionality test |

### Validation Scripts

#### 1. Check SCCM Status

```powershell
# Verify SCCM Removal
$sccmService = Get-Service -Name "CCMExec" -ErrorAction SilentlyContinue
if ($null -eq $sccmService) {
    Write-Host "SCCM client successfully removed"
} else {
    Write-Host "SCCM client still present"
}
```

#### 2. Verify Intune Enrolment

```powershell
# Check MDM Enrolment Status
dsregcmd /status | Select-String -Pattern "MDM.*"
```

## Rollback Procedures

### Rollback Triggers

- Enrolment failure rate exceeds 5%
- Critical application failures
- Security compliance issues
- Unacceptable performance impact

### Rollback Script

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

- [x] 95% enrolment success rate
- [x] 100% policy application
- [x] Zero security compliance issues
- [x] Performance within baseline

### Business Success Metrics

- [x] No increase in support tickets
- [x] User satisfaction above 85%
- [x] All applications functional
- [x] No data loss incidents

---

*Made with ❤️ by Triforce Cloud Team*
