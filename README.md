# OCI Runbook: Windows VM Backup & Restore

This document provides a standardized and validated procedure to perform **backup and restore operations of Windows virtual machines on OCI** while preserving **Active Directory integrity**.

Key objective:

- Ensure **data consistency**
- Avoid **domain trust failures**
- Enable **reliable disaster recovery**


## 1. Architecture Context

- Windows VM hosted on OCI Compute
- Boot volume used for OS and system state
- VM joined to Active Directory (Domain Controller hosted on OCI or on-prem)


## 2. Key Technical Considerations

### 2.1 OCI Boot Volume Backup

- Default behavior: **Crash-consistent snapshot**
- Includes:
  - OS
  - Registry
  - Machine SID
  - AD secure channel (machine password)


### 2.2 Active Directory Behavior

- Machine password automatically rotates every **30 days**
- Backup captures password at time **T0**

/!\ Risk:
If restored later, password mismatch may occur => trust failure

### 2.3 Boot Volume backup of a running Windows instance

### 2.3.1 Crash consistency behavior

Because the snapshot is taken at block level:

- NTFS may not be fully flushed
- Windows registry changes may be incomplete
- open file writes may be interrupted
- Windows Update may be mid-operation

After restore:

In most cases system usually boots normally but you may encounter :

- automatic disk check (chkdsk)
- rollback of Windows updates
- minor file/app inconsistencies

### 2.3.2 Application-level risks

Higher risk if workloads include:

- SQL Server / Oracle / PostgreSQL
- IIS with active sessions
- transactional applications

Potential issues:

- partial transactions
- data inconsistency
- silent logical corruption (rare but possible)


## 3. Backup Procedure (Recommended)

### Step 1 - Graceful Shutdown

Ensure the VM is stopped before backup.

Effects:

- NTFS buffers flushed
- Registry committed
- Services stopped cleanly
- Disk in consistent state

Result:

- Equivalent or better than application-consistent snapshot


### Step 2 - Create Boot Volume Backup

Using OCI Console or CLI:

- Select Boot Volume
- Create Backup (Full)


### Step 3 - Restart VM

Start the instance after backup completion.


## 4. Restore Procedure

### 4.1 Pre-Validation (Optional but Recommended)

On Domain Controller:

```powershell
Get-ADComputer -Identity <VM_NAME> -Properties PasswordLastSet
```

Output:

```
DistinguishedName : CN=WIN-CLIENT,CN=Computers,DC=olygo,DC=local
DNSHostName       : WIN-CLIENT.olygo.local
Enabled           : True
Name              : WIN-CLIENT
ObjectClass       : computer
ObjectGUID        : fc0be536-7dc3-4f8b-a783-adf24287f33d
PasswordLastSet   : 3/24/2026 10:24:24 AM
SamAccountName    : WIN-CLIENT$
SID               : S-1-5-21-2122676175-4166642183-2334546819-1105
UserPrincipalName :
```

Check **'PasswordLastSet'**

| Condition | Risk |
|----------|------|
| < 30 days | Low |
| > 30 days | High |

If the backup is older than the ***'PasswordLastSet'*** then you may encounter the following error:

```
“The trust relationship between this workstation and the domain failed”
```


### 4.2 Restore Steps

#### Step 0 — Terminate Existing VM
Avoid duplication and AD conflicts


#### Step 1 — Restore Boot Volume
Create new boot volume from backup


#### Step 2 — Launch Instance

Requirements:

- Same hostname (recommended)
- Network connectivity to AD
- DNS pointing to Domain Controller


#### Step 3 — Start and Log In


#### Step 4 — Validate Domain Trust

```powershell
Test-ComputerSecureChannel -Verbose
```
Checks the status of the secure channel that the NetLogon service established

```
nltest /sc_verify:yourdomain.local
```

#### Step 5 — Repair Trust (if needed)

```powershell
Test-ComputerSecureChannel -Repair -Credential (Get-Credential)
```

Alternative:

```powershell
Reset-ComputerMachinePassword -Credential (Get-Credential)
```


#### Step 6 — Rejoin Domain (if repair fails)

```powershell
Remove-Computer -UnjoinDomainCredential domain\admin -Restart
```

```powershell
Add-Computer -DomainName domain.local -Credential domain\admin -Restart
```


## 6. VSS Considerations

- OCI does not trigger VSS automatically
- VSS must be used when VM cannot be stopped

Third-party backup tools that support VSS:

- [Veeam](https://helpcenter.veeam.com/docs/agentforwindows/userguide/overview.html?ver=13)
- [Commvault](https://documentation.commvault.com/saas/protecting_oracle_cloud_infrastructure_oci_with_commvault_cloud.html)


## 7. Critical Restrictions

### Do NOT:

- Clone domain-joined VMs (Unless you have terminated the source instance)
- Restore multiple Domain-Joined instances from same backup
- Use OCI clone for Domain Controllers

Risks:

- SID duplication
- AD corruption
- USN rollback


## 8. Troubleshooting

### 8.1 Error:
"The trust relationship between this workstation and the domain failed"

### 8.2 Resolution:

```powershell
Test-ComputerSecureChannel -Repair -Credential (Get-Credential)
```

### 8.3 Validation:

```powershell
Test-ComputerSecureChannel -Verbose
```

Expected:
True


## 9. Recommendations

- Always stop Windows instances before backup
- Always validate trust after restore
- Use AD-aware backup for critical systems


## 10. Appendix

### 10.1 Quick Recovery Script

```powershell
if (-not (Test-ComputerSecureChannel)) {
    Test-ComputerSecureChannel -Repair -Credential (Get-Credential)
}
```
