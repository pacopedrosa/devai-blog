---
title: 'Colas de tareas con RabbitMQ y Dramatiq: cuándo las necesitas'
description: 'Qué problema resuelve una cola de tareas, cómo encajan RabbitMQ y Dramatiq, un ejemplo práctico y las señales que indican que ha llegado el momento de introducirlas.'
pubDate: 'Jul 06 2026'
---

Uso Dramatiq con RabbitMQ en producción para procesar trabajo en segundo plano, así que esto no es una tecnología que haya mirado por encima: es la que tengo delante cuando algo tarda demasiado para resolverse dentro de una petición HTTP. Este artículo explica el concepto —qué es una cola de tareas y qué problema resuelve— y cuándo tiene sentido introducir una, sin entrar en los detalles de ningún proyecto concreto.

## El problema: trabajo que no cabe dentro de una petición

Un servidor web está pensado para responder rápido. Cuando un endpoint hace algo que tarda varios segundos —procesar un archivo grande, enviar un correo, llamar a una API externa lenta, ejecutar un cálculo pesado— tienes un conflicto: el cliente está esperando una respuesta, y esa respuesta no puede llegar tan rápido como debería.

Las malas soluciones a esto son conocidas: subir el timeout del servidor (el cliente sigue esperando, solo que más), o lanzar un hilo suelto dentro de la propia petición (y perder ese trabajo si el proceso se reinicia, sin ninguna garantía de que termine).

La solución que de verdad resuelve el problema es sacar ese trabajo **fuera** del ciclo petición-respuesta: el endpoint encola la tarea y responde inmediatamente ("recibido, se está procesando"), y un proceso separado —un *worker*— recoge esa tarea de la cola y la ejecuta a su ritmo, sin que nadie quede esperando.

## Las dos piezas: el broker y el worker

Una cola de tareas necesita dos componentes que juegan roles distintos.

**El broker** es el intermediario que almacena las tareas pendientes y las entrega a quien las va a procesar. **RabbitMQ** es uno de los brokers de mensajería más usados: recibe mensajes de quien los produce (tu aplicación) y los entrega a quien los consume (los workers), garantizando que un mensaje no se pierde si no hay ningún worker disponible en ese momento — se queda en la cola hasta que alguno lo recoja.

**El worker** es el proceso que ejecuta el trabajo real. Se conecta al broker, recoge tareas de la cola una a una (o varias en paralelo) y las ejecuta. **Dramatiq** es una librería de Python que define cómo se declaran esas tareas y gestiona el ciclo de vida del worker: reintentos, límites de tiempo, concurrencia.

Es la misma separación conceptual que en cualquier sistema productor-consumidor: tu aplicación produce tareas, el broker las almacena y ordena, y uno o varios workers las consumen. La ventaja de tenerlos como piezas separadas es que puedes escalar cada una de forma independiente — añadir más workers cuando hay mucho trabajo pendiente, sin tocar la aplicación que los genera.

## Un ejemplo con Dramatiq

Primero, se declara el broker (RabbitMQ, en este caso) y una tarea:

```python
import dramatiq
from dramatiq.brokers.rabbitmq import RabbitmqBroker

broker = RabbitmqBroker(host="localhost")
dramatiq.set_broker(broker)


@dramatiq.actor(max_retries=3)
def enviar_email(destinatario: str, asunto: str):
    print(f"Enviando '{asunto}' a {destinatario}...")
    # aquí iría la llamada real al servicio de envío de correo
```

El decorador `@dramatiq.actor` convierte una función normal en una tarea que se puede encolar en lugar de ejecutar directamente. `max_retries=3` le dice a Dramatiq que, si la función lanza una excepción, la reintente automáticamente hasta tres veces antes de darla por fallida.

Desde el resto de la aplicación (por ejemplo, un endpoint), en lugar de llamar a la función directamente, se encola:

```python
@app.post("/registro")
def registrar_usuario(datos: UsuarioNuevo):
    crear_usuario(datos)
    enviar_email.send(datos.email, "Bienvenido")
    return {"estado": "registrado"}
```

`.send()` no ejecuta `enviar_email` ahí mismo: publica un mensaje en RabbitMQ y devuelve el control inmediatamente. El endpoint responde al cliente sin haber esperado a que el correo se envíe. En paralelo, un proceso worker independiente recoge ese mensaje y ejecuta la función de verdad:

```bash
dramatiq mi_modulo
```

Ese comando arranca el worker, que se queda escuchando la cola y procesando tareas a medida que llegan — completamente desacoplado del proceso que atiende las peticiones HTTP.

## Por qué esto es mejor que un hilo suelto

La diferencia clave frente a lanzar un hilo dentro de la petición es la **persistencia**. Si el mensaje ya está en RabbitMQ y el worker se cae a mitad de procesarlo, el broker puede reentregar ese mensaje a otro worker (o al mismo, al reiniciar) en lugar de perderlo sin más. Con un hilo en memoria, si el proceso muere, esa tarea desaparece sin dejar rastro, y no hay ninguna forma de saber que hacía falta reintentarla.

Esa garantía de "el trabajo no se pierde, aunque algo falle a mitad" es, en el fondo, la razón de ser de todo este patrón.

## Otras cosas que resuelve bien

- **Tareas programadas y periódicas.** Procesos que deben ejecutarse a una hora concreta sin que nadie los dispare manualmente.
- **Control de concurrencia.** Limitar cuántas tareas de un tipo se ejecutan a la vez, útil cuando el trabajo llama a una API externa con límites de peticiones.
- **Priorización.** Colas separadas para trabajo urgente y trabajo que puede esperar, de forma que lo importante no quede atascado detrás de un lote grande de tareas de baja prioridad.
- **Picos de carga.** Si llegan mil peticiones de golpe que generan mil tareas, estas se acumulan en la cola y los workers las van procesando a su ritmo, en lugar de que el servidor intente hacerlo todo a la vez y se sature.

## Cuándo NO hace falta

Introducir un broker y workers añade una pieza más de infraestructura que hay que desplegar, monitorizar y mantener. No compensa en todos los casos:

- Si la operación tarda milisegundos, no hay ningún problema que resolver.
- Si el proyecto es pequeño y no hay volumen de trabajo en segundo plano, la complejidad añadida no se justifica todavía.
- Si necesitas el resultado de forma síncrona, inmediatamente, dentro de la misma petición — una cola encaja con trabajo que se puede procesar "en algún momento cercano", no con algo que el cliente necesita ya.

## Señales de que ha llegado el momento

- Un endpoint tarda varios segundos porque hace algo que no necesita respuesta inmediata (enviar notificaciones, generar un informe, procesar un archivo subido).
- Estás lanzando hilos o procesos sueltos dentro de las peticiones y ya te ha pasado que se pierde trabajo si el servidor se reinicia.
- Necesitas reintentar automáticamente operaciones que fallan por causas transitorias (una API externa caída un momento, una conexión de red inestable).
- El volumen de trabajo en segundo plano ha crecido lo suficiente como para necesitar escalar esa parte por separado del resto de la aplicación.

## Conclusión

Una cola de tareas no es una pieza exótica de infraestructura: es la solución estándar a un problema muy concreto — trabajo que no debería bloquear a quien lo pide. RabbitMQ se encarga de que ese trabajo no se pierda entre que se encola y se procesa; Dramatiq se encarga de que declarar y ejecutar esas tareas sea tan simple como escribir una función con un decorador encima. La complejidad que añaden merece la pena exactamente cuando el problema que resuelven ya existe en tu proyecto — ni antes, ni después.
