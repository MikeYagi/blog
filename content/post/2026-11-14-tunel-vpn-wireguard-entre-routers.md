---
title: "Túnel VPN WireGuard entre routers MikroTik para una estación remota"
subtitle: "Cómo unir dos casas como si un cable de red las conectara, para operar el Hermes Lite 2 a distancia sin TeamViewer ni AnyDesk."
date: 2026-11-14T12:22:37.364Z
author: "EA1PYM"
layout: "post"
url: "/p/tunel-vpn-wireguard-entre-routers/"
image: "img/post-bg.jpg"
tags: ["wireguard", "mikrotik", "vpn", "estacion-remota", "hermes-lite-2"]
categories: ["Redes"]
---
## Introducción

En la radioafición moderna, la distancia entre el operador y la antena ya no es necesariamente física, sino una distancia de red. Quizá por eso, tras la publicación del artículo “Estación remota basada en Hermes Lite 2”, recibí una decena de mensajes de colegas intrigados por ese hilo digital que unía el transmisor con su operador. Y la verdad es que todos coincidían en algo: entendían el concepto, les interesaba, pero les faltaban los pasos, las claves para llevarlo a cabo.

Este texto nace de esas conversaciones y de ese pulso entre curiosidad e ingenio que tanto define a los radioaficionados. Porque no se trata simplemente de “ver el equipo a distancia” o de manejar un ordenador remoto mediante herramientas como TeamViewer o AnyDesk, soluciones válidas para otros contextos, pero inadecuadas para una estación de radio. Ese enfoque introduce latencias innecesarias, un procesado de audio errático y una dependencia excesiva de capas de software ajenas al propio sistema de radio. Lo que planteo aquí es una arquitectura más sólida, basada en una conexión de red punto a punto, transparente para el software de radio y capaz de transportar los datos de forma nativa y natural.

El propósito es sencillo y ambicioso al mismo tiempo. Mostrar cómo unir dos viviendas mediante un túnel VPN WireGuard utilizando rúters MikroTik, de modo que el flujo de datos viaje con la misma naturalidad que si un cable de red Cat-6e uniera directamente ambos extremos. Un montaje realista, económico y elegante, pensado para quienes no quieren sustituir el rúter de fibra de su operador, sino superponer sobre él una capa de inteligencia flexible y discreta.

## Objetivo y topología del montaje

Para que esta guía sea clara, conviene describir qué pretendemos construir y cómo estarán conectados los elementos. Tendremos dos localizaciones físicas:

- por un lado, la estación remota, donde se encuentra el equipo de radio (por ejemplo, el Hermes Lite 2) y las antenas;

- y por otro, el QTH del operador, que es donde estará el ordenador con Thetis desde el que se controla todo.

En cada lugar ya existe un router de fibra proporcionado por el operador de internet, que no vamos a tocar. Sobre ese router colocaremos un MikroTik hEX S lite, que actuará como un pequeño cerebro auxiliar encargado de crear y mantener el túnel VPN. Este MikroTik no gestionará el tráfico general de la vivienda, sino únicamente el de la estación remota, de modo que todo el montaje es reversible y no interfiere con la red doméstica.

**Nota:** En este artículo nos centraremos exclusivamente en el caso en que ambas localizaciones dispongan de conexión fija a internet (fibra o ADSL). No abordaremos por ahora escenarios en los que una de las sedes —habitualmente la estación remota— utilice conectividad móvil 4G o 5G. Estas configuraciones presentan particularidades como CG-NAT, direcciones IP no enrutables o latencias más variables, y requieren soluciones adicionales (túneles inversos, servidores intermedios, etc.), que se tratarán en un artículo posterior.

### Elección del equipo Mikrotik

A la hora de elegir modelo, la opción más equilibrada en precio y prestaciones es el MikroTik hEX S lite (RB750Gr3). Conviene no confundirlo con el RB750r2, una versión anterior con menor potencia de CPU y sin aceleración por hardware para cifrado. El modelo Gr3 incorpora un procesador de doble núcleo a 880 MHz y 256 MB de RAM, ofreciendo un rendimiento muy solvente en tareas de VPN: puede manejar tráfico WireGuard sostenido del orden de 70–100 Mbps, más que suficiente para una estación remota con audio y control simultáneos. Además, es un equipo robusto y silencioso, pensado para funcionar 24/7.

