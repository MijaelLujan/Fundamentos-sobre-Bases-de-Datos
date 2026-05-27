# Módulo 8 — Transacciones, aislamiento y MVCC

> *"La mayoría de los bugs de producción más difíciles de reproducir no son errores de lógica. Son consecuencias de cómo múltiples transacciones concurrentes interactúan con el modelo de aislamiento del motor. Se manifiestan una vez cada diez mil operaciones, desaparecen al intentar reproducirlos, y dejan rastros inconsistentes en los datos. Entender ACID, los niveles de aislamiento y MVCC no es teoría académica: es la diferencia entre un sistema que funciona correctamente bajo carga y uno que tiene bugs que nunca se pueden reproducir en desarrollo."*

---

## Tabla de contenidos

1. [Por qué las transacciones existen](#1-por-qué-las-transacciones-existen)
2. [ACID desglosado sin simplificaciones](#2-acid-desglosado-sin-simplificaciones)
3. [Fenómenos de concurrencia: el catálogo completo](#3-fenómenos-de-concurrencia-el-catálogo-completo)
4. [Los niveles de aislamiento del estándar SQL](#4-los-niveles-de-aislamiento-del-estándar-sql)
5. [Implementaciones de aislamiento: locks vs MVCC](#5-implementaciones-de-aislamiento-locks-vs-mvcc)
6. [MVCC en profundidad: cómo funciona en PostgreSQL](#6-mvcc-en-profundidad-cómo-funciona-en-postgresql)
7. [Serializable Snapshot Isolation (SSI)](#7-serializable-snapshot-isolation-ssi)
8. [Deadlocks: detección, prevención y patrones](#8-deadlocks-detección-prevención-y-patrones)
9. [El costo de las transacciones largas en MVCC](#9-el-costo-de-las-transacciones-largas-en-mvcc)
10. [Patrones de diseño para concurrencia correcta](#10-patrones-de-diseño-para-concurrencia-correcta)
11. [Resumen y conexión con el resto del curso](#11-resumen-y-conexión-con-el-resto-del-curso)
12. [Ejercicios de comprensión](#12-ejercicios-de-comprensión)

---

## 1. Por qué las transacciones existen

### 1.1 El problema de la concurrencia sin control

Imagina un sistema bancario sin ningún mecanismo de control de concurrencia. Dos cajeros procesan simultáneamente una transferencia de la cuenta A a la cuenta B y un depósito en la cuenta A:

```
Tiempo  Cajero 1 (transferencia A→B, $500)     Cajero 2 (depósito en A, $200)
──────────────────────────────────────────────────────────────────────────────
  t1    Lee saldo A = 1000
  t2                                            Lee saldo A = 1000
  t3    Escribe saldo A = 500 (resta 500)
  t4    Escribe saldo B = saldo_B + 500
  t5                                            Escribe saldo A = 1200 (suma 200)
──────────────────────────────────────────────────────────────────────────────
Resultado final: A = 1200, B = saldo_B + 500
Resultado correcto: A = 700, B = saldo_B + 500
```

El depósito del cajero 2 sobreescribió la transferencia del cajero 1. La cuenta A debería tener 700, pero tiene 1200. Se crearon $500 de la nada. Este es el **lost update** (actualización perdida), uno de los fenómenos que el control de concurrencia previene.

Pero la concurrencia sin control tiene más problemas que los lost updates. Y los mecanismos para prevenirlos tienen costos en rendimiento y complejidad. El diseño correcto requiere entender exactamente qué garantías se necesitan y a qué costo.

### 1.2 El modelo de la transacción

Una **transacción** es una unidad lógica de trabajo que transforma la base de datos de un estado consistente a otro estado consistente. Las operaciones dentro de una transacción son atómicas: o todas ocurren, o ninguna ocurre.

```sql
-- Una transferencia bancaria como transacción
BEGIN;

UPDATE cuentas SET saldo = saldo - 500 WHERE cuenta_id = 'A';
UPDATE cuentas SET saldo = saldo + 500 WHERE cuenta_id = 'B';

-- Si ambos UPDATE tienen éxito, confirmar:
COMMIT;

-- Si algo falla (saldo insuficiente, error de red, caída del servidor):
-- ROLLBACK; ← deshace ambos UPDATE como si no hubieran ocurrido
```

La transacción garantiza que si el sistema falla después del primer `UPDATE` pero antes del segundo, el estado queda como si ninguno de los dos hubiera ocurrido. No existe el estado intermedio "A descontado, B sin acreditar".

---

## 2. ACID desglosado sin simplificaciones

ACID es el acrónimo que describe las cuatro propiedades que garantiza una transacción. Cada propiedad es más sutil de lo que parece a primera vista.

### 2.1 Atomicidad (Atomicity)

**La garantía:** Todas las operaciones de la transacción se confirman, o ninguna. No hay estados intermedios visibles ni parcialmente aplicados.

**Cómo se implementa:** El motor mantiene un **Write-Ahead Log (WAL)**: antes de modificar cualquier dato en disco, escribe la descripción del cambio en el log. Si el sistema cae a mitad de una transacción, al reiniciar el motor lee el WAL y realiza el **undo** de las operaciones no confirmadas (o el **redo** de las confirmadas pero no volcadas a disco).

**Lo que atomicidad NO garantiza:** No garantiza que la transacción no afecte a otras transacciones concurrentes mientras está en curso. Eso lo garantiza el aislamiento. La atomicidad solo dice que el resultado es "todo o nada" desde la perspectiva de la transacción misma.

**El caso del error en la aplicación:**

```sql
BEGIN;
UPDATE cuentas SET saldo = saldo - 500 WHERE cuenta_id = 'A';
-- El código de aplicación tiene un bug y lanza una excepción aquí
-- Si el código no hace ROLLBACK explícito, la transacción queda "abierta"
-- hasta que se cierra la conexión (el motor hace ROLLBACK automáticamente)
-- Pero si el código hace COMMIT a pesar del error... solo el primer UPDATE se aplica.
-- La atomicidad no protege contra código de aplicación buggy que hace COMMIT parcial.
```

### 2.2 Consistencia (Consistency)

**La garantía:** La transacción lleva la base de datos de un estado que satisface todas las restricciones de integridad a otro estado que también las satisface.

**Lo que consistencia realmente significa:** Es la propiedad más mal entendida de ACID. La consistencia **no es una garantía del motor**; es una propiedad que se mantiene si las transacciones están correctamente escritas y el esquema tiene las restricciones correctas declaradas.

El motor garantiza que las restricciones declaradas (`NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`) no serán violadas. Pero si defines una transferencia bancaria que solo actualiza una cuenta y no la otra, el motor no puede saber que eso viola una invariante del negocio ("la suma de todos los saldos debe ser constante"). Esa consistencia de negocio es responsabilidad del código de aplicación y del diseño de la transacción.

**La C de ACID es principalmente una responsabilidad del desarrollador, no del motor.**

### 2.3 Aislamiento (Isolation)

**La garantía:** Las transacciones concurrentes se comportan como si se ejecutaran en serie (una después de la otra). Los cambios intermedios de una transacción no son visibles para otras transacciones.

**La realidad:** El aislamiento completo (serializable) tiene un costo muy alto en rendimiento. En la práctica, los motores ofrecen **niveles de aislamiento** que ofrecen distintos grados de garantía a cambio de mayor o menor rendimiento. Los niveles más bajos permiten que ciertas anomalías de concurrencia ocurran; los niveles más altos las previenen a costa de más bloqueos o reintentos.

Esta es la propiedad más rica y más compleja de ACID, y le dedicaremos la mayor parte del módulo.

### 2.4 Durabilidad (Durability)

**La garantía:** Una vez que una transacción confirma con `COMMIT`, sus cambios son permanentes, incluso si el sistema cae inmediatamente después.

**Cómo se implementa:** El motor garantiza durabilidad escribiendo el registro del commit en el WAL y sincronizando el WAL a disco (fsync) antes de informar al cliente que el commit fue exitoso. Esto es caro en términos de I/O.

**El parámetro `synchronous_commit`:**

```sql
-- El default (on): espera a que el WAL sea sincronizado a disco antes de confirmar
-- Garantiza durabilidad total
SET synchronous_commit = on;

-- off: confirma al cliente sin esperar la sincronización a disco
-- La transacción puede perderse si el sistema cae en los siguientes milisegundos
-- PERO la base de datos nunca queda en estado inconsistente (solo puede perder
-- el commit, no quedar a medias)
SET synchronous_commit = off;
```

`synchronous_commit = off` es útil para cargas de trabajo donde perder los últimos milisegundos de escrituras es aceptable (logs de auditoría no críticos, telemetría). Nunca debe usarse para operaciones financieras o donde la pérdida de datos es inaceptable.

---

## 3. Fenómenos de concurrencia: el catálogo completo

El estándar SQL:1992 definió tres fenómenos de concurrencia. Investigación posterior identificó varios más que el estándar omitió. Conocerlos todos es esencial para entender qué garantiza realmente cada nivel de aislamiento.

### 3.1 Dirty Read (Lectura sucia)

**Definición:** Una transacción lee datos escritos por otra transacción que aún no ha confirmado. Si esa transacción hace rollback, la primera leyó datos que "nunca existieron".

```
T1                              T2
─────────────────────────────────────────────
BEGIN;
UPDATE saldos SET valor = 1000
WHERE cuenta = 'A';
                                BEGIN;
                                SELECT valor FROM saldos WHERE cuenta = 'A';
                                -- Lee 1000 (el valor no confirmado de T1)
ROLLBACK;  ← T1 se deshace
                                -- T2 tomó una decisión basada en 1000,
                                -- pero ese valor nunca existió formalmente
```

**Peligro:** Decisiones tomadas sobre datos que nunca fueron reales.

**Nivel que lo permite:** Solo Read Uncommitted (prácticamente nunca se usa en producción).

### 3.2 Non-repeatable Read (Lectura no repetible)

**Definición:** Una transacción lee la misma fila dos veces y obtiene valores distintos, porque otra transacción la modificó y confirmó entre ambas lecturas.

```
T1                              T2
─────────────────────────────────────────────
BEGIN;
SELECT saldo FROM cuentas
WHERE cuenta = 'A';
-- Lee: 1000
                                BEGIN;
                                UPDATE cuentas SET saldo = 1500
                                WHERE cuenta = 'A';
                                COMMIT;
SELECT saldo FROM cuentas
WHERE cuenta = 'A';
-- Lee: 1500 ← ¡Distinto! La misma fila, el mismo predicado, resultado diferente
COMMIT;
```

**Peligro:** Un informe que lee el mismo dato dos veces y saca conclusiones basadas en que son iguales (p.ej., verificar un saldo dos veces para una transacción).

**Nivel que lo permite:** Read Committed y niveles inferiores.

### 3.3 Phantom Read (Lectura fantasma)

**Definición:** Una transacción ejecuta la misma query (con un predicado de rango) dos veces y obtiene un conjunto de filas diferente, porque otra transacción insertó o eliminó filas que satisfacen el predicado entre ambas ejecuciones.

```
T1                              T2
─────────────────────────────────────────────
BEGIN;
SELECT * FROM pedidos
WHERE monto > 1000;
-- Retorna: 5 filas
                                BEGIN;
                                INSERT INTO pedidos (monto, ...) VALUES (1500, ...);
                                COMMIT;
SELECT * FROM pedidos
WHERE monto > 1000;
-- Retorna: 6 filas ← ¡Una fila nueva "apareció"!
COMMIT;
```

**Diferencia con non-repeatable read:** Non-repeatable read afecta a filas que ya existen (sus valores cambian). Phantom read afecta al conjunto de filas que satisfacen un predicado (filas nuevas aparecen o desaparecen).

**Peligro:** Lógica que asume que el conjunto de filas procesado es estable durante la transacción (p.ej., agregar ítems a una factura mientras otro proceso inserta ítems en la misma factura).

**Nivel que lo permite:** Read Committed y Repeatable Read.

### 3.4 Lost Update (Actualización perdida)

**Definición:** Dos transacciones leen el mismo valor, cada una lo modifica basándose en lo que leyó, y la segunda escritura sobreescribe el resultado de la primera.

```
T1                              T2
─────────────────────────────────────────────
BEGIN;                          BEGIN;
SELECT stock FROM productos
WHERE prod_id = 1;
-- Lee: stock = 100
                                SELECT stock FROM productos
                                WHERE prod_id = 1;
                                -- Lee: stock = 100
UPDATE productos
SET stock = stock - 3
WHERE prod_id = 1;
-- Escribe: 97
COMMIT;
                                UPDATE productos
                                SET stock = stock - 5
                                WHERE prod_id = 1;
                                -- Escribe: 95 ← ¡Sobreescribió el 97!
                                COMMIT;
-- Stock final: 95. Debería ser 92. Se perdieron 3 unidades vendidas por T1.
```

**Peligro:** Inconsistencias en cualquier operación de tipo "leer-modificar-escribir" (contadores, saldos, stock). Es el fenómeno más común y peligroso en aplicaciones reales.

**Nivel que lo permite:** Read Committed (si el UPDATE no usa la condición de raza adecuada).

### 3.5 Write Skew (Sesgo de escritura)

**Definición:** Dos transacciones leen datos superpuestos, cada una verifica una condición basada en lo que leyó, y cada una escribe en partes disjuntas de los datos. El resultado viola una invariante que ninguna transacción violó individualmente.

Este es el fenómeno más sutil y más difícil de detectar.

**Ejemplo clásico: sistema de guardias de turno**

Invariante del negocio: debe haber al menos un médico de guardia en todo momento.

```
Estado inicial: médico Ana = en_guardia, médico Luis = en_guardia

T1 (Ana quiere salir de guardia)    T2 (Luis quiere salir de guardia)
────────────────────────────────────────────────────────────────────
BEGIN;                               BEGIN;
SELECT COUNT(*) FROM guardias
WHERE en_guardia = TRUE;
-- Cuenta: 2 (Ana y Luis)
-- Condición: 2 > 1 → puedo salir
                                     SELECT COUNT(*) FROM guardias
                                     WHERE en_guardia = TRUE;
                                     -- Cuenta: 2 (Ana y Luis)
                                     -- Condición: 2 > 1 → puedo salir
UPDATE guardias
SET en_guardia = FALSE
WHERE medico = 'Ana';
COMMIT;
                                     UPDATE guardias
                                     SET en_guardia = FALSE
                                     WHERE medico = 'Luis';
                                     COMMIT;
────────────────────────────────────────────────────────────────────
Resultado: ningún médico de guardia. La invariante se violó.
```

Cada transacción, individualmente, era válida. Ambas verificaron la condición antes de escribir. Pero escribieron en partes **disjuntas** de los datos (Ana vs Luis), por lo que no hubo conflicto de escritura que el motor pudiera detectar. El resultado viola la invariante del sistema.

**Nivel que lo permite:** Read Committed, Repeatable Read, e incluso Snapshot Isolation (no serializable).

**La única solución:** Serializable isolation, o bloqueos explícitos (`SELECT FOR UPDATE`) que fuercen serialización.

### 3.6 Read Skew (Sesgo de lectura)

**Definición:** Una transacción lee partes de datos relacionados en momentos distintos, y entre esas lecturas otra transacción los modifica, de modo que la primera ve un estado inconsistente del conjunto.

```
T1                              T2
─────────────────────────────────────────────
BEGIN;
SELECT saldo FROM cuenta WHERE id = 'A';
-- Lee: A = 1000
                                BEGIN;
                                UPDATE cuenta SET saldo = 500 WHERE id = 'A';
                                UPDATE cuenta SET saldo = 1500 WHERE id = 'B';
                                COMMIT;
SELECT saldo FROM cuenta WHERE id = 'B';
-- Lee: B = 1500 (el nuevo valor de T2)
-- T1 ve A=1000 (antes de T2) y B=1500 (después de T2): estado inconsistente
-- La suma A+B = 2500, pero debería ser 2000 en el estado original
-- o 2000 también en el estado post-T2.
COMMIT;
```

**Nivel que lo permite:** Read Committed. Snapshot Isolation lo previene porque T1 ve una snapshot consistente del principio de la transacción.

### 3.7 Tabla resumen de fenómenos

| Fenómeno | Descripción breve | Causa raíz |
|---|---|---|
| **Dirty Read** | Lee datos de una transacción no confirmada | Visibilidad prematura |
| **Non-repeatable Read** | La misma fila da valores distintos en dos lecturas | Modificación entre lecturas |
| **Phantom Read** | La misma query da conjuntos de filas distintos | Inserción/eliminación entre lecturas |
| **Lost Update** | Una escritura sobreescribe otra basada en datos obsoletos | Ciclo read-modify-write sin protección |
| **Write Skew** | Dos transacciones violan una invariante compuesta | Lecturas superpuestas, escrituras disjuntas |
| **Read Skew** | Lectura de estado inconsistente entre dos entidades relacionadas | Lecturas en distintos momentos del tiempo |

---

## 4. Los niveles de aislamiento del estándar SQL

### 4.1 Los cuatro niveles definidos por SQL:1992

El estándar SQL define cuatro niveles de aislamiento, ordenados de menor a mayor garantía:

| Nivel | Dirty Read | Non-repeatable Read | Phantom Read |
|---|---|---|---|
| **Read Uncommitted** | Posible | Posible | Posible |
| **Read Committed** | Prevenido | Posible | Posible |
| **Repeatable Read** | Prevenido | Prevenido | Posible |
| **Serializable** | Prevenido | Prevenido | Prevenido |

**El problema con esta tabla:** El estándar de 1992 es incompleto. No incluye write skew, lost update, ni read skew, que son fenómenos reales con consecuencias graves. La tabla oficial dice que Repeatable Read previene phantom reads, pero la implementación de Snapshot Isolation (que la mayoría de los motores usan para Repeatable Read) **no previene write skew**.

### 4.2 Read Uncommitted

El nivel más bajo. Las lecturas no adquieren ningún bloqueo y pueden ver datos de transacciones no confirmadas.

**En la práctica:** PostgreSQL implementa Read Uncommitted igual que Read Committed. Nunca verás dirty reads en PostgreSQL incluso con este nivel declarado. En MySQL InnoDB, sí existe la distinción.

**Cuándo tiene sentido teórico:** Reportes aproximados donde la precisión no importa y la velocidad máxima es lo único que importa. En la práctica, casi nunca es la elección correcta.

### 4.3 Read Committed

Cada statement dentro de la transacción ve una snapshot de los datos confirmados **en el momento en que ese statement comienza**. No en el momento en que comenzó la transacción.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;

-- Primera lectura: ve los datos confirmados en t1
SELECT saldo FROM cuentas WHERE cuenta_id = 'A';
-- Resultado: 1000

-- Mientras tanto, otra transacción actualiza y confirma A = 1500

-- Segunda lectura (nuevo statement): ve los datos confirmados en t2
SELECT saldo FROM cuentas WHERE cuenta_id = 'A';
-- Resultado: 1500 ← non-repeatable read

COMMIT;
```

**Es el nivel por defecto de PostgreSQL, Oracle, y SQL Server.**

**Por qué es el default:** Ofrece un buen balance entre consistencia y rendimiento. Previene dirty reads (el problema más grave) sin requerir snapshots largas. Para la mayoría de las operaciones web (cada request es una transacción corta), es suficiente.

**Cuándo es insuficiente:** Cuando una transacción necesita ver datos consistentes a lo largo de múltiples statements. Reportes, transferencias entre cuentas, cualquier operación de "leer el estado, tomar una decisión, escribir basado en esa decisión".

### 4.4 Repeatable Read (en PostgreSQL: Snapshot Isolation)

La transacción obtiene una **snapshot** del estado de la base de datos en el momento en que comienza, y todas las lecturas dentro de la transacción ven esa misma snapshot, independientemente de qué otras transacciones confirmen mientras tanto.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
-- La snapshot se toma aquí: veremos el estado de la DB en este momento

SELECT saldo FROM cuentas WHERE cuenta_id = 'A';
-- Resultado: 1000

-- Otra transacción actualiza y confirma A = 1500

SELECT saldo FROM cuentas WHERE cuenta_id = 'A';
-- Resultado: 1000 ← La misma snapshot. Non-repeatable read NO ocurre.

COMMIT;
```

**Importante:** En PostgreSQL, `REPEATABLE READ` se implementa con Snapshot Isolation, **no con bloqueos**. Esto significa que:
- No hay dirty reads.
- No hay non-repeatable reads.
- No hay phantom reads (también prevenidos por la snapshot).
- **Sí puede ocurrir write skew** (porque SI no detecta el patrón de read-check-write disjunto).

**El conflicto de escritura en SI:** Si T1 y T2 ambas empezaron con la misma snapshot y ambas intentan modificar la misma fila, la segunda en llegar detecta el conflicto y **aborta con error de serialización**. Esto previene lost updates.

```
ERROR: could not serialize access due to concurrent update
```

El código de aplicación **debe manejar este error y reintentar la transacción**. Esto no es un bug; es el mecanismo de control de concurrencia funcionando correctamente.

### 4.5 Serializable (en PostgreSQL: SSI)

El nivel más alto. Garantiza que el resultado de ejecutar transacciones concurrentes es equivalente a alguna ejecución serial (una después de la otra). Previene **todos** los fenómenos, incluyendo write skew.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

PostgreSQL implementa Serializable con **Serializable Snapshot Isolation (SSI)**, que se explica en la sección 7.

**Cuándo usar Serializable:**
- Cuando la lógica del negocio tiene invariantes compuestas que dependen de múltiples filas (el problema de los médicos de guardia).
- Cuando write skew podría ocurrir por la naturaleza del dominio.
- Cuando la corrección es más importante que el rendimiento máximo.

**El costo:** Mayor overhead de seguimiento de dependencias, mayor tasa de aborts que deben reintentarse.

### 4.6 Cuándo usar cada nivel

| Nivel | Usar cuando... |
|---|---|
| **Read Committed** | Operaciones web típicas, transacciones cortas, CRUD simple. El default razonable. |
| **Repeatable Read** | Reportes que deben ser consistentes, transferencias entre entidades, operaciones que leen múltiples veces el mismo dato. |
| **Serializable** | Invariantes que dependen de múltiples filas, write skew posible, corrección sobre rendimiento. |
| **Read Uncommitted** | Prácticamente nunca. |

---

## 5. Implementaciones de aislamiento: locks vs MVCC

### 5.1 El enfoque basado en locks (modelo clásico)

La implementación clásica del aislamiento usa **bloqueos (locks)**: antes de leer una fila, se adquiere un lock compartido (shared lock, S-lock); antes de escribirla, se adquiere un lock exclusivo (exclusive lock, X-lock).

```
S-lock (lectura):  compatible con otros S-locks, incompatible con X-locks
X-lock (escritura): incompatible con S-locks y X-locks
```

Para Repeatable Read con locks: se mantienen los S-locks hasta el final de la transacción, así ninguna otra transacción puede modificar las filas leídas.

**El problema fundamental de los locks:** Las lecturas bloquean las escrituras y las escrituras bloquean las lecturas. En una aplicación con muchos lectores y varios escritores, esto crea contención y degrada el rendimiento. Una transacción de reporte de larga duración puede bloquear toda la actividad de escritura en las tablas que lee.

### 5.2 El enfoque MVCC (Multi-Version Concurrency Control)

**MVCC** resuelve el problema de raíz: en lugar de que las lecturas bloqueen las escrituras, el motor mantiene **múltiples versiones** de cada fila. Una escritura crea una nueva versión de la fila; las transacciones que necesiten la versión anterior pueden seguir leyéndola.

**La propiedad fundamental de MVCC:**

> Las lecturas no bloquean las escrituras, y las escrituras no bloquean las lecturas.

Esto permite un nivel de concurrencia muy superior al modelo de locks puro, especialmente en cargas de trabajo con muchos lectores (típico en aplicaciones web).

**El trade-off:** MVCC requiere almacenar múltiples versiones de las filas y un mecanismo para eventualmente eliminar las versiones que ya no son necesarias (vacuum en PostgreSQL). También introduce la posibilidad de que las transacciones largas impidan la limpieza de versiones antiguas.

---

## 6. MVCC en profundidad: cómo funciona en PostgreSQL

### 6.1 La estructura de una fila en PostgreSQL

Cada fila en PostgreSQL tiene un conjunto de campos de sistema (ocultos por defecto) que implementan MVCC:

```
┌────────────────────────────────────────────────────────────────────┐
│  xmin  │  xmax  │  cmin  │  cmax  │  ctid  │  datos_de_usuario   │
└────────────────────────────────────────────────────────────────────┘
```

- **`xmin`:** El ID de transacción (XID) que **creó** esta versión de la fila (INSERT o UPDATE que creó esta versión).
- **`xmax`:** El ID de transacción que **eliminó** esta versión (DELETE, o UPDATE que la reemplazó). Si es 0, la versión sigue vigente.
- **`cmin` / `cmax`:** Número de comando dentro de la transacción (para visibilidad dentro de la misma transacción).
- **`ctid`:** La ubicación física de la fila en el heap (block, offset).

```sql
-- Ver los campos de sistema de una tabla
SELECT xmin, xmax, ctid, * FROM empleados LIMIT 5;

-- xmin = 12345 → La transacción 12345 insertó esta fila
-- xmax = 0     → Nadie la ha eliminado todavía
-- ctid = (0,1) → Está en el bloque 0, posición 1 del heap
```

### 6.2 El mecanismo de visibilidad

Cuando una transacción T lee la tabla, para cada versión de cada fila, el motor verifica si esa versión es **visible** para T. La regla general es:

> Una versión de fila es visible para T si:
> - `xmin` corresponde a una transacción **confirmada** antes de que T tome su snapshot, Y
> - `xmax` es 0 (nadie la eliminó), o `xmax` corresponde a una transacción que **no había confirmado** cuando T tomó su snapshot.

En pseudocódigo:
```
visible = (xmin_committed AND xmin_before_snapshot) AND
          (xmax_is_zero OR xmax_not_committed OR xmax_after_snapshot)
```

### 6.3 Snapshots: el mecanismo central

Cuando una transacción comienza (para Read Committed, cuando cada statement comienza; para Repeatable Read / Serializable, cuando la transacción comienza), el motor captura una **snapshot** del estado de las transacciones activas:

```
Snapshot = {
    xmin:   el XID más pequeño de todas las transacciones activas en este momento
    xmax:   el siguiente XID que será asignado (transacciones con XID >= xmax no existen todavía)
    xip:    lista de XIDs de transacciones que están en curso (empezadas pero no confirmadas/abortadas)
}
```

Con esta snapshot, la transacción puede determinar la visibilidad de cualquier versión de fila:
- XIDs menores que `xmin`: confirmdadas antes de la snapshot → visibles.
- XIDs mayores o iguales a `xmax`: no existían en la snapshot → no visibles.
- XIDs en `xip`: estaban en curso en la snapshot → no visibles (sin importar si después confirmaron).

### 6.4 Un ejemplo paso a paso de MVCC

```
Estado inicial: fila con xmin=100, xmax=0, nombre='Ana', salario=5000

Transacción 200 lee la fila:
  - xmin=100 (confirmada antes de la snapshot de T200) → OK
  - xmax=0 → fila activa → visible
  - T200 ve: nombre='Ana', salario=5000

Transacción 201 actualiza la fila (UPDATE):
  1. Crea una NUEVA versión: xmin=201, xmax=0, nombre='Ana', salario=5500
  2. Marca la versión ANTIGUA: xmin=100, xmax=201 (T201 la "eliminó")
  3. T201 ve su propia versión nueva

Transacción 200 lee nuevamente (Repeatable Read):
  - Versión nueva: xmin=201, pero 201 estaba en xip de la snapshot de T200
    → no visible (T200 no veía T201 cuando tomó su snapshot)
  - Versión antigua: xmin=100 (visible), xmax=201 (T201 no estaba confirmada
    cuando T200 tomó su snapshot) → visible
  - T200 sigue viendo: nombre='Ana', salario=5000

T201 confirma (COMMIT)

Transacción 202 comienza después de que T201 confirmó:
  - Versión nueva: xmin=201 (confirmada antes de la snapshot de T202) → visible
  - xmax=0 → fila activa
  - T202 ve: nombre='Ana', salario=5500
  - Versión antigua: xmin=100 (visible), xmax=201 (T201 confirmada antes
    de la snapshot de T202) → la versión antigua es INVISIBLE para T202
```

Esto ilustra perfectamente MVCC: T200, T201, y T202 pueden ejecutarse simultáneamente sin bloquearse entre sí, y cada una ve una vista consistente de los datos.

### 6.5 El UPDATE como DELETE + INSERT en MVCC

Este es uno de los puntos más importantes para entender el comportamiento y los costos de PostgreSQL:

**En PostgreSQL, un UPDATE no modifica la fila existente. Crea una nueva versión de la fila y marca la versión anterior como expirada.**

Consecuencias:
1. La fila antigua sigue ocupando espacio en el heap hasta que el VACUUM la limpie.
2. Si la fila tiene índices, la nueva versión de la fila también necesita entradas en todos los índices (salvo optimizaciones como HOT updates).
3. Un UPDATE masivo que toca muchas filas puede generar una cantidad enorme de "filas muertas" que necesitan vacuuming.

### 6.6 HOT Updates: la optimización para updates frecuentes

**HOT (Heap Only Tuple) update:** Si la actualización no cambia ninguna columna indexada y la nueva versión de la fila cabe en la misma página del heap, PostgreSQL puede hacer un HOT update: crea la nueva versión en la misma página y actualiza la cadena de versiones sin tocar los índices.

```sql
-- Si precio no está indexado, este UPDATE puede ser HOT:
UPDATE productos SET precio = precio * 1.1 WHERE producto_id = 42;

-- Si producto_id (que SÍ está indexado) cambia, NO puede ser HOT:
-- (esto nunca debería ocurrir para una PK, pero ilustra el concepto)
```

El resultado: los índices no necesitan actualizarse, reduciendo el overhead de escritura significativamente para columnas no indexadas que cambian con frecuencia.

### 6.7 VACUUM: el recolector de versiones muertas

Con el tiempo, el heap acumula versiones de filas que ninguna transacción activa puede ver: las "filas muertas" (dead tuples). El proceso `VACUUM` las limpia:

```sql
-- VACUUM manual de una tabla
VACUUM empleados;

-- VACUUM FULL: compacta completamente la tabla (reescribe todo)
-- Muy costoso, requiere lock exclusivo. Solo para casos extremos.
VACUUM FULL empleados;

-- ANALYZE combina las dos tareas: limpieza + actualización de estadísticas
VACUUM ANALYZE empleados;

-- Ver el estado de las tablas: filas vivas vs muertas
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       ROUND(100.0 * n_dead_tup / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS pct_dead
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

El daemon **autovacuum** ejecuta VACUUM automáticamente cuando el número de filas muertas supera un umbral. Por defecto: `50 + 0.2 × n_filas`. Para tablas muy grandes con alta tasa de updates, puede ser necesario ajustar estos umbrales.

```sql
-- Ajustar autovacuum para una tabla específica con alta tasa de updates
ALTER TABLE pedidos SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- 1% en lugar del 20% por defecto
    autovacuum_analyze_scale_factor = 0.005
);
```

---

## 7. Serializable Snapshot Isolation (SSI)

### 7.1 El problema de Snapshot Isolation

Snapshot Isolation es poderoso y eficiente, pero **no es serializable**: permite write skew. Durante años, la comunidad académica y de bases de datos buscó una implementación de serializable que no requiriera bloqueos excesivos.

En 2008, Michael Cahill, Uwe Röhm y Alan Fekete publicaron "Serializable Isolation for Snapshot Databases", describiendo SSI. PostgreSQL 9.1 (2011) fue el primer DBMS de producción en implementarlo.

### 7.2 La idea central de SSI: detectar ciclos de dependencia

SSI observa que write skew siempre involucra un **ciclo de dependencias anti-rw** (anti-read-write). En el ejemplo de los médicos:

- T1 leyó las filas de guardias que T2 escribió (dependencia rw: T1 leyó lo que T2 no había escrito todavía).
- T2 leyó las filas de guardias que T1 escribió (dependencia rw: T2 leyó lo que T1 no había escrito todavía).

Hay un ciclo: T1 →(rw) T2 →(rw) T1. SSI detecta este ciclo y aborta una de las dos transacciones.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;

-- T1: verificar que hay al menos 2 médicos en guardia
SELECT COUNT(*) FROM guardias WHERE en_guardia = TRUE;
-- Resultado: 2

-- T2 hace lo mismo concurrentemente y también obtiene 2

-- T1 decide salir de guardia
UPDATE guardias SET en_guardia = FALSE WHERE medico = 'Ana';

COMMIT;
-- Si T2 también llegó hasta aquí, una de las dos recibirá:
-- ERROR: could not serialize access due to concurrent update
```

### 7.3 Cómo SSI rastrea las dependencias

PostgreSQL SSI mantiene, en memoria, un registro de las dependencias rw entre transacciones activas. Cuando detecta un ciclo de dependencias que podría producir un resultado no serializable, aborta la transacción más "nueva" del ciclo (la que hizo el commit más tarde).

```sql
-- Ver transacciones activas y su nivel de aislamiento
SELECT pid, usename, application_name, state, isolation_level,
       wait_event_type, wait_event
FROM pg_stat_activity
WHERE state != 'idle';
```

### 7.4 El costo real de SSI

SSI agrega overhead porque debe:
1. Rastrear qué filas/páginas leyó cada transacción (para detectar dependencias rw).
2. Verificar en cada escritura si se forma un ciclo.
3. Posiblemente abortar transacciones que causarían un resultado no serializable.

**El overhead en la práctica:** Los benchmarks de PostgreSQL muestran que SSI tiene un overhead del 5-15% sobre SI (Snapshot Isolation) en cargas de trabajo típicas. Para cargas de trabajo donde write skew es muy frecuente (muchos conflictos), el overhead puede ser mayor porque hay más aborts y reintentos.

**Regla práctica:** Usa Serializable cuando el negocio lo requiera, no por defecto. Para la mayoría de las aplicaciones web con operaciones simples (insert/update sobre la fila propia del usuario), Read Committed es suficiente.

---

## 8. Deadlocks: detección, prevención y patrones

### 8.1 Qué es un deadlock

Un **deadlock** ocurre cuando dos o más transacciones se esperan mutuamente, formando un ciclo de espera donde ninguna puede avanzar.

```
T1 tiene lock en fila A, espera lock en fila B
T2 tiene lock en fila B, espera lock en fila A
→ Deadlock: ninguna puede continuar
```

```sql
-- Simulación de deadlock
-- Conexión 1:
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE cuenta_id = 'A';
-- Ahora espera un segundo para que la conexión 2 haga su primer UPDATE

-- Conexión 2 (en paralelo):
BEGIN;
UPDATE cuentas SET saldo = saldo - 100 WHERE cuenta_id = 'B';
UPDATE cuentas SET saldo = saldo + 100 WHERE cuenta_id = 'A';  -- Espera el lock de C1

-- Conexión 1 continúa:
UPDATE cuentas SET saldo = saldo + 100 WHERE cuenta_id = 'B';  -- Espera el lock de C2

-- Resultado: PostgreSQL detecta el deadlock y aborta una de las dos transacciones:
-- ERROR:  deadlock detected
-- DETAIL: Process 12345 waits for ShareLock on transaction 67890;
--         blocked by process 67890.
--         Process 67890 waits for ShareLock on transaction 12345;
--         blocked by process 12345.
-- HINT:   See server log for query details.
```

### 8.2 Detección de deadlocks en PostgreSQL

PostgreSQL detecta deadlocks mediante análisis del **grafo de espera** (wait-for graph): un grafo donde cada nodo es una transacción y existe una arista de T1 a T2 si T1 está esperando un lock que T2 tiene. Un deadlock es un ciclo en este grafo.

La detección se realiza periódicamente (controlada por `deadlock_timeout`, por defecto 1 segundo). Cuando el motor detecta que una transacción ha esperado más de `deadlock_timeout`, construye el grafo de espera y busca ciclos. Si encuentra uno, aborta la transacción más "barata" del ciclo (heurísticamente, la que ha hecho menos trabajo).

```sql
-- Ver el timeout de deadlock detection
SHOW deadlock_timeout;  -- default: 1s

-- Un deadlock_timeout muy alto significa que las transacciones esperarán más
-- antes de que el deadlock sea detectado.
-- Un valor muy bajo puede generar falsos positivos en sistemas lentos.
```

### 8.3 El mensaje de error y la respuesta correcta

Cuando ocurre un deadlock, una de las transacciones recibirá un error. La respuesta correcta del código de aplicación es **siempre reintentar toda la transacción desde el principio**:

```python
MAX_RETRIES = 3

for attempt in range(MAX_RETRIES):
    try:
        with connection.transaction():
            # lógica de la transacción
            execute("UPDATE cuentas SET saldo = saldo - 100 WHERE cuenta_id = 'A'")
            execute("UPDATE cuentas SET saldo = saldo + 100 WHERE cuenta_id = 'B'")
        break  # éxito, salir del loop
    except DeadlockDetected:
        if attempt == MAX_RETRIES - 1:
            raise  # agotamos los reintentos
        time.sleep(0.1 * (2 ** attempt))  # backoff exponencial
```

**Un deadlock no es un error del que "recuperarse" parcialmente. Es una señal de que la transacción debe ejecutarse de nuevo completa.**

### 8.4 Prevención de deadlocks: el orden de adquisición de locks

La causa raíz de la mayoría de los deadlocks en aplicaciones es que distintas partes del código adquieren locks sobre las mismas filas **en distinto orden**.

**La regla de oro para prevenir deadlocks:**

> Si múltiples transacciones necesitan locks sobre el mismo conjunto de filas, **siempre deben adquirirlos en el mismo orden**.

```sql
-- Incorrecto: T1 bloquea A luego B, T2 bloquea B luego A → deadlock posible
-- T1:
UPDATE cuentas SET saldo = saldo - 100 WHERE cuenta_id = 'A';
UPDATE cuentas SET saldo = saldo + 100 WHERE cuenta_id = 'B';

-- T2:
UPDATE cuentas SET saldo = saldo - 50 WHERE cuenta_id = 'B';
UPDATE cuentas SET saldo = saldo + 50 WHERE cuenta_id = 'A';

-- Correcto: siempre actualizar en orden ascendente de cuenta_id
-- T1 y T2 ordenan: min(A,B) primero, luego max(A,B)
-- Nunca se forma el ciclo de espera
```

**En SQL, se puede forzar el orden con `SELECT FOR UPDATE ... ORDER BY`:**

```sql
-- Bloquear las filas en orden consistente antes de actualizarlas
BEGIN;
SELECT * FROM cuentas
WHERE cuenta_id IN ('A', 'B')
ORDER BY cuenta_id  -- ← orden consistente
FOR UPDATE;

-- Ahora ambas filas están bloqueadas en el mismo orden para todos
UPDATE cuentas SET saldo = saldo - 100 WHERE cuenta_id = 'A';
UPDATE cuentas SET saldo = saldo + 100 WHERE cuenta_id = 'B';
COMMIT;
```

### 8.5 `SELECT FOR UPDATE` y sus variantes

`SELECT FOR UPDATE` adquiere un lock exclusivo sobre las filas seleccionadas, impidiendo que otras transacciones las modifiquen o adquieran locks sobre ellas hasta que la transacción termine.

```sql
-- Bloqueo exclusivo (para escritura)
SELECT * FROM inventario WHERE producto_id = 42 FOR UPDATE;

-- Bloqueo compartido (permite otras lecturas, bloquea escrituras)
SELECT * FROM inventario WHERE producto_id = 42 FOR SHARE;

-- NO esperar si el lock no está disponible (falla inmediatamente)
SELECT * FROM inventario WHERE producto_id = 42 FOR UPDATE NOWAIT;
-- Lanza ERROR si la fila ya está bloqueada

-- Saltar filas bloqueadas en lugar de esperar
SELECT * FROM cola WHERE estado = 'pendiente' FOR UPDATE SKIP LOCKED;
-- Retorna solo las filas que no están bloqueadas
-- Útil para implementar colas de trabajo sin contención
```

**El patrón de la cola de trabajo con SKIP LOCKED:**

```sql
-- Worker que toma el siguiente trabajo disponible sin competir con otros workers
BEGIN;
SELECT tarea_id, datos
FROM cola_tareas
WHERE estado = 'pendiente'
ORDER BY prioridad DESC, creado_en ASC
LIMIT 1
FOR UPDATE SKIP LOCKED;

-- Si hay un resultado, procesar la tarea:
UPDATE cola_tareas SET estado = 'en_proceso', worker_id = $1
WHERE tarea_id = $2;

COMMIT;
```

Múltiples workers pueden ejecutar este patrón concurrentemente sin deadlocks: cada uno toma una fila distinta (las que los demás ya bloquearon son "saltadas").

### 8.6 Locks a nivel de tabla

A veces es necesario bloquear una tabla completa, no solo filas individuales:

```sql
-- Lock de acceso compartido: permite lecturas pero bloquea DDL
LOCK TABLE empleados IN SHARE MODE;

-- Lock exclusivo: bloquea lecturas y escrituras
LOCK TABLE empleados IN EXCLUSIVE MODE;

-- Lock de acceso exclusivo: el más restrictivo (bloquea TODO, incluso SELECT)
LOCK TABLE empleados IN ACCESS EXCLUSIVE MODE;
-- Este es el lock que toman los comandos DDL (ALTER TABLE, DROP TABLE, etc.)
```

Los locks de tabla son necesarios en algunos escenarios de mantenimiento, pero en operaciones normales su uso indica casi siempre un problema de diseño.

---

## 9. El costo de las transacciones largas en MVCC

### 9.1 Por qué las transacciones largas son peligrosas en PostgreSQL

En MVCC, las versiones antiguas de las filas solo pueden ser limpiadas por VACUUM cuando **ninguna transacción activa puede necesitarlas**. Una transacción larga que toma una snapshot al principio mantiene viva esa snapshot durante toda su duración. Todas las versiones de filas que existían cuando comenzó esa transacción **no pueden ser limpiadas** hasta que termine.

```
Tiempo t0: T_larga comienza (toma snapshot con xmin=5000)
Tiempo t1: 1,000,000 de UPDATEs y DELETEs en la tabla X
Tiempo t2: VACUUM intenta limpiar tabla X
           → No puede limpiar NADA porque T_larga todavía está activa
             con snapshot xmin=5000
           → Todas las versiones creadas después de t0 son "posiblemente necesarias"
             para T_larga
Tiempo t3: T_larga termina
Tiempo t4: VACUUM puede limpiar → enorme operación de vacuuming
```

El resultado es **table bloat**: la tabla crece porque acumula versiones muertas que no se pueden limpiar. El performance de todos los scans sobre esa tabla se degrada porque hay que leer páginas con muchas filas muertas.

### 9.2 El problema del wraparound de XIDs

PostgreSQL usa IDs de transacción (XIDs) de 32 bits: los valores van de 0 a 4,294,967,295 (~4 billones). Los XIDs son circulares: después del máximo, vuelven a 0. Con una carga de 1,000 transacciones por segundo, un sistema alcanza el límite en ~136 años. Pero con 10,000 TPS, lo alcanza en ~14 años.

El problema es que PostgreSQL usa la diferencia entre XIDs para determinar visibilidad: una transacción "pasada" es visible, una "futura" no. Si los XIDs se "envuelven" (wraparound), transacciones que deberían verse como pasadas se ven como futuras, corrompiendo la visibilidad.

PostgreSQL previene esto con **VACUUM FREEZE**: VACUUM periódicamente "congela" las filas antiguas, reemplazando su `xmin` con un valor especial que significa "esta fila es visible para todas las transacciones", eliminando la dependencia del XID original.

```sql
-- Ver qué tan lejos está cada tabla del wraparound peligroso
SELECT relname,
       age(relfrozenxid) AS xid_age,
       pg_size_pretty(pg_total_relation_size(oid)) AS tamaño
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;
-- Si xid_age se acerca a 200,000,000, hay que hacer VACUUM urgente
```

Una transacción larga que mantiene su XID activo puede impedir que VACUUM FREEZE progrese, acercando el sistema al wraparound peligroso.

### 9.3 Monitoreo de transacciones largas

```sql
-- Ver transacciones activas y cuánto tiempo llevan
SELECT pid,
       now() - xact_start AS duracion,
       state,
       wait_event_type,
       wait_event,
       LEFT(query, 100) AS query_inicio
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND state != 'idle'
ORDER BY xact_start ASC;

-- Alertar si hay transacciones que llevan más de 5 minutos
SELECT pid, now() - xact_start AS duracion, query
FROM pg_stat_activity
WHERE xact_start < NOW() - INTERVAL '5 minutes'
  AND state != 'idle';
```

### 9.4 El parámetro `idle_in_transaction_session_timeout`

Una fuente común de transacciones largas es código de aplicación que abre una transacción, hace trabajo, y luego se "olvida" de hacer COMMIT (p.ej., una excepción no manejada que no cierra la transacción, o código que inicia una transacción antes de una operación lenta de red).

```sql
-- Matar automáticamente transacciones que llevan más de 5 minutos sin hacer nada
-- (en estado 'idle in transaction')
SET idle_in_transaction_session_timeout = '5min';

-- O configurarlo globalmente en postgresql.conf:
-- idle_in_transaction_session_timeout = '5min'

-- También limitar el tiempo total de una transacción:
SET transaction_timeout = '30min';
```

### 9.5 `lock_timeout` y `statement_timeout`

```sql
-- Fallar si no se puede adquirir un lock en 5 segundos
SET lock_timeout = '5s';

-- Fallar si un statement individual tarda más de 30 segundos
SET statement_timeout = '30s';
```

Estos timeouts son esenciales para sistemas de producción: evitan que queries lentas o que esperan locks por mucho tiempo bloqueen otras operaciones.

---

## 10. Patrones de diseño para concurrencia correcta

### 10.1 El patrón optimistic locking

En lugar de bloquear las filas al leer, se verifica en el momento de escribir si alguien modificó los datos desde que se leyeron. Si los datos cambiaron, la operación falla y debe reintentarse.

```sql
-- Tabla con columna de versión
CREATE TABLE productos (
    producto_id  INT PRIMARY KEY,
    nombre       VARCHAR(100),
    stock        INT NOT NULL,
    version      INT NOT NULL DEFAULT 1
);

-- Proceso de actualización optimista:
-- 1. Leer el estado actual (sin locks)
SELECT producto_id, stock, version FROM productos WHERE producto_id = 42;
-- stock = 100, version = 5

-- 2. Hacer el trabajo en la aplicación...

-- 3. Escribir solo si la versión no cambió
UPDATE productos
SET stock = 97, version = version + 1
WHERE producto_id = 42 AND version = 5;  -- ← condición de versión
-- Si rows_affected = 0: alguien modificó el registro → reintentar
-- Si rows_affected = 1: éxito
```

**Cuándo usar optimistic locking:**
- Los conflictos son raros (la mayoría de las veces nadie modifica el mismo registro concurrentemente).
- Las operaciones de lectura son mucho más frecuentes que las de escritura.
- El costo de reintentar es bajo.

**Cuándo NO usarlo:**
- Los conflictos son frecuentes (alta contención sobre las mismas filas): se generan demasiados reintentos.
- Las operaciones de escritura son largas y el reintento es costoso.

### 10.2 El patrón pessimistic locking

Bloquear explícitamente la fila al leerla, antes de procesarla, para garantizar que nadie más pueda modificarla mientras se trabaja.

```sql
BEGIN;
-- Bloquear la fila antes de decidir qué hacer con ella
SELECT producto_id, stock
FROM productos
WHERE producto_id = 42
FOR UPDATE;  -- ← adquiere el lock aquí

-- Si stock >= 3, proceder con la venta
UPDATE productos SET stock = stock - 3 WHERE producto_id = 42;
COMMIT;
```

**Cuándo usar pessimistic locking:**
- Los conflictos son frecuentes y el costo de reintentar es alto.
- La operación requiere tomar decisiones basadas en el estado actual que no pueden cambiar mientras se procesa.
- El tiempo de procesamiento es corto (el lock debe tenerse el menor tiempo posible).

### 10.3 El patrón de contador atómico

Para operaciones de incremento/decremento de contadores, el uso de `UPDATE ... SET col = col + N` es atómico dentro de una transacción y evita el ciclo read-modify-write:

```sql
-- Correcto: atomic update (no hay ciclo read-modify-write vulnerable)
UPDATE inventario SET stock = stock - 3 WHERE producto_id = 42 AND stock >= 3;
-- Si rows_affected = 0: no había suficiente stock

-- Incorrecto: el ciclo read-modify-write crea una ventana de concurrencia
SELECT stock FROM inventario WHERE producto_id = 42;  -- lee 10
-- Aquí otra transacción puede comprar y reducir el stock
UPDATE inventario SET stock = 7 WHERE producto_id = 42;  -- puede ir a negativo
```

### 10.4 Diseño de transacciones cortas

Las transacciones largas causan bloat y contención. Los principios para mantenerlas cortas:

**No hacer trabajo lento dentro de una transacción:**

```python
# Incorrecto: llamada a API externa dentro de la transacción
with connection.transaction():
    datos = db.query("SELECT * FROM pedido WHERE id = $1", pedido_id)
    resultado = llamar_api_externa(datos)  # puede tardar segundos
    db.execute("UPDATE pedido SET estado = $1 WHERE id = $2", resultado, pedido_id)

# Correcto: hacer el trabajo fuera de la transacción
datos = db.query("SELECT * FROM pedido WHERE id = $1", pedido_id)  # sin transacción
resultado = llamar_api_externa(datos)  # fuera de la transacción
with connection.transaction():
    # Solo la escritura final va dentro de la transacción
    db.execute("UPDATE pedido SET estado = $1 WHERE id = $2", resultado, pedido_id)
```

**No pedir input al usuario dentro de una transacción:**

```python
# Incorrecto: la transacción queda abierta mientras el usuario piensa
with connection.transaction():
    items = db.query("SELECT * FROM carrito WHERE usuario_id = $1", usuario_id)
    confirmacion = input("¿Confirmar compra? (s/n): ")  # puede tardar minutos
    if confirmacion == 's':
        db.execute("INSERT INTO pedido ...")
```

**Procesar en lotes para operaciones masivas:**

```sql
-- Incorrecto: una transacción que actualiza millones de filas
BEGIN;
UPDATE productos SET precio = precio * 1.1;  -- 10M filas, tarda minutos
COMMIT;

-- Correcto: procesar en lotes de 10,000 filas
DO $$
DECLARE
    filas_procesadas INT;
BEGIN
    LOOP
        UPDATE productos
        SET precio = precio * 1.1
        WHERE producto_id IN (
            SELECT producto_id FROM productos
            WHERE precio_actualizado = FALSE
            LIMIT 10000
        );
        
        GET DIAGNOSTICS filas_procesadas = ROW_COUNT;
        EXIT WHEN filas_procesadas = 0;
        
        PERFORM pg_sleep(0.1);  -- pequeña pausa entre lotes
    END LOOP;
END $$;
```

### 10.5 El patrón de advisory locks

PostgreSQL ofrece **advisory locks**: mecanismo de bloqueo de propósito general que no está asociado a ninguna fila o tabla específica. Son locks que la aplicación puede adquirir y liberar para coordinar acceso a recursos arbitrarios.

```sql
-- Adquirir un advisory lock (bloquea hasta que esté disponible)
SELECT pg_advisory_lock(12345);  -- 12345 es un ID arbitrario que identifica el recurso

-- Intentar adquirir sin bloquear (retorna FALSE si no está disponible)
SELECT pg_try_advisory_lock(12345);

-- Liberar el lock
SELECT pg_advisory_unlock(12345);

-- Advisory lock a nivel de transacción (se libera automáticamente al COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(12345);
```

**Caso de uso típico:** Garantizar que solo un proceso ejecute una tarea a la vez (p.ej., un job de facturación mensual que no debe ejecutarse dos veces simultáneamente):

```sql
BEGIN;
-- Intentar adquirir el lock para el job de facturación (ID=9999)
-- Si otro proceso ya lo tiene, esto retorna FALSE (y no bloqueamos)
SELECT pg_try_advisory_xact_lock(9999) AS pudo_adquirir;

-- Si pudo_adquirir = TRUE, ejecutar el job
-- Si pudo_adquirir = FALSE, salir: otro proceso ya está ejecutando el job
COMMIT;  -- El lock se libera automáticamente
```

---

## 11. Resumen y conexión con el resto del curso

### Los conceptos clave y cuándo aplicarlos

| Concepto | Aplicación práctica |
|---|---|
| **ACID** | Entender qué garantiza el motor y qué es responsabilidad del código |
| **Read Committed** | El nivel por defecto; adecuado para operaciones web simples |
| **Repeatable Read / SI** | Reportes consistentes, transferencias, read-modify-write patterns |
| **Serializable / SSI** | Invariantes que dependen de múltiples filas; write skew posible |
| **MVCC** | Explica por qué las lecturas no bloquean escrituras; y por qué hay bloat |
| **VACUUM** | Mantenimiento esencial; sin él, el rendimiento y el espacio se degradan |
| **Deadlocks** | Siempre reintentar; prevenir con orden consistente de adquisición de locks |
| **Transacciones cortas** | Principio de diseño para reducir contención y bloat |
| **SELECT FOR UPDATE** | Bloqueo explícito para operaciones read-modify-write críticas |
| **SKIP LOCKED** | Patrón para colas de trabajo sin contención |

### La meta-lección del módulo

El control de concurrencia es el área donde más frecuentemente el comportamiento del sistema en producción diverge del comportamiento esperado en desarrollo. Hay tres razones:

1. **Los bugs de concurrencia son no deterministas:** Ocurren solo bajo carga y condiciones de timing específicas que son difíciles de reproducir.
2. **El nivel de aislamiento por defecto (Read Committed) no previene lost updates ni write skew:** Operaciones que parecen correctas en desarrollo pueden silenciosamente producir resultados incorrectos bajo concurrencia real.
3. **Las transacciones largas tienen efectos colaterales que no son obvios:** Bloat, bloqueo de vacuum, wrap-around risk.

La respuesta no es usar siempre Serializable (el overhead puede ser inaceptable). La respuesta es entender qué garantías necesita cada operación y elegir el nivel y los mecanismos adecuados:

- Para operaciones de solo lectura: Read Committed es suficiente.
- Para operaciones read-modify-write sobre filas individuales: el UPDATE atómico (`SET col = col + N`) o `SELECT FOR UPDATE` con Read Committed.
- Para invariantes que abarcan múltiples filas: Serializable o diseño explícito con locks.
- Para todo lo demás: Repeatable Read con manejo de errores de serialización.

### Conexión con módulos futuros

- **Módulo 9 (Modelado físico e índices):** Los locks de fila y de página interactúan con la estructura física de los índices. Un index scan puede adquirir locks en páginas del índice y del heap que causen contención inesperada con operaciones de mantenimiento como `REINDEX` o `CREATE INDEX CONCURRENTLY`.

- **Módulo 10 (OLTP vs OLAP):** Una de las razones por las que los sistemas OLAP se separan de los OLTP es exactamente el conflicto de concurrencia: una query analítica que escanea toda la tabla durante minutos, incluso sin bloquear lecturas (MVCC), genera bloat y mantiene snapshots que impiden el VACUUM.

- **Módulo 13 (Sistemas distribuidos):** En sistemas distribuidos, las propiedades ACID son mucho más costosas de mantener. El teorema CAP y los modelos de consistencia eventual son, en parte, consecuencias del costo de mantener aislamiento serializable a través de múltiples nodos.

- **Módulo 16 (El oficio):** El diagnóstico de bugs de concurrencia en producción, el diseño de pruebas de carga que reproduzcan condiciones de concurrencia, y la lectura de los logs de deadlocks son habilidades prácticas fundamentales.

---

## 12. Ejercicios de comprensión

**Ejercicio 1.** Dado el siguiente escenario de transferencia bancaria:

```sql
BEGIN;
SELECT saldo FROM cuentas WHERE cuenta_id = 'A';  -- lee 1000
-- La aplicación verifica que 1000 >= 500
UPDATE cuentas SET saldo = saldo - 500 WHERE cuenta_id = 'A';
UPDATE cuentas SET saldo = saldo + 500 WHERE cuenta_id = 'B';
COMMIT;
```

a) ¿Este código tiene una condición de carrera bajo Read Committed? Describe el escenario exacto en que falla.
b) ¿El problema desaparece bajo Repeatable Read? ¿Por qué sí o por qué no?
c) Reescribe la transacción para que sea correcta bajo Read Committed sin cambiar el nivel de aislamiento.

---

**Ejercicio 2.** Considera el siguiente sistema de reservas de asientos en un avión con capacidad de 150 pasajeros:

```sql
CREATE TABLE reservas (
    reserva_id  SERIAL PRIMARY KEY,
    vuelo_id    INT NOT NULL,
    pasajero_id INT NOT NULL,
    asiento     VARCHAR(4) NOT NULL,
    UNIQUE (vuelo_id, asiento)
);
```

a) ¿Qué fenómeno de concurrencia podría causar que se vendan más de 150 asientos, a pesar de la constraint UNIQUE?
b) Diseña la transacción correcta para reservar un asiento, incluyendo el nivel de aislamiento apropiado y los mecanismos de locking necesarios.
c) ¿Cómo detectarías en producción que este problema está ocurriendo?

---

**Ejercicio 3.** El escenario del write skew de los médicos de guardia:

```sql
CREATE TABLE guardias (
    medico_id  INT PRIMARY KEY,
    nombre     VARCHAR(100),
    en_guardia BOOLEAN NOT NULL
);
```

La invariante: siempre debe haber al menos un médico de guardia.

a) Demuestra (con el sequence exacto de operaciones SQL) cómo dos transacciones bajo Snapshot Isolation pueden violar esta invariante.
b) ¿Por qué Repeatable Read (que en PostgreSQL es SI) no es suficiente para prevenir esto?
c) Propón dos soluciones distintas: una usando Serializable, otra usando locks explícitos con Read Committed.

---

**Ejercicio 4.** Analiza el siguiente código Python que implementa un sistema de puntos de fidelidad:

```python
def canjear_puntos(usuario_id, puntos_a_canjear):
    with db.transaction():
        usuario = db.query_one("SELECT puntos FROM usuarios WHERE id = %s", usuario_id)
        if usuario.puntos < puntos_a_canjear:
            raise ValueError("Puntos insuficientes")
        db.execute("UPDATE usuarios SET puntos = puntos - %s WHERE id = %s",
                   puntos_a_canjear, usuario_id)
        db.execute("INSERT INTO canjes (usuario_id, puntos) VALUES (%s, %s)",
                   usuario_id, puntos_a_canjear)
```

a) ¿Qué condición de carrera existe en este código?
b) ¿El problema ocurre bajo Read Committed? ¿Bajo Repeatable Read?
c) Identifica al menos dos formas de corregirlo, con sus respectivos trade-offs.

