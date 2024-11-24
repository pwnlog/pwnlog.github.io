---
title: Linux Privilege Escalation - Port Forwarding for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: ""
license: ""
images: []
date: 2021-12-28 11:33:00 +0800
categories: [Linux Privilege Escalation]
tags: [privilege escalation, ssh, local port forwarding, remote port forwarding, dynamic port forwarding, defend]

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

summary: "Services can go outside too? (\\*<\\*)"
---

# Port Forwarding

When there's a service listening on localhost and not externally, it doesn't mean that we cannot access this service. Port forwarding as it name implies it allow us to foward a port to another socket. If this particular service is vulnerable to anything related an elevation of privilege, then it can be used as an elevation path.

# Apache2 Scenario

Install apache2 with apt:

```shell
sudo apt install apache2
```

Configure the apache2 listening ports:

```shell
user@pwn:~$ cat /etc/apache2/ports.conf
# If you just change the port or add more ports here, you will likely also
# have to change the VirtualHost statement in
# /etc/apache2/sites-enabled/000-default.conf

Listen 127.0.0.1:80

<IfModule ssl_module>
	Listen 127.0.0.1:443
</IfModule>

<IfModule mod_gnutls.c>
	Listen 127.0.0.1:443
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

Enable the service, restart the service, and check its status:

```shell
sudo systemctl enable apache2 && sudo systemctl restart apache2 && sudo systemctl status apache2
```

Check if port 80 is listening on the loopback interface:

```shell
user@pwn:~$ ss -tnlp | grep 80
LISTEN  0        511            127.0.0.1:80             0.0.0.0:*    
```

The options above are the following:
- -t: Display TCP sockets.
- -n: Do not try to resolve service names.
- -l: Display only listening sockets
- -p: Show process using socket.

Verify if it's working by going to the browser and opening the localhost URL `http://localhost:80`.

Do a nmap scan from an attacker host to prove that's listening only on localhost:

```shell
❯ sudo nmap -p80 192.168.146.131
[sudo] password for pwnlog:
...<SNIP>...

PORT   STATE  SERVICE
80/tcp closed http
MAC Address: 00:0C:29:6A:00:F9 (VMware)

Nmap done: 1 IP address (1 host up) scanned in 0.25 seconds
```

As you can see the port is closed externally.

# Privilege Escalation via Local Port Forwarding

Local Port Forwarding means forwarding the port that's listening on the target host loopback interface to our host. This is done by connecting to the target host.

- Local: Connect to the target host from your adversary host.

This technique involves forwarding the port that's listening on the loopback interface in the target/victim host to our remote host:

```shell
user@parrot:~$ ssh -L <your-port>:127.0.0.1:<port-to-forward> user@<target-ip>

user@parrot:~$ ssh -L 9898:127.0.0.1:80 user@192.168.146.131
```

After authenticating successfully to the target host. We can see that our adversary host now has established a socket:

```shell
❯ ss -tlnp | grep 9898
LISTEN 0      128        127.0.0.1:9898      0.0.0.0:*    users:(("ssh",pid=491131,fd=5))
LISTEN 0      128            [::1]:9898         [::]:*    users:(("ssh",pid=491131,fd=4))
```

In our adversary host, we can enumerate the response headers to see if the `Server` header indicates that is running Apache2 in Ubuntu:

```shell
❯ curl -I http://localhost:9898
HTTP/1.1 200 OK
Date: Tue, 20 Dec 2021 20:26:08 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Tue, 20 Dec 2021 00:11:10 GMT
ETag: "2aa6-5d429a897fe9c"
Accept-Ranges: bytes
Content-Length: 10918
Vary: Accept-Encoding
Content-Type: text/html
```

As we can see from the output above we have successfully forwarded port 80 from the target machine (Ubuntu) to our adversary host on port 9898 (ParrotOS/Kali Linux).

When we're done we can close this connection by exiting the SSH session.

# Privilege Escalation via Remote Port Forwarding

