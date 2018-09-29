---
title: "Generating a certificate request with SAN via PowerShell"
tags:
  - pki
  - powershell
---

This is pretty simple. Drop this `domainrole.ps1` wherever you place your facts.

```powershell
$name = "ws01.ad.piccola.us"
$san  = "mysite.piccola.us"
$csrPath = Join-Path -Path 'c:\windows\temp' -ChildPath "$name.csr"
$infPath = Join-Path -Path 'c:\windows\temp' -ChildPath "$name.inf"

$inf =
@"
[Version]
Signature = '$Windows NT$'

[NewRequest]
Subject = "CN=$name, O=Piccola, L=Denver, S=Colorado, C=US"
KeySpec = 1
KeyLength = 2048
Exportable = TRUE
MachineKeySet = TRUE
SMIME = False
PrivateKeyArchive = FALSE
UserProtected = FALSE
UseExistingKeySet = FALSE
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
HashAlgorithm = SHA256
ProviderType = 12
RequestType = PKCS10
KeyUsage = 0xa0

[EnhancedKeyUsageExtension]
OID=1.3.6.1.5.5.7.3.1

[Extensions]
2.5.29.17 = "{text}"
_continue_ = "dns=$name&"
_continue_ = "dns=$san&"
"@

$inf | out-file -filepath $infPath -force
certreq -new $infPath $csrPath
```