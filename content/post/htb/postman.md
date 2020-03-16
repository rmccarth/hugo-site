---
title: "Postman HTB [Medium]"
date: 2020-01-03T18:10:03-05:00
draft: false
description: "Postman is a vulnerable machine rated Easy on Hack the Box (hackthebox.eu). | API and Python RCE, SQL Query Vulnerabilities"
---

If the screenshots here are critical to your success in implementing this penetration test, you can find the full writeup here on my medium channel: 

> https://medium.com/@rob.mccarthy31/postman-write-up-hack-the-box-49785d2ea78f

Postman is a vulnerable machine rated Easy on Hack the Box (hackthebox.eu).

Hostname: Postman
IP Address: 10.10.10.160
Release Date: 02 Nov 2019
Points: 20

> High-level Summary

The 10.10.10.160 host is a linux machine running a web server on port 80. There is a Webmin portal on port 10000 that is vulnerable to credential-reuse and an exposed Redis database on port 6379 which does not require authentication. The machine is running SSH on port 22 which is used to gain an initial foothold on the system.
The penetration tester was able to authenticate via SSH public-private key pairing after uploading a public key to the Redis database. Authenticating as the Redis user allowed the penetration tester to enumerate the local file system and discover an encrypted backup of a private key. Cracking the password for the key provided a means of authentication to the Webmin portal on port 10000. This access allowed for the penetration tester to use publicly-available exploit code to compromise the Webmin service which is running as the root user.

>Attack Path

Initial Scan(s):
```bash
nmap -n -v --open -p- 10.10.10.160 |tee nmap-discovery
nmap -n -v -sV -sC -p22,80,443,6379,10000 10.10.10.160 |tee nmap-svc
```

Initial Foothold:
The Redis service can be exploited as the database is write-able and does not require authentication. The author of Redis says that if Redis is exposed to the internet, then it is vulnerable, so lets get cracking (Packet Storm, 2015).

```bash
$ sudo apt install redis-server
$ redis-cli -h 10.10.10.160
$ echo "testing-command-execution"
```

We get a response, so we know we can execute commands unauthenticated on the Redis database. It lets us navigate to /var/lib/redis/.ssh/ so we know this directory exists. Lets try to write to it.

At this point we can exit the redis-cli and generate our ssh keys for the initial foothold.

```bash
$ ssh-keygen -t rsa -C slixperi@htb
```

I used id_rsa as the name of my private key, and provided no password when given the option by ssh-keygen.

```bash
$ (echo -e "\n\n"; cat id_rsa.pub; echo -e "\n\n") > keytext.txt
```

Now that we have a public key stored on the database by the name of “crackit”, we can save the value stored in crackit as a file; for the exploit to work this needs to be stored as authorized_hosts so that we can connect over SSH as the Redis user.


We can now connect over SSH using the private key that is paired to the public key we uploaded.

```bash
$ ssh -i id_rsa redis@10.10.10.160
```

I uploaded an enumeration script to gather data for me.

```bash
$ scp -i id_rsa LinEnum.sh redis@10.10.10.160:/tmp
```

SSH back in as the redis user and execute LinEnum.sh.


```bash
$ chmod +x LinEnum.sh && ./LinEnum.sh
```

Carefully reviewing the output we can see that there is a backup of a private key in the filesystem, which we can read, however it is encrypted.

We can crack encrypted SSH keys with JohnTheRipper but first we have to put it in the John format using SSH2John:

I first copied the SSH key into a new directory called matt, and named the SSH key id_rsa.

I located SSH2John using “locate ssh2john”.

```bash
$ locate ssh2john
$ python /usr/share/john/ssh2john.py id_rsa > johnkey
$ locate rockyou.txt
$ john --wordlist=/usr/share/wordlists/rockyou.txt johnkey
```

It turns out Matt is using this password to authenticate on the machine locally. We can now “su matt” and use “computer2008” to access his files on the machine. But the big kicker here is the password reuse. Matt is a user on the Webmin portal on port 10000. To access this we have to add “Postman” to our /etc/hosts file like so:

We can now access the Webmin console over port 10000 providing the credentials Matt:computer2008 to login.
The Webmin console is vulnerable to a publicly-available authenticated remote code execution which we can find on Metasploit. Since the Webmin console is running as the root user. We gain root-level access upon popping the shell.

Sources:
1) https://packetstormsecurity.com/files/134200/Redis-Remote-Command-Execution.html


