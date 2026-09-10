---
title: 'PostgreSQL + Python: errores comunes de principiantes'
description: 'Los errores más frecuentes al conectar Python con PostgreSQL: conexiones no cerradas, problemas N+1, falta de índices y transacciones mal gestionadas, con ejemplo y solución de cada uno.'
pubDate: 'Jul 18 2026'
---

La mayoría de los problemas de rendimiento y estabilidad que aparecen al usar PostgreSQL desde Python no son fallos exóticos — son un puñado de patrones que se repiten una y otra vez, casi siempre invisibles mientras el proyecto es pequeño y la base de datos tiene pocos datos. Esta guía recoge los más comunes, con el síntoma, la causa y la solución de cada uno.

## Error 1: no cerrar las conexiones

**Síntoma.** La aplicación funciona bien al principio y, tras un rato en marcha, empieza a fallar con errores del tipo `too many connections` o similares, sobre todo bajo algo de carga.

**Causa.** Cada conexión a PostgreSQL consume recursos en el servidor de base de datos, y hay un límite máximo de conexiones simultáneas configurado (por defecto, no es un número enorme). Si cada petición abre una conexión nueva y nunca la cierra explícitamente, las conexiones se van acumulando hasta agotar ese límite — y a partir de ahí, ninguna petición nueva consigue conectar, ni siquiera las que no tienen nada que ver con el código que causó el problema.

```python
# Mal: la conexión se abre y nunca se cierra si algo falla a mitad
def obtener_usuario(id):
    conn = psycopg.connect(DATABASE_URL)
    cur = conn.cursor()
    cur.execute("SELECT * FROM usuarios WHERE id = %s", (id,))
    return cur.fetchone()
```

Si `execute` lanza una excepción, `conn.close()` nunca se ejecuta porque no había ningún mecanismo que lo garantizara.

**Solución.** Usar siempre un gestor de contexto (`with`), que garantiza el cierre incluso si algo falla a mitad:

```python
def obtener_usuario(id):
    with psycopg.connect(DATABASE_URL) as conn:
        with conn.cursor() as cur:
            cur.execute("SELECT * FROM usuarios WHERE id = %s", (id,))
            return cur.fetchone()
```

Y en producción, la solución completa no es solo cerrar bien cada conexión, sino no abrir una nueva en cada petición: usar un **pool de conexiones** (por ejemplo, con `psycopg_pool` o el pool que traiga tu ORM), que mantiene un conjunto de conexiones ya abiertas y las reutiliza, en lugar de pagar el coste de abrir y cerrar una conexión TCP en cada petición.

## Error 2: el problema N+1

**Síntoma.** Una página que lista, por ejemplo, veinte pedidos con el nombre del cliente de cada uno, tarda muchísimo más de lo que debería — y al mirar los logs de consultas, hay **veintiuna** consultas SQL en lugar de una o dos.

**Causa.** Es el error de rendimiento más común en cualquier ORM. Ocurre cuando cargas una lista de registros y, después, accedes a una relación de cada uno dentro de un bucle:

```python
# Mal: 1 consulta para los pedidos + N consultas, una por cada cliente
pedidos = session.query(Pedido).all()
for pedido in pedidos:
    print(pedido.cliente.nombre)  # dispara una consulta nueva en cada iteración
```

Con veinte pedidos, esto son veintiuna consultas (1 + N) donde con una consulta bien construida bastaría con una o dos. El problema es que cada consulta implica un viaje de ida y vuelta a la base de datos, y esa latencia se multiplica por cada elemento de la lista.

**Solución.** Cargar la relación por adelantado, en la misma consulta o en una consulta adicional bien planificada, en lugar de una por elemento. Con SQLAlchemy:

```python
from sqlalchemy.orm import joinedload

pedidos = session.query(Pedido).options(joinedload(Pedido.cliente)).all()
for pedido in pedidos:
    print(pedido.cliente.nombre)  # ya está cargado, sin consulta adicional
```

`joinedload` le dice al ORM que traiga la relación `cliente` en la misma consulta (mediante un `JOIN`), así que el bucle posterior no dispara ninguna consulta extra. El nombre técnico de esta técnica es *eager loading*, y la señal para saber si hace falta es siempre la misma: si estás accediendo a una relación dentro de un bucle sobre una lista, revisa cuántas consultas se están generando de verdad.

## Error 3: falta de índices

**Síntoma.** Una consulta que filtra o busca por una columna concreta funciona rápido con pocos datos de prueba, y se vuelve progresivamente más lenta a medida que la tabla crece — hasta tardar segundos enteros con una tabla de tamaño moderado.

**Causa.** Sin un índice sobre la columna por la que filtras, PostgreSQL tiene que recorrer la tabla entera comparando fila a fila (un *sequential scan*) para encontrar las que coinciden. Con pocas filas esto es instantáneo, así que el problema pasa desapercibido durante todo el desarrollo — y aparece de golpe cuando la tabla ya tiene datos reales.

