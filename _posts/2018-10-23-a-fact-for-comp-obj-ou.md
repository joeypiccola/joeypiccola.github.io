---
title: "A Puppet fact for the OU location of a server in Active Directory"
tags:
  - puppet
  - powershell
---

Drop the fact code wherever you place your facts (e.g. site/profile/facts.d). This will generate a fact named `activedirectory_meta` with the following JSON formatted data. The fact is pretty useful if you want to scope / target your Puppet code based on the location of the node in Active Directory. Credit to the [OneScript Team](https://social.technet.microsoft.com/profile/onescript%20team/) for some of the `System.DirectoryServices.DirectorySearcher` logic.

```json
{
  "dn" : "CN=DEN3-NODE-3,OU=puppet-nodes,OU=LabStuff,OU=Servers,DC=ad,DC=piccola,DC=us",
  "ou" : "OU=puppet-nodes,OU=LabStuff,OU=Servers,DC=ad,DC=piccola,DC=us",
  "whenChanged" : "10/22/2018 5:28:37 AM",
  "whenCreated" : "8/30/2018 5:17:09 PM"
}
```


The fact code.
```powershell
# get the domain role of the machine
$DomainRole = (Get-WmiObject -Class Win32_ComputerSystem -Property DomainRole).DomainRole
# if the machine is not a Standalone Workstation or a Standalone Server then attempt to query ad
if ($DomainRole -notmatch '^(0|2)') {
    # query ad
    $directorySearcher = New-Object System.DirectoryServices.DirectorySearcher
    $directorySearcher.Filter = "(&(objectCategory=Computer)(Name=$env:ComputerName))"
    $searcherPath = $directorySearcher.FindOne()
    $getDirectoryEntry = $searcherPath.GetDirectoryEntry()

    # make the results pretty
    $dn = $getDirectoryEntry.distinguishedName
    $compobj = [PSCustomObject]@{
        dn          = $getDirectoryEntry.distinguishedName.ToString()
        ou          = $dn.substring(($dn.split(',')[0].length + 1), ($dn.Length - ($dn.split(',')[0].length + 1)))
        whenCreated = $getDirectoryEntry.whenCreated.ToString()
        whenChanged = $getDirectoryEntry.whenChanged.ToString()
    }
    $adobj = [PSCustomObject]@{
        activedirectory_meta = $compobj
    }
    # write it out
    Write-Output ($adobj | ConvertTo-Json)
}
```