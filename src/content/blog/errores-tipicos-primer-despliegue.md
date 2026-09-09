---
title: 'Errores típicos al desplegar tu primer proyecto (y cómo los resolví)'
description: 'Cinco errores reales que te vas a encontrar al crear y desplegar tu primer proyecto web, con el síntoma, la causa y la solución exacta de cada uno.'
pubDate: 'Sep 07 2026'
---

Desplegar tu primer proyecto tiene una parte que nadie te cuenta: el código suele ser lo fácil. Lo que te hace perder la tarde son los detalles del entorno — el shell que usas, el sistema de archivos donde compilas, la configuración que sigue apuntando a un dominio de ejemplo.

Y hay algo importante que conviene decir antes de empezar: **encontrarte con estos errores no significa que lo estés haciendo mal**. Significa que estás saliendo del tutorial, donde todo funciona a la primera porque el autor ya limpió el camino. En cuanto tu entorno se desvía un poco del de la documentación —usas otro gestor de paquetes, otro shell, trabajas en una máquina virtual—, aparecen fricciones que ninguna guía anticipa.

En este artículo recojo los errores que me encontré montando este mismo blog con Astro, con el síntoma exacto, la causa real y la solución. Los tres primeros me pasaron literalmente a mí; los dos últimos son trampas clásicas de despliegue que conviene conocer antes de tropezar con ellas.

## Error 1: el instalador se niega a usar tu carpeta actual

**Síntoma.** Ya tenías la carpeta del proyecto creada, con un `README.md` y un `git init` hecho de antemano. Lanzas el asistente indicando el directorio actual:

```bash
pnpm create astro@latest .
```

Y el asistente se niega a continuar porque el directorio **no está vacío**, aunque le indiques explícitamente `.` como destino. No hay ninguna opción de "forzar" en el propio wizard.

**Causa.** Muchos generadores de proyectos (`create-astro`, `create-next-app` y compañía) comprueban que el destino esté vacío antes de escribir nada. Es una protección deliberada: evitan sobrescribirte archivos por accidente. El problema es que un flujo perfectamente razonable —crear el repo primero, con su README y su `.gitignore`, y añadir el proyecto después— choca de frente con esa comprobación.

**Solución.** Crear el proyecto en una carpeta temporal vacía y mover el contenido después:

```bash
pnpm create astro@latest devai-blog-temp
```

Con esto el asistente funciona sin quejarse. Lo interesante viene al mover los archivos, que es justo el siguiente error.

## Error 2: `zsh: event not found` al mover archivos ocultos

**Síntoma.** Tienes el proyecto en `devai-blog-temp/` y quieres mover todo su contenido a la carpeta raíz, incluidos los archivos ocultos (`.gitignore`, `.vscode/`...). Usas el patrón clásico para capturar dotfiles y el shell te responde con un error críptico:

```
zsh: event not found
```

**Causa.** El patrón habitual para archivos ocultos es `.[!.]*`, que excluye `.` y `..`. En **zsh**, el carácter `!` activa la expansión de historial (`!!`, `!123`, `!comando` para reejecutar comandos anteriores). El shell intenta interpretar `!.` como una referencia al historial, no la encuentra, y aborta antes siquiera de ejecutar `mv`. En bash el mismo patrón funciona sin problema, por eso circula tanto en guías antiguas: fueron escritas asumiendo bash.

**Solución.** En lugar de pelearte con el escapado, activa `dotglob`, que hace que el `*` normal incluya también los archivos ocultos:

```bash
setopt dotglob
mv devai-blog-temp/* .
unsetopt dotglob
rmdir devai-blog-temp
```

Con `dotglob` activado, `devai-blog-temp/*` captura tanto `src/` y `package.json` como `.gitignore` y `.vscode/`, y `mv` los mueve todos de una sola vez. Acuérdate de desactivarlo después: dejarlo activo cambia el comportamiento de todos los `*` de esa sesión, y eso puede darte sorpresas desagradables en un `rm`.

## Error 3: `Unknown command` al pasar flags a través del gestor de paquetes

**Síntoma.** Necesitas que el servidor de desarrollo escuche en todas las interfaces de red, no solo en `localhost`. Intentas pasar el flag a través del script de `package.json`:

```bash
pnpm run dev -- --host
```

Y obtienes un error de `Unknown command`.

**Causa.** El `--` es la convención para decir "lo que viene después son argumentos para el comando subyacente, no para el gestor de paquetes". Pero el comportamiento real depende del gestor, de su versión y de cómo esté definido el script: a veces el flag se reenvía, a veces se lo come el intermediario y el binario final recibe algo que no sabe interpretar. Es una fuente de fricción sorprendentemente frecuente al cambiar de npm a pnpm o a yarn.

**Solución.** Sáltate el script intermedio e invoca directamente el binario:

```bash
pnpm exec astro dev --host
```

`pnpm exec` ejecuta el binario del proyecto pasándole los argumentos tal cual, sin capa intermedia que los reinterprete. Con esto el servidor queda accesible desde otros equipos de la red.

