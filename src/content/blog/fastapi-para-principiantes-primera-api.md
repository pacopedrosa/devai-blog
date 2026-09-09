---
title: 'FastAPI para principiantes: tu primera API en 15 minutos'
description: 'Tutorial práctico de FastAPI desde cero: instalación, primer endpoint, validación con Pydantic, documentación automática y por qué elegirlo frente a otras opciones de Python.'
pubDate: 'Aug 28 2026'
---

FastAPI se ha convertido en la opción por defecto para construir APIs en Python, y no por moda. Yo lo uso a diario para microservicios de machine learning en producción, y hay una razón muy concreta por la que se impuso tan rápido: **usa las anotaciones de tipos de Python para hacer trabajo real**, en lugar de tratarlas como documentación decorativa.

En este tutorial vas a construir una API funcional de principio a fin. Todo el código es ejecutable; puedes copiarlo tal cual e ir siguiéndolo.

## Por qué FastAPI y no otra cosa

Antes de escribir código, conviene entender qué te da FastAPI que no te dan las alternativas clásicas.

**Frente a Flask.** Flask es minimalista y flexible, pero no valida nada por ti. Si esperas un entero y llega la cadena `"abc"`, te enteras cuando revienta a mitad de la función. En FastAPI, la petición se rechaza antes de entrar en tu código, con un error claro que indica qué campo falla y por qué.

**Frente a Django REST Framework.** DRF es potentísimo, pero viene con todo el ecosistema Django detrás: ORM, migraciones, admin, settings. Para un microservicio que expone tres endpoints, es mucha maquinaria para poco.

**Lo que aporta FastAPI:**

- **Validación automática** a partir de las anotaciones de tipos, sin escribir código de validación.
- **Documentación interactiva gratis**: genera un esquema OpenAPI y una interfaz para probar la API desde el navegador, siempre sincronizada con el código.
- **Soporte nativo de async**, útil cuando tu API pasa la mayor parte del tiempo esperando a una base de datos o a otro servicio.
- **Autocompletado real en el editor**, porque todo está tipado de verdad.

Ese último punto se nota mucho más de lo que parece leído así. Cuando la respuesta de tu endpoint es un modelo tipado, el editor te dice qué campos tiene y te avisa si te equivocas al escribirlos.

## Instalación

Crea una carpeta, un entorno virtual e instala FastAPI:

```bash
mkdir mi-api && cd mi-api
python -m venv .venv
source .venv/bin/activate   # en Windows: .venv\Scripts\activate

pip install "fastapi[standard]"
```

El extra `[standard]` incluye el servidor Uvicorn y la CLI de FastAPI, que es lo que vas a usar para desarrollar.

Trabajar dentro de un entorno virtual no es opcional: sin él instalas paquetes a nivel global y acabas con conflictos de versiones entre proyectos. Es la primera costumbre que conviene automatizar.

## Tu primer endpoint

Crea un archivo `main.py`:

```python
from fastapi import FastAPI

app = FastAPI(title="Mi primera API")


@app.get("/")
def raiz():
    return {"mensaje": "La API funciona"}
```

Arráncalo:

```bash
fastapi dev main.py
```

Abre `http://127.0.0.1:8000` y ahí está tu respuesta JSON. El modo `dev` recarga automáticamente al guardar cambios.

Ahora abre `http://127.0.0.1:8000/docs`. Eso es documentación interactiva completa, generada a partir de tu código, sin que hayas escrito una sola línea para conseguirla. Puedes lanzar peticiones desde ahí mismo. A medida que añadas endpoints, se actualizará sola.

## Parámetros de ruta y de consulta

Los tipos que declaras no son adorno: FastAPI los usa para convertir y validar.

```python
@app.get("/articulos/{articulo_id}")
def obtener_articulo(articulo_id: int):
    return {"articulo_id": articulo_id, "tipo": type(articulo_id).__name__}
```

Visita `/articulos/42` y recibirás `articulo_id` como **entero**, no como cadena — FastAPI lo ha convertido por ti, porque lo anotaste como `int`.

Prueba ahora `/articulos/abc`. En lugar de una excepción a mitad de tu función, obtienes un `422` con un mensaje que explica exactamente qué esperaba y qué recibió. No has escrito nada para que eso ocurra.

Los parámetros que no aparecen en la ruta se interpretan como parámetros de consulta:

```python
@app.get("/articulos")
def listar_articulos(limite: int = 10, publicados: bool = True):
    return {"limite": limite, "publicados": publicados}
```

Una llamada a `/articulos?limite=5&publicados=false` te da `limite` como entero y `publicados` como booleano de Python. Y como ambos tienen valor por defecto, son opcionales.

## Validación con Pydantic: aquí está la potencia real

Para recibir datos en el cuerpo de la petición se declara un modelo de Pydantic. Aquí es donde FastAPI se separa de verdad de las alternativas:

```python
from fastapi import FastAPI
from pydantic import BaseModel, Field

app = FastAPI(title="Mi primera API")


class ArticuloNuevo(BaseModel):
    titulo: str = Field(min_length=3, max_length=120)
    contenido: str
    minutos_lectura: int = Field(gt=0, le=120)
    publicado: bool = False


@app.post("/articulos", status_code=201)
def crear_articulo(articulo: ArticuloNuevo):
    return {"creado": articulo.titulo, "minutos": articulo.minutos_lectura}
```

