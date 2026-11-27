---
title: "Mis cuentas, en mi casa"
subtitle: "Cómo hacer que los bancos te cuenten cada mañana qué ha pasado con tu dinero, usando PSD2, Enable Banking y un servidor del tamaño de un libro."
date: 2026-11-27
author: "EA1PYM"
layout: "post"
url: "/p/finanzas-autoalojadas-psd2/"
image: "img/post-bg.jpg"
tags: ["psd2", "securo", "enable-banking", "proxmox", "homelab", "python"]
categories: ["Redes"]
draft: false
---

## La hoja de cálculo

Durante años llevé las cuentas de casa en una hoja de cálculo. No era una mala hoja: tenía pestañas por año, una columna de categorías con desplegable y unas fórmulas que alguna vez entendí. Su único defecto era que dependía de mí, y yo dependo del humor. Cada fin de mes había que entrar en la web de cada banco, descargar el extracto en un formato distinto, limpiar las columnas, pegar y categorizar a mano doscientos movimientos que se parecían sospechosamente a los doscientos del mes anterior. Lo hice con disciplina durante un tiempo y, después, con la disciplina de quien hace dieta: los lunes.

La hoja acabó como acaban esas cosas, con tres meses en blanco y la promesa de ponerla al día en vacaciones. Lo que no sabía entonces, y sospecho que la mayoría de la gente tampoco sabe, es que desde hace unos años los bancos europeos están obligados por ley a contarle a uno lo que hacen con su dinero a través de una API. No por una web, no por un PDF: por una interfaz programática, con autenticación fuerte y un formato estándar. La obligación tiene nombre de formulario, PSD2, y es probablemente la norma europea más útil que casi nadie ha usado.

Este artículo cuenta cómo la usé para montar en casa un sistema que, cada mañana, recoge los movimientos de todas mis cuentas, los clasifica solo y me deja delante un panel con saldos, gastos por categoría y tendencias. Sin hojas de cálculo, sin descargar nada y sin que mis datos financieros salgan de un servidor que está en una estantería del salón.

## Qué dice la PSD2 y por qué nos importa

La Segunda Directiva de Servicios de Pago, en vigor desde 2018, obligó a los bancos a abrir el acceso a las cuentas a terceros autorizados. Distingue dos papeles. Los PISP pueden iniciar pagos en nombre del cliente, y los AISP, proveedores de información de cuentas, pueden leer saldos y movimientos. Para esto segundo, que es lo que nos interesa, el banco no puede negarse si el cliente da su consentimiento y el tercero tiene licencia.

Conviene tener claras tres cosas antes de seguir. La primera es que el consentimiento es explícito y caduca: uno autoriza el acceso desde la propia app del banco, con la autenticación reforzada de siempre, y esa autorización dura noventa o ciento ochenta días según la entidad. Después hay que renovarla, una molestia pequeña que, bien mirada, es también una garantía. La segunda es que el acceso es de solo lectura. Un AISP ve, pero no toca, y no puede mover un euro.

La tercera es la trampa para el aficionado. El tercero tiene que estar regulado, y un particular no puede pedir una licencia de AISP para conectarse a su propio banco. Necesita un intermediario que la tenga.

Ese intermediario, en mi caso, es Enable Banking.

## Enable Banking, el notario

Enable Banking es una empresa finlandesa con licencia de AISP que ofrece una API unificada sobre cientos de bancos europeos. Cada banco implementa la PSD2 a su manera, con sus propios formatos, sus particularidades y sus caídas de los domingos por la noche, y Enable Banking hace de traductor: uno habla con su API y ella habla con todos los demás.

Lo interesante para un uso personal es que permite registrar una aplicación propia sin coste para conectar las cuentas de uno mismo. El proceso es sobrio, casi administrativo:

