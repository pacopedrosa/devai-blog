---
title: 'Server-Sent Events (SSE) explicado con un caso práctico'
description: 'Qué son los Server-Sent Events, en qué se diferencian de WebSockets, y cómo implementar una barra de progreso en tiempo real con un ejemplo completo de servidor y cliente.'
pubDate: 'Jul 10 2026'
---

Cuando se necesita que el servidor envíe actualizaciones al cliente sin que este las pida una y otra vez, la primera opción que suele venir a la cabeza es WebSockets. Pero WebSockets es una solución bidireccional completa para un problema que, muchas veces, solo va en una dirección: el servidor informa, el cliente escucha. Para ese caso concreto, **Server-Sent Events (SSE)** es una opción más simple, nativa del navegador y con mucho menos código de por medio.

Este tutorial explica el concepto y lo construye con un ejemplo concreto: una barra de progreso que se actualiza en tiempo real mientras el servidor procesa una tarea larga.

## Qué es SSE, en una frase

SSE es un mecanismo para que el servidor envíe eventos al cliente a través de una **conexión HTTP normal**, mantenida abierta, en una sola dirección: del servidor al navegador. No hace falta ningún protocolo especial ni librería adicional — es HTTP con un tipo de contenido concreto (`text/event-stream`) y una API del navegador (`EventSource`) que sabe interpretarlo.

## SSE frente a WebSockets: cuándo cada uno

| | **SSE** | **WebSockets** |
|---|---|---|
| **Dirección** | Servidor → cliente, únicamente | Bidireccional |
| **Protocolo** | HTTP normal | Su propio protocolo, sobre una conexión que empieza como HTTP |
| **Reconexión automática** | Sí, integrada en el navegador | Hay que implementarla a mano |
| **Formato de datos** | Texto | Texto o binario |
| **Complejidad de implementación** | Baja | Mayor |
| **Casos de uso típicos** | Notificaciones, progreso de una tarea, feeds en vivo | Chats, juegos en tiempo real, colaboración con edición simultánea |

La pregunta que decide cuál usar es simple: **¿el cliente necesita enviar datos al servidor por ese mismo canal, además de recibirlos?** Si la respuesta es no —el cliente solo necesita enterarse de cosas que pasan en el servidor—, SSE resuelve el problema con bastante menos complejidad. Si el cliente también necesita enviar datos continuamente por el mismo canal (un chat, por ejemplo), WebSockets es la herramienta correcta.

## El formato del protocolo

Un evento SSE es texto plano con un formato muy simple:

```
data: este es un mensaje

data: {"progreso": 50}

event: completado
data: {"resultado": "ok"}

```

Cada evento termina con una línea en blanco. El campo `data` lleva el contenido (normalmente JSON); el campo opcional `event` permite distinguir tipos de evento en el cliente, para reaccionar de forma distinta según cuál llegue.

## Caso práctico: progreso de una tarea en tiempo real

El ejemplo: el cliente dispara una tarea larga (por ejemplo, procesar un archivo) y quiere ver una barra de progreso que se actualiza sola, sin refrescar la página ni hacer polling cada segundo.

### Servidor (Python, con Flask)

```python
from flask import Flask, Response
import time

app = Flask(__name__)


def generar_progreso():
    for porcentaje in range(0, 101, 10):
        yield f"data: {{\"progreso\": {porcentaje}}}\n\n"
        time.sleep(0.5)  # simula trabajo real en curso

    yield "event: completado\ndata: {}\n\n"


@app.route("/progreso")
def progreso():
    return Response(generar_progreso(), mimetype="text/event-stream")
```

Tres detalles que hacen que esto funcione:

- **`mimetype="text/event-stream"`** es lo que le dice al navegador que interprete la respuesta como un flujo de eventos SSE, y no como una respuesta normal.
- **El uso de un generador** (`yield`) es lo que permite enviar datos progresivamente, según se van produciendo, en lugar de acumularlos y mandarlos todos de golpe al final.
- **El formato exacto** (`data: ...\n\n`) tiene que respetarse; el salto de línea doble es el que marca dónde termina cada evento.

### Cliente (JavaScript, con `EventSource`)

```javascript
const barra = document.getElementById('barra-progreso');
const origen = new EventSource('/progreso');

origen.onmessage = (evento) => {
  const datos = JSON.parse(evento.data);
  barra.value = datos.progreso;
};

origen.addEventListener('completado', () => {
  console.log('Tarea terminada');
  origen.close();
});

origen.onerror = () => {
  console.error('Conexión con el servidor perdida');
};
```

`EventSource` es una API nativa del navegador: no hace falta ninguna librería para consumir SSE. `onmessage` recibe los eventos sin nombre (el `data:` suelto del ejemplo del servidor); `addEventListener('completado', ...)` recibe específicamente los eventos etiquetados como `completado`. Al llegar ese evento, se cierra la conexión con `.close()` porque ya no hace falta seguir escuchando.

## La reconexión automática, y por qué importa

Es la característica de SSE que más tiempo ahorra y menos se menciona: si la conexión se corta —por una red inestable, un reinicio del servidor— el navegador **reintenta la conexión automáticamente**, sin que tengas que escribir ningún código para ello. Con WebSockets, esa lógica de reconexión hay que implementarla a mano.

Esto tiene una consecuencia práctica que conviene tener en cuenta: tras una reconexión automática, el flujo del servidor **empieza de nuevo** salvo que gestiones explícitamente dónde se quedó el cliente. Para eso existe el campo `id` en el protocolo y la cabecera `Last-Event-ID`, que el navegador envía automáticamente al reconectar para que el servidor sepa desde dónde continuar. Para el caso de una tarea de progreso como la de este ejemplo, normalmente basta con que el servidor vuelva a enviar el último estado conocido al reconectar.

## Limitaciones que conviene conocer

- **Solo texto.** Si necesitas enviar datos binarios, hay que codificarlos (por ejemplo, en base64), lo cual añade sobrecarga.
- **Una sola dirección.** Si el cliente necesita responder por el mismo canal, SSE no es la herramienta — habría que combinarlo con peticiones HTTP normales aparte, o usar WebSockets directamente.
- **Límite de conexiones simultáneas por dominio en HTTP/1.1.** Los navegadores limitan cuántas conexiones SSE (y HTTP en general) puede haber abiertas a la vez contra el mismo dominio. En HTTP/2 este límite prácticamente desaparece, así que en producción conviene servir la aplicación sobre HTTP/2.
- **Algunos proxies y balanceadores de carga cortan conexiones largas por defecto.** Si SSE no funciona en producción aunque funcione en local, es el primer sitio donde mirar: hay que configurar explícitamente que esas rutas permitan conexiones de larga duración.

## Conclusión

SSE ocupa un hueco muy concreto y muy común: actualizaciones del servidor hacia el cliente, sin necesidad de que el cliente hable por el mismo canal. Para ese caso —notificaciones, progreso de una tarea, un feed que se actualiza solo— resuelve el problema con una API nativa del navegador, reconexión automática incluida, y sin la complejidad adicional de gestionar un protocolo bidireccional que en realidad no necesitas. Antes de reservar WebSockets para un problema, merece la pena confirmar si de verdad hace falta esa bidireccionalidad, o si el flujo real es de servidor a cliente y nada más.
