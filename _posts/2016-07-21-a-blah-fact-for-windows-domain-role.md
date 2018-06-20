---
title: "A Puppet fact for the Windows Domain Role"
tags:
  - puppet
  - powershell
---

This is pretty simple. Drop this `domainrole.ps1` wherever you place your facts.
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