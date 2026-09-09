---
title: 'Cómo uso Claude Code en mi día a día como developer'
description: 'Para qué tareas funciona bien un asistente de código agéntico, cómo darle buen contexto, sus límites reales y en qué casos es mejor no delegarle el trabajo.'
pubDate: 'Aug 24 2026'
---

Sobre las herramientas de IA para programar hay básicamente dos discursos, y los dos son inútiles. El primero dice que ya no hace falta aprender a programar. El segundo, que son un juguete que genera código malo y te va a arruinar el proyecto.

Uso Claude Code de forma habitual, integrado en mi flujo de trabajo. Este artículo es el punto intermedio: para qué me resulta realmente útil, cómo consigo que dé buenos resultados, dónde están sus límites de verdad y cuándo directamente no le delego una tarea.

## Qué es Claude Code, en una frase

Es un asistente de programación **agéntico** que funciona en el terminal: a diferencia de un autocompletado que sugiere la siguiente línea, puede leer los archivos de tu proyecto, editarlos, ejecutar comandos y verificar el resultado de lo que acaba de hacer.

Esa diferencia entre "sugiere" y "actúa" es la que importa. Un autocompletado te ayuda mientras escribes. Un agente puede recibir una tarea, explorar el código para entender el contexto, hacer el cambio en varios archivos, ejecutar los tests y corregirse si fallan. También significa que puede equivocarse en más sitios a la vez, y de ahí sale buena parte de lo que cuento más abajo.

## Para qué funciona realmente bien

**Entender código que no conoces.** Es, con diferencia, el uso que más rentabilidad me da. Aterrizar en un repositorio ajeno y preguntar cómo fluye una petición, dónde se valida algo o por qué existe un módulo, es mucho más rápido que ir abriendo archivos a ciegas. Aquí el riesgo además es bajo: si la explicación es incorrecta, lo descubres al leer el código que te señala.

**Cambios mecánicos repetitivos.** Renombrar un concepto en veinte archivos, migrar llamadas de una API antigua a una nueva, adaptar un patrón que ya existe a tres sitios más. Es trabajo tedioso, propenso a erratas por despiste, y fácil de verificar después porque sabes exactamente qué esperabas.

**Tests.** Escribir tests para código que ya funciona es de esas tareas que todo el mundo pospone. Delegarlo funciona bien porque los tests son *autoverificables*: o pasan o no pasan. Eso sí, hay que revisarlos igual — un test que no comprueba nada útil también pasa.

**Boilerplate y configuración.** Un Dockerfile, un workflow de CI, la estructura inicial de un módulo siguiendo el patrón que ya usa el proyecto. Cosas donde hay una forma bastante estándar de hacerlo y el valor no está en la originalidad.

**Depuración con acceso a ejecución.** Poder ejecutar el código, ver el error real y probar una hipótesis es cualitativamente distinto a razonar sobre un fragmento pegado en un chat. Para errores reproducibles funciona muy bien.

**Documentación.** Explicar en un README lo que hace un módulo, o dejar por escrito los pasos de un despliegue. Especialmente útil cuando hay que mantenerlo en dos idiomas.

## Cómo dar buen contexto (aquí está casi todo)

La diferencia entre un resultado útil y uno que hay que tirar rara vez está en el modelo. Está en el encargo.

**Explica el porqué, no solo el qué.** "Cambia esta función para que devuelva null si no encuentra el usuario" produce un cambio literal. "Los llamantes están reventando cuando el usuario no existe; quiero que ese caso sea explícito en lugar de una excepción" da el objetivo, y permite proponer algo mejor que lo que se te había ocurrido a ti.

**Sé concreto con los archivos.** Señalar dónde vive el problema ahorra una fase entera de búsqueda y evita que se toque código que no venía al caso.

**Di explícitamente qué NO debe tocarse.** Es de los consejos que más resultado dan y menos se aplican. Si hay un módulo delicado, una migración a medias o una convención rara que tiene su razón de ser, decirlo por adelantado evita "mejoras" no pedidas.

**Pide que verifique, no que prometa.** "Ejecuta los tests y arregla lo que falle" es un encargo comprobable. Un agente que puede ejecutar código debería usarlo; si no lo hace, pídelo.

**Trocea las tareas.** Un encargo mediano bien acotado sale bien. Uno enorme y difuso ("refactoriza el módulo de pagos") sale mal, y además es imposible de revisar. Si tú no serías capaz de revisar el diff resultante en una sentada, es demasiado grande.

