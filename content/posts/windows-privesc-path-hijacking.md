---
title: Windows Local Privilege Escalation - PATH Hijacking for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: "Windows Local Privilege Escalation - PATH Hijacking"
license: ""
images: []
date: 2022-03-20 11:33:00 +0800
categories: [Windows Local Privilege Escalation]
tags: [path hijacking, local privilege escalation, windows]

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
summary: "File systems path permissions and replacements. Here we go again (@_@)"
---

# %PATH% Environment Variable

The `%PATH%` environment variable is a critical component for the operating system. It specifies the directories that the system searches to find executable files for commands, tools, or programs. When a user types a command in the terminal or command prompt, the system looks through the directories listed in the `%PATH%` variable to locate the executable file associated with that command.

This environment variable is particularly useful for running programs without needing to specify their full paths. Instead of typing the complete path to an executable, users can simply type the command name, and the system will search the directories in the `%PATH%` variable to find and execute the program.

The `%PATH%` variable is used for relative paths, meaning it searches through the specified directories in the order they are listed. If the executable is found in one of these directories, it is executed. If not, the system returns an error indicating that the command was not found.

For example, if the `%PATH%` variable includes directories like `C:\Windows\System32` and `C:\Program Files`, the system will search these directories for the executable file when a command is entered. This allows for efficient and convenient execution of programs without needing to navigate to their specific directories.

Properly configuring the `%PATH%` variable is essential for ensuring that the system can locate and run the necessary programs efficiently. It is also important to avoid adding unnecessary or insecure directories to the `%PATH%` variable, as this can pose security risks.

In summary, the environment variable `%PATH%` is used for **relative paths** and not **absolute paths**. 

This is a relative path:

```cmd
cmd.exe
```

This is an absolute path:

```cmd
C:\Windows\System32\cmd.exe
```

When we use a relative path the system needs to search for the executable file and to search for it, it uses the %PATH% environment variable. The first directory listed in the %PATH% environment variable will be the first directory that will be searched, here is an example:

```cmd
C:\Windows\system32 # First search in here, if its not here then continue
C:\Windows          # Second search in here, if its not here then continue
C:\Windows\System32\Wbem # If its here, then execute the command and stop searching
C:\Windows\System32\WindowsPowerShell\v1.0\
C:\Windows\System32\OpenSSH\
C:\Program Files\dotnet\
```

> Note: The %PATH% environment variable is used to search the name of the executable, it doesn't verify the integrity of the file, which means that we can create an illegitimate executable as long as we can hijack the %PATH%.

The **user environment variables** can be found in the registry `HKEY_CURRENT_USER\Environment`:

```reg
PS C:\Users\user> reg query "HKEY_CURRENT_USER\Environment"

HKEY_CURRENT_USER\Environment
    Path    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Microsoft\WindowsApps;%USERPROFILE%\.dotnet\tools
    TEMP    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Temp
    TMP    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Temp
    OneDrive    REG_EXPAND_SZ    C:\Users\user\OneDrive
```

The **system environment variables** can be found in the registry `"HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment"`:

```reg
PS C:\Users\user> reg query "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment"

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment
    ComSpec    REG_EXPAND_SZ    %SystemRoot%\system32\cmd.exe
    DriverData    REG_SZ    C:\Windows\System32\Drivers\DriverData
    OS    REG_SZ    Windows_NT
    Path    REG_SZ    C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Program Files\dotnet\
    PATHEXT    REG_SZ    .COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC
    PROCESSOR_ARCHITECTURE    REG_SZ    AMD64
    PSModulePath    REG_EXPAND_SZ    %ProgramFiles%\WindowsPowerShell\Modules;%SystemRoot%\system32\WindowsPowerShell\v1.0\Modules
    TEMP    REG_EXPAND_SZ    %SystemRoot%\TEMP
    TMP    REG_EXPAND_SZ    %SystemRoot%\TEMP
    USERNAME    REG_SZ    SYSTEM
    windir    REG_EXPAND_SZ    %SystemRoot%
    NUMBER_OF_PROCESSORS    REG_SZ    4
    PROCESSOR_LEVEL    REG_SZ    6
    PROCESSOR_IDENTIFIER    REG_SZ    Intel64 Family 6 Model 165 Stepping 5, GenuineIntel
    PROCESSOR_REVISION    REG_SZ    a505


PS C:\Users\user> (echo " C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Program Files\dotnet\").split(";")
 C:\Windows\system32
C:\Windows
C:\Windows\System32\Wbem
C:\Windows\System32\WindowsPowerShell\v1.0\
C:\Windows\System32\OpenSSH\
C:\Program Files\dotnet\
```

The **current process %PATH% environment variable** can be found with:

```powershell
PS C:\Users\user> ($env:Path).split(";")
C:\Windows\system32
C:\Windows
C:\Windows\System32\Wbem
C:\Windows\System32\WindowsPowerShell\v1.0\
C:\Windows\System32\OpenSSH\
C:\Program Files\dotnet\
C:\Users\user\AppData\Local\Microsoft\WindowsApps
C:\Users\user\.dotnet\tools
```

