---
title: 'Docker para developers que nunca lo han usado (guía sin humo)'
description: 'Qué es Docker realmente, la diferencia entre imagen y contenedor, un Dockerfile comentado línea a línea y los comandos que de verdad usas cada día.'
pubDate: 'Sep 01 2026'
---

Docker es una de esas herramientas que llevas años viendo en ofertas de trabajo y en READMEs ajenos, y que sigues posponiendo porque las explicaciones que encuentras o son demasiado abstractas ("contenedorización de cargas de trabajo") o demasiado profundas (namespaces y cgroups del kernel de Linux).

Esta guía va por otro camino. Uso Docker a diario para desplegar plataformas de IA full-stack en producción, y lo que necesitas para empezar a ser productivo es bastante menos de lo que parece: entender qué problema resuelve, tener clara una única distinción conceptual, y manejar unos ocho comandos.

## El problema que resuelve Docker

"En mi máquina funciona."

Ese es literalmente el problema. Tu aplicación no es solo tu código: es tu código **más** una versión concreta de Python o Node, más un conjunto de librerías del sistema, más unas variables de entorno, más una configuración concreta. Cuando mueves el código a otra máquina —la de un compañero, un servidor, un runner de CI— todo eso cambia, y las cosas se rompen de formas difíciles de diagnosticar.

Docker resuelve esto empaquetando la aplicación **junto con su entorno completo** en una unidad que se ejecuta igual en cualquier sitio donde haya Docker instalado.

La comparación que se suele hacer es con las máquinas virtuales, y es útil para entender la diferencia clave: una VM incluye un sistema operativo entero, con su propio kernel. Arranca en minutos y ocupa gigas. Un contenedor comparte el kernel del sistema anfitrión y solo aísla el espacio de usuario: arranca en milisegundos y puede ocupar decenas de megas. Por eso es viable tener quince contenedores corriendo en tu portátil, y no quince máquinas virtuales.

## La distinción que importa: imagen vs contenedor

Si te quedas con una sola cosa de este artículo, que sea esta.

Una **imagen** es una plantilla inmutable de solo lectura: el sistema de archivos empaquetado con tu aplicación, sus dependencias y las instrucciones de arranque. No se ejecuta. Existe en disco.

Un **contenedor** es una instancia en ejecución de una imagen. Tiene su propio sistema de archivos escribible (una capa fina por encima de la imagen), sus procesos y su red.

La analogía habitual es la de clase e instancia en programación orientada a objetos: la imagen es la clase, el contenedor es el objeto. Otra que funciona bien: la imagen es la receta y el contenedor el plato concreto que has cocinado. De una misma receta puedes hacer muchos platos, y si tiras uno a la basura la receta sigue intacta.

Esto tiene una consecuencia práctica muy importante: **los contenedores son desechables**. Todo lo que escribas dentro de un contenedor desaparece cuando lo eliminas, salvo que lo guardes explícitamente fuera (con volúmenes, que veremos más abajo). No es un bug, es el modelo: si un contenedor se estropea, no lo reparas, lo tiras y levantas otro.

## Comprobar que lo tienes instalado

```bash
docker --version
docker run hello-world
```

El segundo comando descarga una imagen mínima de prueba y la ejecuta. Si ves un mensaje de bienvenida, Docker funciona.

Fíjate en lo que ha pasado ahí, porque es el ciclo completo en miniatura: Docker no encontró la imagen `hello-world` en local, la descargó de Docker Hub (el registro público de imágenes), creó un contenedor a partir de ella, lo ejecutó y el contenedor terminó.

Prueba ahora algo más interesante:

```bash
docker run -it ubuntu bash
```

Estás dentro de un shell de Ubuntu, aunque tu máquina sea otra distribución o macOS. Haz `ls`, crea un archivo, y sal con `exit`. Ese sistema de archivos ya no existe: acabas de comprobar en la práctica que los contenedores son desechables.

## Tu primer Dockerfile, comentado línea a línea

