---
title: 'Cómo crear un blog gratis con Astro y desplegarlo en Netlify (guía 2026)'
description: 'Guía práctica, paso a paso y con los errores reales que me encontré, para montar un blog con Astro, subirlo a GitHub y desplegarlo gratis en Netlify.'
pubDate: 'Sep 09 2026'
---

Si quieres un blog rápido, gratis y sin mantener un servidor propio, la combinación **Astro + Netlify** es de las mejores opciones en 2026. Astro genera HTML estático (ideal para Core Web Vitals y SEO) y Netlify lo despliega automáticamente cada vez que haces push a GitHub, sin coste, con HTTPS y CDN incluidos.

Este mismo blog, DevAI Blog, está montado exactamente así. En esta guía te cuento el proceso paso a paso, incluyendo los tropiezos reales con los que me encontré (y cómo los resolví) para que no pierdas el tiempo que perdí yo.

## Por qué Astro + Netlify para un blog

- **Gratis para empezar**: el plan gratuito de Netlify cubre de sobra un blog personal (build automático, CDN, HTTPS).
- **Rápido de verdad**: Astro renderiza a HTML estático por defecto, así que no hay JavaScript innecesario cargando en el navegador del lector.
- **Sin servidor que mantener**: nada de VPS, nginx ni actualizaciones de sistema. Git push y listo.
- **Escrito en Markdown**: los artículos son archivos `.md`/`.mdx`, cómodos para versionar con Git igual que el resto del código.

## Requisitos previos

- **Node.js** 22.12 o superior (Astro 7 lo exige).
- **pnpm** como gestor de paquetes. Yo uso pnpm en lugar de npm porque es más rápido y ahorra espacio en disco con su almacén de dependencias compartido. Si no lo tienes instalado:

```bash
corepack enable
corepack prepare pnpm@latest --activate
pnpm --version
```

- Una cuenta de **GitHub** y otra de **Netlify** (puedes crear la de Netlify con tu cuenta de GitHub directamente).

## Crear el proyecto Astro paso a paso

El asistente oficial de creación de proyectos permite elegir plantilla desde el propio terminal. Yo elegí la plantilla **"blog"** (no la básica), porque ya trae de fábrica el listado de posts, el layout de artículo, RSS y colecciones de contenido en Markdown configuradas — es decir, justo lo que necesita un blog, sin tener que montarlo desde cero.

```bash
pnpm create astro@latest devai-blog-temp
```

Durante el asistente:

1. Selecciona la plantilla **Blog**.
2. Confirma **TypeScript** y el modo **Strict**.
3. Deja que instale las dependencias (`pnpm install`).
4. Puedes decir que no a inicializar un repositorio Git nuevo si ya vas a gestionarlo tú manualmente (ver la sección de errores más abajo, es justo lo que me pasó).

## Errores comunes y cómo los resolví

### 1. El asistente se niega a crear el proyecto en la carpeta actual

Yo ya había hecho `git init` en la carpeta del proyecto y tenía un `README.md` y un `.gitignore` creados de antemano. Al lanzar:

```bash
pnpm create astro@latest .
```

el asistente detecta que el directorio **no está vacío** y se niega a continuar, aunque le indiques explícitamente "." como destino. No hay ninguna opción de "forzar" en el propio wizard.

**Solución**: crear el proyecto en una carpeta temporal vacía y luego mover los archivos a la carpeta real:

```bash
pnpm create astro@latest devai-blog-temp
```

### 2. Mover los archivos (incluidos los ocultos) sin que zsh se queje

Una vez creado el proyecto en `devai-blog-temp/`, el siguiente paso es mover todo su contenido a la carpeta raíz del repositorio, incluyendo archivos ocultos como `.gitignore` o `.vscode/`. El patrón clásico `.[!.]*` para capturar dotfiles **falla en zsh** con un error críptico:

```
zsh: event not found
```

Esto ocurre porque zsh interpreta el `!` como expansión de historial de comandos, no como parte del patrón glob. La forma más simple de evitarlo es activar `dotglob`, que hace que el `*` normal también capture archivos ocultos:

