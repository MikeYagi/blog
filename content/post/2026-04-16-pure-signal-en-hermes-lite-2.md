---
title: "Pure Signal en Hermes Lite 2"
subtitle: "Cuando las matemáticas limpian la señal"
date: 2026-04-16T19:17:15.555Z
author: "EA1PYM"
layout: "post"
url: "/p/pure-signal-en-hermes-lite-2/"
image: "img/home-bg.jpg"
tags: ["hermes-lite-2", "sdr", "puresignal", "dsp"]
categories: ["Radio"]
---
Una de las ventajas de operar con un equipo SDR es disponer de un panadapter en la pantalla de nuestro ordenador, permitiéndonos visualizar el espectro en tiempo real con alta resolución. Esta capacidad, más allá de lo meramente descriptivo, aporta una ventaja clara: facilita identificar de forma inmediata si una señal está correctamente transmitida o presenta problemas de calidad.

Y si hablamos de tipos de señales, es muy habitual encontrase con las que tienen un *splatter* casi ensordecedor: ocupan más ancho de banda del que deberían (2,7Khz-2,9Khz), invaden frecuencias adyacentes, y tienen ese aspecto “deshilachado” que delata que algo no va bien.

Esas señales que vemos tan molestas, no son más que la manifestación directa de un problema muy concreto: **una amplificación no lineal**. Es decir, que ese interlocutor lo que está haciendo es amplificar la señal y su amplificador no está comportándose como debería.