1. Se crea una aplicación en su portal. Al crearla se genera un par de claves RSA. La pública se queda en Enable Banking y la privada, en un fichero PEM, se la queda uno. Con esa clave se firman las peticiones a la API.
2. Se declara una URL de redirección, que es adonde vuelve el navegador después de que el banco haya pedido la autenticación.
3. Se vinculan en el portal las cuentas propias. Cada vinculación pasa por la web o la app del banco, con su autenticación reforzada.

A partir de ahí, cualquier programa que tenga el identificador de la aplicación y la clave privada puede pedir saldos y movimientos de esas cuentas mientras dure el consentimiento. La clave privada es, por tanto, lo más delicado de todo el montaje: quien la tenga puede leer mis cuentas. Volveré sobre esto.

## Securo, la casa

Faltaba alguien que hablara con Enable Banking, guardara los datos y los enseñara de forma decente. Lo encontré en [Securo](https://github.com/securo-finance/securo), un gestor de finanzas personales de código abierto (AGPL-3.0) pensado para autoalojarse. Por debajo es una aplicación web clásica y bien resuelta: un backend en FastAPI, PostgreSQL para los datos, Redis y Celery para las tareas periódicas y un frontend aparte. Sincroniza bancos europeos a través de Enable Banking de forma nativa, importa extractos en OFX, QIF, CAMT o CSV, admite reglas de categorización, presupuestos, activos que no son cuentas (un fondo, una vivienda) y espacios de trabajo separados con roles de usuario.

Esto último, los roles, resultó ser más importante de lo que parecía. Lo explico más abajo.

### Dónde vive

En casa tengo un pequeño servidor con Proxmox que hace de todo: domótica, DNS, fotos y otros servicios. A Securo le di un contenedor LXC propio, sin privilegios, con dos núcleos y dos gigas de memoria, y dentro, Docker con las imágenes oficiales. Hay quien prefiere el script de instalación que compila todo en el sitio. Yo prefiero imágenes con la versión fijada, porque actualizar se reduce a cambiar una etiqueta y porque deshacer también:

```yaml
# docker-compose.override.yml (extracto)
x-img: &backend-img ghcr.io/securo-finance/securo-backend:0.19.0
services:
  backend:
    image: *backend-img
    ports: !override ["127.0.0.1:8000:8000"]
  frontend:
    image: ghcr.io/securo-finance/securo-frontend:0.19.0
    ports: !override ["127.0.0.1:3000:8080"]
```

Fijarse en los puertos: ni el backend ni el frontend escuchan en la red. Solo en `127.0.0.1`. Delante hay un Caddy que sirve HTTPS con una autoridad de certificación interna, y el cortafuegos del contenedor solo deja pasar el 443 desde la red de casa. Desde fuera se entra por la VPN WireGuard que ya uso para la estación de radio, o no se entra. No hay un solo puerto abierto a Internet.

Antes de cada actualización hago dos copias: un volcado de PostgreSQL y una copia completa del contenedor. Las migraciones de base de datos de Securo solo van hacia delante, y la única vuelta atrás posible es restaurar. Leer el diff entre versiones antes de subir de una a otra, especialmente las rutas de la API y las migraciones, ha resultado ser una costumbre barata y tranquilizadora.

### El problema de la URL de redirección

Hubo un momento de duda que merece la pena contar. Enable Banking necesita una URL de redirección, y mi Securo vive en un nombre interno que ningún servidor de Internet sabe resolver. Pensé que eso iba a obligarme a publicar el servicio o a montar un dominio público con certificado solo para esto.

No hizo falta. La redirección la hace el navegador, no los servidores de Enable Banking: después de autenticarse en el banco, es mi propio navegador, dentro de mi red, el que vuelve a la dirección interna. Enable Banking nunca necesita alcanzarla. Es uno de esos detalles que se entienden en cuanto uno dibuja el diagrama de secuencia y que, sin dibujarlo, cuestan una tarde.

### La clave privada

El PEM de Enable Banking vive en un único sitio: un directorio del contenedor con permisos 700, en un fichero 600 que solo lee el servicio. No está en copias en la nube, no ha pasado por ningún chat ni por ningún correo, y no está en el repositorio de configuración. Si algún día sospecho de él, se revoca en el portal y se genera otro en cinco minutos. Es el mismo criterio que aplico a las claves privadas de WireGuard: lo que firma por mí no viaja.

## Lo que no habla PSD2

La normativa cubre las cuentas de pago. Fuera se quedan cosas que, en el día a día, también son dinero: ciertas tarjetas de crédito emitidas por entidades que no ofrecen acceso PSD2 en España, y los productos de inversión.

### La tarjeta sin API

Una de mis tarjetas no está disponible en Enable Banking. La solución es menos elegante, pero igual de fiable: una vez al mes descargo el CSV del extracto y lo subo a una pequeña página web propia que vive en el mismo contenedor. Un script lo importa en Securo con el mapeo de columnas correcto (el emisor invierte el signo de los importes y usa fechas en formato día/mes/año, por supuesto), descarta duplicados por la referencia de cada operación y busca en la cuenta corriente el cargo mensual de la tarjeta para enlazarlo como transferencia interna. Sin ese enlace, cada compra con tarjeta contaría dos veces: una al pagar y otra al liquidar el extracto. Si a los pocos días del corte no he subido el fichero, Home Assistant me lo recuerda en el móvil.

### Las inversiones

Mis fondos indexados están en una gestora que tiene una API REST con token personal. El problema es que esa API no ofrece tokens de solo lectura: el mismo token que consulta la cartera podría, en teoría, hacer otras cosas. Lo acepté sabiendo el riesgo y lo encerré lo mejor que supe. Hay un usuario Linux dedicado y sin shell, un servicio de systemd con el sistema de ficheros en solo lectura, sin capacidades y sin acceso a los directorios personales, y un script que solo hace dos peticiones GET. Cada mañana lee el valor de cada cartera y lo anota en Securo como el valor de un activo, con un usuario de Securo que solo sirve para eso. El resultado es que el patrimonio total del panel incluye las inversiones al día, sin que yo haga nada.

## La IA, con correa corta

Con los datos entrando solos, el cuello de botella pasó a ser la categorización. Securo tiene reglas (si la descripción contiene tal texto, asigna tal categoría), y empecé escribiendo unas sesenta. Cubren casi todo, pero siempre aparece el comercio nuevo, la tienda de la que uno ya no se acuerda, la cafetería del aeropuerto.

Aquí metí un modelo de lenguaje, pero con unas condiciones que decidí antes de escribir una línea de código y que sigo creyendo que son las correctas para cualquier cosa que toque dinero:

1. Las cuentas las hace el código. Python suma, resta y compara. El modelo no hace aritmética nunca.
2. Al modelo solo llega lo mínimo. Un nombre de comercio limpio y un importe redondeado. Nunca un IBAN, un saldo, una transferencia o un Bizum, y nunca un nombre de persona: un filtro local descarta cualquier descripción que lo parezca antes de que salga nada del contenedor.
3. La IA lee con un usuario de solo lectura. En Securo, un usuario con rol de lector no puede modificar nada por diseño: todas las rutas de escritura exigen un rol con permiso de escritura.
4. Nada se aplica sin mi permiso.

El flujo queda así. Cada madrugada, un script recoge los movimientos sin categoría. Primero intenta resolverlos con el propio historial, en local: si ese mismo comercio ya tiene categoría en otros movimientos, propone esa. Solo lo que queda sin resolver se envía al modelo, y la respuesta se valida contra la lista real de categorías: si el modelo se inventa una, se descarta. Cada propuesta me llega al móvil como una notificación de Home Assistant con tres botones: *Sí*, *Sí y crea una regla* o *No*. Al pulsar, Home Assistant llama a un pequeño servicio aprobador dentro del contenedor, que solo acepta identificadores de la lista de propuestas pendientes y solo atiende peticiones del propio Home Assistant, autenticadas con una clave compartida. Ese servicio es el único que escribe, con un usuario de Securo propio y limitado. Las propuestas rechazadas no vuelven a aparecer.

En la práctica, categorizar ha pasado de ser una tarea de fin de mes a ser un gesto de diez segundos en el móvil mientras espera uno el café, y que cada vez hay menos que categorizar, porque cada *Sí y crea una regla* enseña algo al sistema para siempre.

El siguiente paso, ya diseñado, es un informe mensual con el mismo principio. El script calcula un JSON con los totales del mes, la tasa de ahorro y el gasto por categoría frente a su media de seis meses, y el modelo solo redacta. Después el script extrae todos los números del texto y comprueba que cada uno esté en el JSON de partida. Si el modelo ha escrito una cifra que no existe, se repite la redacción, y si vuelve a pasar el informe llega marcado como no verificado. Un modelo de lenguaje es un buen redactor y un contable poco fiable, y el diseño tiene que asumir las dos cosas.

## Lo que se ve al final

Todo este andamiaje existe para una pantalla. La de Securo, abierta cualquier mañana, enseña el saldo de todas las cuentas a la hora en que se sincronizaron, el patrimonio con las inversiones valoradas ese mismo día, el gasto del mes por categorías frente a la media y la evolución mes a mes. No hay nada especialmente sofisticado en esos gráficos. Lo sofisticado es que estén al día sin que nadie haya tocado nada.

Con datos completos y limpios empiezan a verse cosas que la hoja de cálculo, con sus meses en blanco, nunca enseñó: qué recibos han subido sin avisar, cuánto pesan de verdad las suscripciones cuando se suman, en qué meses del año se va el dinero y por qué. La estadística doméstica tiene poco de glamurosa, pero tiene la virtud de no dejarse engañar por la memoria, que siempre recuerda que este mes hemos gastado poco.

## Lo que aprendí por el camino

Las importaciones masivas de reglas las carga el diablo. Importar un fichero en modo *sobrescribir* borra todas las reglas existentes, cosa que al menos avisa. Lo que no avisa es que el importador descarta en silencio cualquier regla cuya categoría no exista con el nombre exacto, letra por letra. Así desaparecieron las reglas de suministros, sin un mensaje de error y sin que yo me diera cuenta hasta que los recibos de la luz empezaron a aparecer sin categoría. Desde entonces exporto antes de importar, hago los cambios con scripts que por defecto solo simulan y comparo el antes y el después.

El consentimiento caduca, y conviene enterarse antes que el banco. Descubrir un mes después que una cuenta lleva semanas sin sincronizar es exactamente el tipo de sorpresa que todo esto pretendía eliminar.

Cada tarea tiene su usuario: uno de solo lectura para la IA, otro para las inversiones, otro para la tarjeta y otro para las aprobaciones. Si algo falla o se filtra, se sabe por dónde y se corta solo eso.

Y, sobre todo, conviene leer el código antes de fiarse. Garantías como que un usuario lector no pueda escribir las comprobé en el código de la versión que tengo instalada, no en la documentación, porque la documentación cuenta lo que el autor quería hacer y el código cuenta lo que hizo.

## Por qué cuento esto

Porque estoy convencido de que la mayoría de la gente no sabe que esto existe. Pagamos con el móvil, consultamos el saldo diez veces al día y seguimos creyendo que la única forma de sacar los datos del banco es un PDF con logotipo. La PSD2 lleva años abriendo esa puerta y casi nadie fuera de las fintech ha pasado por ella.

Montar algo así lleva unas cuantas tardes y exige cierto cuidado con la seguridad, que es justo la parte que más he disfrutado. A cambio, uno recupera algo que parecía perdido: la posibilidad de mirar su propio dinero con sus propias herramientas, en su propio servidor, con el detalle que quiera y sin pedirle permiso a nadie más que a sí mismo. Y, de paso, sin volver a abrir una hoja de cálculo un lunes.
