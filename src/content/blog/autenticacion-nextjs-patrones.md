---
title: 'Autenticación en Next.js: patrones que funcionan'
description: 'Guía de los patrones habituales de autenticación en Next.js: sesiones frente a JWT, proveedores OAuth, protección de rutas y dónde guardar el token de forma segura.'
pubDate: 'Jul 14 2026'
---

La autenticación es de esas partes de una aplicación donde "que funcione" y "que sea segura" no son lo mismo, y la distancia entre ambas se paga cara si se descubre tarde. Esta guía recorre los patrones habituales para autenticar usuarios en Next.js: qué problema resuelve cada uno, sus ventajas y sus riesgos concretos.

## Sesiones vs JWT: la decisión de fondo

Todo lo demás se apoya en esta elección.

**Sesiones (con estado en el servidor).** Al iniciar sesión, el servidor genera un identificador de sesión, lo guarda (en memoria, Redis o una base de datos) junto con los datos del usuario, y le entrega al navegador solo ese identificador, normalmente en una cookie. En cada petición, el servidor busca la sesión correspondiente a ese identificador.

- **Ventaja principal**: se puede revocar al instante. Si borras la sesión del almacén, el usuario queda desconectado inmediatamente, sin importar cuántas peticiones haga.
- **Coste**: necesitas un almacén compartido (Redis es la opción habitual) si tu aplicación corre en varias instancias, porque todas necesitan poder consultar la misma sesión.

**JWT (JSON Web Tokens, sin estado).** El servidor genera un token firmado que contiene los datos del usuario directamente dentro. El cliente lo guarda y lo envía en cada petición; el servidor solo necesita verificar la firma, sin consultar ningún almacén.

- **Ventaja principal**: no necesita almacén compartido, lo cual simplifica escalar horizontalmente.
- **Coste**: revocar un token antes de que expire es difícil por diseño — el servidor no "recuerda" qué tokens emitió, así que no hay un sitio único donde borrarlo. Se puede mitigar con listas de revocación, pero eso reintroduce el estado compartido que el JWT pretendía evitar.

La regla práctica: si necesitas poder cerrar la sesión de un usuario de forma inmediata y fiable (por ejemplo, tras detectar actividad sospechosa), las sesiones con estado encajan mejor. Si tu prioridad es escalar sin almacén compartido y puedes convivir con tokens de vida corta, JWT tiene sentido — especialmente combinado con un token de refresco, como se ve más abajo.

## Dónde guardar el token en el cliente

Esta es la decisión donde más fallos de seguridad se cuelan, y merece su propio apartado.

**`localStorage`: la opción más simple y la menos recomendable.** Es accesible desde JavaScript, lo cual la hace muy cómoda de usar — y esa misma accesibilidad es el problema. Si tu aplicación tiene una vulnerabilidad XSS en cualquier punto (un script de un tercero, contenido no escapado correctamente), ese script puede leer `localStorage` y robar el token sin más obstáculos.

**Cookies `httpOnly`: la opción recomendada.** Una cookie marcada como `httpOnly` no es accesible desde JavaScript en absoluto — ni siquiera tu propio código puede leerla. El navegador la envía automáticamente en cada petición al dominio correspondiente, y un script malicioso inyectado por XSS no puede robarla porque, sencillamente, no puede verla.

```typescript
// Al iniciar sesión, en un Route Handler:
import { cookies } from 'next/headers';

export async function POST(request: Request) {
  const token = await generarToken(/* ... */);

  const cookieStore = await cookies();
  cookieStore.set('sesion', token, {
    httpOnly: true,
    secure: true,       // solo se envía por HTTPS
    sameSite: 'lax',    // mitiga CSRF en la mayoría de los casos
    path: '/',
  });

  return Response.json({ ok: true });
}
```

Tres flags que no son opcionales en producción: `httpOnly` (evita el robo por XSS), `secure` (evita que la cookie viaje por HTTP sin cifrar) y `sameSite` (mitiga ataques de tipo CSRF, donde otro sitio intenta hacer peticiones a tu API aprovechando que el navegador adjunta la cookie automáticamente).

## Proteger rutas con middleware

Next.js permite interceptar peticiones antes de que lleguen a la página con un `middleware.ts` en la raíz del proyecto. Es el sitio natural para comprobar autenticación de forma centralizada, en lugar de repetir la comprobación en cada página:

```typescript
import { NextResponse } from 'next/server';
import type { NextRequest } from 'next/server';

export function middleware(request: NextRequest) {
  const sesion = request.cookies.get('sesion');

  if (!sesion) {
    const url = new URL('/login', request.url);
    url.searchParams.set('redirect', request.nextUrl.pathname);
    return NextResponse.redirect(url);
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/cuenta/:path*'],
};
```