![Router MikroTik hEX](https://substack-post-media.s3.amazonaws.com/public/images/9d5ea761-d33e-496e-9868-6511c8e7b020_1384x914.jpeg)
*Router MikroTik hEX*

### Arquitectura general

En la estación remota, el esquema será así:

- El router de fibra mantiene la conexión a internet de la vivienda (con IP, por ejemplo, 192.168.1.1).

- El MikroTik se conecta con un cable Ethernet Cat6 a una de las bocas LAN del router y adopta una dirección dentro de esa red, digamos 192.168.1.10.

- A continuación, el MikroTik crea su propia subred interna, separada, donde conectaremos el Hermes Lite 2 y cualquier otro dispositivo auxiliar. Esa subred podría ser 192.168.10.0/24, con el propio MikroTik actuando como puerta de enlace en la 192.168.10.1.

- El Hermes Lite 2 tendrá, por ejemplo, la dirección 192.168.10.10.

De este modo, el tráfico de la estación (I/Q, CAT, audio, etc.) se aísla en una red privada, pero el MikroTik puede enviarlo cifrado hacia fuera a través del router principal de fibra.

En el QTH del operador haremos algo simétrico, pero obviamente con direcciones diferentes:

- El router de fibra habitual de la vivienda (con IP 192.168.2.1) seguirá dando servicio a los dispositivos domésticos.

- El MikroTik local se conectará también a ese router, con dirección 192.168.2.10, y creará su propia subred —por ejemplo, 192.168.20.0/24— donde estará el ordenador de control, con IP 192.168.20.20.

Una vez configurados ambos extremos, el objetivo será que el túnel WireGuard una directamente los dos MikroTik, formando un canal cifrado punto a punto con direcciones internas, por ejemplo 10.0.0.1 en el lado remoto y 10.0.0.2 en el lado del operador.

Gracias a ese enlace, el PC del operador podrá comunicarse con el Hermes Lite 2 (y con cualquier otro dispositivo conectado en la estación remota) de manera completamente transparente, como si ambos estuvieran en la misma red local.

## Configurando el Router Mikrotik

Antes de teclear comandos, conviene hacer un breve repaso conceptual. WireGuard no es simplemente otra VPN: es una revisión moderna del concepto. Frente a soluciones veteranas como OpenVPN o IPsec, apuesta por la simplicidad y la solidez criptográfica. Su código es ligero y eficiente: menos complejidad implica menos puntos de fallo.

Cada extremo del túnel, lo que WireGuard denomina peer, dispone de un par de claves: una privada, que nunca sale del equipo, y una pública, que se comparte con el otro extremo. No hay contraseñas ni certificados: dos dispositivos que se reconocen mediante matemáticas. En la práctica, el túnel actúa como un coaxial digital: por él viajan comandos CAT, audio o tramas I/Q encapsulados en datagramas UDP cifrados. Para el sistema operativo o el software de radio, la diferencia es inexistente: comunicarse con el Hermes Lite 2 será tan natural como si estuviera conectado en la misma mesa.

### Preparación del entorno y acceso al MikroTik

El primer paso es establecer una sesión con el router. Si el MikroTik es nuevo, vendrá con configuración de fábrica y una dirección IP por defecto (192.168.88.1). Para acceder por primera vez, habrá que conectar el ordenador directamente por cable Ethernet y asignarle temporalmente una IP del mismo rango, por ejemplo 192.168.88.2. Una vez dentro del router (por SSH o interfaz web), podremos cambiar su IP a la que vayamos a usar en la red, por ejemplo 192.168.1.10. Al finalizar, conviene recordar devolver el ordenador a su configuración original (DHCP o la IP previa).

Con el MikroTik conectado al router de fibra y al ordenador dentro de la misma red local —por ejemplo, 192.168.1.10 para el MikroTik y 192.168.1.20 para el PC— nos comunicaremos con él mediante SSH, que permite abrir una consola segura directamente en el dispositivo.

En Windows:

1.  Pulsa la tecla Inicio y escribe cmd.

2.  Se abrirá una ventana negra: es el símbolo del sistema.

3.  Escribe el siguiente comando y pulsa Enter:

&nbsp;

    ssh admin@192.168.1.10

En Mac:

1.  Abre la aplicación Terminal

2.  Escribe el mismo comando indicado anteriormente para Windows.

3.  Acepta la conexión con indicando Yes.

El usuario por defecto en MikroTik es admin, y en las unidades nuevas no tiene contraseña. Una vez conectado, se verá un prompt similar a esto:

    [admin@MikroTik] >

Ahora ya estamos “dentro” del router y podremos configurarlo a nuestro gusto. Para salir en cualquier momento, escribe quit.

### Conectando al MikroTik

Vamos a definir las direcciones que usará el MikroTik para hablar con el router de fibra y con los dispositivos que cuelguen de él (por ejemplo, el Hermes Lite 2). En la estación remota, escribimos lo siguiente:

    /ip address add address=192.168.1.10/24 interface=ether1 comment="Conexión al router de fibra"
    /ip address add address=192.168.10.1/24 interface=ether2 comment="Red interna HL2"
    /ip pool add name=dhcp_pool10 ranges=192.168.10.10-192.168.10.100
    /ip dhcp-server add name=dhcp10 interface=ether2 address-pool=dhcp_pool10 disabled=no
    /ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1

¿Qué estamos haciendo aquí? En los routers MikroTik, los conectores Ethernet se identifican como ether1, ether2, ether3, etc. Cada uno es una boca física RJ45 del router, igual que los puertos de cualquier switch doméstico. Podemos pensar que cada ether es una “puerta” que conecta el MikroTik con una red diferente.

- ether1 es la puerta que mira hacia el router de fibra, el que nos da acceso a Internet. Por ella saldrán los datos cifrados del túnel.

- ether2 es la puerta que conecta con la red de la estación: ahí enchufaremos el Hermes Lite 2 y cualquier otro dispositivo local (un acoplador automático, un medidor remoto, etc.).

- Las líneas que empiezan por /ip address add asignan las direcciones IP correspondientes a cada una de esas interfaces. Así, el MikroTik tendrá la 192.168.1.10 en la red del router de fibra y la 192.168.10.1 en su propia red interna.

El siguiente bloque crea un pequeño servidor DHCP, que no es más que un mecanismo para repartir direcciones IP automáticamente. Gracias a eso, cualquier dispositivo que conectemos a la boca ether2 recibirá su dirección sin que tengamos que configurarla manualmente.

### Creando la interfaz Wireguard en el router Mikrotik

WireGuard actúa como una interfaz más, igual que si fuera otro puerto del router, solo que este canal no es físico: es un enlace cifrado que viajará por Internet. Para crearla, escribe:

    /interface wireguard add name=wg0 listen-port=51820

Esto crea la interfaz y le asigna un puerto de escucha, el 51820, que es el valor por defecto en WireGuard. Ahora necesitamos saber las claves criptográficas que ha generado el router para este túnel:

    /interface wireguard print detail

Verás algo parecido a esto:

    0 name="wg0" mtu=1420 listen-port=51820 private-key="x..." public-key="y..."

Apunta la clave pública, porque la necesitaremos en el otro extremo. Guárdala en un archivo txt o Word. Lo siguiente es asignar una dirección IP interna al túnel:

    /ip address add address=10.0.0.1/24 interface=wg0

### Configurando el peer remoto

Cada extremo del túnel debe conocer la clave pública del otro. Supongamos que en el lado del operador la clave pública es ABCD1234…. (Mucho ojo con los ceros y las oes, las eles y las íes, si se tiene dudas, se puede copiar la clave y acudir a alguna web identificadora de caracteres para confirmarlo)

    /interface wireguard peers add interface=wg0 public-key="ABCD1234..." allowed-address=10.0.0.2/32,192.168.20.0/24 persistent-keepalive=25

Con esto, el MikroTik remoto sabrá que el otro extremo del túnel (10.0.0.2) y la red detrás de él (192.168.20.0/24) son accesibles a través de este enlace.

La opción persistent-keepalive=25 hace que el router envíe cada 25 segundos un pequeño paquete para mantener el túnel activo, algo especialmente útil si alguno de los routers está detrás de un NAT.

### Añadiendo la ruta y NAT

Por último, debemos decirle al router cómo llegar a la red del operador y permitir que los paquetes salgan correctamente a Internet.

    /ip route add dst-address=192.168.20.0/24 gateway=10.0.0.2
    /ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade

La primera línea añade la ruta hacia el otro extremo.

La segunda realiza un *masquerade,* es decir, traduce las direcciones internas cuando el tráfico sale hacia fuera.

![Arquitectura del sistema montado y sus conexiones](https://substack-post-media.s3.amazonaws.com/public/images/69152ee4-0b8f-4d2e-a2c6-e73d5479df3e_2100x1060.jpeg)
*Arquitectura del sistema montado y sus conexiones*

### Repitiendo la configuración en el lado QTH del Operador

El procedimiento en el lado del operador es simétrico, solo cambian las direcciones, procedemos a teclear lo siguiente:

    /ip address add address=192.168.2.10/24 interface=ether1 comment="Conexión al router de fibra"
    /ip address add address=192.168.20.1/24 interface=ether2 comment="Red interna operador"
    /ip pool add name=dhcp_pool20 ranges=192.168.20.10-192.168.20.100
    /ip dhcp-server add name=dhcp20 interface=ether2 address-pool=dhcp_pool20 disabled=no
    /ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1
    /interface wireguard add name=wg0 listen-port=51820
    /ip address add address=10.0.0.2/24 interface=wg0
    /interface wireguard peers add interface=wg0 public-key="CLAVE_PUBLICA_REMOTA" endpoint-address="IP_PUBLICA_REMOTA" endpoint-port=51820 allowed-address=10.0.0.1/32,192.168.10.0/24 persistent-keepalive=25
    /ip route add dst-address=192.168.10.0/24 gateway=10.0.0.1
    /ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade

Si el router remoto no tiene IP fija, se puede usar el servicio DDNS integrado de MikroTik con:

    /ip cloud enable

y usar el nombre que te asigne (por ejemplo, nombre.sn.mynetname.net) en lugar de la IP pública.

### Verificar el túnel

Llegados a este punto, ya deberías tener configurados ambos routers MikroTik: uno en la estación remota y otro en el QTH del operador. Antes de conectar el software de radio, conviene comprobar que el túnel WireGuard realmente está funcionando. Estas pruebas se hacen con el comando ping, una herramienta muy sencilla pero extremadamente útil, que nos permite saber si un dispositivo responde en red. Conéctate por SSH al MikroTik que tienes junto al ordenador del operador (por ejemplo, 192.168.2.10). Desde ahí, escribe:

    ping 10.0.0.1

Este comando envía pequeños paquetes hacia la dirección **10.0.0.1**, que es la IP del túnel en el extremo remoto. Si el túnel está bien establecido, verás algo como:

    SEQ HOST SIZE TTL TIME STATUS
    0 10.0.0.1 56 64 32ms
    1 10.0.0.1 56 64 31ms
    2 10.0.0.1 56 64 33ms

Eso significa que el canal cifrado entre ambos MikroTik está **activo y operativo**.

Si en cambio aparece timeout, revisa que las claves públicas estén bien copiadas en cada extremo y que el puerto 51820 esté abierto en el router de fibra remoto.

Ahora procedemos a verificar la conexión entre redes internas. Una vez confirmado que el túnel funciona, el siguiente paso es probar si **las redes detrás de cada MikroTik pueden verse entre sí**. Sigue conectado al MikroTik del operador y ejecuta:

    ping 192.168.10.10

Aquí estamos intentando llegar al **Hermes Lite 2**, que cuelga del MikroTik remoto y tiene la IP 192.168.10.10 (según nuestra configuración anterior).

Si el túnel está correctamente enrutado, deberías ver respuesta del dispositivo.

Si no responde, pero el túnel sí funciona (es decir, el *ping* a 10.0.0.1 sí contestaba), probablemente el problema esté en el firewall o en la configuración de IP del equipo remoto. Puedes verificarlo conectándote al MikroTik remoto y haciendo un *ping inverso* hacia el ordenador del operador:

    ping 192.168.20.20

De esta manera confirmas que el tráfico puede fluir en ambos sentidos. Por último, podemos comprobar el estado del túnel Wireguard. WireGuard tiene un comando específico para mostrar si los peers están intercambiando datos.

En cualquiera de los dos MikroTik, escribe:

    /interface wireguard peer sprint

Verás una salida similar a esta:

    Flags: X - disabled
    0 interface=wg0 public-key="ABCD123..." endpoint=203.0.113.15:51820
    allowed-address=10.0.0.1/32,192.168.10.0/24
    latest-handshake=1m23s ago
    rx=124.3KiB tx=118.6KiB
    persistent-keepalive=25

Aquí conviene fijarse en tres datos importantes:

- latest-handshake: muestra cuándo fue el último intercambio de claves.

Si ves “1m ago” o “a few seconds ago”, el túnel está vivo.

- rx **y** tx indican los bytes recibidos y transmitidos. Si aumentan cada vez que haces *ping*, todo funciona correctamente.

- endpoint indica la IP pública y el puerto del otro extremo: comprueba que coincide con la configuración que introdujiste.

Por último, no está de más probar la conexión desde el ordenador del operador. Para ello, sal del Mikrotik y, desde el ordenador del operador (por ejemplo, tendrá una IP 192.168.20.20) abre una ventana de comandos o terminal y escribe:

    ping 192.168.10.10

Este *ping* no lo lanza el router, sino tu propio ordenador, lo que te confirma que el túnel está funcionando **de extremo a extremo**: desde tu PC, pasando por el MikroTik local, el túnel WireGuard, el MikroTik remoto y, finalmente, hasta el Hermes Lite 2.

Si la respuesta llega, ya puedes abrir el software que uses para operar la estación (Thetis, Quisk, SparkSDR, etc.) y configurar como dirección del transceptor remoto **192.168.10.10**. El programa se conectará igual que si el equipo estuviera enchufado en la misma red local.
