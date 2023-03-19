---
title: Cómo conectar con HTB (VPN)
author: skkkajenen
date: 2023-03-19 11:00:00 +0100
categories: [Hacking, Varios]  # Categoría principal , categoría secundaria
tags: [htb,vpn,startingpoing]     # TAG names should always be lowercase
image: /htb_connect_vpn_title.png   # Mantener la proporción 1.91 : 1
img_path: /posts/connect_vpn
published: false
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
: Esta será la opción elegida. Requiere (como hemos adelantado) un sistema Linux corriendo sobre una máquina virtual en la que necesitaremos tener instalada una versión de **OpenVPN** (OpenSource). Puedes visitar su web [aquí](openvpn)

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

El protocolo, a priori, elegiremos el **UDP 1337**. Si dicho protocolo nos presenta problemas entonces probaremos con TCP 443



***

(... continuará)



[htb]: https://www.hackthebox.com
[vmware]: https://www.vmware.com
[virtualbox]: https://www.virtualbox.org
[openvpn]: https://openvpn.net