![](https://substack-post-media.s3.amazonaws.com/public/images/b09c93d7-3663-4551-9d06-fa4b31e6b1ff_1610x675.jpeg)

Si pensamos en un amplificador ideal, la idea sería trivial: una caja que multiplica la señal de entrada por un número y listo. Pero los amplificadores reales no funcionan así. Su comportamiento se parece más a una curva que a una recta. Acudimos en este caso al mítico Señales y Sistemas que tantos usamos en la carrera, y que estoy encantado de revisitar con los años.

> Pequeño paréntesis personal: recuerdo perfectamente cuando hace casi 20 años estudiaba ingeniería y me enfrentaba a asignaturas como Sistemas Lineales o Procesado Digital de Señal. Aparecían una y otra vez desarrollos en series, aproximaciones polinómicas… En definitiva, modelos que en aquel momento resultaban innecesariamente complicados. ¿Por qué nadie supo explicármelo usando el caso de un amplificador de radio para poder entenderlo? Durante bastante tiempo tuve la sensación de que todo aquello pertenecía a un mundo más teórico que práctico, y sin embargo han tenido que pasar veinte años para que vuelva a ello. Y lo entienda por fín.

Volviendo al artículo: decíamos que el comportamiento de un amplificador debería parecerse a una curva, una descrita de esta forma:

y(t) = a1​x(t) + a2​x2(t) + a3​x3(t) + ⋯

No hace falta entrar en muchos detalles para entender lo importante:

- El primer término es el “bueno”

- Los demás son los que introducen distorsión

Y esa distorsión no se queda dentro del equipo, sino que sale al aire y se traduce en:

- Productos de intermodulación

- Ensanchamiento de la señal

- Energía fuera de banda

Es decir, exactamente lo que se ve arriba definido como *splatter*. Y aquí es donde entra PureSignal. Si el amplificador distorsiona, basta con crear por nuestro lado una distorsión justo al contrario que neutralice la distorsión generada inicialmente. En términos matemáticos:

![](https://substack-post-media.s3.amazonaws.com/public/images/c44a4769-ab8d-4ea3-a33e-e0ab241ba2dd_346x102.png)

Y si el ajuste es bueno:

![](https://substack-post-media.s3.amazonaws.com/public/images/2860b6b2-79fc-444c-a980-f08a307dc04c_296x78.png)

Digamos que el sistema tiene una observación permanente de sí mismo con permanente corrección. Los pasos son los siguientes:

1.  Transmitimos la señal

2.  Tomamos una muestra de RF a la salida del amplificador

3.  La digitalizamos

4.  La comparamos con la señal original del equipo (Hermes Lite 2 en nuestro caso)

5.  Ajustamos el modelo

#### PureSignal en la práctica 

Sorprendentemente, en la red hay pocos artículos en castellano hablando de cómo implementar Pure Signal en equipos Hermes Lite 2. Pareciera que hubiese un conformismo con la señal de salida del amplificador lineal, sin esperanza de poder mejorarla. Pues bien, llevarlo a cabo es sencillo.

PureSignal necesita una cosa por encima de todo: ver qué está pasando a la salida del sistema. Necesita comparar lo que queríamos transmitir con lo que realmente estamos transmitiendo. Para eso hay que devolver una muestra de RF al equipo. Si trabajamos sin amplificador externo, el propio sistema puede hacerlo internamente. Pero en cuanto aparece un amplificador, la cosa cambia: la distorsión importante ocurre ahí, y si no la medimos, no la corregimos.

En ese caso es imprescindible tomar una muestra de RF a la salida del amplificador, adecuadamente atenuada, e introducirla de vuelta en el sistema, al Hermes vaya. Esto se resuelve mediante un “RF Sampler” o “RF coupler” un elemento aparentemente menor pero que, en realidad, condiciona todo el sistema.

![](https://substack-post-media.s3.amazonaws.com/public/images/bd5edf5f-ccc1-4ab9-a8fe-f50b2b91c5e5_1024x768.jpeg)

Su función no es otra que extraer una pequeña fracción de la señal de RF a la salida del amplificador —típicamente del orden de -40 a -60 dB— y entregarla de vuelta al receptor del propio equipo. Lo importante es que esa muestra sea fiel a lo que realmente está saliendo al aire.

Aquí conviene detenerse un instante, porque es precisamente en este punto donde muchas implementaciones fallan sin que resulte evidente. Si la muestra no es lineal, si introduce su propia distorsión, si está contaminada por retornos o por problemas de adaptación, el sistema de corrección trabaja sobre una referencia incorrecta. Y cuando eso ocurre, el resultado no es una mejora parcial: es, directamente, una corrección errónea. Dicho de otra forma: PureSignal no corrige “la señal”, corrige lo que cree que es la señal.

En el contexto del Hermes Lite 2, la implementación resulta especialmente interesante por su sencillez conceptual. El equipo permite introducir la señal de retorno a través de la entrada de **PureSignal** disponible en la placa I/O adicional, que puede montarse mediante zócalo en su interior. Existe, es cierto, una alternativa más artesanal basada en cablear directamente esa entrada, pero por coste y, sobre todo, por la limpieza del montaje, la solución con la I/O board merece claramente la pena, al permitir una integración prácticamente *plug & play*. A partir de ese momento, el algoritmo de predistorsión entra en funcionamiento: ajusta coeficientes, recalcula el modelo y vuelve a evaluar el resultado. Todo ello ocurre en tiempo real, de forma continua y sin intervención del operador más allá de la configuración inicial.

![](https://substack-post-media.s3.amazonaws.com/public/images/da274c15-34f6-4cfd-a657-b173ff5df261_1000x750.jpeg)

El resultado, cuando el sistema está correctamente ajustado, es inmediatamente visible en el panadapter. Donde antes había una señal con bordes difusos, aparecen límites definidos. Donde había energía derramándose hacia frecuencias adyacentes, vuelve a haber silencio. No es una mejora sutil: es, en muchos casos, un cambio cualitativo. Supone que, independientemente de la potencia con la que emitamos, a antena que tengamos, o el micro con el que modulemos la señal, la gente aprecia una señal limpia y eficiente.

#### Puesta en marcha en Thetis

Llegados a este punto, la teoría deja de tener interés si no somos capaces de llevarla al aire. Afortunadamente, en Thetis la activación de PureSignal no es compleja, aunque sí exige respetar una cierta secuencia y entender qué se está haciendo en cada paso.

El equipo tiene que ser capaz de “escucharse” a sí mismo durante la transmisión. Por eso, en Thetis deben activarse dos funciones básicas: **DUPLEX** y **PS-A**. La primera permite que el sistema siga recibiendo mientras transmite; la segunda activa el modo de autocalibración de PureSignal.

El siguiente paso, aunque pueda parecer menor, es importante: en el control de atenuación hay que trabajar en **A-ATT** y no en **S-ATT**. Y es que si el sistema no está en el modo adecuado de atenuación automática, PureSignal puede no converger correctamente o hacerlo de forma deficiente.

A partir de ahí conviene abrir la ventana de **Linearity**, que es donde realmente se supervisa el funcionamiento del sistema. Es en ese panel donde veremos si la realimentación llega con el nivel adecuado, si el algoritmo está calibrando y si la corrección se mantiene de forma estable. En la práctica, el procedimiento más razonable consiste en situarse en una frecuencia libre, transmitir y observar cómo los indicadores de feedback pasan a actividad durante el proceso de medición y ajuste.

El resultado, cuando todo está en su sitio, suele ser inmediato. La señal se estrecha, los hombros laterales descienden y el transmisor deja de “ocupar” más espectro del necesario. No es únicamente una mejora estética en el panadapter: es una mejora real de calidad de transmisión, y también una forma de respeto al resto de la banda. En las pruebas llevadas a cabo, la reducción de espurios alrededor de la portadora llegó a moverse entre **10 y 30 dB**, una magnitud suficientemente elocuente como para que la diferencia deje de ser teórica.
