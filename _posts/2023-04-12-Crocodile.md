---
title: Crocodile - Starting Point
author: skkkajenen
date: 2023-04-12 20:00:00 +0200
categories: [Hacking, Writeups]  # Categoría principal , categoría secundaria
tags: [htb,linux,startingpoint,ftp,apache]     # TAG names should always be lowercase
image: /htb_crocodile.png   # Mantener la proporción 1.91 : 1
img_path: /posts/crocodile
published: true
---

# Introducción

Seguimos con la serie de **Starting Point** de [HackTheBox][htb]. En esta máquina **Linux** combinaremos la exploración a través de dos servicios (**FTP** y **WEB**) para acabar obteniendo credenciales y así acceder a un panel a priori no visible. Aquí se empieza a uscar el desarrollar habilidades de **OSINT**.

En estas máquinas el nivel de detalle del presente **write-up** es mayor ya que se pretende dar un **'paso a paso'** para principiantes más que una simple guía para un usuario algo avanzado. Además, durante la explotación de la máquina tendremos que ir respondiendo a preguntas que ayudarán en el camino hasta la consecución. En este caso son **7** las preguntas a responder. y éstas se añadirán al final del **write-up**.

<div>
    <img src="https://img.shields.io/badge/Sistema-Linux-orange" alt="Sistema-Linux-orange" />
    <img src="https://img.shields.io/badge/Dificultad-VeryEasy-brightgreen" alt="Dificultad-VeryEasy-brightgreen" />
</div>

# Resolución

## Conexión al VPN de HTB

Comenzamos estableciendo conexión por VPN con la plataforma [HackTheBox][htb].
Una vez tengamos IP asignada de la VPN de HTB procedemos a ir a la web de Starting Point para iniciar la máquina **Appointment**.
Puede trascurrir hasta 2 minutos en reconocer la web que estamos conectados por VPN, llegando en ocasiones a tener que refrescar la página `F5`

> Si tienes dudas sobre conectar con la **VPN** de [HackTheBox][htb] te lo explico [**aquí**][vpn]
{: .prompt-tip}

## Iniciar la máquina

Iniciamos la máquina pulsando sobre el botón

![Botón Spawn Machine](spawn_machine.png)

Tras unos cuantos segundos (en ocasiones puede tardar más) se habrá iniciado correctamente la máquina y nos mostrará la dirección ip de la *máquina víctima*. En nuestro caso será **10.129.143.21**

### Settarget

Haciendo uso de la herramienta previamente establecida en la ZSHRC llamada '*settarget*' dejaremos anotada la IP y nombre de la *máquina víctima* en la parte superior de la Polybar para tener así mayor eficacia. La sintaxis es: `settarget (IP_VICTIMA) (NOMBRE_MAQUINA_VICTIMA)`

~~~bash
settarget 10.129.143.21 Crocodile
~~~

![settarget](settarget.png)

De esta manera en la Polybar superior tendremos siempre bien referenciada la *máquina víctima*

### Ping

Comprobamos que efectivamente tenemos visible la *máquina víctima* lanzando un `ping`sobre ella. Con el parámetro `-c 1` le indicamos que tan solo haga un ping. No es necesario más.

~~~bash
ping -c 1 10.129.143.21
~~~

Como resultado obtenemos lo siguiente de lo cual destacatemos el TTL y el número de paquetes enviados / recibidos:

![Ping](ping.png)

* `TTL = 63`
: Este valor (cercano a 64) indica que nos encontramos ante una máquina **Linux**

> Por defecto valores de `TTL` en torno a ***64*** indica que la máquina es **Linux** mientras que valores de ***128*** indicaría que la máquina es **Windows**
{: .prompt-info}

* `1 packets transmitted, 1 received`
: Indica que el paquete enviado ha sido recibido con éxito por lo tanto => La *máquina víctima* está encendida y tenemos conexión con ella

## Reconocimiento con NMAP (PortDiscovery)

