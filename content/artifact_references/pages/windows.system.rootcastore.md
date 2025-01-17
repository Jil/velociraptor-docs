---
title: Windows.System.RootCAStore
hidden: true
tags: [Client Artifact]
---

Enumerate the root certificates in the Windows Root store.


```yaml
name: Windows.System.RootCAStore
description: |
   Enumerate the root certificates in the Windows Root store.

reference:
   - "ATT&CK: T1553"
   - https://attack.mitre.org/techniques/T1553/004/

parameters:
   - name: CertificateRootStoreGlobs
     type: csv
     default: |
       Accessor,Glob
       reg,HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\SystemCertificates\ROOT\Certificates\**\Blob
       reg,HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\SystemCertificates\ROOT\Certificates\**\Blob
       reg,HKEY_USERS\*\Software\Microsoft\SystemCertificates\Root\Certificates\**\Blob
       reg,HKEY_USERS\*\Software\Policies\Microsoft\SystemCertificates\Root\Certificates\**\Blob

sources:
  - precondition:
      SELECT OS From info() where OS = 'windows'

    query: |
        LET profile = '''[
        ["Record", "x=>x.Length + 12", [
          ["Type", 0, "uint32"],
          ["Length", 8, "uint32"],
          ["Data", 12, "String", {
              length: "x=>x.Length",
              term: "",
          }],
          ["UnicodeString", 12, "String", {
              encoding: "utf16",
          }]
        ]],
        ["Records", 0, [
          ["Items", 0, "Array", {
              type: "Record",
              count: 20,
          }]
        ]]
        ]'''

        // Parse the types from the certificate record itself, as well as the X509 cert structure.
        LET GetCert(CertData) = SELECT parse_x509(data=Data)[0] AS Cert
          FROM foreach(row=parse_binary(filename=CertData,
                       accessor="data", profile=profile, struct="Records").Items)
          WHERE Type = 32

        // Format the fingerprint as a hex string
        LET GetFinger(CertData) = SELECT format(format="%x", args=Data) AS FingerPrint
          FROM foreach(row=parse_binary(filename=CertData,
                       accessor="data", profile=profile, struct="Records").Items)
          WHERE Type = 3

        LET GetName(CertData) = SELECT UnicodeString AS Name
          FROM foreach(row=parse_binary(filename=CertData,
                       accessor="data", profile=profile, struct="Records").Items)
          WHERE Type = 11

        // Glob for certificates in all the locations we know about.
        SELECT * FROM foreach(row=CertificateRootStoreGlobs,
        query={
          SELECT FullPath AS _RegistryValue, ModTime,
               GetName(CertData=Data.value)[0].Name AS Name,
               GetFinger(CertData=Data.value)[0].FingerPrint AS FingerPrint,
               GetCert(CertData=Data.value)[0].Cert AS Certificate
          FROM glob(globs=Glob, accessor=Accessor)
        })

```
