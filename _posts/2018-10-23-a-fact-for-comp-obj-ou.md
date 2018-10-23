---
title: "A Puppet fact for a the OU location of a server in Active Directory"
tags:
  - puppet
  - powershell
---

Drop the fact code wherever you place your facts (e.g. site/profile/facts.d). This will generate a fact named `activedirectory_meta` with the JSON formatted data. The fact is pretty useful if you want to scope / target your Puppet code based on the location of the server in Active Directory.

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
$directorySearcher = New-Object System.DirectoryServices.DirectorySearcher
$directorySearcher.Filter = "(&(objectCategory=Computer)(Name=$env:ComputerName))"
$searcherPath = $directorySearcher.FindOne()
$getDirectoryEntry = $searcherPath.GetDirectoryEntry()

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

Write-Output ($adobj | ConvertTo-Json)
```