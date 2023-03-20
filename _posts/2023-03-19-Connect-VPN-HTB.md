---
title: Cómo conectar con HTB (VPN)
author: skkkajenen
date: 2023-03-19 21:30:00 +0100
categories: [Hacking, Varios]  # Categoría principal , categoría secundaria
tags: [htb,vpn,startingpoing]     # TAG names should always be lowercase
image: /htb_connect_vpn_title.png   # Mantener la proporción 1.91 : 1
img_path: /posts/connect_vpn
published: true
---

En esta guía te explico paso a paso y sin ninguna complicación los pasos que tienes que seguir para conectarte a la VPN de [Hack The Box][htb] y poder hacer uso de la plataforma.

> Conviene conocer que para la plataforma de *Starting Point*, para el acceso a los laboratorios donde están las *máquinas* principales y para los *Challenges de Temporada* se usan **diferentes ficheros de configuración** pero todos se obtienen y configuran de la misma manera
{: .prompt-warning }

> Elige el fichero de configuración que va a emplear. En este ejemplo nos centraremos en el **Starting Point**
{: .prompt-info}

> Esta guía tutorial está pensada para el caso que vayas a trabajar con una máquina virtual ([VMware][vmware] o [VirtualBox][virtualbox]) que es lo habitual para trabajar en estos laboratorios.
{: .prompt-info}

# Guía paso a paso

Entramos a la web de [Hack The Box][htb] y localizamos en la zona superior derecha el botón **Connect to HTB**

![Paso_1](1.png)

Se nos desplegará un menú similar a este

![Paso_2](2.png)

Damos click en el grupo que nos interese, en este caso de ejemplo elegimos **Starting Point** y veremos un menú en el que tendremos **dos opciones**. Te detallo cuales son:

![Paso_3](3.png)

* OpenVPN ⭐️
: Esta será la opción elegida. Requiere (como hemos adelantado) un sistema Linux corriendo sobre una máquina virtual en la que necesitaremos tener instalada una versión de **OpenVPN** (OpenSource). Puedes visitar su web [aquí][openvpn]

* Pwnbox
: Es un entorno de Parrot Linux ejecutado desde la propia web de [HTB][htb] con la que poder desarrollar el laboratorio. Tiene un **uso limitado** para los usuarios **Free** y hasta 24h de uso para los que son **VIP**

Ya dentro de OpenVPN tendremos para elegir la región y el servidor

![Paso_4](4.png)

Las opciones que se nos despliegan las opciones de:
* VPN Access
    * EU - Starting Point ⭐️
    * US - Starting Point

![Paso_5](5.png)

Seleccionamos EU para el caso de Europa, y US en cualquier otro caso

En cuanto al VPN Server elegimos el que menos usuarios:

![Paso_6](6.png)

> En función de la ocupación existente, [Hack The Box][htb] pondrá a disposición de los usuarios 1, 2 o más servidores para cada una de las áreas (Starting Point, etc.)
{: .prompt-info}

El protocolo, a priori, elegiremos el **UDP 1337**. Si dicho protocolo nos presenta problemas entonces probaremos con TCP 443.

Daremos click en el botón de **Download VPN**

![Paso_7](7.png)

Se nos descargará un fichero con extensión **.ovpn** compuesto por el área para la que descargamos el fichero (en nuestro caso **starting_point**) seguido de un guión bajo **'_'** y nuestro nombre de usuario en HTB. Algo tal que así.

![Paso_8](8.png)

Este fichero tiene todo lo necesario para conectarnos directamente usando [OpenVPN][openvpn]

Ya que hemos descargado el fichero **.ovpn** ahora tendremos que hacerlo llegar a nuestra máquina virtual. Este paso se puede realizar de múltiples maneras:
* Copiar el fichero en una **memoria USB** y copiarlo a la máquina virtual
* **Copiar directamente el fichero** a la máquina virtual (si tenemos esa opción habilitada)
* **Mandar el fichero a nuestro correo** o nuestra nube para descargarlo posteriormente en la VM
* Levantar un servidor **HTTP** con **Python** y descargarlo desde la **VM**

Explicaremos este último caso (servidor con Python) ya que puede resultar más rápido y cómodo.

> Deberemos tener instalado Python3 en nuestro equipo. Sino lo tienes puedes descargarlo desde [**aquí**][python]
{: .prompt-info}

Para MAC: Abrimos Terminal y ejecutamos los siguientes comandos:
1. `cd ~/Downloads`
    : Suponiendo que se ha descargado a la ubicación por defecto, si fuera otra la reemplazaríamos donde pone `Downloads`
2. `python3 -m http.server 8080`
    : Con esto servimos todo el contenido de la carpeta donde estemos situados a través del **puerto 8080** por el **protocolo HTTP**

![Paso_9](9.png)

Ahora nos cambiamos a nuestra **máquina virtual** y nos movemos al directorio donde queramos descargar el fichero. Para ello ejecutaremos:

```python
wget http://(ip_máquina_host):(puerto)/(nombre_fichero_ovpn)
#
# ejemplo:
wget http://192.168.1.67:8080/starting_point_Skkkajenen.ovpn
```

Si todo va bien se descargará de manera casi instantánea y veremos esto:

![Paso_10](10.png)

Y aquí concluiría el proceso de configuración de la **VPN** de [HTB][htb] en nuestra **máquina virtual** ¿Sencillo? Pues ahora toca hacer una conexión para comprobar que efectivamente todo está como debería.

En el mismo directorio donde tenemos descargado el fichero **.ovpn** ejecutamos lo siguiente:

![Paso_11](11.png)

> La inclusión del comando **sudo** aplica privilegios a la ejecución de la conexión
{: prompt-info}

Comenzará la conexión que puede tomar desde unos cuantos segundos hasta un par de minutos. Si la conexión tarda más convendría acabar con el proceso (`ctrl + c`) y revisar.

![Paso_12](12.png)

También en la interfaz de [Hack The Box][htb] aparecerá como **Conectado**

![Paso_13](13.png)

Y a partir de aquí ya podemos comenzar a trabajar con las máquinas 😉

> Para terminar la conexión ejecutamos (`ctrl + c`)
{: prompt-tip}


[htb]: https://www.hackthebox.com
[vmware]: https://www.vmware.com
[virtualbox]: https://www.virtualbox.org
[openvpn]: https://openvpn.net
[python]: https://www.python.org/downloads/
