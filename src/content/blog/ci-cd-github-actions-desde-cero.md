---
title: 'CI/CD con GitHub Actions desde cero: tutorial paso a paso'
description: 'Tutorial práctico para montar un pipeline de integración y despliegue continuo con GitHub Actions: build, tests y despliegue, con un workflow comentado línea a línea.'
pubDate: 'Jun 27 2026'
---

CI/CD son las siglas de *Continuous Integration* y *Continuous Deployment* (o *Delivery*), pero el nombre completo explica menos que un ejemplo concreto: cada vez que haces push, algo comprueba automáticamente que tu código sigue funcionando y, si corresponde, lo despliega — sin que nadie tenga que ejecutar esos pasos a mano.

Este tutorial monta ese flujo de principio a fin con GitHub Actions, que es la opción más directa cuando tu código ya vive en GitHub porque no requiere configurar nada externo.

## Por qué automatizar esto

Antes de un pipeline, el ciclo típico es: escribes código, ejecutas los tests tú mismo (si te acuerdas), hace el build a mano, y despliegas a mano. Cada uno de esos pasos manuales es una oportunidad para saltárselo por prisa, y es exactamente en los momentos de prisa cuando más falta hace no saltárselo.

Con CI/CD, esos pasos ocurren siempre, de la misma forma, sin depender de que alguien se acuerde. La integración continua (CI) comprueba que el código funciona en cada cambio; el despliegue continuo (CD) lo lleva a producción cuando esa comprobación pasa.

## Conceptos básicos de GitHub Actions

- **Workflow**: el proceso completo, definido en un archivo YAML dentro de `.github/workflows/`.
- **Trigger**: el evento que dispara el workflow (`push`, `pull_request`, una programación, o manualmente).
- **Job**: un conjunto de pasos que se ejecuta en una máquina virtual concreta. Un workflow puede tener varios jobs, y por defecto se ejecutan en paralelo.
- **Step**: cada acción individual dentro de un job (instalar dependencias, ejecutar tests...).
- **Action**: un paso reutilizable, publicado por GitHub o por la comunidad, que encapsula una tarea común (por ejemplo, `actions/checkout` para descargar el código del repositorio).

## Tu primer workflow: solo CI

Crea el archivo `.github/workflows/ci.yml`:

```yaml
name: CI

# Se ejecuta en cada push y en cada pull request contra main
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      # Descarga el código del repositorio en la máquina del runner
      - name: Checkout
        uses: actions/checkout@v4

      # Instala la versión de Python que use el proyecto
      - name: Configurar Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'

      - name: Instalar dependencias
        run: pip install -r requirements.txt

      - name: Ejecutar tests
        run: pytest
```

Con esto ya tienes CI de verdad: cada push a `main` y cada pull request contra `main` dispara este workflow, que instala las dependencias y ejecuta los tests en una máquina limpia. Si algo falla, el pull request lo muestra directamente, sin que nadie tenga que ir a mirarlo por su cuenta.

Fíjate en un detalle que evita muchos dolores de cabeza: los tests corren en una máquina **limpia**, sin nada del entorno de tu portátil. Si el proyecto funciona en tu máquina pero falla aquí, casi siempre es porque dependía de algo local que no está declarado en ningún sitio (una variable de entorno, una versión distinta de una herramienta).

## Cachear dependencias para builds más rápidos

Instalar todas las dependencias desde cero en cada ejecución es lento. `setup-python` puede cachear el gestor de paquetes directamente:

```yaml
- name: Configurar Python
  uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'
```

Con `cache: 'pip'` los pasos siguientes reutilizan la caché de paquetes descargados cuando el archivo de dependencias no ha cambiado, en lugar de descargarlos de nuevo cada vez. La diferencia se nota especialmente en proyectos con muchas dependencias.

## Varios jobs: separar build, test y lint

Un workflow no tiene por qué ser un único job. Separarlos deja el resultado más claro y permite que se ejecuten en paralelo cuando no dependen entre sí:

```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - run: pip install ruff
      - run: ruff check .

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: pip install -r requirements.txt
      - run: pytest
```

`lint` y `test` no dependen el uno del otro, así que GitHub Actions los ejecuta en paralelo por defecto, reduciendo el tiempo total de espera.

## Variables de entorno y secretos

Nunca escribas credenciales directamente en el YAML — quedaría en el historial de Git, visible para siempre. Los valores sensibles se guardan como **secrets** del repositorio (en `Settings → Secrets and variables → Actions`) y se referencian así:

```yaml
- name: Ejecutar tests con base de datos
  env:
    DATABASE_URL: ${{ secrets.DATABASE_URL }}
  run: pytest
```

Los secrets no aparecen en los logs del workflow, ni siquiera si el propio script los imprimiera por error: GitHub Actions los enmascara automáticamente.

## Añadir el despliegue (CD)

Una vez que CI funciona de forma fiable, el paso natural es añadir el despliegue como un job que solo se ejecuta si los anteriores han ido bien, y solo en la rama principal:

```yaml
jobs:
  test:
    # ... el job de test de antes ...

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Desplegar
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: |
          echo "Aquí van los comandos de despliegue reales:"
          echo "por ejemplo, construir y publicar una imagen, o llamar al CLI de tu plataforma de despliegue."
```

Dos claves de este bloque:

- **`needs: test`** hace que `deploy` espere a que `test` termine con éxito. Si los tests fallan, el despliegue simplemente no se ejecuta.
- **`if: github.ref == 'refs/heads/main'`** restringe el despliegue a la rama principal, para que un push a una rama de feature no despliegue nada por accidente.

El contenido exacto del paso de despliegue depende por completo de dónde despliegues: puede ser construir y publicar una imagen de Docker, llamar a la CLI de una plataforma de hosting, o ejecutar un script de despliegue propio. La estructura del workflow (esperar a los tests, restringir la rama) es la misma independientemente de eso.

## Un workflow completo, de principio a fin

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'
      - run: pip install -r requirements.txt
      - run: pytest

  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Desplegar
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
        run: echo "Comandos de despliegue aquí"
```

Cada push a una rama de feature o cada pull request ejecuta solo `test`. Cada push a `main`, tras pasar los tests, ejecuta también `deploy`. Es exactamente el ciclo que describía al principio del artículo, ahora como código versionado junto al proyecto.

## Buenas prácticas

- **Fija las versiones de las actions** (`actions/checkout@v4`, no `@main`), por la misma razón que fijas versiones de dependencias: reproducibilidad.
- **No dupliques configuración entre jobs** si crece mucho; GitHub Actions permite componer pasos reutilizables, pero para empezar, la duplicación explícita es más fácil de leer que una abstracción prematura.
- **Haz que el pipeline falle rápido.** Ejecuta primero lo más barato y rápido (lint) antes que lo más lento (tests de integración), para no esperar minutos a un fallo que un linter habría detectado en segundos.
- **Protege la rama principal.** En la configuración del repositorio, exige que el workflow de CI pase antes de poder fusionar un pull request. Sin eso, el pipeline existe pero nada obliga a respetarlo.

## Conclusión

Con esto tienes un pipeline funcional de principio a fin: cada cambio se comprueba automáticamente, y los que llegan a `main` y pasan esa comprobación se despliegan solos. Lo que cambia respecto al flujo manual no es solo la comodidad — es que la comprobación deja de depender de que alguien se acuerde de ejecutarla, que es precisamente el punto en el que suelen colarse los errores.
