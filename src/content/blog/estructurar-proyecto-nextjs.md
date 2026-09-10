---
title: 'Cómo estructurar un proyecto Next.js real'
description: 'Buenas prácticas de organización de carpetas y convenciones para un proyecto Next.js que crece: dónde colocar componentes, lógica de negocio, tipos y utilidades.'
pubDate: 'Jul 22 2026'
---

Next.js no impone una estructura de carpetas más allá de lo que necesita el propio router (`app/`), y esa libertad es cómoda al empezar y un problema en cuanto el proyecto crece sin ninguna convención. Esta guía recoge una organización que escala razonablemente bien, con el razonamiento detrás de cada decisión — para que puedas adaptarla, no solo copiarla.

## El principio de fondo: separar rutas de todo lo demás

La carpeta `app/` en el App Router tiene un trabajo muy concreto: definir rutas. Cada carpeta dentro de `app/` es un segmento de URL, y eso es una responsabilidad ya de por sí suficiente. El error más habitual al empezar es dejar que `app/` cargue además con toda la lógica de negocio, los componentes reutilizables y las utilidades — mezclando "qué URLs existen" con "cómo funciona la aplicación", que son dos preguntas distintas.

La estructura que mejor separa ambas cosas:

```
mi-proyecto/
├── app/                    # Solo rutas: páginas, layouts, route handlers
│   ├── (marketing)/
│   │   ├── page.tsx
│   │   └── precios/
│   │       └── page.tsx
│   ├── (app)/
│   │   ├── dashboard/
│   │   │   └── page.tsx
│   │   └── cuenta/
│   │       └── page.tsx
│   ├── api/
│   │   └── webhooks/
│   │       └── route.ts
│   ├── layout.tsx
│   └── globals.css
├── components/             # Componentes de UI reutilizables
│   ├── ui/                 # Componentes genéricos: Button, Input, Card...
│   └── dashboard/          # Componentes específicos de una sección
├── lib/                    # Lógica de negocio y utilidades
│   ├── db.ts
│   ├── auth.ts
│   └── validaciones.ts
├── types/                  # Tipos compartidos entre varias partes
│   └── index.ts
└── middleware.ts
```

## Grupos de rutas: `(marketing)` y `(app)`

Los paréntesis en el nombre de una carpeta dentro de `app/` crean un **grupo de rutas**: organizan el código sin afectar a la URL final. `app/(marketing)/precios/page.tsx` sigue sirviéndose en `/precios`, no en `/marketing/precios`.

La utilidad real de esto no es solo cosmética: permite que cada grupo tenga su propio `layout.tsx`. La parte de marketing (landing, precios) normalmente necesita un layout distinto al de la aplicación autenticada (con barra lateral, navegación de usuario). Sin grupos de rutas, ese layout compartido se vuelve un condicional creciente dentro de un único archivo; con grupos, cada sección tiene el suyo, limpio.

```
app/
├── (marketing)/
│   ├── layout.tsx     # Layout público: header simple, footer
│   └── page.tsx
└── (app)/
    ├── layout.tsx     # Layout de la app: sidebar, navegación de usuario
    └── dashboard/
        └── page.tsx
```

## `components/`: separar lo genérico de lo específico

Una distinción que evita que esta carpeta se convierta en un cajón de sastre: separar componentes **genéricos**, sin conocimiento de ningún dominio concreto (un botón, un input, una tarjeta), de componentes **específicos** de una funcionalidad (una tabla de pedidos, un formulario de checkout).

```
components/
├── ui/                    # No saben nada del dominio de la app
│   ├── button.tsx
│   ├── input.tsx
│   └── card.tsx
└── dashboard/              # Conocen el dominio: pedidos, usuarios...
    ├── tabla-pedidos.tsx
    └── resumen-ventas.tsx
```

Un componente en `ui/` debería poder copiarse a otro proyecto sin cambiar una línea. Si un componente de `ui/` empieza a importar algo de `lib/` relacionado con el dominio (por ejemplo, el tipo `Pedido`), es la señal de que en realidad pertenece a una carpeta más específica, no a la genérica.

## `lib/`: donde vive la lógica que no es de interfaz

Aquí va todo lo que no es un componente visual: acceso a la base de datos, llamadas a APIs externas, validaciones, funciones de utilidad. Es la carpeta que hace que la lógica de negocio sea testeable de forma aislada, sin tener que renderizar nada.

