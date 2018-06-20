---
title: "a puppet fact for windows domain role"
categories:
  - puppet
tags:
  - powershell
  - script
  - puppet
---

### the fact
below is a snippit of PowerShell that could be dropped in `domainrole.ps1`.


```powershell
switch ((Get-WmiObject -Class Win32_ComputerSystem -Property DomainRole).DomainRole) {
    '0' {
        Write-Output 'domainrole=Standalone Workstation'
    }
    '1' {
        Write-Output 'domainrole=Member Workstation'
    }
    '2' {
        Write-Output 'domainrole=Standalone Server'
    }
    '3' {
        Write-Output 'domainrole=Member Server'
    }
    '4' {
        Write-Output 'domainrole=Backup Domain Controller'
    }
    '5' {
        Write-Output 'domainrole=Primary Domain Controller'
    }
}
```