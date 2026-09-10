---
title: 'Cómo orquestar tareas con Prefect: introducción práctica'
description: 'Tutorial de introducción a Prefect: qué problema resuelve un orquestador de tareas, flows y tasks, reintentos, programación y el panel de observabilidad.'
pubDate: 'Jun 23 2026'
---

Uso Prefect a diario para orquestar pipelines de machine learning en producción, así que esto no es una herramienta que haya probado una tarde: es la que tengo abierta la mayoría de los días. Lo que cuento aquí es la introducción que me hubiera gustado tener al empezar: qué problema resuelve exactamente un orquestador, y cómo se usa en la práctica con lo mínimo necesario para ser productivo.

## El problema que resuelve un orquestador

Imagina un proceso con varios pasos: descargar unos datos, transformarlos, entrenar o actualizar un modelo, y publicar el resultado. Puedes escribir eso como un script que llama a cuatro funciones seguidas. Funciona, hasta que necesitas alguna de estas cosas:

- Que se ejecute automáticamente cada noche, sin que nadie lo lance a mano.
- Que si un paso falla por un motivo transitorio (una API que no responde, una conexión que se cae), se reintente solo en lugar de fallar el proceso entero.
- Saber, sin adivinar, cuánto tardó cada paso la última vez y por qué falló ayer a las tres de la madrugada.
- Que dos pasos que no dependen entre sí se ejecuten en paralelo, y los que sí dependen esperen en el orden correcto.

Un script suelto no te da nada de esto gratis: hay que construirlo a mano, y es fácil que ese código "de infraestructura" acabe siendo más grande y más frágil que la lógica real del proceso. Un orquestador como Prefect asume esa parte por ti.

## Instalación

```bash
pip install -U prefect
```

Con eso ya tienes lo necesario para ejecutar flujos en local. Prefect también ofrece un servidor con interfaz web para ver el histórico de ejecuciones, que se levanta con:

```bash
prefect server start
```

## Los dos conceptos que hay que entender: `flow` y `task`

Todo en Prefect gira alrededor de estos dos decoradores.

Un **task** es la unidad de trabajo más pequeña: una función que hace una cosa concreta (descargar un archivo, consultar una base de datos, entrenar un modelo).

Un **flow** es la función que orquesta: llama a las tasks en el orden que corresponda y define el proceso completo.

```python
from prefect import flow, task


@task
def extraer_datos():
    print("Descargando datos...")
    return {"filas": 1000}


@task
def transformar(datos):
    print(f"Transformando {datos['filas']} filas...")
    return {"filas_validas": datos["filas"] - 10}


@task
def cargar(datos):
    print(f"Cargando {datos['filas_validas']} filas válidas.")


@flow(name="pipeline-diario")
def pipeline():
    datos_crudos = extraer_datos()
    datos_limpios = transformar(datos_crudos)
    cargar(datos_limpios)


if __name__ == "__main__":
    pipeline()
```

Ejecuta el script directamente (`python pipeline.py`) y ya tienes un flujo funcionando. La diferencia frente a llamar a esas mismas tres funciones seguidas en un script normal es que Prefect ahora conoce la estructura del proceso: sabe qué tareas hay, en qué orden se ejecutaron, cuánto tardó cada una y si alguna falló. Eso es lo que te permite construir todo lo que viene después.

## Reintentos: la razón principal para usar esto en producción

Cualquier paso que dependa de algo externo (una API, una base de datos, la red) puede fallar de forma transitoria. Repetirlo automáticamente, con una breve espera entre intentos, resuelve la mayoría de esos fallos sin intervención humana:

```python
@task(retries=3, retry_delay_seconds=10)
def consultar_api_externa():
    ...
```

Con eso, si la tarea lanza una excepción, Prefect la reintenta hasta tres veces, esperando diez segundos entre cada intento, antes de darla por fallida de verdad. Es una línea que sustituye bastante código de manejo de errores que, de otra forma, tendrías que escribir a mano en cada sitio donde pueda fallar algo.