En mi caso esto no era un capricho: trabajo en una máquina virtual de Linux y navego desde el navegador de Windows accediendo por la IP local de la VM. Sin `--host`, el servidor solo escucha en `localhost` y desde fuera de la VM no se ve absolutamente nada — la pestaña se queda cargando sin ningún error que te oriente.

## Error 4: el sitio se despliega, pero las URLs internas apuntan a otro dominio

**Síntoma.** El sitio compila y se despliega sin errores. Todo parece correcto hasta que miras el `sitemap.xml`, el feed RSS o las etiquetas `canonical` del HTML: apuntan a `https://example.com`, no a tu dominio real.

**Causa.** Las plantillas de Astro incluyen un dominio de ejemplo en `astro.config.mjs`:

```js
export default defineConfig({
  site: 'https://example.com',
  integrations: [mdx(), sitemap()],
});
```

Ese campo `site` no afecta a cómo se ve el sitio en el navegador, así que es fácil no darse cuenta de que sigue sin tocar. Pero es el que se usa para **generar URLs absolutas**: sitemap, canonicals y RSS. Y esas tres cosas son exactamente las que leen los buscadores.

Es un fallo silencioso: no rompe nada visible, no aparece en la consola y no falla el build. Simplemente le estás diciendo a Google que tu contenido canónico vive en otro dominio.

**Solución.** Actualiza `site` con la URL real en cuanto la tengas, y verifica el resultado en el build en lugar de fiarte:

```js
export default defineConfig({
  site: 'https://devai-blog.netlify.app',
  integrations: [mdx(), sitemap()],
});
```

```bash
pnpm astro build
grep -o '<loc>[^<]*' dist/sitemap-0.xml | head -3
```

Si ahí sigue apareciendo `example.com`, es que no has guardado el cambio o estás mirando un `dist/` antiguo.

## Error 5: funciona en tu máquina y falla en el build de producción

Este no me lo encontré en este proyecto, pero es probablemente el error de despliegue más común que existe, y merece la pena conocerlo antes de que te pase.

**Síntoma.** El proyecto compila perfectamente en local, haces push, y el build en el servidor falla con un error del tipo `Cannot find module './components/Header'` o `Module not found`. El archivo existe. Lo estás viendo en tu editor.

**Causa.** El sistema de archivos. macOS y Windows usan sistemas **insensibles a mayúsculas** por defecto: para ellos, `Header.astro` y `header.astro` son el mismo archivo. Linux —que es donde compilan Netlify, Vercel y prácticamente cualquier CI— **sí distingue mayúsculas**. Si tu archivo se llama `Header.astro` y en algún import escribiste `./header`, en tu portátil funciona y en el servidor no existe.

**Solución.** Comprueba que cada import coincide **carácter a carácter** con el nombre real del archivo. Y ojo con un detalle traicionero: si renombras un archivo cambiando solo mayúsculas, Git puede no registrar el cambio en un sistema insensible. En ese caso hay que forzarlo:

```bash
git mv --force header.astro Header.astro
```

La regla práctica que evita el problema entero: elige una convención de nombres (por ejemplo, `PascalCase` para componentes y `kebab-case` para todo lo demás) y respétala sin excepciones.

## Checklist antes de dar por bueno un despliegue

Que el build pase en verde no significa que el sitio esté bien. Antes de darlo por cerrado:

- [ ] **El build local pasa limpio** (`pnpm astro build` sin errores ni warnings que no entiendas).
- [ ] **El campo `site` apunta a tu dominio real**, no al de ejemplo de la plantilla.
- [ ] **El sitemap y el RSS contienen tu dominio.** Compruébalo en el HTML generado, no de memoria.
- [ ] **Los imports coinciden en mayúsculas** con el nombre real de cada archivo.
- [ ] **La versión de Node del servidor coincide con la tuya.** Si en local usas Node 22 y el servidor arranca con Node 18, vas a ver fallos que no tienen ningún sentido. Fíjala explícitamente (por ejemplo, con la variable `NODE_VERSION` en Netlify).
- [ ] **Has abierto el sitio ya desplegado**, no solo el preview local. Navega, entra en un artículo, prueba en móvil.
- [ ] **Ninguna clave ni archivo `.env` ha entrado en el repositorio.** Revisa `git status` antes de cada commit.

## Conclusión

Ninguno de estos cinco errores es un fallo de programación: son fricciones de entorno. Y esa es justo la razón por la que cuesta tanto encontrarlos buscando el mensaje de error en Google — el mensaje habla del síntoma (`event not found`, `Unknown command`) y nunca de la causa real (la expansión de historial de zsh, el reenvío de argumentos del gestor de paquetes).

La estrategia que mejor me funciona es esta: cuando algo falla de forma inexplicable, en lugar de buscar el mensaje literal, pregúntate **qué componente del entorno está entre tú y el comando que crees estar ejecutando**. Casi siempre la respuesta está ahí: el shell, el gestor de paquetes, el sistema de archivos o la configuración que nunca llegaste a tocar.

Si estás montando tu primer proyecto con Astro y quieres el proceso completo de principio a fin, lo tienes paso a paso en la [guía para crear un blog con Astro y desplegarlo en Netlify](/blog/crear-blog-astro-netlify/).