Alternatively, both the user environment variables and the system environment variables can be found using the GUI: 

-   **Windows XP** - Right-click My Computer, and then click Properties → Advanced → Environment variables → Choose New, Edit or Delete.
-   **Windows 7** - Click on Start → Computer → Properties → Advanced System Settings → Environment variables → Choose New, Edit or Delete.
-   **Windows 8** - Right click on the bottom left corner to get Power User Task Menu → Select System → Advanced System Settings → Environment variables → Choose New, Edit or Delete.
-   **Windows 10** - Right click on Start Menu to get Power User Task Menu → Select System → Advanced System Settings → Environment variables → Choose New, Edit or Delete.

In **Windows 11** is at the following path `Control Panel\System and Security\System\About\Advanced Settings` or `Settings > System > About`:

{{< image src="/images/posts/advanced-system-settings.png" caption="Advanced System Settings " src_s="/images/posts/advanced-system-settings.png" src_l="/images/posts/advanced-system-settings.png" >}}

System Properties -> Advanced -> Environment Variables...:

{{< image src="/images/posts/advanced-system-settings-2.png" caption="Advanced System Settings " src_s="/images/posts/advanced-system-settings-2.png" src_l="/images/posts/advanced-system-settings-2.png" >}}

Now we can see both (user and system) environment variables:

{{< image src="/images/posts/advanced-settings-3.png" caption="Advanced System Settings " src_s="/images/posts/advanced-settings-3.png" src_l="/images/posts/advanced-settings-3.png" >}}

## %PATH% Usage 

We can change the user environment variable with setx and since this environment variable belongs to our current user we don't need to execute this as administrator:

```powershell
PS C:\Users\user> reg query "HKEY_CURRENT_USER\Environment"

HKEY_CURRENT_USER\Environment
    Path    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Microsoft\WindowsApps;%USERPROFILE%\.dotnet\tools
    TEMP    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Temp
    TMP    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Temp
    OneDrive    REG_EXPAND_SZ    C:\Users\user\OneDrive

PS C:\Users\user> setx PATH "C:\Temp;%PATH%"

SUCCESS: Specified value was saved.
PS C:\Users\user> reg query "HKEY_CURRENT_USER\Environment"

HKEY_CURRENT_USER\Environment
    Path    REG_EXPAND_SZ    C:\Temp;%PATH%
    TEMP    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Temp
    TMP    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Temp
    OneDrive    REG_EXPAND_SZ    C:\Users\user\OneDrive
```

Alternatively, we can use PowerShell to change the user environment variable:

```powershell
PS C:\Users\user> [Environment]::SetEnvironmentVariable( 'Path', 'C:\Temp;%USERPROFILE%\AppData\Local\Microsoft\WindowsApps;%USERPROFILE%\.dotnet\tools', [EnvironmentVariableTarget]::User )

PS C:\Users\user> reg query "HKEY_CURRENT_USER\Environment"

HKEY_CURRENT_USER\Environment
    Path    REG_SZ    C:\Temp;%USERPROFILE%\AppData\Local\Microsoft\WindowsApps;%USERPROFILE%\.dotnet\tools
    TEMP    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Temp
    TMP    REG_EXPAND_SZ    %USERPROFILE%\AppData\Local\Temp
    OneDrive    REG_EXPAND_SZ    C:\Users\user\OneDrive
```

Changing the system %PATH% environment variable requires to be executed as administrator:

```powershell
PS C:\Windows\system32> [Environment]::SetEnvironmentVariable( 'Path', 'C:\Temp', [EnvironmentVariableTarget]::Machine )


PS C:\Windows\system32> reg query "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" | findstr "Path"
    Path    REG_SZ    C:\Temp
    PSModulePath    REG_EXPAND_SZ    %ProgramFiles%\WindowsPowerShell\Modules;%SystemRoot%\system32\WindowsPowerShell\v1.0\Modules
```

There is a problem with the above configuration and that is the value of the `Path` registry key (it only has one directory), it must not disturb other programs that use this environment variable therefore let's change its value:

```powershell
PS C:\Windows\system32> [Environment]::SetEnvironmentVariable( 'Path', 'C:\Temp;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Program Files\dotnet\', [EnvironmentVariableTarget]::Machine )

PS C:\Windows\system32> reg query "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\Environment" | findstr "Path"
    Path    REG_SZ    C:\Temp;C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Program Files\dotnet\
    PSModulePath    REG_EXPAND_SZ    %ProgramFiles%\WindowsPowerShell\Modules;%SystemRoot%\system32\WindowsPowerShell\v1.0\Modules
```

Now it won't disturb other programs that use the system %PATH% environment variable and we will be able to hijack our target program.

We can also change the current process %PATH% environment variable and this can be executed as our current user:

```powershell
PS C:\Users\user> $env:Path = "C:\Temp;$env:Path"
PS C:\Users\user> ($env:Path).split(";")
C:\Temp
C:\Windows\system32
C:\Windows
C:\Windows\System32\Wbem
C:\Windows\System32\WindowsPowerShell\v1.0\
C:\Windows\System32\OpenSSH\
C:\Program Files\dotnet\
C:\Users\user\AppData\Local\Microsoft\WindowsApps
C:\Users\user\.dotnet\tools
```