Con esas pocas líneas has definido un contrato completo. FastAPI ahora:

1. Comprueba que el cuerpo es JSON válido.
2. Verifica que están todos los campos obligatorios.
3. Convierte cada valor al tipo declarado.
4. Aplica las restricciones (`min_length`, `gt`, `le`...).
5. Devuelve un `422` detallado si algo falla, señalando el campo exacto.
6. Añade el esquema a la documentación automática.

Pruébalo con datos inválidos:

```bash
curl -X POST http://127.0.0.1:8000/articulos \
  -H "Content-Type: application/json" \
  -d '{"titulo": "ok", "contenido": "texto", "minutos_lectura": 0}'
```

La respuesta te dice que `titulo` es demasiado corto y que `minutos_lectura` debe ser mayor que cero. Ese es exactamente el código de validación que **no** has tenido que escribir.

## Controlar la respuesta con `response_model`

Igual que validas la entrada, conviene declarar la salida. Es especialmente importante para no filtrar campos sin querer:

```python
from pydantic import BaseModel, EmailStr


class UsuarioEntrada(BaseModel):
    email: EmailStr
    nombre: str
    password: str


class UsuarioSalida(BaseModel):
    email: EmailStr
    nombre: str


@app.post("/usuarios", response_model=UsuarioSalida)
def crear_usuario(usuario: UsuarioEntrada):
    # Aunque devuelvas el objeto completo, response_model filtra la salida:
    # el campo password NUNCA sale en la respuesta.
    return usuario
```

Este patrón —un modelo para entrada y otro para salida— es una de las mejores costumbres que puedes adoptar desde el principio. Convierte "no filtrar la contraseña" en algo garantizado por el tipo, en lugar de algo que hay que recordar en cada endpoint.

## Errores explícitos

Cuando algo no existe o no está permitido, se lanza una excepción HTTP:

```python
from fastapi import HTTPException

articulos_db = {1: {"titulo": "Primer artículo"}}


@app.get("/articulos/{articulo_id}")
def obtener_articulo(articulo_id: int):
    if articulo_id not in articulos_db:
        raise HTTPException(status_code=404, detail="Artículo no encontrado")
    return articulos_db[articulo_id]
```

## `async def` o `def`: cuál usar

Esta es la duda más habitual al empezar, y la regla práctica es sencilla:

- Usa **`async def`** cuando dentro del endpoint hagas operaciones de espera con librerías asíncronas (`await` a una base de datos async, a un cliente HTTP async).
- Usa **`def`** normal cuando trabajes con librerías síncronas o hagas cálculo intensivo.

Lo importante es que **no elijas `async def` "porque es más rápido"**. Si declaras un endpoint como `async` y dentro haces una llamada bloqueante, bloqueas el bucle de eventos y degradas el rendimiento de toda la aplicación. Con `def` normal, FastAPI ejecuta la función en un pool de hilos y no bloquea nada.

En caso de duda: `def` normal. Es la opción segura.

## Un ejemplo algo más realista

Juntando las piezas, así queda un servicio pequeño pero completo:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel, Field

app = FastAPI(title="API de artículos", version="1.0.0")


class Articulo(BaseModel):
    id: int
    titulo: str = Field(min_length=3, max_length=120)
    minutos_lectura: int = Field(gt=0, le=120)


class ArticuloNuevo(BaseModel):
    titulo: str = Field(min_length=3, max_length=120)
    minutos_lectura: int = Field(gt=0, le=120)


db: dict[int, Articulo] = {}
siguiente_id = 1


@app.get("/salud")
def salud():
    return {"estado": "ok"}


@app.get("/articulos", response_model=list[Articulo])
def listar(limite: int = 10):
    return list(db.values())[:limite]


@app.post("/articulos", response_model=Articulo, status_code=201)
def crear(datos: ArticuloNuevo):
    global siguiente_id
    articulo = Articulo(id=siguiente_id, **datos.model_dump())
    db[siguiente_id] = articulo
    siguiente_id += 1
    return articulo


@app.delete("/articulos/{articulo_id}", status_code=204)
def borrar(articulo_id: int):
    if articulo_id not in db:
        raise HTTPException(status_code=404, detail="Artículo no encontrado")
    del db[articulo_id]
```

Un aviso para que no te lleves una sorpresa: ese diccionario `db` vive **en memoria**. Se borra al reiniciar el proceso y no se comparte entre varios workers. Para cualquier cosa real necesitas una base de datos de verdad — PostgreSQL es la opción por defecto razonable.

El endpoint `/salud` tampoco es decorativo: es la convención estándar para que orquestadores y balanceadores comprueben si tu servicio sigue vivo. Cuesta tres líneas y lo vas a necesitar en cuanto despliegues.

## Siguientes pasos

Con esto ya tienes una API funcionando y validada. Los pasos naturales a partir de aquí son:

1. **Persistencia real** con PostgreSQL, mediante SQLAlchemy o SQLModel.
2. **Autenticación**, normalmente con tokens JWT.
3. **Separar el código en módulos** con `APIRouter` cuando el archivo empiece a crecer.
4. **Tests** con `pytest` y el `TestClient` que trae FastAPI.
5. **Despliegue**, y aquí lo estándar es meterlo en un contenedor.

Para ese último punto, en la [guía de Docker para developers](/blog/docker-para-developers-guia-sin-humo/) hay un Dockerfile comentado línea a línea, construido precisamente para una API de FastAPI como esta.
