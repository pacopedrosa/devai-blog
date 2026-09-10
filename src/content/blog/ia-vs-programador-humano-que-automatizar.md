---
title: 'IA vs programador humano: qué tareas automatizar y cuáles no'
description: 'Artículo de opinión sobre dónde la IA aporta valor real en programación y dónde sigue haciendo falta criterio humano, sin afirmaciones categóricas en ningún sentido.'
pubDate: 'Aug 10 2026'
---

Sobre este tema circulan sobre todo dos posturas, y las dos simplifican de más. Una dice que la IA ya escribe código mejor que la mayoría de programadores y que el oficio, tal como se conocía, tiene los días contados. La otra dice que es una moda sobrevalorada que produce código mediocre y que un desarrollador serio no debería depender de ella. Ninguna resiste mucho contacto con el uso real de estas herramientas.

Este artículo es un intento de argumentar el término medio con algo más de precisión que "depende", explicando en qué se apoya cada lado de la balanza.

## Lo que la IA hace claramente bien

**Trabajo mecánico y de patrón reconocible.** Boilerplate, renombrados, migraciones de una API a otra, adaptar un patrón que ya existe en un sitio del proyecto a otro sitio similar. Es trabajo donde el valor no está en la originalidad de la solución, sino en aplicarla correctamente y sin erratas — exactamente lo que un modelo hace bien y un humano encuentra tedioso.

**Explorar código desconocido.** Aterrizar en un proyecto ajeno y preguntar cómo fluye una petición, dónde se valida un dato o por qué existe un módulo concreto es mucho más rápido con un asistente que puede leer el código y resumir que abriendo archivos a ciegas. El riesgo aquí es bajo: si la explicación es incorrecta, se descubre al leer el código real al que apunta.

**Trabajo autoverificable.** Tests para código que ya funciona, o cualquier tarea donde el resultado se puede comprobar ejecutándolo. La IA se equivoca con la misma confianza tanto si acierta como si no, pero cuando el resultado se puede verificar objetivamente (pasa o no pasa), ese riesgo queda acotado.

**Primeras versiones y punto de partida.** Un borrador de función, una estructura inicial de proyecto, una primera aproximación a un algoritmo. No tiene por qué ser la versión final, pero partir de un boceto razonable en segundos, en lugar de la página en blanco, ahorra tiempo real.

## Lo que sigue necesitando criterio humano

**Decisiones de arquitectura.** Elegir entre un monolito o servicios separados, qué base de datos usar, cómo modelar el dominio del negocio. Son decisiones que dependen de restricciones del equipo, del negocio y de un futuro que solo se puede prever con juicio, no con patrones vistos en código de otros proyectos. Son también caras de revertir, lo cual las hace especialmente sensibles a delegarse sin criterio propio detrás.

**Evaluar si algo es correcto cuando no hay forma objetiva de comprobarlo.** Si no hay tests, ni manera de ejecutar el resultado, ni criterio propio para juzgarlo, delegar esa tarea no traslada el riesgo a la herramienta — lo elimina de la vista, que es peor. El problema no es la capacidad del modelo, es que nadie ha verificado el resultado.

**Contexto no escrito en ningún sitio.** Que un servicio se cae si le llegan más de cierto volumen de peticiones, que hay un cliente con una integración frágil que no soporta cambios en un formato de respuesta, por qué se tomó una decisión rara hace tiempo. Nada de esto está en el código ni en ningún prompt, salvo que alguien se acuerde de mencionarlo — y es precisamente el tipo de conocimiento que un equipo acumula con el tiempo y que ninguna herramienta trae de fábrica.

**Cualquier cosa que toque producción de forma difícil de revertir.** Migraciones destructivas, borrados masivos, cambios en infraestructura compartida, gestión de secretos. No por desconfianza hacia la herramienta en sí, sino porque el coste de un error ahí es asimétrico frente al beneficio de haberlo hecho un poco más rápido.

**Aquello que se está intentando aprender.** Es el matiz que menos se menciona en este debate y probablemente el más importante a nivel individual. Delegar todos los ejercicios mientras se aprende un concepto nuevo da código que funciona y cero aprendizaje real; la sensación de productividad es genuina, pero el conocimiento no se queda. Cuando el objetivo es aprender, la relación con la herramienta tiene que ser distinta a cuando el objetivo es producir.

## La zona gris: donde de verdad está el debate

Entre esos dos extremos hay una franja mucho más amplia y mucho más interesante que cualquiera de los dos discursos: cambios de tamaño medio, en código que sí se puede verificar pero cuya corrección no es trivial a simple vista. Ahí la respuesta honesta no es "sí" o "no" categóricos, sino que el resultado depende directamente de cuánto criterio se aplica **después** de que la IA produzca algo — de si alguien lo revisa con la misma exigencia con la que revisaría el código de un compañero, o si se acepta porque "tiene buena pinta" y compila.

Esto tiene una implicación incómoda para el argumento de que la IA "sustituye" al programador: en esa zona gris, que es donde vive la mayor parte del trabajo real, la herramienta no elimina la necesidad de criterio técnico — lo desplaza hacia la revisión en lugar de hacia la escritura. Se escribe menos código a mano, se lee y se juzga bastante más. Eso exige la misma competencia técnica que escribirlo desde cero, aplicada de otra forma.

## Por qué ninguno de los dos discursos extremos se sostiene

El discurso de "ya no hace falta programar" ignora que alguien tiene que poder evaluar si el resultado es correcto, y esa evaluación exige el mismo conocimiento que se necesitaría para escribirlo. Sin ese criterio, lo que se obtiene no es productividad, es confianza sin fundamento en código que nadie ha verificado de verdad.

El discurso de "es un juguete que no sirve para nada serio" ignora que hay categorías enteras de trabajo —mecánico, repetitivo, autoverificable— donde el riesgo de delegar es bajo y el ahorro de tiempo es real y medible en la práctica diaria de quien usa estas herramientas de forma habitual.

Los dos discursos comparten el mismo defecto: tratan "la IA en programación" como una única cosa, cuando en realidad es una herramienta cuyo valor cambia radicalmente según el tipo de tarea a la que se aplique.

## Una pregunta más útil que "¿sustituye la IA al programador?"

La pregunta que de verdad ayuda a decidir, tarea por tarea, no es si la IA "sabrá hacer esto". Casi siempre, en algún grado, sabe. La pregunta útil es otra: **¿sabré yo, después, si lo ha hecho bien?**

Cuando la respuesta es sí —porque hay tests, porque el resultado se puede ejecutar y comprobar, porque el ámbito es lo bastante acotado como para revisarlo con atención real— delegar suele compensar. Cuando la respuesta es no, ese es exactamente el trabajo que sigue tocando hacer a una persona: no porque la herramienta no pueda intentarlo, sino porque nadie estaría en condiciones de saber si el resultado es de fiar.

## Conclusión

La IA no sustituye el criterio técnico; lo desplaza hacia un lugar distinto del proceso. Automatiza bien lo mecánico, lo repetitivo y lo verificable, y deja intacta —o incluso más importante que antes— la parte del oficio que consiste en decidir con buen juicio y en saber juzgar si un resultado es correcto. Esa capacidad de evaluación no es un residuo del oficio que quede tras automatizar el resto: es, cada vez más, el oficio en sí.
