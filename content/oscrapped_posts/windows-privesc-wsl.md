---
title: Windows Local Privilege Escalation - WSL
draft: true
author: "pwnlog"
authorLink: ""
description: "Windows Local Privilege Escalation - WSL"
license: ""
images: []
date: 2022-03-20 11:33:00 +0800
categories: [Windows Local Privilege Escalation]
tags: [wsl, local privilege escalation, windows]

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
---



# Lab Information

This lab was created and tested on the following environment:

| Target Operating System                 | Attacking Operating System | Network Class | Virtualization Environment  |
|-----------------------------------------|----------------------------|---------------|-----------------------------|
| Windows 11 Enterprise Evalution Edition | Kali Linux 2021.4          | IPv4 Class C  | VMware Workstation 17.0 Pro |

The date of this writing.

![[date.png]]

Windows operating system version.

![[winver.png]]

Windows updates information.

![[hotfix.png]]

# Configuration

Open PowerShell or Windows Command Prompt as **administrator** by right-clicking and selecting "Run as administrator" and enter the following command.

```powershell
wsl --install
```

During the installation process, a UAC prompt may appear to install WSL in the system.

![[wsl-uac.png]]

By default, it'll attempt to install Ubuntu.

![[wsl-ubuntu.png]]

After a system has been installed, it'll finish the installation process.

![[wsl-installed.png]]

Now, we need to reboot the system to apply the changes.

```powershell
Restart-Computer
```

After rebooting the system we'll wait a few seconds or minutes for the Ubuntu installation to complete. However, an error code `0x80370102` may appear.

![[installing-error.png]]