**Escribe las convenciones del proyecto una sola vez.** Claude Code lee un archivo `CLAUDE.md` en la raíz del repositorio. Dejar ahí cómo se ejecutan los tests, qué gestor de paquetes se usa o qué patrones sigue el proyecto evita repetirlo en cada conversación — y evita que se asuma npm cuando usas pnpm.

## Los límites reales

Esta es la parte que suele faltar en los artículos sobre estas herramientas.

**Puede equivocarse con total seguridad.** No hay diferencia de tono entre una respuesta correcta y una incorrecta. Un humano duda, matiza, dice "creo que". Un modelo afirma igual de convencido en ambos casos. Esto es lo que hace peligroso delegar en un área donde no puedes evaluar el resultado: no vas a notar el error por cómo suena.

**No conoce tu contexto no escrito.** No sabe que ese servicio se cae si le llegan más de X peticiones, que hay un cliente con una integración frágil, ni por qué se tomó una decisión rara hace dos años. Solo conoce lo que puede leer y lo que le cuentas.

**La fiabilidad baja según crece el cambio.** Para un cambio acotado el resultado suele ser bueno. Para una reescritura grande que toca muchos archivos, la probabilidad de que algo salga sutilmente mal en alguna parte crece bastante.

**El contexto tiene un límite.** En sesiones largas o repositorios grandes no cabe todo. Puede perderse un detalle que se dijo mucho antes, o no haber leído el archivo justo donde estaba la respuesta.

**Tiende a hacer de más.** Si le pides arreglar un bug, es fácil que además reorganice imports, añada comentarios y "mejore" cosas de alrededor. Ese ruido dificulta la revisión y mete cambios que nadie pidió. Se corrige diciéndolo explícitamente.

## Cuándo NO conviene delegar

**Decisiones de arquitectura.** Elegir entre monolito o servicios, qué base de datos usar, cómo modelar el dominio. Estas decisiones dependen de restricciones del equipo, del negocio y del futuro previsible, y son caras de revertir. Como interlocutor para contrastar opciones va bien; como decisor, no.

**Código donde no puedes verificar la corrección.** Si no tienes tests, ni forma de probarlo, ni criterio propio para juzgarlo, estás añadiendo código que nadie ha validado. Aquí el problema no es la herramienta: es que no tienes forma de saber si está bien.

**Nada que toque producción de forma difícil de revertir.** Migraciones destructivas, borrados, cambios en infraestructura compartida, manejo de secretos. No por desconfianza en la herramienta, sino porque el coste de un error es asimétrico: revisas el plan tú, y ejecutas tú.

**Aquello que estás intentando aprender.** Este es el más importante y el que menos se dice. Si estás aprendiendo un framework y delegas todos los ejercicios, obtienes código que funciona y cero aprendizaje. La sensación de productividad es real; el aprendizaje no ocurre. Cuando el objetivo es aprender, la herramienta debería explicarte, no resolverte.

## El flujo que a mí me funciona

1. **Encargos pequeños y acotados**, con el objetivo y el porqué explícitos.
2. **Revisar siempre el diff**, línea a línea. No "parece bien": leerlo. Es la parte no negociable.
3. **Tests como red de seguridad.** El valor de tener tests sube mucho cuando parte del código no lo has escrito tú.
4. **Commits pequeños.** Si algo se tuerce, revertir es trivial.
5. **Si algo no me convence, no lo negocio: lo escribo yo.** A veces explicar exactamente lo que quieres cuesta más que hacerlo, y ahí la respuesta correcta es hacerlo.

## Conclusión

Después de usarla de forma habitual, mi valoración es bastante prosaica: es una herramienta que quita tiempo de trabajo mecánico y acelera mucho el entender código ajeno, a cambio de añadir una responsabilidad nueva — **revisar bien**.

Eso desplaza el trabajo, no lo elimina. Se escribe menos código a mano y se lee bastante más. Y hace más valiosa, no menos, la capacidad de juzgar si un código es correcto: si no puedes evaluar lo que te devuelve, la herramienta no te está ayudando, te está dando confianza sin fundamento.

La pregunta útil antes de delegar una tarea no es "¿sabrá hacer esto?", sino **"¿sabré yo si lo ha hecho bien?"**. Cuando la respuesta es sí, suele merecer la pena. Cuando es no, ese es justo el trabajo que te toca hacer a ti.
