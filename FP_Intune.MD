# Change Request: SCCM to Intune Bulk Migration
**CR Number**: [To be assigned]  
**Date Submitted**: 13/02/2025  
**Requested By**: [Requester Name]  
**Change Owner**: [Change Owner Name]  
**Priority**: High  
**Risk Level**: Medium  

## 1. Change Description
Bulk migration of devices from SCCM to Microsoft Intune using automated GPO deployment and Device Enrollment Manager (DEM) account.

## 2. Business Justification
- Centralise device management in Intune
- Reduce management overhead
- Enable modern workplace capabilities
- Streamline device management processes

## 3. Impact Assessment
**Services Affected**:
- Device Management
- Security Policies
- Software Distribution
- Compliance Monitoring

**Users Affected**:
- All users in targeted OUs
- IT Support Staff
- Service Desk

**Systems Affected**:
- Client Computers
- Active Directory
- SCCM Infrastructure
- Intune Environment

## 4. Detailed Implementation Plan

### Phase 1: Pre-Implementation (Duration: 1 day)
1. Documentation and Backup
   - Export current SCCM device list
   - Document current policies
   - Verify backup systems operational
   - Take system state backup of SCCM server

2. Environment Verification
   - Test network connectivity to Intune endpoints
   - Verify Azure AD Connect sync status
   - Check SentinelOne policy configurations
   - Validate GPO replication

### Phase 2: DEM Account Setup (Duration: 2 hours)
1. Create AD Account
   ```powershell
   New-ADUser -Name "SVC_IntuneEnroll" -UserPrincipalName "SVC_IntuneEnroll@domain.com" -AccountPassword $securePassword -Enabled $true -PasswordNeverExpires $true -CannotChangePassword $true
   ```

2. Configure Account Properties
   - Set description: "Intune Device Enrollment Manager Account"
   - Add to required security groups
   - Document account details securely

3. Intune Configuration
   - Add account as DEM in Intune portal
   - Verify DEM permissions
   - Test account access

### Phase 3: GPO Creation (Duration: 4 hours)
1. Create GPO
   ```powershell
   New-GPO -Name "Intune-Migration-Automation" -Comment "Automated SCCM to Intune migration"
   ```

2. Configure GPO Settings
   - Computer Configuration > Preferences > Windows Settings
   - Create Scheduled Task
   - Import PowerShell script
   - Set security filtering

3. GPO Testing
   - Run GPO modeling
   - Verify GPO replication
   - Test GPO application

### Phase 4: Pilot Implementation
1. Select Test Group
   - Identify 5-10 test devices
   - Create test OU
   - Move test devices to OU

2. Deploy to Test Group
   - Link GPO to test OU
   - Force GPO update
   - Monitor progress

3. Validation
   - Check device enrollment
   - Verify policy application
   - Test software deployment
   - Document results

### Phase 5: Full Implementation 
1. Preparation
   - Schedule out-of-hours window
   - Notify service desk
   - Prepare monitoring tools

2. Deployment Steps
   a. Stage 1 - First Wave (20% of devices)
      - Link GPO to target OU
      - Monitor for 4 hours
      - Validate success
   
   b. Stage 2 - Second Wave (30% of devices)
      - Continue monitoring
      - Validate enrollments
   
   c. Stage 3 - Final Wave (Remaining devices)
      - Complete deployment
      - Full validation

3. Post-Implementation
   - Verify all devices enrolled
   - Check policy application
   - Monitor help desk tickets

## 5. Testing Plan
### Pre-Implementation Testing
- GPO application test
- Script execution test
- Network connectivity test
- DEM account access test

### During Implementation Testing
- Device enrollment verification
- Policy application check
- Application deployment test
- User access validation

### Post-Implementation Testing
- Full functionality test
- Security policy verification
- Application availability check
- Performance monitoring

## 6. Risk Assessment and Mitigation

### High Risks
1. **Device Management Loss**
   - Mitigation: Phased deployment
   - Rollback plan in place
   - Continuous monitoring

2. **Script Execution Failure**
   - Mitigation: Pre-tested scripts
   - SentinelOne exceptions
   - Error logging

### Medium Risks
1. **Network Issues**
   - Mitigation: Bandwidth monitoring
   - Multiple enrollment attempts
   - Failure logging

2. **User Impact**
   - Mitigation: Out-of-hours deployment
   - Clear communication
   - Service desk preparation

## 7. Rollback Plan
### Trigger Conditions
- More than 10% enrollment failures
- Critical system accessibility issues
- Security policy failures

### Rollback Steps
1. Disable Migration GPO
   ```powershell
   Disable-GPO -Name "Intune-Migration-Automation"
   ```

2. Deploy Rollback GPO
   - Enable SCCM reinstallation
   - Remove Intune enrollment
   - Restore original settings

3. Verify Systems
   - Check SCCM connectivity
   - Validate policy application
   - Test application access

## 8. Success Criteria
### Required Outcomes
- 100% devices visible in Intune
- SCCM agent removed from all devices
- Policies applying correctly
- No user disruption
- Applications functional

### Validation Methods
- Intune portal device count
- Policy report verification
- Application deployment test
- User access validation

## 9. Resource Requirements
### Personnel
- Change Owner
- Technical Implementation Team
- Service Desk Support
- Security Team Representative

### Tools
- Azure AD/Intune Access
- SCCM Admin Access
- AD Admin Rights
- Monitoring Tools

## 10. Communication Plan
### Pre-Change
- Notify all stakeholders
- Brief service desk
- Update change calendar
- Prepare user communications

### During Change
- Regular status updates
- Issue reporting channel
- Emergency contact list
- Progress monitoring

### Post-Change
- Completion notification
- Success metrics report
- Lessons learned document
- Future recommendations

## 11. Change Schedule
**Start Date**: [Date]  
**End Date**: [Date]  
**Implementation Window**: [Time]  
**Duration**: [Hours/Days]  