```bash
setopt dotglob
mv devai-blog-temp/* .
unsetopt dotglob
rmdir devai-blog-temp
```

Con `dotglob` activado, `devai-blog-temp/*` incluye tanto los archivos normales (`src/`, `package.json`...) como los ocultos (`.gitignore`, `.vscode/`), y `mv` los mueve todos de una sola vez a la carpeta actual.

### 3. Arrancar el servidor de desarrollo accesible desde otro equipo de la red

Yo trabajo en una VM de Linux, pero navego desde el navegador de Windows accediendo por la IP local de la VM. Por defecto, `astro dev` solo escucha en `localhost`, así que hace falta el flag `--host` para que escuche en todas las interfaces de red.

El problema es que pasar el flag a través del script de pnpm no funciona como cabría esperar:

```bash
pnpm run dev -- --host
```

Esto devuelve un error de `Unknown command`. La forma que sí funciona es invocar directamente el binario de Astro con `pnpm exec`, saltándote el script intermedio de `package.json`:

```bash
pnpm exec astro dev --host
```

Con esto, el servidor de desarrollo queda accesible desde `http://<IP-de-la-VM>:4321` y puedes abrirlo desde el navegador de Windows sin problema.

## Subir el proyecto a GitHub

Con el proyecto ya funcionando localmente, el siguiente paso es subirlo a un repositorio de GitHub:

```bash
git add .
git commit -m "chore: initial Astro blog setup"
```

Crea el repositorio en GitHub (puedes hacerlo desde la web, o con la CLI `gh` si la tienes instalada):

```bash
gh repo create devai-blog --public --source=. --remote=origin
git push -u origin main
```

Si prefieres no usar la CLI de GitHub, crea el repositorio vacío desde github.com, copia la URL y ejecuta:

```bash
git remote add origin https://github.com/<tu-usuario>/devai-blog.git
git push -u origin main
```

## Desplegar en Netlify paso a paso

1. Entra en [app.netlify.com](https://app.netlify.com) e inicia sesión con tu cuenta de GitHub.
2. Pulsa **Add new site → Import an existing project**.
3. Autoriza a Netlify a acceder a tu cuenta de GitHub y selecciona el repositorio `devai-blog`.
4. Netlify suele detectar Astro automáticamente y rellenar la configuración de build. Si no lo hace, indícala a mano:
   - **Build command**: `pnpm run build`
   - **Publish directory**: `dist`
5. Como usamos pnpm, conviene asegurarse de que Netlify lo use en el build. En la configuración del sitio (**Site settings → Build & deploy → Environment**), añade la variable de entorno:
   - `NODE_VERSION` → `22`
   
   Netlify detecta el `pnpm-lock.yaml` del repositorio y usa pnpm automáticamente para instalar dependencias; si no fuera así, puedes forzarlo añadiendo la variable `NETLIFY_USE_PNPM` a `true`.
6. Pulsa **Deploy site**. Netlify instalará dependencias, ejecutará `astro build` y publicará el contenido de `dist/` en su CDN.

A partir de aquí, cada `git push` a la rama `main` dispara un nuevo build y despliegue automático, sin que tengas que tocar nada más.

## Conclusión y próximos pasos

En menos de una hora tienes un blog funcionando en producción, gratis, rápido y con despliegue continuo desde Git. Los tropiezos que me encontré —el asistente rechazando una carpeta no vacía, el `event not found` de zsh al mover dotfiles, y el `--host` que solo funciona con `pnpm exec`— son justo el tipo de fricción que no aparece en la documentación oficial pero que te hace perder media tarde si no sabes qué está pasando.

Con el sitio ya desplegado, los siguientes pasos naturales son:
- Configurar un **dominio propio** en Netlify en lugar de la URL `.netlify.app` por defecto.
- Añadir las páginas legales necesarias (política de privacidad, contacto) si piensas monetizar con publicidad.
- Empezar a escribir contenido real y consistente antes de solicitar la revisión de servicios como Google AdSense.
