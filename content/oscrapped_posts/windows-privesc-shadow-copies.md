---
title: Windows Local Privilege Escalation - Shadow Copies
draft: false
author: "pwnlog"
authorLink: ""
description: "Windows Local Privilege Escalation - Shadow Copies"
license: ""
images: []
date: 2022-03-20 11:33:00 +0800
categories: [Windows Local Privilege Escalation]
tags: [shadow copies, local privilege escalation, windows]


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

# Shadow Copies

The Volume Shadow Copy Service (VSS), which was introduced in Windows Server 2003, is known by multiple names:

- Volume Shadow Copy Service
- Volume Snapshot Service (VS)
- Shadow Copies

Shadow Copies (also known as Volume Snapshot Service, Volume Shadow Copy Service or VSS) are snapshots or copies of computer volumes and files. Therefore, Shadow Copies are used to create backups of a system manually or automatically. These can then be restored. The problems lies in the fact that if we gained administrator privileges we could gather sensitive files which may contain sensitive information.

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

Open System Properties with the `sysdm.cpl` command.

![[sysdm.png]]

Click on the `System Protection` tab.

![[system-protection-tab.png]]

Click on the `Configure` button.

![[configure-button.png]]

Enable system protection and configure the max usage.

![[turn-on-system-protection.png]]

Apply the changes and click on the `Create` button.

![[create-restore-point.png]]

Create a backup name.

![[restore-point-name.png]]

Then it'll start creating the restore point.

![[creating-a-restore-point.png]]

Once it has completed, we'll receive a window prompt.

![[restore-point-was-created.png]]

Alternatively, we could create a backup of a `C:\` with WMIC.

```
wmic shadowcopy call create volume='C:\'
```

![[wmic-shadow-copy.png]]

# Working with Shadow Copies

We can list shadow copies using `vssadmin`. However, we need administrator privileges.

```powershell
vssadmin list shadows
```

![[vssadmin-list-shadows.png]]

If we want to delete a restore point/shadow. We can do it again using System Properties.

![[delete-restore-point-gui.png]]

Alternatively, we could use `vssadmin` to delete a restore point/shadow.

```
vssadmin delete shadows /Shadow={shadow copy ID}
```

![[vssadmin-delete-shadow.png]]

> Note: I used `cmd.exe` to execute the command above.

If we want to delete all the restore points/shadows, we could also do it.

```
vssadmin delete shadows /all
```

## Symlink Shadow Copy

We can create a symlink to access a shadow easily.

```
mklink /d C:\ShadowCopy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy2\
```

![[mklink-shadow-copy.png]]

> Note: In the image above, I didn't use a backslash (`\`) at the end of the command. This will throw an error the following error:

```
C:\Windows\system32>cd C:\ShadowCopy
The parameter is incorrect.
```

We can fix this by removing the directory and creating a symlink with the backslash (`\`) at the end.

![[shadow-copy-linked.png]]

# Privilege Escalation via Shadow Copies

We're looking for sensitive information such as credentials, memory dumps, or other.

## Dump SAM & SYSTEM Backups

SAM and SYSTEM files cannot normally be accessed on an active system but they can be accessed in backups. They are commonly located in these directories:

```
%SYSTEMROOT%\repair\SAM
%SYSTEMROOT%\System32\config\RegBack\SAM
%SYSTEMROOT%\System32\config\SAM
%SYSTEMROOT%\repair\system
%SYSTEMROOT%\System32\config\SYSTEM
%SYSTEMROOT%\System32\config\RegBack\system
```

Once these files has been exfiltrated from the system, we could use impacket to crack them.

```bash
impacket-secretsdump -sam SAM -system SYSTEM LOCAL
```

## Others

The creativity here is up to the adversary and what he/she can find.

## Detection

We can log the command line events by configuring Group Policy. Take into consideration the following policies.

| Policy                                          | Path                                                                                                                                                                          | Setting | Short Description                 |
|-------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------|-----------------------------------|
| Audit Process Creation                          | Local Computer Policy -> Computer Configuration -> Windows Settings -> Security Settings -> Advanced Audit Policy Configuration -> System Audit Policies -> Detailed Tracking | Success | Logs every process that's created |
| Include command line in process creation events | Local Computer Policy -> Computer Configuration -> Administrative Templates -> System -> Audit Process Creation                                                               | Enabled | Logs the command line commands and arguments   |

Access to the following path is immediatly suspicious because it's not common for a process to access shadow copies.

```powershell
 \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy<Number>
```

# Defense

There are a few things that we could consider:
- Harden ACLs
- Avoid storing potential sensitive files that could be leveraged to elevate privileges

# Mitigation/Remediation

Backups are really important for system administrators. Therefore, deleting backups it's not recommended. One of the best solutions would be store backups in cold storage/offline storage. However, that might not always be an option. 