>[NMAP][nmap] es una herramienta ampliamente extendida en el reconocimiento de redes así como en el escaneo de puertos, hosts y servicios

Ejecutaremos la instrucción básica de reconocimiento con los parámetros que se explican a continuación:

~~~bash
nmap -p- -sS --min-rate 5000 -n -Pn -vvv --open 10.129.143.21 -oG allPorts
~~~

* `-p-`
: Indicamos que queremos analizar el rango completo de puertos. Del **1** al **65535**

* `-sS`
: Especificación del tipo de escaneo. En este caso SYN Scan que es rápido a la vez que sigiloso ya que no termina de establecer la conexión y de tal manera puede pasar desapercibido en los logs de conexión

* `--min-rate 5000`
: Acelera el proceso de escane ya que no tramita paquetes cuya velocidad sea inferior a 5000 paquetes por segundo. Esto puede hacer menos sigiloso el escaneo pero más rápido

* `-n`
: Indica que *no* queremos aplicar resolución *DNS*. Más velocidad de escaneo

* `-Pn`
: Escaneo de puertos incluso si el host no responde a los ping. Ignora si el host está o no online

* `-vvv`
: *Triple verbose* para que los resultados que vaya obteniendo los vaya mostrando por pantalla sin necesidad de acabar el escaneo

* `--open`
: Un puerto principalmente puede estar **Abierto**, **Cerrado** o **Filtrado**. En un principio nos centraremos en aquellos que están **Abiertos**

* `-oG allPorts`
: Salida del resultado en un fichero llamado ***allPorts*** en formato *'grepable'* lo cual aportará ventajas a la hora de tratarlo posteriormente

El resultado del comando ejecutado sería el siguiente:

![NMAP sS](nmap_ss.png)

Observamos que tiene unicamente un **puerto abierto**, se trata del **puerto 80**.

* `Puerto 80 => HTTP`
: El servicio **HTTP**, acrónimo de **HyperText Transfer Protocol** es el protocolo de comunicación que permite las transferencias de información a través de archivos (XML, HTML…) en la World Wide Web. Indica que podremos conectarnos usando un navegador para ver si hay una web corriendo en este puerto

* `Puerto 21 => FTP`
: El servicio **FTP**, acrónimo de **File Transfer Protocol** es el que por defecto funciona sobre el **puerto 21** (abierto en esta máquina). El servicio **FTP** es usado para realizar transferencia de ficheros entre una máquina orígen y una máquina destino conectadas ambas por red.

> **FTP** a pesar de tener posibilidad de proteger el acceso con usuario y contraseña no se podría decir que sea especialmente seguro. Especialmente prestaremos atención a si el usuario **anonymous** está habilitado, en cuyo caso podríamos acceder sin necesidad de proporcionar contraseña
{: .prompt-warning}

## Puertos abiertos (versiones)

Sobre aquellos puertos abiertos en la *máquina víctima* lanzaremos un comando de NMAP enfocado al descubrimiento de versiones de los servicios que estén corriendo sobre los puertos abiertos

~~~bash
nmap -sCV -p21,80 10.129.143.21 -oN targeted
~~~

* `-sCV` (también se puede poner como `-sC -sV`)
: Realiza un escaneo en busca de versiones y comparando con una base de datos de servicios conocidos

* `-p21,80`
: Indica que se analizará en esta caso solamente el **puerto 80** y el **puerto 21**

* `-oN targeted`
: Salida del resultado en un fichero llamado ***targeted*** en formato normal de NMAP

El resultado del comando ejecutado sería el siguiente:

![NMAP sCV](nmap_scv.png)

Este escaneo ha aportado información para la consecución de la máquina:

* `Puerto 80 -> Apache httpd 2.4.41 ((Ubuntu))`
: Esta es la versión del servidor Apache corriendo en la *máquina víctima*. Lo tendremos en cuenta más adelante tanto para la resolución de las preguntas como en la búsqueda de posibles vulnerabilidades.

