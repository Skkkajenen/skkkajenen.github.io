---
title: Appointment - Starting Point
author: skkkajenen
date: 2023-04-03 20:00:00 +0200
categories: [Hacking, Writeups]  # Categoría principal , categoría secundaria
tags: [htb,linux,startingpoint,sql,sql-injection,php]     # TAG names should always be lowercase
description: Writeup de Appointment en HackTheBox Starting Point. Explota una vulnerabilidad de SQL Injection en un formulario de login con Apache en Linux.
image: /htb_appointment.png   # Mantener la proporción 1.91 : 1
img_path: /posts/appointment
published: true
---

# Introducción

**Appointment** es la quinta de las máquinas de **Starting Point** en [HackTheBox][htb] y la primera del segundo bloque que se nos habilita tras haber completado con éxito las primeras cuatro máquinas. Es una máquina **Linux** en la que está corriendo un servidor **Apache** con un formulario de login vulnerable a **SQL Injection** el cual tendremos que comprometer para llegar a obtener la **flag**. Además, durante la explotación de la máquina tendremos que ir respondiendo a preguntas que ayudarán en el camino hasta la consecución. En este caso son **10** las preguntas a responder.

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

Tras unos cuantos segundos (en ocasiones puede tardar más) se habrá iniciado correctamente la máquina y nos mostrará la dirección ip de la *máquina víctima*. En nuestro caso será **10.129.29.33**

### Settarget

Haciendo uso de la herramienta previamente establecida en la ZSHRC llamada '*settarget*' dejaremos anotada la IP y nombre de la *máquina víctima* en la parte superior de la Polybar para tener así mayor eficacia. La sintaxis es: `settarget (IP_VICTIMA) (NOMBRE_MAQUINA_VICTIMA)`

~~~bash
settarget 10.129.29.33 Appointment
~~~

![settarget](settarget.png)

De esta manera en la Polybar superior tendremos siempre bien referenciada la *máquina víctima*

### Ping

Comprobamos que efectivamente tenemos visible la *máquina víctima* lanzando un `ping`sobre ella. Con el parámetro `-c 1` le indicamos que tan solo haga un ping. No es necesario más.

~~~bash
ping -c 1 10.129.61.206
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
nmap -p- -sS --min-rate 5000 -n -Pn -vvv --open 10.129.29.33 -oG allPorts
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

## Puertos abiertos (versiones)

Sobre aquellos puertos abiertos en la *máquina víctima* lanzaremos un comando de NMAP enfocado al descubrimiento de versiones de los servicios que estén corriendo sobre los puertos abiertos

~~~bash
nmap -sCV -p80 10.129.29.33 -oN targeted
~~~

* `-sCV` (también se puede poner como `-sC -sV`)
: Realiza un escaneo en busca de versiones y comparando con una base de datos de servicios conocidos

* `-p80`
: Indica que se analizará en esta caso únicamente el **puerto 80**

* `-oN targeted`
: Salida del resultado en un fichero llamado ***targeted*** en formato normal de NMAP

El resultado del comando ejecutado sería el siguiente:

![NMAP sCV](nmap_scv.png)

Este escaneo ha aportado información para la consecución de la máquina:

* `Apache httpd 2.4.38 ((Debian))`
: Esta es la versión del servidor Apache corriendo en la *máquina víctima*. Lo tendremos en cuenta más adelante tanto para la resolución de las preguntas como en la búsqueda de posibles vulnerabilidades.

## Reconocimiento del Puerto 80

### Whatweb

Haciendo uso de la herramienta **whatweb** exploramos la dirección IP obteniendo el siguiente resultado:

![Reconocimiento con Whatweb](whatweb.png)

* No hay información que a priori sea sustancioso

* El servidor es un **Apache** en una versión 2.4.38 como ya habíamos visto con **NMAP**

* Tiene corriendo una versión de **jQuery** relativamente reciente frente a vulnerabilidades

### Navegador

