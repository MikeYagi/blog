---
title: "Diseño y montaje de una estación remota basada en Hermes Lite 2"
subtitle: "Este artículo fue publicado en la revista Radioaficionados URE - Octubre 2025"
date: 2026-04-13T10:22:22.528Z
author: "EA1PYM"
layout: "post"
url: "/p/diseno-y-montaje-de-una-estacion/"
image: "img/post-bg.jpg"
tags: []
categories: ["Radio"]
---
**1. Introducción**

El concepto de estación remota en radioafición ha ganado relevancia en los últimos años, principalmente gracias al desarrollo de las redes de telecomunicaciones y a la popularización de los equipos SDR (Software Defined Radio). Ya no es necesario que el operador esté junto al transmisor para disfrutar de la radio: hoy es posible operar una estación a cientos o miles de kilómetros con una experiencia casi indistinguible de la local. Y esto no solo aporta comodidad, sino que es una solución práctica para quienes no pueden instalar antenas en su localización habitual, o desean operar desde ubicaciones con mejor propagación o menor ruido.

En este artículo se describe el análisis, diseño y puesta en marcha de una estación remota basada en el transceptor SDR Hermes Lite 2. A partir de una experiencia personal, se explican los fundamentos de la solución, sus ventajas, limitaciones, componentes clave y aspectos técnicos críticos como latencia, control remoto y fiabilidad.

Si Marconi levantara la cabeza, probablemente frunciría el ceño al no ver una sola chispa, ni un dial giratorio, ni latiguillos sobre la mesa. Y confieso que incluso yo a veces echo de menos un *shack* con una estantería llena de equipos, frontales con decenas de botones, con sus clics táctiles y su luz ámbar. Pero lo cierto es que hoy, con un ordenador modesto en nuestro cuarto de radio, tenemos al alcance una potencia de cómputo que iguala ,y a menudo supera, las prestaciones de transceptores de alta gama. El milagro de la radio sigue vivo, solo que ha cambiado de forma.

**2. Visión general de arquitecturas posibles**

Podemos dividir las estaciones remotas en dos grandes tipos: las basadas en equipos tradicionales y las que usan tecnología SDR.

Las primeras buscan extender el control físico del equipo a distancia. Suelen incluir software de control CAT, cámaras IP para monitorizar y sistemas de relés para encendido, cambio de antenas, amplificadores, etc. Y aunque funcionales, estas soluciones requieren una infraestructura compleja y mantenimiento continuo. En algunos casos, se opta por colocar un ordenador junto al equipo y acceder a él mediante herramientas de escritorio remoto como VNC o TeamViewer. Aunque puede parecer una solución sencilla, este enfoque presenta importantes limitaciones: el manejo del audio es mucho más errático, la latencia es elevada y la seguridad, en general, queda en manos de sistemas pensados para otros entornos, no para la radio.

Las estaciones SDR, en cambio, eliminan esa barrera física desde el diseño. La radio convierte directamente la RF en datos digitales que viajan por red. Todo el control se realiza desde el software del operador, sin necesidad de tocar mandos físicos ni de supervisión visual. Este enfoque, especialmente con una arquitectura IP bien diseñada, permite soluciones remotas robustas, modulares y escalables.

**3. La solución adoptada: Hermes Lite 2 como núcleo**

**3.1 ¿Qué es el Hermes Lite 2?**

El Hermes Lite 2 es un transceptor SDR de código abierto para bandas de HF. Derivado del proyecto Hermes de OpenHPSDR, utiliza un ADC de alta calidad y una FPGA para el procesamiento digital. Se conecta exclusivamente por Ethernet, y permite transmisión/recepción entre 160 y 10 m con una potencia de salida de hasta 5 W. Su arquitectura soporta muestreo directo de RF, múltiples receptores simultáneos y recepción en banda ancha.