Alternatively, if we have access to the GUI we can go to:
- System Properties -> Advanced -> Environment Variables... -> System variables

# %PATH% Hijacking Privilege Escalation

To create a vulnerable $PATH hijacking scenario we need the following requirements:

- PATH contains a writable folder.
- The writable folder is **before or above** the folder that contains the legitimate binary.

The "vulnerability" or "problem" lies in the fact that if we're able to write to a directory that's before or above the valid program/command/tool/application, we can then create a malicious binary with the same name as the valid program/command/tool/application. Once it is found Windows will stop searching for the program since it will assume that it found it and it will execute it:

```powershell
C:\Windows\system32 # First search in here, if its not here then continue
C:\Windows          # If we have write access then we can create a malicious binary with the same name as the valid binary. Once the binary -name- is found Windows will stop searching for the program since it will assume that it found it and it will execute it.
C:\Windows\System32\Wbem 
C:\Windows\System32\WindowsPowerShell\v1.0\
C:\Windows\System32\OpenSSH\ # Valid binary is here but it won't be executed since it already found it and it stopped searching.
C:\Program Files\dotnet\
```

Run PowerShell and create a directory:

```powershell
mkdir C:\Temp
```

Add the `modify` (M) permission:

```powershell
PS C:\Windows\system32> icacls.exe "C:\Temp" /grant "NT AUTHORTY\Authenticated Users:M"
NT AUTHORTY\Authenticated Users: No mapping between account names and security IDs was done.
Successfully processed 0 files; Failed processing 1 files

PS C:\Windows\system32> icacls.exe "C:\Temp"
C:\Temp BUILTIN\Administrators:(I)(OI)(CI)(F)
        NT AUTHORITY\SYSTEM:(I)(OI)(CI)(F)
        BUILTIN\Users:(I)(OI)(CI)(RX)
        NT AUTHORITY\Authenticated Users:(I)(M)
        NT AUTHORITY\Authenticated Users:(I)(OI)(CI)(IO)(M)

Successfully processed 1 files; Failed processing 0 files
```

Copy `calc.exe` to `C:\Temp\cmd.exe`:

```powershell
PS C:\Windows\system32> where.exe calc.exe
C:\Windows\System32\calc.exe

PS C:\Windows\system32> copy C:\Windows\System32\calc.exe C:\Temp\cmd.exe
```

Change the `$PATH` environment variable of the current process:

```powershell
PS C:\> $env:Path = "C:\Temp;$env:Path"
PS C:\> ($env:Path).split(";")
C:\Temp
C:\Windows\system32
C:\Windows
C:\Windows\System32\Wbem
C:\Windows\System32\WindowsPowerShell\v1.0\
C:\Windows\System32\OpenSSH\
C:\Program Files\dotnet\
```

> Note: Remember this is the PATH environment variable of our current process, if this executable was executed by a Service, Task Scheduler, or something else. We would need to use the system %PATH% environment variable instead, depending on the configuration.

Review which `cmd.exe` will be loaded first:

```powershell
PS C:\> cd \
PS C:\> where.exe cmd.exe
C:\Temp\cmd.exe
C:\Windows\System32\cmd.exe
```

Because the directory "`C:\Temp`" is before "`C:\Windows\system32\`" in the PATH environment variable, the next time the user runs "cmd.exe", our calculator will run, instead of the legitimate one in the system32 folder.

```powershell
cmd.exe
```

As expected the calculator was spawned:

{{< image src="/images/posts/calculator.png" caption="Calculator " src_s="/images/posts/calculator.png" src_l="/images/posts/calculator.png" >}}

Now, let's try it with the same binary (cmd.exe):

```powershell
PS C:\> copy C:\Windows\System32\cmd.exe C:\Temp\cmd.exe
```

Execute the cmd.exe again:

```powershell
PS C:\> cmd.exe
The system cannot find message text for message number 0x2350 in the message file for Application.

(c) Microsoft Corporation. All rights reserved.
Not enough memory resources are available to process this command.
```

Using Process Hacker as Administrator we can see the process tree:

{{< image src="/images/posts/process-hacker-process-tree.png" caption="Process Hacker Process Tree " src_s="/images/posts/process-hacker-process-tree.png" src_l="/images/posts/process-hacker-process-tree.png" >}}

We can also view the token of the `cmd.exe` process and from there we can see that is running with High-Integrity Level and we have more privileges since `cmd.exe` was executed from an administrator PowerShell console:

{{< image src="/images/posts/process-hacker-token 1.png" caption="Process Hacker Token " src_s="/images/posts/process-hacker-token 1.png" src_l="/images/posts/process-hacker-token 1.png" >}}

If this was executed by a service with the LocalSystem user (System Integrity Level) or by a task scheduler running with the highest level (High Integrity Level), or simply by an administrator (High Integrity Level) we could elevate our privileges as long as we replace the `cmd.exe` command with our payload or implant.