```
lib/
├── db.ts             # Conexión y queries a la base de datos
├── auth.ts           # Lógica de autenticación
├── validaciones.ts   # Esquemas de validación (con Zod, por ejemplo)
└── api/
    └── pagos.ts       # Cliente para un servicio de pagos externo
```

La prueba práctica de si algo está bien colocado aquí: si puedes importar esa función en un test unitario sin necesidad de renderizar ningún componente de React, probablemente está en el sitio correcto.

## Colocar código junto a la ruta que lo usa, cuando es exclusivo de ella

Next.js permite colocar archivos auxiliares dentro de una carpeta de ruta sin que se conviertan en rutas ellos mismos, siempre que no se llamen `page.tsx`, `layout.tsx` u otro nombre reservado:

```
app/(app)/dashboard/
├── page.tsx
├── _components/
│   └── grafico-ventas.tsx   # Solo se usa en esta página
└── _lib/
    └── calcular-metricas.ts
```

El guion bajo inicial (`_components`, `_lib`) es una convención para marcar explícitamente "esto no es una ruta", y evita cualquier ambigüedad con el router. La regla para decidir entre esto y las carpetas globales (`components/`, `lib/`) es sencilla: si algo se usa en un único sitio, vive junto a ese sitio; en cuanto se necesita en un segundo lugar, se promueve a la carpeta compartida.

## `types/`: tipos que cruzan varias partes del proyecto

Los tipos que solo se usan dentro de un componente o una función pueden (y deben) declararse ahí mismo, junto a su uso. Los que se comparten entre varias partes —el modelo de un `Usuario`, la respuesta de una API— merecen un sitio centralizado:

```typescript
// types/index.ts
export interface Usuario {
  id: string;
  nombre: string;
  email: string;
  rol: 'admin' | 'miembro';
}
```

Evita duplicar la misma forma de datos en varios archivos con nombres ligeramente distintos — es una fuente silenciosa de bugs cuando un campo cambia en un sitio y no en los demás.

## Route Handlers: solo para lo que de verdad necesita ser una API

Con Server Actions disponibles para mutaciones desde formularios y componentes, `app/api/` debería reservarse para lo que realmente necesita ser un endpoint HTTP: webhooks de servicios externos, integraciones que otro sistema va a llamar, o endpoints que consume un cliente que no es la propia aplicación Next.js (una app móvil, por ejemplo).

```
app/api/
└── webhooks/
    └── stripe/
        └── route.ts
```

Usar un Route Handler para una mutación que solo dispara tu propio formulario, cuando una Server Action haría lo mismo con menos código de por medio (sin tener que gestionar manualmente el `fetch` desde el cliente), es añadir una capa que no aporta nada.

## Convenciones de nombres

- **Componentes**: `PascalCase` para el nombre exportado, y `kebab-case` para el nombre del archivo (`tabla-pedidos.tsx` exportando `TablaPedidos`) es la convención más extendida en proyectos Next.js recientes, aunque `PascalCase` también en el archivo es perfectamente válido — lo importante es elegir una y no mezclarlas.
- **Utilidades y hooks**: `camelCase` (`formatearFecha.ts`, `usePedidos.ts`).
- **Constantes**: `SCREAMING_SNAKE_CASE` para valores verdaderamente constantes (`MAX_INTENTOS = 3`).

## Una señal de que la estructura necesita revisarse

Si para hacer un cambio pequeño en una funcionalidad concreta tienes que tocar archivos en cinco carpetas distintas sin relación aparente entre sí, es una señal de que la organización actual no refleja cómo se agrupa realmente el trabajo en tu proyecto. La estructura de carpetas no es un fin en sí misma — es una herramienta para que encontrar y modificar código sea rápido, y cuando deja de cumplir eso, merece revisarse, aunque sea "la forma estándar" de organizarlo.

## Conclusión

No existe una única estructura correcta para un proyecto Next.js, pero sí un principio que se sostiene en casi todos los proyectos que crecen bien: mantener `app/` centrado en rutas, sacar la lógica de negocio a `lib/` donde se puede testear de forma aislada, y separar los componentes genéricos de los específicos de cada funcionalidad. El resto —nombres exactos de carpetas, cuándo colocar algo localmente o promoverlo a compartido— son detalles que conviene decidir una vez, como equipo, y respetar con consistencia a partir de ahí.