## Ejecución en paralelo con `.submit()`

Si dos tareas no dependen entre sí, no hace falta que se ejecuten una detrás de otra. Usando `.submit()` en lugar de llamar a la task directamente, Prefect las lanza de forma concurrente:

```python
@flow
def pipeline_paralelo():
    resultado_a = tarea_a.submit()
    resultado_b = tarea_b.submit()

    # Aquí se espera a que ambas terminen antes de continuar
    combinar(resultado_a, resultado_b)
```

`.submit()` devuelve inmediatamente algo similar a un futuro; Prefect resuelve las dependencias automáticamente cuando pasas ese resultado a otra tarea, así que no hace falta gestionar la sincronización a mano.

## Programar ejecuciones

Para que un flujo se ejecute solo, sin intervención manual, se despliega con una programación asociada:

```python
from prefect import flow
from prefect.schedules import Cron


@flow
def pipeline():
    ...


if __name__ == "__main__":
    pipeline.serve(
        name="pipeline-diario",
        schedule=Cron("0 3 * * *"),  # todos los días a las 3:00
    )
```

Esto deja un proceso corriendo que espera a que llegue la hora programada y lanza el flujo. Para entornos de producción reales, lo habitual es desplegar esto en un worker gestionado en lugar de dejarlo corriendo en tu propia máquina, pero el concepto de fondo — la programación asociada a un flujo desplegado — es el mismo.

## El panel: por qué importa tanto como el código

Levanta el servidor local (`prefect server start`) y abre la URL que indica en la terminal. Ahí aparece el histórico de ejecuciones de cada flujo: estado, duración, logs de cada task y, si algo falló, el error concreto y en qué paso ocurrió.

Esto es, en la práctica, la razón principal por la que un orquestador da tranquilidad en producción frente a un script con un cron detrás: cuando algo falla a las tres de la madrugada, no hace falta ir a buscar en logs sueltos de un servidor — hay un sitio único donde ver qué pasó, en qué task, y con qué error exacto.

## Manejo de errores más allá del reintento automático

No todos los fallos se resuelven reintentando. Para esos casos, Prefect deja engancharte al ciclo de vida del flujo:

```python
from prefect import flow

def notificar_fallo(flow, flow_run, state):
    print(f"El flujo {flow.name} ha fallado: {state.message}")


@flow(on_failure=[notificar_fallo])
def pipeline_critico():
    ...
```

En un caso real, ese `notificar_fallo` normalmente envía una alerta (a un canal del equipo, a un sistema de monitorización) en lugar de solo imprimir por consola. La idea es la misma: que un fallo real, tras agotar los reintentos, no pase desapercibido.

## Cuándo tiene sentido introducir un orquestador

No todo necesita esto. Un script que ejecutas tú a mano de vez en cuando no lo necesita. La señal de que ha llegado el momento suele ser alguna de estas:

- El proceso tiene que ejecutarse solo, sin que nadie lo lance manualmente.
- Hay pasos que fallan de forma intermitente y quieres que se reintenten sin intervención.
- El proceso tiene varios pasos con dependencias entre sí, y necesitas ver claramente dónde falló cuando algo sale mal.
- Varios procesos comparten infraestructura y quieres un único sitio desde el que observarlos todos.

Si tu caso encaja en alguno de estos puntos, migrar de un script suelto a un flujo de Prefect suele ser cuestión de añadir un par de decoradores, no de reescribir la lógica.

## Conclusión

Lo que da valor a un orquestador no es la sintaxis — los decoradores `@flow` y `@task` son, deliberadamente, casi transparentes sobre el código que ya tendrías. El valor está en lo que obtienes gratis a cambio de usarlos: reintentos, paralelismo, programación y, sobre todo, un histórico donde ver qué pasó exactamente cuando algo falla. Para un proceso que corre sin supervisión, esa observabilidad no es un extra — es la diferencia entre depurar un fallo en minutos o perder la tarde reconstruyendo qué ocurrió a partir de logs sueltos.