Un `Dockerfile` es la receta: un archivo de texto con las instrucciones para construir tu imagen. Vamos con uno real para una API de Python con FastAPI, comentado paso a paso:

```dockerfile
# Imagen base: Python 3.12 en su variante "slim" (mucho más ligera que la
# completa, porque prescinde de herramientas de compilación y documentación).
FROM python:3.12-slim

# Directorio de trabajo dentro del contenedor. Los comandos siguientes
# se ejecutan aquí, y evita tener que escribir rutas absolutas.
WORKDIR /app

# Copiamos SOLO el archivo de dependencias, antes que el código.
# Esto no es un capricho de orden: es la optimización más importante
# del archivo. Lo explico justo debajo.
COPY requirements.txt .

# Instalamos dependencias. --no-cache-dir evita guardar el caché de pip
# dentro de la imagen, que ya no vas a necesitar y solo ocupa espacio.
RUN pip install --no-cache-dir -r requirements.txt

# Ahora sí, copiamos el código de la aplicación.
COPY . .

# Documenta que la aplicación escucha en el puerto 8000. Es informativo:
# no publica el puerto por sí solo, eso se hace al ejecutar con -p.
EXPOSE 8000

# Comando que se ejecuta al arrancar el contenedor.
# --host 0.0.0.0 es imprescindible: si escuchara solo en localhost,
# escucharía dentro del contenedor y no podrías acceder desde fuera.
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Por qué el orden de las instrucciones importa tanto

Cada instrucción del Dockerfile crea una **capa**, y Docker cachea las capas. Al reconstruir, reutiliza todas las capas anteriores al primer cambio y solo rehace de ahí en adelante.

Por eso copiamos `requirements.txt` antes que el código. Tus dependencias cambian cada pocas semanas; tu código cambia cada pocos minutos. Con este orden, editar un archivo `.py` invalida solo la capa del `COPY . .` y la instalación de dependencias se reutiliza del caché: la reconstrucción tarda segundos.

Si copiaras todo de golpe (`COPY . .` antes del `RUN pip install`), cualquier cambio mínimo en el código invalidaría la capa de dependencias y reinstalarías todo desde cero en cada build. Es la diferencia entre esperar tres segundos o dos minutos, cada vez que cambias una línea.

### El `.dockerignore`

Igual de importante y mucho menos conocido. Funciona como un `.gitignore` y evita que basura innecesaria acabe dentro de la imagen:

```
.git
.venv
__pycache__
*.pyc
.env
node_modules
```

Esto no es solo cuestión de tamaño: `.env` en esa lista es una cuestión de **seguridad**. No quieres tus credenciales horneadas dentro de una imagen que quizá acabes publicando en un registro.

## Construir y ejecutar

```bash
# Construye la imagen y la etiqueta con un nombre (-t de "tag").
# El punto final es el contexto de build: el directorio que se envía a Docker.
docker build -t mi-api .

# Ejecuta un contenedor a partir de la imagen.
# -p 8000:8000 mapea el puerto 8000 de tu máquina al 8000 del contenedor.
docker run -p 8000:8000 mi-api
```

Abre `http://localhost:8000` y ahí está tu aplicación, corriendo dentro del contenedor.

El mapeo de puertos (`-p host:contenedor`) es de las cosas que más confunden al principio. El contenedor tiene su propia red aislada: aunque tu app escuche en el 8000 dentro del contenedor, desde tu máquina no existe hasta que publiques ese puerto explícitamente. Puedes mapearlo a otro distinto — `-p 3000:8000` te lo sirve en `localhost:3000`.

## Los comandos que usas de verdad cada día

Toda la documentación del mundo, y al final acabas usando estos:

```bash
# Ver contenedores en ejecución (añade -a para ver también los parados)
docker ps
docker ps -a

# Ver los logs de un contenedor. -f los sigue en tiempo real,
# como un tail -f. Es tu primera parada cuando algo no funciona.
docker logs -f <nombre_o_id>

# Abrir un shell DENTRO de un contenedor que ya está corriendo.
# Imprescindible para depurar: te deja mirar desde dentro.
docker exec -it <nombre_o_id> bash

# Parar y eliminar un contenedor
docker stop <nombre_o_id>
docker rm <nombre_o_id>

# Listar y borrar imágenes
docker images
docker rmi <imagen>

# Liberar espacio: elimina contenedores parados, redes sin usar,
# imágenes huérfanas y caché de build. Docker acumula MUCHO con el tiempo.
docker system prune -a
```

Dos consejos prácticos sobre estos comandos. El primero: usa `--name` al lanzar contenedores (`docker run --name api -p 8000:8000 mi-api`) y te ahorras copiar identificadores hexadecimales todo el rato. El segundo: `docker run --rm` elimina el contenedor automáticamente al terminar, perfecto para pruebas rápidas que no quieres ir limpiando después.

## Volúmenes: cuando sí necesitas que algo persista

Si los contenedores son desechables, ¿dónde guardas la base de datos? En un **volumen**: almacenamiento gestionado por Docker que vive fuera del ciclo de vida del contenedor.

```bash
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secreto \
  -v datos_pg:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:16
```

Aquí `datos_pg` es un volumen con nombre. Puedes destruir y recrear el contenedor tantas veces como quieras: los datos siguen ahí.

Existe una variante muy útil en desarrollo, el *bind mount*, que monta una carpeta de tu máquina dentro del contenedor:

```bash
docker run -p 8000:8000 -v $(pwd):/app mi-api
```

Con eso, editas un archivo en tu editor y el cambio se refleja dentro del contenedor al instante, sin reconstruir la imagen. Es el patrón estándar para desarrollar dentro de contenedores.

## Cuando son varios servicios: Docker Compose

En cuanto tu proyecto tiene API más base de datos, lanzar contenedores a mano se vuelve incómodo. Docker Compose describe todo el conjunto en un archivo:

```yaml
services:
  api:
    build: .
    ports:
      - '8000:8000'
    environment:
      DATABASE_URL: postgresql://postgres:secreto@db:5432/app
    depends_on:
      - db

  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secreto
    volumes:
      - datos_pg:/var/lib/postgresql/data

volumes:
  datos_pg:
```

Y todo el stack arranca con un comando:

```bash
docker compose up      # añade -d para dejarlo en segundo plano
docker compose down    # para y elimina todo
```

Fíjate en un detalle importante: la API se conecta a la base de datos usando `db` como nombre de host, no `localhost`. Compose crea una red interna donde cada servicio es alcanzable por su nombre. Es de las cosas que más despistan al empezar.

## Buenas prácticas que evitan problemas

- **Usa imágenes base ligeras.** `python:3.12-slim` en lugar de `python:3.12` reduce el tamaño de forma notable, y menos superficie significa también menos vulnerabilidades que parchear.
- **Fija las versiones.** `python:3.12-slim`, no `python:latest`. Con `latest` tu build deja de ser reproducible: la misma orden puede dar resultados distintos con un mes de diferencia.
- **No ejecutes como root.** Por defecto los procesos del contenedor corren como root. Crea un usuario sin privilegios y cambia a él antes del `CMD`.
- **Nunca metas secretos en la imagen.** Ni en el Dockerfile ni copiando un `.env`. Pásalos como variables de entorno en tiempo de ejecución.
- **Aprende builds multi-etapa** cuando necesites compilar algo. Compilas en una imagen con todas las herramientas y copias solo el resultado a una imagen final mínima.

## Por dónde seguir

Con lo de aquí ya puedes contenerizar una aplicación y trabajar con ella cómodamente. El siguiente paso natural, cuando eso te resulte cómodo, es integrarlo en CI/CD: construir la imagen automáticamente en cada push y desplegarla.

Y si quieres una aplicación real que meter dentro de un contenedor, en la [guía de FastAPI para principiantes](/blog/fastapi-para-principiantes-primera-api/) construimos exactamente la API que usa el Dockerfile de este artículo.
