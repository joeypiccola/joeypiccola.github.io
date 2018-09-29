---
title: "Adding a certificate to the Puppet Enterprise console."
tags:
  - puppet
  - openssl
  - pki
---

The following discusses how to add a certificate to the Puppet Enterprise Console.

1. generate a csr `puppet.piccola.us.csr` and submit it to a CA. in this example a cert request was generated with PowerShell and submitted to a Microsoft CA.
2. download it `puppet.piccola.us.cer` in `.der` format. once downloaded add it to the personal store of a windows machine then export it with keys and extensions as a `.pfx` (e.g. `puppet.piccola.us.pfx`).
3. at this point you should have the following three files.
```
C:\scripts\puppet_ec_cert\pec> l
    Directory: C:\scripts\puppet_ec_cert\pec
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        6/12/2018   8:17 PM           1224 puppet.piccola.us.cer
-a----        6/12/2018   8:16 PM           1384 puppet.piccola.us.csr
-a----        6/12/2018   8:20 PM           6049 puppet.piccola.us.pfx
```
4. install and use openssl to convert pfx to pem.
```
choco install openssl.light
openssl pkcs12 -in puppet.piccola.us.pfx -clcerts -nokeys -out public-console.cert.pem
Enter Import Password: *****
```
5. use openssl to create passphrased private key.
```
openssl pkcs12 -in puppet.piccola.us.pfx -nocerts -out public-console.private_key_PASSPHRASED_.pem
Enter Import Password: *****
Enter PEM pass phrase: *****
```
6. use openssl to remove passphrase from private key.
```
openssl rsa -in public-console.private_key_PASSPHRASED_.pem -out public-console.private_key.pem
Enter pass phrase for public-console.private_key_PASSPHRASED_.pem: *****
writing RSA key
```
7. at this point you should have the following six files.
```
 C:\scripts\puppet_ec_cert\pec> l
    Directory: C:\scripts\puppet_ec_cert\pec
Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        6/12/2018   9:30 PM           2034 public-console.cert.pem
-a----        6/12/2018   9:34 PM           1706 public-console.private_key.pem
-a----        6/12/2018   9:33 PM           2145 public-console.private_key_PASSPHRASED_.pem
-a----        6/12/2018   8:17 PM           1224 puppet.piccola.us.cer
-a----        6/12/2018   8:16 PM           1384 puppet.piccola.us.csr
-a----        6/12/2018   8:20 PM           6049 puppet.piccola.us.pfx
```
8. copy `public-console.cert.pem` and `public-console.private_key.pem` to `/opt/puppetlabs/server/data/console-services/certs` on the Puppet server.
```
root@puppet:/opt/puppetlabs/server/data/console-services/certs# ls -la
total 36
drwx------ 3 pe-console-services pe-console-services 4096 Jun 12 21:40 .
drwxrwx--- 3 pe-console-services pe-console-services 4096 Sep  8  2017 ..
drwxr-xr-x 2 root                root                4096 Jun 12 21:40 old
-rw-r--r-- 1 root                root                2034 Jun 12 20:33 public-console.cert.pem
-rw-r--r-- 1 root                root                1706 Jun 12 20:51 public-console.private_key.pem
-r-------- 1 pe-console-services pe-console-services 2086 Jun 12 21:40 puppet.piccola.us.cert.pem
-r-------- 1 pe-console-services pe-console-services 3243 Jun 12 21:40 puppet.piccola.us.private_key.pem
-r-------- 1 pe-console-services pe-console-services 2374 Jun 12 21:40 puppet.piccola.us.private_key.pk8
-r-------- 1 pe-console-services pe-console-services  800 Jun 12 21:40 puppet.piccola.us.public_key.pem
```
9. Use the console to edit the parameters of the puppet_enterprise::profile::console class.
    1. Click Classification, and in the PE Infrastructure group, select the PE Console group.
    2. On the Configuration tab, in the `puppet_enterprise::profile::console` class, add the following parameters:

| Parameter | Value |
| --- | --- |
| `browser_ssl_cert` | `/opt/puppetlabs/server/data/console-services/certs/public-console.cert.pem` |
| `browser_ssl_private_key` | `/opt/puppetlabs/server/data/console-services/certs/public-console.private_key.pem` |

10. run `puppet agent -t` on the master.
```
root@puppet:~# puppet agent -t
```
11. use nmap's "ssl-cert.nse" script to verify.
```
c:\> nmap --script=ssl-cert.nse -p 443 puppet.piccola.us
PORT    STATE SERVICE
443/tcp open  https
| ssl-cert: Subject: commonName=puppet.piccola.us
| Subject Alternative Name: DNS:puppet.piccola.us
| Issuer: commonName=Piccola Subordinate CA I
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2018-06-13T02:09:48
| Not valid after:  2020-06-12T02:09:48
| MD5:   e5a6 bb29 8870 b967 49d2 ecf1 0514 c2a2
|_SHA-1: eaf5 df14 db39 d2dc e502 3276 862b 0532 1ce1 557e
MAC Address: 00:50:56:89:9F:17 (VMware)
Nmap done: 1 IP address (1 host up) scanned in 1.45 seconds
```