![Ilustración 1- Equipo SDR Hermes Lite 2](https://substack-post-media.s3.amazonaws.com/public/images/7f46abfa-e0db-4fe7-9285-9f701e36c44f_1600x1066.jpeg)
*Ilustración 1- Equipo SDR Hermes Lite 2*

Su diseño sin pantalla ni controles físicos está completamente orientado a operación remota. Además, su precio es muy competitivo (400€) frente a soluciones SDR comerciales.

**3.2 Ventajas frente a equipos tradicionales**

En los equipos clásicos, incluso con control remoto, muchas funciones siguen dependiendo de botones físicos o interfaces CAT limitadas. El HL2 externaliza completamente la interfaz y el procesamiento al PC, que se convierte en el panel de control del operador. Esto permite separar físicamente el operador del equipo sin comprometer calidad ni funcionalidad. Además, al operar exclusivamente por red, se eliminan muchos elementos típicos del control remoto tradicional (relés, webcams, etc.). El mantenimiento también mejora: basta con actualizar *software* para incorporar nuevas funciones. La flexibilidad es máxima, y la estación puede funcionar durante meses sin intervención física.

**3.3 Ethernet como habilitador**

El verdadero cambio de paradigma lo permite la conexión Ethernet del HL2, que transforma la radio en un dispositivo de red, operable tanto localmente como a través de Internet.

A diferencia del cable coaxial, donde la distancia degrada la señal, en el mundo IP lo importante es la latencia y la estabilidad. El audio y control se encapsulan en paquetes IP que viajan por fibra o redes móviles sin pérdida de integridad. El operador puede estar en la ciudad y la estación en un entorno rural con condiciones óptimas, sin que ello afecte al rendimiento.

Naturalmente, se necesita conexión a Internet en la ubicación remota, pero eso hoy es cada vez menos problemático: España es el tercer país de la OCDE en cobertura de fibra, con un 88 % de penetración. Y para localizaciones todavía más remotas, siempre se puede acudir a la red móvil.

**4. Diagrama de bloques de la estación remota**

La arquitectura se basa en diferentes bloques interconectados. En la ubicación remota se sitúa el HL2, alimentado con 13,8 V y conectado a una antena EFHW de 40 m con transformador 49:1 (de nuestro compañero EB4HRA). Para alcanzar 100 W se emplea un amplificador Neptune 100 con detección automática de bandas (por voltajes) y activado desde el HL2 mediante un cable PTT. El ATU automático ATM100 asegura una correcta adaptación sin intervención manual.

![Ilustración 2- Diagrama de bloques con conexiones entre estación remota y estación operadora](https://substack-post-media.s3.amazonaws.com/public/images/1af8faa3-ced5-45ea-a398-71c8edeac5d0_3642x1678.jpeg)
*Ilustración 2- Diagrama de bloques con conexiones entre estación remota y estación operadora*

El HL2 se conecta por Ethernet a un router Mikrotik que establece un túnel WireGuard hacia otro Mikrotik en el QTH del operador. Desde ahí, un PC con Thetis proporciona el control completo de la radio, incluyendo *waterfall*, VFOs y configuración operativa.

**5. Elección del software: Thetis, latencia y calidad de audio**

La elección del software es clave. Pensemos que este programa supone el 90% de la experiencia de usuario usando una estación remota, por lo que su elección resulta clave para una operación satisfactoria. Tras probar varias opciones, personalmente opto por Thetis, una evolución moderna de PowerSDR con soporte nativo para el HL2. Destaca por su interfaz configurable, receptores simultáneos, *waterfall* fluido y buena integración con sistemas de audio virtual y CAT.

Uno de los aspectos que más sorprende a los usuarios de Thetis es el excelente funcionamiento de filtros digitales de alto nivel mediante procesamiento en tiempo real. Utiliza transformadas rápidas de Fourier (FFT) y filtrado adaptativo apoyándose en la CPU del PC. Para quienes hemos sido estudiantes de ingeniería y nos hemos peleado con ventanas temporales, resolución espectral o leakage, ver estas técnicas aplicadas a en la vida real es francamente reconfortante.

![Ilustración 3 - Thetis en activo. Se pueden apreciar dos columnas amarillas que actúan como Multi-tracking Notch Filters elimitando señal con ancho de banda a elegir, en este caso en los contornos de la señal que buscamos escuchar, para reducir ruido.](https://substack-post-media.s3.amazonaws.com/public/images/fc8e1558-7f58-4ab9-aca6-ee2f6c686c11_5112x2782.jpeg)
*Ilustración 3 - Thetis en activo. Se pueden apreciar dos columnas amarillas que actúan como Multi-tracking Notch Filters elimitando señal con ancho de banda a elegir, en este caso en los contornos de la señal que buscamos escuchar, para reducir ruido.*

En cuanto a latencia, el sistema se comporta muy bien con una buena conexión a internet. Con fibra simétrica de 100 Mbps y baja fluctuación, la latencia total (audio + control) ronda los 140 ms. La calidad de audio es excelente, comparable a equipos comerciales de gama alta. A resaltar la importancia de una buena configuración del software: fijar a 48kHz todos los elementos de audio del pc, así como seguir un correcto calibrado de las diferentes opciones del menú. Este tema se revisará en futuros artículos ya que cubrirlo en detalle requiere de explicaciones extensas.

**6. Conexión fiable: red, VPN y CG-NAT**

Para una operación remota segura, la comunicación entre el operador y la estación debe ser fiable. El HL2 usa Ethernet y protocolos TCP/UDP estándar, pero exponerlo directamente a Internet plantea riesgos.

Una solución sencilla, si se dispone de IP pública, es usar DNS dinámico y reenvío de puertos. Pero esto deja servicios expuestos, con riesgo de ataques, por lo que la recomendación pasa por emplear un método seguro como es un túnel VPN punto a punto.

Una VPN (Virtual Private Network) consiste básicamente en establecer un “túnel” de datos entre el QTH donde se encuentra la estación y el QTH donde está el operador, como si estuvieran en la misma red local, pero usando internet como medio. Hay diferentes formas y métodos de establecer este túnel, personalmente elijo WireGuard por su ligereza, facilidad de configuración y buen rendimiento.

Si se está detrás de un CG-NAT, algunas operadoras permiten salir de él mediante solicitud. Si no, se puede usar un servidor VPS intermedio en la nube, pero esto se escapa del alcance de este artículo, que será revisado en futuras ediciones.

**7. Alimentación, encendido remoto y seguridad**

Uno de los aspectos clave en una estación remota es la gestión de la alimentación y el encendido a distancia. No hay posibilidad de reiniciar manualmente si algo falla, por lo que se debe asegurar la autonomía del sistema.

En mi caso, todos los dispositivos (HL2, amplificador, ATU, etc.) están alimentados desde una fuente lineal de 13,8 V, controlada a su vez por un enchufe inteligente con wifi. Este es el único punto de encendido necesario y permite activar todo el sistema de forma remota.

![Ilustración 4 - Alojamiento de los equipos. Al no ser necesario su localización en cuarto de radio, pueden posicionarse cercanos al punto de despliegue de antena (con las protecciones necesarias que implican estar a la intemperie)](https://substack-post-media.s3.amazonaws.com/public/images/818d01e4-dac3-40d6-abf6-39de7ce3250f_1600x1200.jpeg)
*Ilustración 4 - Alojamiento de los equipos. Al no ser necesario su localización en cuarto de radio, pueden posicionarse cercanos al punto de despliegue de antena (con las protecciones necesarias que implican estar a la intemperie)*

**10. Experiencia práctica, resultados y puntos de mejora**

Tras varios meses de operación, la estación remota basada en el Hermes Lite 2 y el software Thetis ha demostrado ser una solución robusta y eficiente, permitiendo disfrutar de condiciones de propagación óptimas y un bajo nivel de ruido, lo que ha incrementado significativamente el número de contactos y la calidad de la experiencia como operador. Sin embargo, como en cualquier instalación técnica avanzada, existen oportunidades de mejora que pueden potenciar aún más su rendimiento, seguridad y fiabilidad.

Una de las prioridades pasaría por reforzar la redundancia en la alimentación eléctrica. Aunque la fuente lineal de 13,8 V y el control remoto a través de un enchufe WiFi son suficientes para la mayoría de las situaciones, la integración de un sistema de alimentación ininterrumpida (SAI) mitigaría los riesgos asociados a cortes de energía. Además, implementar sensores de voltaje y corriente permitiría monitorizar el estado de la alimentación y prevenir fallos antes de que afecten al equipo.

La protección contra descargas atmosféricas es otro aspecto clave. A pesar de contar con protecciones básicas, sería recomendable instalar un relé de desconexión automática de antenas en caso de tormentas eléctricas, así como revisar periódicamente las tomas a tierra del equipamiento.

La gestión de la temperatura en el alojamiento de los equipos también es relevante; incorporar ventiladores controlados por termostato o sensores de temperatura que alerten remotamente sobre sobrecalentamientos evitaría fallos por estrés térmico, especialmente en verano o en ubicaciones con poca ventilación.

Finalmente, la monitorización de todos estos aspectos podría gestionarse mediante una Raspberry Pi con sensores de temperatura, humedad y consumo eléctrico, permitiendo detectar incidencias en tiempo real y facilitando el mantenimiento preventivo.

**11. Notas finales**

No faltan quienes ven en las estaciones remotas una amenaza al romanticismo de la radioafición. Para algunos, operar sin estar físicamente junto al equipo, sin escuchar el clic de los relés o sin sentir el calor del transmisor, es poco menos que una herejía. Se cuestiona si, al delegar buena parte de la experiencia en el software y la conectividad IP, no estaremos perdiendo el contacto con la esencia misma de nuestra afición.

Pero esa visión, aunque comprensible desde la nostalgia, olvida algo fundamental: la radioafición siempre ha sido una afición de pioneros, de ingenieros autodidactas, de quienes experimentan, adaptan y empujan los límites técnicos con los medios disponibles en cada época. El uso de SDR, redes privadas, túneles cifrados o procesamiento digital no aleja al operador de la radioafición; lo inserta, precisamente, en la frontera actual del conocimiento y la experimentación técnica.

Tal vez no haya mayor símbolo del espíritu radioaficionado que esta reinvención remota: una antena en un prado, un operador en un apartamento, y entre ambos, un mar de transformadas de Fourier, filtros digitales y latencias cuidadosamente calibradas. Marconi hablaba de unir continentes; hoy, un SDR como el Hermes Lite 2 puede unir al operador con su pasión, aunque los separen cientos de kilómetros y más de una capa de red. Porque al final, más allá del lugar donde está la caja negra que emite, lo que define a un radioaficionado es su capacidad para conectar, innovar y construir puentes entre mundos aparentemente distantes.