---

**Ejercicio 5.** El siguiente sistema de procesamiento de pagos tiene un problema de deadlock recurrente en producción:

```sql
-- Transacción A (procesar pago):
BEGIN;
UPDATE cuentas SET saldo = saldo - monto WHERE cuenta_id = origen;
UPDATE transacciones SET estado = 'procesado' WHERE trans_id = X;
COMMIT;

-- Transacción B (registrar comisión, se ejecuta concurrentemente con A):
BEGIN;
UPDATE transacciones SET comision = monto * 0.02 WHERE trans_id = X;
UPDATE cuentas SET saldo = saldo - comision WHERE cuenta_id = origen;
COMMIT;
```

a) Explica exactamente cómo se produce el deadlock.
b) Propón la solución mínima que elimine el deadlock sin cambiar la lógica del negocio.
c) ¿Hay otras mejoras que harías al diseño de estas transacciones?

---

**Ejercicio 6.** Un sistema de e-commerce tiene una tabla `pedidos` con 50 millones de filas. El DBA observa que la tabla ocupa 40 GB pero `pg_stat_user_tables` muestra que solo hay 8 millones de filas vivas (`n_live_tup = 8M`, `n_dead_tup = 42M`).

a) ¿Cuál es la causa más probable de esta situación?
b) ¿Por qué el autovacuum no ha limpiado las filas muertas?
c) ¿Cuáles son las consecuencias de rendimiento de este estado?
d) ¿Qué pasos concretos tomarías para diagnosticar y resolver el problema? Incluye las queries de diagnóstico y los comandos de mantenimiento.

---

*Próximo módulo: Modelado físico e índices — donde descenderemos al nivel de bytes y páginas para entender cómo la estructura física de los datos y los índices determina qué operaciones son rápidas y cuáles son inevitablemente lentas, y cómo esas decisiones se conectan de vuelta al modelo lógico.*