Remote Port Forwarding means forwarding the port that's listening on the target host loopback interface to our host. This is done by connecting to our host **from** the target host.

- Remote: Connect to your adversary host from the target host.

This technique involves forwarding the port that's listening on the loopback interface in the target/victim host to our remote host:

```shell
user@victim-host:~$ ssh -R <your-port>:localhost:<port-to-forward> your_username@your_adversary_host_IP

user@pwn:~$ ssh -R 9999:localhost:80 user@10.10.10.14
```

Now go to your adversary host and verify that port 9999 is listening on your localhost:

```shell
❯ echo "This host is `hostname`" && echo '' &&  ss -tnlp | grep 9999
This host is kali

LISTEN 0      128        127.0.0.1:9999      0.0.0.0:*
LISTEN 0      128            [::1]:9999         [::]:*
```

When we `exit` the ssh connection that's in the victim machine then the port will stop being forwarded.

# Privilege Escalation via Dynamic Port Forwarding

Dynamic port forwarding means that it'll forward all the ports from the remote host to our SOCKS port.

SOCKS works on the OSI Layer 5 (Session Layer), so don't expect things like ICMP, ARP or the half-open reset that SYN scan on Nmap to work. To scan via proxy with Nmap we need to do a TCP connect scan with the option `-sT` and we should ignore ICMP with `-Pn`.

There are different tools to help we out when Dynamic port forwarding is being used. The most common one is `proxychains` which are available for Linux and Mac but also for Windows.

Dynamic port forwarding allows we to create a socket on the local (ssh client) machine, which acts as a SOCKS proxy server. When a client connects to this port, the connection is forwarded to the remote (ssh server) machine, which is then forwarded to a dynamic port on the destination machine.

This way, all the applications using the SOCKS proxy will connect to the SSH server, and the server will forward all the traffic to its actual destination.

Install the `proxychains` package:

```shell
sudo apt install proxychains
```

