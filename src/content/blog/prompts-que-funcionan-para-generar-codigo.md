---
title: 'Prompts que funcionan para generar código útil'
description: 'Guía práctica de cómo pedir código a un asistente de IA para obtener resultados usables: qué información incluir, ejemplos de prompts buenos y malos, y errores frecuentes.'
pubDate: 'Jul 27 2026'
---

La diferencia entre un prompt que produce código directamente usable y uno que produce algo que hay que reescribir casi entero rara vez está en la herramienta — está en cuánta información relevante contiene el encargo. Esta guía es una colección de patrones concretos, con ejemplos de prompts flojos y sus versiones mejoradas, para pedir código de forma que el resultado necesite menos retrabajo.

## El problema de fondo: un buen prompt sustituye el contexto que el modelo no tiene

Un asistente de IA no conoce tu proyecto por intuición. No sabe qué convenciones sigues, qué librerías ya usas, ni qué casos límite importan en tu dominio, salvo que se lo digas o que pueda leerlo directamente en el código. Un prompt vago obliga al modelo a rellenar esos huecos con suposiciones genéricas — y las suposiciones genéricas son, casi por definición, las que menos encajan con un proyecto concreto.

## Da el objetivo, no solo la instrucción literal

**Flojo:**
> Escribe una función que valide un email.

**Mejor:**
> Necesito validar el email en un formulario de registro. Que rechace formatos claramente inválidos, pero sin ser tan estricto que rechace emails reales con dominios poco comunes. Es para el frontend, en TypeScript.

La segunda versión no solo pide "una función" — explica para qué se usa, lo que permite al modelo elegir un nivel de rigor razonable (una validación de formato básica, no una verificación exhaustiva según el RFC completo de emails, que en la práctica suele rechazar direcciones válidas). El primer prompt puede darte cualquiera de los dos extremos, y no tienes forma de saber cuál hasta ver el resultado.

## Da el contexto técnico: lenguaje, librerías, versión

**Flojo:**
> Haz una petición HTTP a esta API.

**Mejor:**
> Haz una petición GET a esta API usando `fetch` nativo (no quiero añadir axios como dependencia), en un Server Component de Next.js 15 con App Router. Necesito manejar el caso de que la API devuelva un 429 con un reintento.

Sin ese contexto, el modelo tiene que adivinar el entorno de ejecución (¿cliente o servidor?), la librería preferida y si existen restricciones sobre añadir dependencias nuevas. Cada suposición equivocada es una ronda extra de corrección.

## Muestra el patrón existente, no lo describas de memoria

Cuando ya tienes una función o un componente similar en el proyecto, pegar ese ejemplo real en el prompt vale más que describir la convención de palabra:

**Flojo:**
> Sigue el estilo del resto del proyecto para este nuevo endpoint.

**Mejor:**
> Aquí tienes un endpoint existente del proyecto:
> ```python
> @router.get("/pedidos/{pedido_id}")
> def obtener_pedido(pedido_id: int, db: Session = Depends(get_db)):
>     pedido = db.query(Pedido).filter(Pedido.id == pedido_id).first()
>     if not pedido:
>         raise HTTPException(404, "Pedido no encontrado")
>     return pedido
> ```
> Necesito uno equivalente para `Cliente`, siguiendo exactamente el mismo patrón: mismo manejo de 404, misma forma de inyectar la sesión de base de datos.

Con el ejemplo real delante, el modelo no tiene que inferir la convención a partir de una descripción — la copia directamente, con mucho menos margen de error.

## Sé explícito sobre lo que NO debe tocar

**Flojo:**
> Arregla el bug en la función de cálculo de precios.

**Mejor:**
> Hay un bug en `calcular_precio_final`: no está aplicando el descuento por volumen correctamente. Arréglalo sin cambiar la firma de la función (otros módulos la llaman) ni tocar `calcular_impuestos`, que es una función delicada que no quiero modificar en este cambio.

Es uno de los ajustes con mayor retorno y menos esfuerzo: decir qué está fuera de los límites del encargo evita "mejoras" no pedidas en código adyacente, que luego hay que revisar igual y que amplían el diff sin necesidad.

## Pide el manejo de casos límite explícitamente

**Flojo:**
> Escribe una función que divida el total entre el número de participantes.

**Mejor:**
> Escribe una función que divida el total entre el número de participantes. Contempla: participantes = 0 (debe lanzar un error explícito, no devolver infinito o NaN), total negativo (no debería darse, pero que falle de forma clara si pasa), y que el resultado tenga como máximo dos decimales.

Sin especificarlo, es fácil que el código generado cubra solo el camino feliz — no porque el modelo no sepa manejar esos casos, sino porque no sabe si te importan en este contexto concreto, y por defecto tiende a producir la versión más simple que satisface el pedido literal.

## Pide que se verifique, no que se prometa

**Flojo:**
> Añade tests para esta función.

**Mejor:**
> Añade tests para esta función con pytest, cubriendo el caso normal, una lista vacía y un valor `None`. Ejecuta los tests después y confírmame que pasan.

Si el asistente tiene acceso a ejecutar código, pedir explícitamente que ejecute y muestre el resultado convierte una afirmación ("esto debería funcionar") en un hecho comprobado. Es la diferencia entre confiar en una promesa y ver la prueba.

## Trocea los encargos grandes

**Flojo:**
> Refactoriza el módulo de autenticación para usar el nuevo sistema de permisos.

**Mejor:**
> Primero, muéstrame qué archivos del módulo de autenticación referencian el sistema de permisos antiguo. Después iremos módulo a módulo.

Un encargo enorme y difuso produce, casi siempre, un resultado enorme y difícil de revisar — y si tú no puedes revisar el diff completo con atención en una sola sesión, tampoco vas a poder confiar en él del todo. Pedir primero una exploración y avanzar por partes da resultados más pequeños, más fáciles de verificar, y con margen para corregir el rumbo entre un paso y el siguiente.

## Tabla resumen: qué añadir y qué evitar

| Añade esto | En vez de esto |
|---|---|
| El objetivo real detrás del pedido | Solo la instrucción literal |
| Lenguaje, framework y versión concretos | Asumir que el modelo lo va a adivinar bien |
| Un ejemplo real del patrón del proyecto | Una descripción de palabra del estilo |
| Qué archivos o funciones no se deben tocar | Confiar en que no va a tocar nada de más |
| Los casos límite que te importan | Solo el caso normal |
| Petición explícita de ejecutar y verificar | Confiar en la promesa de que "debería funcionar" |
| Un encargo del tamaño de un diff revisable | Una tarea enorme y ambigua de una vez |

## Conclusión

No hay una fórmula mágica de palabras que garantice un buen resultado — lo que hay es una regla general: cuanta más de la información que tú tienes en la cabeza (el objetivo, las restricciones, las convenciones, los casos que importan) consiga pasar al prompt, menos tiene que adivinar el modelo, y menos rondas de corrección hacen falta después. Escribir un buen prompt no es un truco aparte de programar bien — es, en el fondo, la misma disciplina de especificar con precisión lo que ya deberías aplicar al escribir un ticket o una especificación para otra persona.
