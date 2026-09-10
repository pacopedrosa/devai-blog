---
title: 'TypeScript en modo strict: por qué merece la pena'
description: 'Qué activa realmente el modo strict de TypeScript, ejemplos de errores que detecta antes de ejecutar el código, y por qué compensa incluso en proyectos pequeños.'
pubDate: 'Jul 01 2026'
---

Es habitual empezar un proyecto con TypeScript y dejar `strict` desactivado, sobre todo si se viene de JavaScript y el objetivo es "ir rápido". El problema es que sin `strict`, TypeScript te da mucha menos protección de la que crees que tienes — y la sensación de seguridad, sin la seguridad real detrás, es el peor de los dos mundos.

## Qué es exactamente `strict`

En `tsconfig.json`, `strict` no es una comprobación aislada: es un interruptor que activa un conjunto de comprobaciones a la vez.

```json
{
  "compilerOptions": {
    "strict": true
  }
}
```

Entre las más relevantes:

- `strictNullChecks`: `null` y `undefined` dejan de ser asignables a cualquier tipo por defecto.
- `noImplicitAny`: obliga a tipar explícitamente lo que el compilador no puede inferir, en lugar de dejarlo como `any` silenciosamente.
- `strictFunctionTypes`: comprobación más rigurosa de la compatibilidad entre tipos de funciones.
- `strictPropertyInitialization`: las propiedades de una clase deben inicializarse, o declararse explícitamente como opcionales.
- `alwaysStrict`: emite el código en modo estricto de JavaScript (`'use strict'`).

De todas ellas, `strictNullChecks` es la que más impacto tiene en la práctica, así que merece un ejemplo aparte.

## `strictNullChecks`: el que evita el error más común de JavaScript

Sin `strict`, este código compila sin ninguna queja:

```typescript
function obtenerUsuario(id: number): Usuario {
  return usuarios.find((u) => u.id === id);
}
```

El problema: `Array.find` devuelve `Usuario | undefined`, no `Usuario`. Si no se encuentra el usuario, la función devuelve `undefined` mientras promete devolver un `Usuario`. Cualquier código que llame a esta función y confíe en el tipo declarado va a intentar acceder a una propiedad de `undefined` en tiempo de ejecución — el clásico `Cannot read properties of undefined`, ahora escondido detrás de una firma de tipos que decía que esto no podía pasar.

Con `strictNullChecks` activado, esa misma función **no compila**:

```
Type 'Usuario | undefined' is not assignable to type 'Usuario'.
```

El compilador te obliga a decidir qué hacer con el caso de "no encontrado" antes de que el código llegue a ejecutarse:

```typescript
function obtenerUsuario(id: number): Usuario | undefined {
  return usuarios.find((u) => u.id === id);
}

// Y en el punto de uso, TypeScript te obliga a comprobarlo:
const usuario = obtenerUsuario(42);
if (usuario) {
  console.log(usuario.nombre); // aquí ya sabe que no es undefined
}
```

Esto es, en esencia, lo que aporta `strict`: convierte una clase entera de errores que hoy descubrirías en producción (o, con suerte, en un test) en un error de compilación que ves mientras escribes.

## `noImplicitAny`: que `any` sea una decisión, no un descuido

Sin esta comprobación, un parámetro sin tipo se convierte silenciosamente en `any`, y `any` desactiva el sistema de tipos para esa variable por completo:

```typescript
// Sin noImplicitAny: 'datos' es 'any' sin que nadie lo haya decidido
function procesar(datos) {
  return datos.valor.toUpperCase();
}
```

Ese código compila igual si `datos` es un objeto con un campo `valor` de tipo string, si es `undefined`, o si es un número. El error, si lo hay, aparece en tiempo de ejecución, exactamente donde TypeScript debería haberlo evitado.

Con `noImplicitAny`, ese mismo código da un error de compilación pidiendo un tipo explícito. Puedes seguir usando `any` si de verdad lo necesitas — pero ahora es una elección visible (`datos: any`), no un valor por defecto que se cuela sin que nadie se dé cuenta.

## Un ejemplo más completo: antes y después

```typescript
// Sin strict: compila sin avisos
interface Pedido {
  id: number;
  cliente?: { nombre: string };
}

function saludarCliente(pedido: Pedido) {
  return `Hola, ${pedido.cliente.nombre}`;
}
```

`cliente` es opcional (`?`), así que puede no existir. Sin `strictNullChecks`, TypeScript no te avisa de que estás accediendo a `.nombre` sobre algo que podría ser `undefined`. Con `strict` activado, este código no compila hasta que lo manejas explícitamente:

```typescript
function saludarCliente(pedido: Pedido) {
  if (!pedido.cliente) {
    return 'Hola';
  }
  return `Hola, ${pedido.cliente.nombre}`;
}
```

La diferencia entre ambas versiones es exactamente la diferencia entre un error que revienta con un pedido real sin cliente asociado, y un código que contempla ese caso porque el compilador no le dejó otra opción.

## "Pero así voy más lento"

Es la objeción más habitual, y tiene algo de cierto en el momento de escribir: con `strict` activado tienes que pensar en los casos de `null`/`undefined` según escribes, en lugar de dejarlo para más tarde.

La cuenta completa, sin embargo, no es esa. Ese tiempo que "ahorras" al no pensarlo ahora no desaparece: se traslada a depurar un fallo en producción, con menos contexto del que tenías al escribir la función y, normalmente, con más prisa. `strict` no añade trabajo, lo adelanta al momento en el que es más barato de arreglar.

Hay además un efecto colateral que se nota con el tiempo: con `strict` activado, el autocompletado del editor es mucho más útil, porque los tipos que ve reflejan la realidad (incluyendo los `undefined` posibles) en lugar de una versión optimista del código.

## Migrar un proyecto existente sin que sea una migración enorme

Activar `strict` de golpe en un proyecto grande puede generar cientos de errores de una vez, lo cual desanima a terminarlo. Hay una vía más gradual: activar las comprobaciones individuales una por una, empezando por las de más impacto:

```json
{
  "compilerOptions": {
    "strictNullChecks": true,
    "noImplicitAny": true
  }
}
```

Corrige los errores que aparecen, y ve añadiendo el resto de flags de `strict` de una en una según los vas resolviendo, hasta llegar a `"strict": true` completo. Es un proceso mecánico y verificable: cada error que el compilador señala es un sitio concreto donde el tipo no reflejaba la realidad del código.

## Por qué empezar directamente con `strict` en un proyecto nuevo

En un proyecto que arranca de cero no hay ninguna razón para posponerlo: no hay deuda que migrar, y el coste de tener `strict` desde el primer commit es cero comparado con activarlo después de meses de código escrito sin esa disciplina. La plantilla oficial de `tsc --init` incluye `"strict": true` comentado por defecto — quitarle el comentario es literalmente todo lo que hace falta.

## Conclusión

`strict` no añade restricciones arbitrarias: hace que el sistema de tipos de TypeScript describa lo que tu código puede hacer de verdad, en lugar de una versión simplificada que asume que nada es `null`, que las funciones siempre reciben lo que esperan y que cada `any` fue una decisión consciente. Sin `strict`, tienes la sintaxis de tipos de TypeScript pero no buena parte de las garantías que la hacen valiosa. Con él, el compilador se convierte en la primera línea de revisión de código, antes de que nadie más lo lea.
