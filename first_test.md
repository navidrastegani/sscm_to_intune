# Change Management Document - SCCM to Intune Migration

## Document Control Information

### Document Details

* **Document Title:** SCCM to Intune Migration Change Request
* **Document ID:** CHG-2025-0212
* **Version:** 1.0
* **Status:** Draft
* **Classification:** Internal
* **Author:** [Author Name]
* **Department:** IT Infrastructure
* **Created Date:** February 12, 2025
* **Last Modified:** February 12, 2025

### Version History

| Version | Date | Author | Changes |
|---------|------|--------|----------|
| 0.1 | 2025-02-12 | [Name] | Initial draft |
| 1.0 | 2025-02-12 | [Name] | Final version |

## 1. Executive Summary

This change request outlines the migration of endpoint device management from System Center Configuration Manager (SCCM) to Microsoft Intune. The migration will modernize our device management capabilities while reducing infrastructure complexity and enhancing security controls.

### Change Overview

* **Type:** Standard Change
* **Priority:** Medium
* **Risk Level:** Medium
* **Impact:** Medium

## 2. Change Description

### 2.1 Purpose

To transition from on-premises SCCM to cloud-based Intune for endpoint management, enabling modern management capabilities while reducing infrastructure overhead.

### 2.2 Scope

#### In Scope

* Windows 10 devices under SCCM management
* Security policy migration
* SCCM client removal
* Intune enrollment automation
* User communication and training

#### Out of Scope

* Non-Windows devices
* Application packaging
* Network infrastructure changes
* Data migration
* Hardware upgrades

### 2.3 Requirements

#### Technical Requirements

* Windows 10 version 1803 or higher
* Valid Intune licenses
* Azure AD Premium licenses
* Network connectivity to Microsoft endpoints
* Local administrative rights

#### Business Requirements

* Minimal user disruption
* Maintained security compliance
* Continuous application availability
* Clear communication process

## 3. Business Justification

### 3.1 Current State Challenges

#### Infrastructure Complexity

* Maintenance-heavy SCCM infrastructure
* Multiple management points
* Complex policy management
* High operational overhead

#### Operational Challenges

* Limited remote management
* Complex troubleshooting
* Resource-intensive updates
* Limited modern features

### 3.2 Benefits

#### Financial Benefits

* Reduced infrastructure costs
* Lower maintenance overhead
* Optimized licensing
* Reduced operational costs

#### Technical Benefits

* Modern management capabilities
* Enhanced security features
* Improved remote management
* Simplified administration

#### Strategic Benefits

* Cloud alignment
* Modern workplace enablement
* Enhanced mobility support
* Improved user experience

## 4. Implementation Plan

### 4.1 Pre-Implementation

#### Environment Preparation

1. License Verification
   * Confirm Intune licensing
   * Verify Azure AD Premium
   * Check Windows 10 versions

2. Network Assessment
   * Validate connectivity
   * Check bandwidth
   * Verify endpoints

3. Security Validation
   * Review security policies
   * Check compliance requirements
   * Verify tool compatibility

### 4.2 Technical Implementation Steps

#### Step 1: SCCM Removal

```powershell
# Stop SCCM Service
Stop-Service -Name "CCMExec" -Force

# Uninstall SCCM Client
c:\windows\ccmsetup\ccmsetup.exe /uninstall

# Verify Removal
Get-Service -Name "CCMExec" -ErrorAction SilentlyContinue
```

#### Step 2: Intune Enrollment

```powershell
# Set Registry Keys
$registryPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\CurrentVersion\MDM"
New-Item -Path $registryPath -Force
Set-ItemProperty -Path $registryPath -Name "AutoEnrollMDM" -Value 1 -Type DWord
Set-ItemProperty -Path $registryPath -Name "UseAADCredentialType" -Value 1 -Type DWord

# Remove from AAD
dsregcmd.exe /leave /debug

# Trigger Enrollment
C:\Windows\system32\deviceenroller.exe /c /AutoEnrollMDM

# Wait and Restart
Start-Sleep -Seconds 120
Restart-Computer -Force
```

## 5. Test Plan

### 5.1 Test Cases

| ID | Description | Expected Result | Pass/Fail |
|----|-------------|----------------|-----------|
| TC1 | SCCM Removal | Clean uninstall | |
| TC2 | Registry Config | Keys set correctly | |
| TC3 | Intune Enrollment | Successful enrollment | |
| TC4 | Policy Application | All policies applied | |
| TC5 | Application Access | Apps functional | |

### 5.2 Validation Steps

#### Technical Validation

1. SCCM Status
   * Check services
   * Verify removal
   * Check registry

2. Intune Status
   * Check enrollment
   * Verify policies
   * Test compliance

3. Application Status
   * Launch applications
   * Verify functionality
   * Check updates

#### Business Validation

1. User Experience
   * Login process
   * Application access
   * Performance impact
   * Remote access

2. Support Impact
   * Ticket volume
   * Resolution time
   * User feedback
   * System performance

## 6. Rollback Plan

### 6.1 Rollback Triggers

* Enrollment failure rate exceeds 5%
* Critical application failures
* Security compliance issues
* Unacceptable performance impact

### 6.2 Rollback Procedure

#### Step 1: Initiate Rollback

1. Stop deployment
2. Notify stakeholders
3. Activate support team
4. Document issues

#### Step 2: Technical Rollback

```powershell
# Remove from Intune
.\Remove-IntuneEnrollment.ps1

# Reinstall SCCM
\\sccm-server\client\ccmsetup.exe

# Verify SCCM Client
Get-Service -Name "CCMExec"
```

#### Step 3: Validation

1. Verify SCCM status
2. Check policy application
3. Test applications
4. Confirm security compliance

## 7. Success Criteria

### 7.1 Technical Success

* 95% enrollment success rate
* 100% policy application
* Zero security compliance issues
* Performance within baseline

### 7.2 Business Success

* No increase in support tickets
* User satisfaction above 85%
* All applications functional
* No data loss incidents

## 8. Approval

### Pre-Implementation Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Change Manager | | | |
| Technical Lead | | | |
| Security Officer | | | |
| Business Owner | | | |

### Post-Implementation Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| Change Manager | | | |
| Technical Lead | | | |
| Security Officer | | | |
| Business Owner | | | |

## 9. Reference Documents

1. Technical Documentation
2. Security Baselines
3. Support Procedures
4. Training Materials

---

*End of Document*