```sql
-- Sin índice sobre email, esto recorre toda la tabla
SELECT * FROM usuarios WHERE email = 'ejemplo@correo.com';
```

**Solución.** Crear un índice sobre las columnas que se usan habitualmente en `WHERE`, `JOIN` o `ORDER BY`:

```sql
CREATE INDEX idx_usuarios_email ON usuarios(email);
```

Con el índice, PostgreSQL puede localizar las filas coincidentes sin recorrer la tabla entera, y la diferencia de rendimiento se vuelve cada vez más notable cuanto más crece la tabla.

Dos avisos importantes: primero, un índice no es gratis — acelera las lecturas pero añade coste a cada escritura (insertar o actualizar implica también actualizar el índice), así que no tiene sentido indexar columnas que nunca se consultan. Segundo, la forma fiable de confirmar si un índice está ayudando de verdad —y no solo asumirlo— es mirar el plan de ejecución:

```sql
EXPLAIN ANALYZE SELECT * FROM usuarios WHERE email = 'ejemplo@correo.com';
```

Si el plan muestra `Seq Scan` donde esperabas un `Index Scan`, algo no está usando el índice como debería (a veces porque no existe, otras porque la consulta está escrita de una forma que impide usarlo).

## Error 4: transacciones mal gestionadas

**Síntoma.** Una operación que debería ser todo-o-nada —por ejemplo, descontar stock y crear un pedido— deja el sistema en un estado inconsistente cuando algo falla a mitad: el stock se descontó, pero el pedido nunca se creó.

**Causa.** Cada paso se ejecutó como una operación independiente, sin agruparlos en una transacción. Si el segundo paso falla, el primero ya se confirmó (hizo *commit*) y no hay forma automática de deshacerlo.

```python
# Mal: si crear_pedido falla, el stock ya se descontó y queda así
descontar_stock(producto_id, cantidad)
crear_pedido(usuario_id, producto_id, cantidad)
```

**Solución.** Agrupar los pasos relacionados en una única transacción, de forma que si cualquiera falla, se deshacen todos (*rollback*):

```python
with psycopg.connect(DATABASE_URL) as conn:
    with conn.transaction():
        descontar_stock(conn, producto_id, cantidad)
        crear_pedido(conn, usuario_id, producto_id, cantidad)
    # si ambas operaciones terminan sin excepción, se confirma (commit)
    # si cualquiera lanza una excepción, se deshace todo automáticamente
```

La regla práctica: cualquier conjunto de escrituras que deban ser consistentes entre sí —o se aplican todas, o no se aplica ninguna— pertenece a la misma transacción. Es exactamente lo que las bases de datos relacionales están diseñadas para garantizar, y no aprovecharlo es renunciar a su garantía más importante sin necesidad.

## Error 5: usar `SELECT *` cuando solo hacen falta dos columnas

**Síntoma.** No es un error que rompa nada, pero consume ancho de banda y memoria de forma innecesaria, y se nota especialmente en tablas con columnas grandes (texto largo, JSON, binarios).

**Causa.** `SELECT *` trae todas las columnas de la tabla, incluidas las que no vas a usar. Si la tabla tiene un campo de texto largo o una columna JSON pesada, cada fila que traes de más tiene un coste real, multiplicado por cuántas filas devuelva la consulta.

**Solución.** Seleccionar explícitamente solo las columnas que se van a usar:

```sql
-- En lugar de SELECT * FROM usuarios
SELECT id, nombre, email FROM usuarios WHERE activo = true;
```

Además de ahorrar recursos, esto documenta con precisión qué datos necesita realmente esa parte del código — algo que un `SELECT *` oculta por completo.

## Checklist rápida

- [ ] Toda conexión se abre con un gestor de contexto (`with`), o pasa por un pool de conexiones.
- [ ] Al acceder a una relación dentro de un bucle, se ha comprobado cuántas consultas se generan de verdad.
- [ ] Las columnas usadas en `WHERE`, `JOIN` y `ORDER BY` con frecuencia tienen un índice, y se ha verificado con `EXPLAIN ANALYZE`.
- [ ] Las operaciones que deben ser todo-o-nada están agrupadas en una transacción explícita.
- [ ] Las consultas seleccionan solo las columnas que realmente se necesitan.

## Conclusión

Ninguno de estos cinco errores es difícil de entender una vez que se conoce — el problema es que todos ellos son invisibles mientras el volumen de datos es pequeño, que es precisamente la fase en la que se escribe la mayor parte del código. Se comportan bien en desarrollo y se degradan en producción, así que la única forma fiable de detectarlos a tiempo es no fiarse de que "funciona rápido en mi máquina" y revisar de vez en cuando, con datos de un volumen realista, cuántas consultas se están generando y qué plan de ejecución sigue cada una.
