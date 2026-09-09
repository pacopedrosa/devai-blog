---
title: 'Astro vs Next.js: cuándo usar cada uno'
description: 'Comparativa práctica entre Astro y Next.js: rendimiento, casos de uso, curva de aprendizaje y una recomendación clara según el tipo de proyecto que vayas a construir.'
pubDate: 'Sep 04 2026'
---

"¿Astro o Next.js?" es una pregunta mal planteada, aunque se haga constantemente. Es como preguntar si es mejor un destornillador o una llave inglesa: depende por completo de lo que tengas delante.

Uso los dos en contextos distintos. Este blog está hecho con Astro. En mi trabajo construyo aplicaciones full-stack con Next.js. Y la elección en cada caso no fue una cuestión de preferencia, sino de qué tipo de cosa estaba construyendo.

Este artículo explica esa diferencia para que puedas decidir sin tener que probar los dos.

## La diferencia de fondo: qué asume cada uno por defecto

Todo lo demás se deriva de aquí.

**Astro asume que tu página es contenido.** Por defecto renderiza a HTML estático y envía **cero JavaScript** al navegador. Si un componente necesita interactividad, se lo indicas explícitamente y solo ese componente se hidrata. Es el modelo de *islands*: un mar de HTML estático con islas puntuales de interactividad.

**Next.js asume que tu página es una aplicación.** Está construido sobre React, con todo lo que eso implica: estado compartido, componentes que reaccionan a la interacción del usuario, navegación del lado del cliente. Con el App Router y los React Server Components la cantidad de JavaScript enviado ha bajado mucho, pero el marco mental sigue siendo el de una aplicación React.

Fíjate en la asimetría, porque es la clave práctica: en Astro, la interactividad es la excepción que declaras. En Next.js, es el punto de partida.

## Rendimiento

Aquí conviene ser preciso, porque se repite mucho "Astro es más rápido" sin matizar y eso lleva a decisiones malas.

**Para sitios de contenido, Astro gana con claridad.** No por magia, sino por aritmética: si no envías JavaScript, no hay nada que descargar, parsear ni ejecutar. Eso se traduce directamente en mejores métricas de Core Web Vitals, sobre todo en móviles modestos y conexiones lentas. Un blog en Astro puede servir páginas con literalmente cero KB de JS.

**Para aplicaciones interactivas, la ventaja se diluye.** Si tu página necesita un dashboard con estado compartido, filtros que se comunican entre sí y actualizaciones en tiempo real, ese JavaScript hay que enviarlo igualmente. Construir eso con islas de Astro es posible, pero acabas peleándote para compartir estado entre islas — justo el problema que React resuelve de forma natural.

La conclusión honesta: **la ventaja de rendimiento de Astro es enorme cuando tu página es mayoritariamente contenido, y tiende a desaparecer a medida que la interactividad crece**. Elegir Astro para una aplicación muy interactiva no te va a dar el rendimiento que viste en los benchmarks de blogs.

## Comparativa directa

| | **Astro** | **Next.js** |
|---|---|---|
| **Modelo mental** | HTML por defecto, JS opcional | Aplicación React |
| **JS enviado por defecto** | Cero | El runtime de React (reducido con RSC) |
| **Ideal para** | Blogs, docs, landings, marketing, portfolios, e-commerce con catálogo | Dashboards, SaaS, apps con sesión, e-commerce con carrito complejo |
| **Framework de UI** | React, Vue, Svelte, Solid o ninguno | React |
| **Backend / API** | Endpoints básicos; no es su foco | Route Handlers, Server Actions, middleware |
| **Autenticación y sesiones** | Posible, pero manual | Ecosistema maduro |
| **Contenido en Markdown** | Nativo, con colecciones tipadas | Requiere librerías externas |
| **Curva de aprendizaje** | Suave si sabes HTML/CSS | Media-alta: RSC, caché, límites cliente/servidor |
| **Despliegue** | Estático en cualquier CDN | Óptimo en plataformas con soporte para su runtime |

## Cuándo elegir Astro

