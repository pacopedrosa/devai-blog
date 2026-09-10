---
title: 'Cómo revisar código generado por IA sin fiarte a ciegas'
description: 'Guía práctica de qué comprobar siempre en código escrito por un asistente de IA: seguridad, dependencias, tests y lógica de negocio, antes de darlo por bueno.'
pubDate: 'Jun 19 2026'
---

Que un fragmento de código compile, se ejecute y produzca el resultado esperado en el caso feliz no significa que esté bien. Esto es cierto para cualquier código, lo haya escrito quien lo haya escrito, pero se vuelve especialmente relevante con código generado por IA por una razón concreta: **no hay diferencia de tono entre una respuesta correcta y una incorrecta**. Un modelo no duda, no dice "no estoy seguro de esto", y presenta con la misma confianza un cambio bien pensado que uno con un fallo de seguridad.

Esta guía no trata de convencerte de no usar estas herramientas — trata de dar una lista concreta de qué comprobar antes de fusionar algo que no has escrito tú línea a línea.

## El principio de fondo: revisa como si lo hubiera escrito un becario con mucha confianza

Es la analogía que mejor funciona. Alguien capaz técnicamente, rápido, que conoce muchos patrones, pero que no conoce **tu** proyecto: no sabe qué decisiones se tomaron por una razón concreta, no conoce las restricciones de negocio no escritas, y no tiene ningún filtro interno que le haga decir "esto no me convence, mejor pregunto". La responsabilidad de detectar eso sigue siendo tuya.

Con esa idea en la cabeza, el resto de la guía son categorías concretas de cosas que fallan.

## 1. Seguridad

Es lo primero que hay que mirar, porque es lo que tiene coste más alto si se cuela.

- **Inyección.** SQL, comandos de shell, HTML sin escapar (XSS). Si el código construye una consulta o un comando concatenando strings con datos que vienen del usuario, es una señal de alerta inmediata, sin excepciones. Debe usarse parametrización o el mecanismo seguro equivalente de la librería que estés usando.
- **Validación de entrada.** ¿Qué pasa si el campo que se espera como número llega vacío, o como texto, o con un valor fuera de rango? Un modelo tiende a escribir el camino feliz salvo que le pidas explícitamente que cubra los bordes.
- **Secretos hardcodeados.** Claves de API, contraseñas o tokens escritos directamente en el código en lugar de en variables de entorno o un gestor de secretos. Pasa más de lo que parece, sobre todo en ejemplos rápidos donde el foco estaba en la funcionalidad, no en cómo se despliega.
- **Permisos y autenticación.** Si el cambio toca un endpoint, comprueba explícitamente que sigue exigiendo la autenticación y los permisos que debería. Es fácil que una refactorización rompa una comprobación de permisos sin que nada lo señale, porque el código sigue "funcionando" para el usuario que tiene todos los permisos.
- **Datos sensibles en logs o respuestas.** Que un objeto que se devuelve o se registra en logs no incluya campos que no deberían salir (contraseñas, tokens, datos personales de más).

## 2. Dependencias

- **¿Es una dependencia nueva o ya existía en el proyecto?** Si el código añade una librería que no estaba, pregúntate si hacía falta o si el mismo resultado se podía conseguir con lo que ya tenías. Añadir una dependencia tiene un coste de mantenimiento que no siempre se justifica para resolver algo pequeño.
- **¿Existe de verdad?** Los modelos pueden sugerir nombres de paquetes que suenan plausibles pero no existen, o que existen con un propósito distinto al que el código asume. Comprueba el nombre exacto en el registro correspondiente (PyPI, npm...) antes de instalar nada a ciegas.
- **Versión y mantenimiento.** Una dependencia sin actualizar en años, con pocas descargas o sin repositorio público es un riesgo, tanto de seguridad como de que deje de funcionar sin aviso.

## 3. Tests

- **¿El test comprueba algo real, o solo pasa?** Un test que hace `assert True` o que comprueba un valor trivial pasa siempre y no aporta nada. Lee qué afirma cada test, no solo si está en verde.
- **¿Cubre los casos límite, o solo el camino feliz?** Entrada vacía, valores negativos donde no deberían darse, listas vacías, condiciones de carrera si aplica.
- **¿Los tests existentes se ejecutaron de verdad?** Pide explícitamente que se ejecuten y se muestre el resultado, no una afirmación de que "deberían pasar". Un agente con acceso a ejecución puede y debe comprobarlo.

## 4. Lógica de negocio

Esta es la categoría más difícil de automatizar, porque requiere conocimiento que no está en el código: solo tú (o tu equipo) lo tenéis.

- **¿El comportamiento coincide con la regla de negocio real, no solo con lo que pediste literalmente?** Si pediste "que el descuento se aplique al total" y la regla real es que hay categorías de producto excluidas, un modelo no lo sabe si no se lo dices.
- **¿Se han manejado bien los casos frontera del dominio?** Por ejemplo, ¿qué pasa con un pedido de importe cero, con una fecha en el pasado, con un usuario sin ningún registro previo? Estas reglas suelen vivir en la cabeza del equipo, no en el código que el modelo puede leer.
- **¿Contradice alguna decisión previa del proyecto?** Un cambio puede ser correcto en aislamiento y aun así romper una convención o una invariante que el resto del sistema da por hecha.

## 5. Legibilidad y mantenimiento

No es solo estética: código difícil de entender es código difícil de revisar de verdad, y por tanto más probable que esconda un error que nadie detecta.

- ¿El cambio es del tamaño de un diff que puedes leer completo, con atención, en una sentada? Si es demasiado grande, pide que se trocee.
- ¿Se ha tocado solo lo necesario, o se ha aprovechado para reorganizar imports, renombrar variables o "mejorar" cosas alrededor que no venían al caso? Ese ruido dificulta ver qué cambió realmente.
- ¿El nombre de las funciones y variables refleja lo que hacen, o son nombres genéricos que obligan a leer el cuerpo para entender la intención?

## Una checklist para usar de verdad

- [ ] No hay concatenación de strings con datos externos en consultas o comandos.
- [ ] Los datos de entrada se validan, incluidos los casos vacíos o fuera de rango.
- [ ] No hay secretos ni credenciales escritos directamente en el código.
- [ ] Los endpoints sensibles siguen exigiendo la autenticación y los permisos correctos.
- [ ] Cualquier dependencia nueva es necesaria, existe de verdad y está mantenida.
- [ ] Los tests comprueban algo real y se han ejecutado, no solo escrito.
- [ ] El comportamiento coincide con la regla de negocio real, no solo con el encargo literal.
- [ ] El diff es del tamaño que puedes revisar completo y solo toca lo necesario.

## Conclusión

Nada de esto es exclusivo del código generado por IA — es, en el fondo, la misma disciplina de revisión que debería aplicarse a cualquier pull request. Lo que cambia es la tentación: cuando el código llega ya formateado, con nombres razonables y una explicación convincente, es más fácil bajar la guardia justo donde no deberías.

La pregunta que merece la pena hacerse antes de aprobar un cambio no es "¿tiene buena pinta?", sino **"¿puedo explicar por qué es correcto, punto por punto?"**. Si la respuesta es sí, adelante. Si no, esa es la parte que todavía te toca revisar a ti.