* `Puerto 21 -> Anonymous FTP login allowed`
: En este caso el FTP tiene habilitado la conexión de manera anónima lo cual va a ser el punto de entrada para comenzar a explorar la *máquina víctima*

## Reconocimiento del Puerto 21 - FTP

* Conectamos de manera anónima usando `ftp 10.129.143.21` y proporcionando `anonymous` como nombre.

* Listamos el contenido disponible en el servicio FTP con `ls` (también válido `dir`). Encontramos dos ficheros.

* Procedemos a descargar los dos ficheros con `get allowed.userlist` y `get allowed.userlist.passwd` al directorio de trabajo

* Cerramos conexión con `exit`

![FTP](ftp.png)

Listamos el contenido de ambos ficheros con `cat allow*`

![Files](files.png)

A priori, podemos pensar que se trata de usuarios válidos a nivel de sistema y sus correspondientes contraseñas

## Reconocimiento del Puerto 80 - WEB

Entramos en la web desde el navegador (http://10.129.143.21). Vemos una landing page que a priori no parece aportar información relevante

Con `whatweb` vemos la información que obtenemos:

![whatweb](whatweb.png)

Trataremos de localizar ficheros PHP usando la herramienta **gobuster**

### Fuzzing de ficheros y directorios con Gobuster

Usaremos la herramienta **[Gobuster][gobuster]** para localizar ficheros en este caso con extensión **php**

~~~bash
gobuster dir -u http://10.129.143.21/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20 -x php
~~~

* `-u http://10.129.143.21/`
: `-u` indica la URL sobre la que reconocer directorios

* `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`
: `-w` indica la wordlist a utilizar

* `-t 20`
: `-t` indica el número de hilos a utilizar (**por defecto 10** si no se indica otro valor)

* `-x php`
: Indicamos que queremos localizar ficheros con extensión **php**

![gobuster](gobuster.png)

Nos encuentra con código de estado 200 (OK) un par de ficheros:

* /login.php

* /config.php

Volvemos al navegador para ver la pinta que tiene ahora ese *login* => `http://10.129.143.21/login.php`

Vemos un simple formulario de login

![login](login.png)

Probaremos las distintas parejas de usuario y contraseña que habíamos obtenido en el FTP.

|User|Pass|Resultado|
|---|---|---|
|aron|root|❌|
|pwnmeow|Supersecretpassword1|❌|
|egolisticalsw|@BaASD&9032123sADS|❌|
|admin|rKXM59ESxesUFHAd|✅|

La última de los user-pass nos da acceso a un dashboard que contiene la flag
![dashboard](dashboard.png)

## Flag

![Root Flag](https://img.shields.io/badge/RootFlag-c7110277ac44d78b6a9fff2232434d16-red)

## Tasks

Estas **Tasks** hacen las funciones de *mini-flags* y ayudan guiando la resolución de la máquina:

1. What Nmap scanning switch employs the use of default scripts during a scan?
: `sC`

2. What service version is found to be running on port 21?
: `vsftpd 3.0.3`

3. What FTP code is returned to us for the "Anonymous FTP login allowed" message?
: `230`

4. After connecting to the FTP server using the ftp client, what username do we provide when prompted to log in anonymously?
: `anonymous`

5. After connecting to the FTP server anonymously, what command can we use to download the files we find on the FTP server?
: `get`

6. What is one of the higher-privilege sounding usernames in 'allowed.userlist' that we download from the FTP server?
: `admin`

7. What version of Apache HTTP Server is running on the target host?
: `Apache httpd 2.4.41`

8. What switch can we use with Gobuster to specify we are looking for specific filetypes?
: `-x`

9. Which PHP file can we identify with directory brute force that will provide the opportunity to authenticate to the web service?
: `login.php`

[htb]: https://www.hackthebox.com
[vpn]: https://www.skkkajenen.com/posts/Connect-VPN-HTB/
[nmap]: https://nmap.org
[gobuster]: https://github.com/OJ/gobuster