Elige Astro si tu proyecto es **principalmente contenido que se lee**:

- Blogs y publicaciones. Trae colecciones de contenido con esquemas tipados y validados, generación de RSS y sitemap. Este blog usa exactamente eso.
- Documentación técnica.
- Landings y sitios de marketing, donde cada kilobyte impacta en la conversión y el SEO.
- Portfolios y webs corporativas.
- Tiendas donde el catálogo es sobre todo navegación y ficha de producto.

La señal clara: **si puedes describir tu sitio como "páginas que la gente lee, con algo de interactividad puntual", Astro es la opción correcta.**

Un detalle que suele decidir la elección: Astro es agnóstico respecto al framework de UI. Puedes usar componentes React donde de verdad los necesites y HTML plano en el resto. No estás obligado a meter React en toda la casa para tener un formulario interactivo en una esquina.

## Cuándo elegir Next.js

Elige Next.js si tu proyecto es **una aplicación con la que se interactúa**:

- Dashboards y paneles de administración.
- Productos SaaS con usuarios, sesiones y permisos.
- Aplicaciones con datos que cambian en tiempo real.
- Cualquier cosa con formularios complejos, estado compartido entre vistas y flujos de varios pasos.
- Proyectos donde el frontend y el backend viven en el mismo repositorio y quieres Server Actions o Route Handlers.

La señal clara: **si el usuario va a pasar la mayor parte del tiempo interactuando en lugar de leyendo, y hay sesión de por medio, Next.js.**

Hay un factor adicional que no es técnico pero pesa: el ecosistema. Autenticación, pasarelas de pago, ORMs, componentes... la mayoría de librerías del mundo React documentan primero para Next.js. En un proyecto de trabajo con plazos, eso cuenta.

## Curva de aprendizaje

**Astro es más fácil de empezar**, sobre todo si vienes de HTML y CSS. Los archivos `.astro` son HTML con un bloque de JavaScript arriba para los datos. El modelo de islas es explícito y se entiende rápido: si no pones una directiva, no hay JavaScript. Puedes ser productivo el primer día.

**Next.js exige más.** No es que sea inaccesible, pero hay conceptos que hay que interiorizar sí o sí: la diferencia entre Server Components y Client Components, cuándo hace falta `"use client"`, cómo funciona el caché, qué código acaba en el servidor y cuál en el navegador. Esos límites son la fuente principal de errores de quien empieza.

Dicho esto, la curva de Next.js es una inversión con retorno: son conceptos que estructuran cómo piensas las aplicaciones web modernas en general.

## No es obligatorio elegir uno solo

Un punto que se pasa por alto: **muchos proyectos usan los dos**. Es un patrón perfectamente razonable tener el sitio de marketing, el blog y la documentación en Astro —optimizados para SEO y velocidad— y la aplicación en sí en Next.js, sirviéndose desde un subdominio o una ruta.

Cada parte usa la herramienta adecuada y ninguna paga el coste de la otra. Si tu proyecto tiene una parte pública de contenido y otra privada de aplicación, seguramente sea la mejor opción disponible.

## Recomendación práctica

Resumido en decisiones concretas:

- **Blog, documentación o landing** → Astro, sin dudarlo.
- **Aplicación con login, sesiones y datos del usuario** → Next.js.
- **Sitio de contenido con algún componente interactivo suelto** (buscador, formulario, calculadora) → Astro, con islas de React solo donde haga falta.
- **Aplicación muy interactiva con algo de contenido** → Next.js para todo; no merece la pena partirlo.
- **Ambas cosas, y son grandes** → Astro para el contenido, Next.js para la aplicación.
- **No lo tienes claro y es un proyecto pequeño** → empieza por Astro. Es más fácil añadir React donde haga falta que quitarlo de donde sobra.

Y una advertencia final: casi ninguna decisión de framework es tan irreversible como parece cuando la estás tomando. Se pierde bastante más tiempo comparando que migrando después un proyecto pequeño. Si dudas, elige el que mejor encaje con el tipo de proyecto según la tabla de arriba, y ponte a construir.