Tratamos de acceder a la web desde la dirección IP dada: `http://10.129.29.33`. Esta es ahora la apariencia que tiene la web:

![Web](web.png)

Observamos un típico formulario de entrada de **login** y **password**. El botón de `Forgot Password?` no funciona

### Wappalyzer

Con la extensión **Wappalyzer** podemos analizar desde el navegador los servicios y plugin que corren en la web de manera similar que con **whatweb** lo hacíamos desde línea de comandos

Y estas son las tecnologías que Wappalyzer ha reconocido:

![Wappalyzer](wappalyzer.png)

## Explorando directorios y ficheros con GoBuster

Haciendo uso de la herramienta GoBuster exploraremos posibles directorios y ficheros accesibles en el servidor. Usaremos la lista de directory-list-2.2-medium que recoge un total de 220K líneas de las más probables.
Para ello lanzaremos el siguiente comando:

~~~bash
gobuster dir -u http://10.129.29.33/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 20
~~~

* `-u http://10.129.29.33/`
: `-u` indica la URL sobre la que reconocer directorios

* `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt`
: `-w` indica la wordlist a utilizar

* `-t 20`
: `-t` indica el número de hilos a utilizar (**por defecto 10** si no se indica otro valor)

![GoBuster](gobuster.png)

El resultado no revela nada especialmente de interés.

## Inyección SQL

Ya que vemos el formulario para un **usuario** y **contraseña** damos por supuesto de que tiene que haber por detrás corriendo una **base de datos** con alguna tabla que contenga la información para validar los valores introducidos por el usuario.

> Una inyección SQL consiste en jugar con el campo disponible al usuario para introducir información para modificar el comportamiento de la query que la web ejecutará para traer información de la base de datos
{: .prompt-info}

Probamos con un listado de inyecciones básicas:

* `admin' --` En el campo **User**. No funciona
* `admin' #` En el campo **User**. No funciona
* `admin'/*` En el campo **User**. No funciona
* `' or 1=1--` En el campo **Password**. Funciona !! 🎉
* `' or 1=1#`
* `' or 1=1/*`
* `') or '1'='1--`
* `') or ('1'='1--`

![Flag](flag.png)

### Explicación de la inyección SQL => `' or 1=1--`

Aplicado sobre el campo password, aplicando un `'` al principio se puede decir que cierra con un *vacío* el campo del password y le une un `or` con una función que es siempre `true` como es el `1=1`. Finalmente se añaden un doble guión `--` al final para que no interprete lo que haya predefinido después.

# Flag
![Root Flag](https://img.shields.io/badge/RootFlag-e3d0796d002a446c0e622226f42e9672-red)

## Tasks

Estas **Tasks** hacen las funciones de *mini-flags* y ayudan guiando la resolución de la máquina:

1. What does the acronym SQL stand for?
: `Structured Query Language`

2. What is one of the most common type of SQL vulnerabilities?
: `SQL injection`

3. What does PII stand for?
: `Personally Identifiable Information`

4. What is the 2021 OWASP Top 10 classification for this vulnerability?
: `A03:2021-Injection`

5. What does Nmap report as the service and version that are running on port 80 of the target?
: `Apache httpd 2.4.38 ((Debian))`

6. What is the standard port used for the HTTPS protocol?
: `443`

7. What is a folder called in web-application terminology?
: `directory`

8. What is the HTTP response code is given for 'Not Found' errors?
: `404`

9. Gobuster is one tool used to brute force directories on a webserver. What switch do we use with Gobuster to specify we're looking to discover directories, and not subdomains?
: `dir`

10. What single character can be used to comment out the rest of a line in MySQL?
: `#`

11. If user input is not handled carefully, it could be interpreted as a comment. Use a comment to login as admin without knowing the password. What is the first word on the webpage returned?
: `Congratulations`

[htb]: https://www.hackthebox.com
[vpn]: https://www.skkkajenen.com/posts/Connect-VPN-HTB/
[nmap]: https://nmap.org