According to the [documentation](https://learn.microsoft.com/en-us/windows/wsl/troubleshooting#error-0x80370102-the-virtual-machine-could-not-be-started-because-a-required-feature-is-not-installed), this error is described as the following:

> Error: 0x80370102 The virtual machine could not be started because a required feature is not installed.

There are two fixes for this error:
1. Use WSL version 1.0
2. Enable nested virtualization (Problem: It may not work with third party software)

## Method 1: WSL 1.0 (Recommended)

However, changing WSL to version 1.0 is not straight forward.

![[wsl-version.png]]

This error above happens because WSL installed but its `optional component` feature is not enabled . We can use PowerShell as administrator to enable it.

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

![[wsl-enable-feature.png]]

After the system has rebooted, we'll open a PowerShell console and run the command again.

```powershell
wsl --set-default-version 1
```

![[wsl-set-default-version.png]]

This time around the operation completed successfully.

## Method 2: Optional Features (Hyper-V Only)

> This steps apply to Hyper-V and not VMware.

Therefore, we have to enabled the required feature which is Hyper-V. We can install this through `optional features`. We can launch the Windows Features program by running the following command.

![[optionalfeatures.png]]

We can see that a few Hyper-V features are greyed out.

![[hyper-v-gray.png]]

This is because this system has Second-Level Address Translation (SLAT) capabilites enabled. The solution to this issue is to enable nested virtualization with PowerShell as administrator. However, we'll need a Hyper-V PowerShell module, which can be installed by selecting the following in your (host) system and not your (guest/VM) system.

![[hyper-v-powershell-module.png]]

The following [answer](https://learn.microsoft.com/en-us/windows/wsl/faq#can-i-run-wsl-2-in-a-virtual-machine-) at Microsoft, specifies that we need to run the following command from host. Again this command is runned from your host and not your guest/VM.

```
Set-VMProcessor -VMName <VMName> -ExposeVirtualizationExtensions $true
```

 Enable `Virtual Machine Platform` feature. This enables virtualization support for the current system.

```
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

![[virtualmachineplatform.png]]

After enabling this feature, we'll need to reboot the system to apply the changes.

```powershell
Restart-Computer
```

Ensure that the hypervisor is enabled in our boot configuration.

![[boot-configuration.png]]

If this didn't solve this error for you, please visit the following documentations:

- Error link: https://learn.microsoft.com/en-us/windows/wsl/troubleshooting#error-0x80370102-the-virtual-machine-could-not-be-started-because-a-required-feature-is-not-installed
- Nested virtualization link: https://learn.microsoft.com/en-us/virtualization/hyper-v-on-windows/user-guide/nested-virtualization#configure-nested-virtualization

## Installing Ubuntu in WSL

Now we'll install Ubuntu in our WSL.

```powershell
wsl --install -d ubuntu
```

![[wsl-ubuntu-launch.png]]

I'm going to enter a new UNIX username.

![[wsl-ubuntu-setup.png]]

The password will be `easypass`.

![[wsl-passwords.png]]

When the password don't match, you're allowed to input them again.

![[wsl-linux.png]]

The installation is now done. The credentials that I used for this WSL system are the following:

- Username: user-wsl
- Password: easypass

# Interacting with WSL

We have installed Ubuntu in WSL. We can use it in many ways. In this section, we'll go over a few common methods in which we can use this image.

When WSL is installed, it'll create an executable in the system, which is to be expected.

![[where-wsl.png]]

When a Linux distribution is installed, it'll also install some components outside the image such as bash.

![[where-bash.png]]

Because this executables are outside the image, we can use them from our host.

![[wsl-ip.png]]

we can learn more about WSL commands in the documentation:

- https://learn.microsoft.com/en-us/windows/wsl/basic-commands

# Privilege Escalation via WSL

## Configuration

If the default user of WSL image is `root`, it'll immediatly make it vulnerable to "privilege escalation" since we'll land as `root`. This can be configured from a Command Prompt or PowerShell console running as administrator.

```powershell
ubuntu.exe config --default-user root
```

![[wsl-default-user.png]]

As we saw in the previous section, the default user was `low-wsl` but now it's root.

![[wsl-whoami.png]]

This means that we literally interact as the `root` user.

![[wsl-bash.png]]

## Enumeration

_What information do we need to find?_
- _If WSL is installed and enabled_
- _The WSL Linux distribution_
- _WSL default user_

We can if WSL is installed, enabled, and its Linux images with one command.

```
wsl --list --verbose
```

![[wsl-list.png]]

It also tells us its WSL version, which is great. Now we need information about its default user.

```
wsl whoami
```

![[wsl-whoami-root.png]]

The WSL system default user is root.

## Attack

The attack can be done in multiple ways and its all on the hands of the adversary. In this case, we could either use `wsl` or `bash` to execute an implant. This implant is up to the users creativity.

```
wsl.exe <implant>
```

If you're using this in a CTF, we may use a reverse shell.

```
wsl.exe <reverse_shell_code/command>
```

A few TTPs that can be done with WSL can be any of the following:
- Installing Utilities
- Installing adversary Distributions
- Hijack Execution Flow by Redirecting to Linux Utilities

WSL also allows us to mix Linux and Windows utilities.

## Detection

We can log the command line events by configuring Group Policy. Take into consideration the following policies.

| Policy                                          | Path                                                                                                                                                                          | Setting | Short Description                            |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------- | -------------------------------------------- |
| Audit Process Creation                          | Local Computer Policy -> Computer Configuration -> Windows Settings -> Security Settings -> Advanced Audit Policy Configuration -> System Audit Policies -> Detailed Tracking | Success | Logs every process that's created            |
| Include command line in process creation events | Local Computer Policy -> Computer Configuration -> Administrative Templates -> System -> Audit Process Creation                                                               | Enabled | Logs the command line commands and arguments |

Additonally, EDR systems may be to use MITRE ATT&CK techniques IDs to identify suspicious behaviours. A few techniques are documented in MITRE:

- Indirect Command Execution: https://attack.mitre.org/techniques/T1202/

# Defense

These are few defenses that we could implement in our current WSL configuration:
- Change the default user to a user privilege user.
- Disable WSL (if possible).
- Create a restricted bash such as rbash as the default shell. This would limit the commands that a particular user can execute.

# Migitation/Remediation

The most straight remediation to this problem is to disable WSL with the following PowerShell cmdlet:

```
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux
```

Alternatively, it can de disabled with DISM.

```
dism.exe /online /disable-feature /featurename:Microsoft-Windows-Subsystem-Linux
```

It could also be also disabed by unchecking the `Windows Subsystem for Linux` in Windows Features. This can launched using the `optionalfeatures` command.

![[wsl-subsystem.png]]

Additionally, Windows Features can disabled in the following registry paths:

```
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Policies\Programs
HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\Programs
```

The setting that needs to be changed is _NoWindowsFeatures_ value to one (1). This one is not recommended as it may affect production systems.