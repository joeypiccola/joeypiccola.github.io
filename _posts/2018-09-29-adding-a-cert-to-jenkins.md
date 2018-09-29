---
title: "Adding a certificate to Jenkins on Windows."
tags:
  - jenkins
  - pki
---

The following assumes you requested a certificate from a Microsoft CA. The requested certificate was downloaded as base 64 and saved to D:\install_files\cert\jenkins.ad.piccola.us. This certificate was imported into jenkins.ad.piccola.us's personal cert store and then exported with 1) the private key 2) extended properties, and 3) all certificates in the certification path.

1. Generate a JKS keystore from the previously exported D:\install_files\cert\jenkins.ad.piccola.us.pfx.
```
D:\Program Files (x86)\Jenkins\jre\bin>keytool.exe -importkeystore -srckeystore D:\install_files\cert\jenkins.ad.piccola.us.pfx -srcstoretype pkcs12 -destkeystore jenkins.ad.piccola.us.jks -deststoretype JKS
Enter destination keystore password:
Re-enter new password:
Enter source keystore password:
Entry for alias certreq-6008260b-a0d5-4948-b329-9200fad7f20a successfully imported.
Import command completed:  1 entries successfully imported, 0 entries failed or cancelled
```

2. Move the generated JKS file from D:\Program Files (x86)\Jenkins\jre\bin to D:\install_files\cert\jenkins.ad.piccola.us.jks. We do this just to keep track of it.
```
D:\Program Files (x86)\Jenkins\jre\bin>move jenkins.ad.piccola.us.jks d:\install_files\cert
        1 file(s) moved.
```

3. Copy the genereated JKS file from D:\install_files\cert\jenkins.ad.piccola.us.jks to D:\Program Files (x86)\Jenkins\jenkins.ad.piccola.us.jks (aka JENKINS_HOME).
```
d:\install_files\cert>copy jenkins.ad.piccola.us.jks "d:\Program Files (x86)\Jenkins"
        1 file(s) copied.
```

4. Update `D:\Program Files (x86)\Jenkins\jenkins.xml`.
```xml
41 | <arguments>-Xrs -Xmx256m -Dhudson.lifecycle=hudson.lifecycle.WindowsServiceLifecycle -jar "%BASE%\jenkins.war" --httpPort=-1 --httpsPort=443 --httpsKeyStore="%JENKINS_HOME%\jenkins.ad.piccola.us.jks" --httpsKeyStorePassword="--REMOVED--" --webroot="%BASE%\war"</arguments>
```

5. Files used in this process.
```
d:\install_files\cert>tree /F
Folder PATH listing for volume Data
Volume serial number is 0C30-53CA
D:.
    jenkins.ad.piccola.us.cer   <-- downloaded cert from web enrollment
    jenkins.ad.piccola.us.csr   <-- certificate signing request (not discussed in this post)
    jenkins.ad.piccola.us.inf   <-- inf file used for certificate signing request (not discussed in this post)
    jenkins.ad.piccola.us.jks   <-- password protected java key store
    jenkins.ad.piccola.us.pfx   <-- password protected exported cert
No subfolders exist
```