Alternatively, we can build it from the [source code](https://github.com/haad/proxychains) from GitHub.

Let's configure the proxychains:

```shell
sudo vim /etc/proxychains.conf
```

Add the following configuration:

```shell
socks4  127.0.0.1 9090
```

In Linux, macOS, and other Unix systems to create a dynamic port forwarding (SOCKS) pass the `-D` option to the `ssh` client:

```shell
ssh -D [LOCAL_IP:]LOCAL_PORT [USER@]SSH_SERVER
```

The options used are as follows:

-   `[LOCAL_IP:]LOCAL_PORT` - The local machine IP address and port number. When `LOCAL_IP` is omitted, the ssh client binds on localhost.
-   `[USER@]SERVER_IP` - The remote SSH user and server IP address.

The following command will create a SOCKS tunnel on port `9090` and connect to the target machine, we're gonna execute this command from our adversary host:

```shell
ssh -D 9090 -N -f user@10.10.10.14
```

The options used are as follows:
- -N: Do not execute a remote command.  This is useful for just forwarding ports.
- -f: Requests ssh to go to the background just before command execution.  This is useful if ssh is going to ask for passwords or passphrases, but the user wants it in the background. 

Verify that the socket has been established, and execute this command in your adversary host:

```shell
❯ ss -tnlp | grep 9090
LISTEN 0      128        127.0.0.1:9090      0.0.0.0:*    users:(("ssh",pid=500777,fd=5))
LISTEN 0      128            [::1]:9090         [::]:*    users:(("ssh",pid=500777,fd=4))
```

Once the tunneling is established, we can configure your application to use it. [This article](https://linuxize.com/post/how-to-setup-ssh-socks-tunnel-for-private-browsing/) explains how to configure Firefox and Google Chrome browsers to use the SOCKS proxy.

The port forwarding has to be separately configured for each application that we want to tunnel the traffic through it.

In the victim host, I have these ports open:

```shell
user@pwn:~$ ss -tnlp
State               Recv-Q              Send-Q                             Local Address:Port                             Peer Address:Port              Process              
LISTEN              0                   511                                    127.0.0.1:80                                    0.0.0.0:*                                      
LISTEN              0                   4096                               127.0.0.53%lo:53                                    0.0.0.0:*                                      
LISTEN              0                   128                                      0.0.0.0:22                                    0.0.0.0:*                                      
LISTEN              0                   5                                      127.0.0.1:631                                   0.0.0.0:*                                      
LISTEN              0                   128                                         [::]:22                                       [::]:*                                      
LISTEN              0                   5                                          [::1]:631                                      [::]:*             
```

We can do port scans and service scans with nmap:

```shell
❯ sudo proxychains nmap -sT -Pn 127.0.0.1 -vvv 2>/dev/null
Starting Nmap 7.92 ( https://nmap.org ) at 2021-12-28 15:48 EST
Initiating Connect Scan at 15:48
Scanning localhost (127.0.0.1) [1000 ports]
Discovered open port 22/tcp on 127.0.0.1
Discovered open port 80/tcp on 127.0.0.1
Discovered open port 631/tcp on 127.0.0.1
Completed Connect Scan at 15:48, 1.30s elapsed (1000 total ports)
Nmap scan report for localhost (127.0.0.1)
Host is up, received user-set (0.0011s latency).
Scanned at 2021-12-28 15:48:24 EST for 2s
Not shown: 997 closed tcp ports (conn-refused)
PORT    STATE SERVICE REASON
22/tcp  open  ssh     syn-ack
80/tcp  open  http    syn-ack
631/tcp open  ipp     syn-ack

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 1.35 seconds
```

Noticed how I used the options:
- -sT = TCP connect scan, rather than the default `-sS` SYN scan. The SYN won’t work because the proxy doesn't pass the TCP handshake packets back to our adversary host, an SYN scan, which sends the SYN packet, sees the ACK, and then ends the connection, this won’t be passed back over the proxy, therefore, SYN scans don't work with proxies.
- -Pn = Ignore ICMP request/response because ICMP doesn't go through the proxy.

Without these options the scan will fail because the proxy will drop any SYN scans, we need the full TCP handshake to know if the port is open or not.

We can try to run Firefox with `proxychains`:

```shell
proxychains firefox
```

Now we can access port 80 from the target host with the SOCKS port that is defined in our `proxychains` configuration file so we can navigate to `http://localhost:80` in our adversary host browser and we will see the apache2 default page.

This can be seen by using curl with proxychains and enumerating the response headers:

```shell
❯ proxychains curl -I http://localhost:80
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.15
[proxychains] Strict chain  ...  127.0.0.1:9090  ...  127.0.0.1:80  ...  OK
HTTP/1.1 200 OK
Date: Tue, 20 Dec 2021 21:00:29 GMT
Server: Apache/2.4.41 (Ubuntu)
Last-Modified: Tue, 20 Dec 2021 00:11:10 GMT
ETag: "2aa6-5d429a897fe9c"
Accept-Ranges: bytes
Content-Length: 10918
Vary: Accept-Encoding
Content-Type: text/html
```

The advantage of using `proxychains` with dynamic port forwarding is that we can access all the ports in the target dynamically. Let's try to log in to SSH using the `proxychains` to prove this:

```shell
❯ proxychains ssh user@localhost
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.15
[proxychains] Strict chain  ...  127.0.0.1:9090  ...  127.0.0.1:22  ...  OK
user@localhost's password:
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.11.0-43-generic x86_64)

<...SNIP...>

user@pwn:~$ hostname
ubuntu
```

As we can see from the output above we're able to log in to SSH via `proxychains`. This is almost the same as executing `ssh user@localhost` in the victim machine.

We can close this connection by killing its PID, first finding the PID of the process:

```shell
❯ ps -ef | grep 'ssh -D 9090' | grep -v 'grep' 2>/dev/null
kali      500777       1  0 15:42 ?        00:00:02 ssh -D 9090 -N -f user@10.10.10.14
```

The PID is the one in the second field from the output above, in this case, is (500777). Now kill the process:

```shell
❯ kill -9 500777
```