`matcher` limita en qué rutas se ejecuta el middleware — no tiene sentido pagar ese coste en páginas públicas. Un detalle importante: el middleware es un buen sitio para la comprobación rápida de "¿hay algo parecido a una sesión?", pero la validación completa (¿es válida de verdad? ¿ha expirado? ¿tiene los permisos que hacen falta para esta acción concreta?) debe repetirse en el propio servidor al ejecutar la acción, no confiarse únicamente al middleware.

## Server Components y Server Actions: comprobar en el sitio correcto

Con el App Router, gran parte de la lógica vive en el servidor por defecto, lo cual es una ventaja para autenticación: el código que decide qué puede ver o hacer un usuario no llega nunca al navegador.

```typescript
// app/dashboard/page.tsx (Server Component)
import { cookies } from 'next/headers';
import { redirect } from 'next/navigation';

export default async function Dashboard() {
  const cookieStore = await cookies();
  const sesion = cookieStore.get('sesion');

  if (!sesion) {
    redirect('/login');
  }

  const usuario = await validarSesion(sesion.value);

  return <div>Hola, {usuario.nombre}</div>;
}
```

Y en una Server Action, la comprobación de permisos debe repetirse siempre, sin asumir que si el usuario llegó hasta ahí es porque el middleware ya lo validó:

```typescript
'use server';

export async function eliminarCuenta(usuarioId: string) {
  const usuario = await obtenerUsuarioActual();

  if (usuario.id !== usuarioId && !usuario.esAdmin) {
    throw new Error('No autorizado');
  }

  // ... lógica de eliminación
}
```

La razón para no confiar solo en el middleware o en que "la página ya comprobó la sesión": una Server Action se puede invocar directamente, y el hecho de que el usuario haya llegado a ver la página no garantiza que tenga permiso para ejecutar esa acción concreta sobre ese recurso concreto. La comprobación de autorización pertenece al punto donde ocurre la acción, no solo a la puerta de entrada.

## Proveedores OAuth (Google, GitHub, etc.)

Delegar la autenticación en un proveedor externo evita tener que gestionar contraseñas propias — con todo lo que eso implica: hashing seguro, recuperación de contraseña, protección contra fuerza bruta. El flujo estándar (OAuth 2.0 / OpenID Connect) es, a grandes rasgos:

1. El usuario hace clic en "Continuar con Google" y se le redirige al proveedor.
2. El usuario se autentica en el proveedor (no en tu aplicación) y autoriza el acceso.
3. El proveedor redirige de vuelta a tu aplicación con un código de autorización.
4. Tu servidor intercambia ese código por un token, y con él obtiene los datos básicos del usuario (email, nombre).
5. Tu aplicación crea su propia sesión (o JWT) a partir de esos datos — el token del proveedor no sustituye a tu propio sistema de sesión, es solo la forma de confirmar la identidad.

Implementar este flujo a mano es posible pero laborioso y fácil de hacer mal en los detalles de seguridad (validación del `state` para evitar CSRF, verificación de la firma del token). En el ecosistema de Next.js existen librerías dedicadas a esto que gestionan el flujo completo con varios proveedores preconfigurados; para un proyecto real, conviene evaluarlas antes de escribir el flujo OAuth desde cero.

## Tokens de refresco: vida corta para el token de acceso

Un patrón habitual cuando se usa JWT es combinar dos tokens con vidas distintas:

- **Token de acceso**: vida corta (minutos), es el que se envía en cada petición.
- **Token de refresco**: vida larga (días o semanas), guardado de forma más restringida (cookie `httpOnly`), que solo se usa para pedir un nuevo token de acceso cuando el actual expira.

Esto acota el daño si un token de acceso se filtra: expira pronto por sí solo. El token de refresco, al usarse con mucha menos frecuencia y viajar solo hacia un endpoint concreto, tiene menos superficie de exposición.

## Checklist antes de dar por buena una implementación de autenticación

- [ ] El token o la sesión se guarda en una cookie `httpOnly`, no en `localStorage`.
- [ ] Las cookies llevan `secure` y `sameSite` configurados.
- [ ] Cada Server Action que modifica datos valida permisos en el propio servidor, no solo en el middleware.
- [ ] Las contraseñas, si las gestionas tú, se guardan con un algoritmo de hashing pensado para contraseñas (bcrypt, argon2), nunca en texto plano ni con un hash genérico como SHA-256 sin más.
- [ ] Hay una forma de revocar una sesión o token comprometido antes de que expire por sí solo.
- [ ] Los mensajes de error de login no revelan si el fallo fue el email o la contraseña (evita facilitar enumeración de usuarios).

## Conclusión

No existe un único patrón correcto de autenticación — sesiones, JWT y OAuth resuelven variantes distintas del mismo problema, y la elección depende de si necesitas revocación inmediata, de si escalas horizontalmente sin almacén compartido, o de si prefieres delegar la gestión de identidad en un proveedor externo. Lo que sí es constante entre todos los patrones es dónde se cuelan los fallos: guardar el token en un sitio accesible desde JavaScript, y confiar la autorización a una única comprobación de entrada en lugar de repetirla en cada acción sensible.
