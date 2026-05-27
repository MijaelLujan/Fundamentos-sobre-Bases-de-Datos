# Módulo 9 — Modelado físico e índices

> *"El modelo lógico describe qué guardar y cómo se relaciona. El modelo físico describe cómo eso se traduce en bytes sobre un disco. Un modelo lógico impecable puede ser irrelevante si el modelo físico es incorrecto: el mejor esquema del mundo, sin los índices adecuados o con tipos de datos desperdiciadores, puede ser tan lento como si no hubiera diseño alguno. Y un modelo físico sobre-indexado puede arruinar el rendimiento de escritura de un sistema que de otra forma funcionaría bien."*

---

## Tabla de contenidos

1. [El modelo físico como capa separada](#1-el-modelo-físico-como-capa-separada)
2. [Tipos de datos: costos de almacenamiento y alineación](#2-tipos-de-datos-costos-de-almacenamiento-y-alineación)
3. [Organización física del heap](#3-organización-física-del-heap)
4. [Índices B-tree: estructura interna y comportamiento](#4-índices-b-tree-estructura-interna-y-comportamiento)
5. [Índices hash](#5-índices-hash)
6. [Índices bitmap](#6-índices-bitmap)
7. [Índices GIN: para búsquedas en arrays y texto completo](#7-índices-gin-para-búsquedas-en-arrays-y-texto-completo)
8. [Índices GiST: para geometría, rangos y tipos complejos](#8-índices-gist-para-geometría-rangos-y-tipos-complejos)
9. [Índices compuestos y el orden de columnas](#9-índices-compuestos-y-el-orden-de-columnas)
10. [Índices parciales](#10-índices-parciales)
11. [Índices de expresión](#11-índices-de-expresión)
12. [Index-only scans y covering indexes](#12-index-only-scans-y-covering-indexes)
13. [Cuándo un índice destruye el rendimiento de escritura](#13-cuándo-un-índice-destruye-el-rendimiento-de-escritura)
14. [Particionamiento de tablas](#14-particionamiento-de-tablas)
15. [Clustering físico y fill factor](#15-clustering-físico-y-fill-factor)
16. [El proceso de diseño del modelo físico](#16-el-proceso-de-diseño-del-modelo-físico)
17. [Resumen y conexión con el resto del curso](#17-resumen-y-conexión-con-el-resto-del-curso)
18. [Ejercicios de comprensión](#18-ejercicios-de-comprensión)

---

## 1. El modelo físico como capa separada

### 1.1 Qué pertenece al modelo físico

En el Módulo 1 establecimos la separación entre el nivel conceptual (el modelo lógico: relaciones, atributos, restricciones) y el nivel interno (el modelo físico: páginas, índices, archivos). El módulo físico cubre exactamente ese nivel interno: las decisiones que no cambian qué datos guarda el modelo, pero sí con qué eficiencia se almacenan y se acceden.

Las decisiones del modelo físico incluyen:
- **Tipos de datos concretos:** `INT` vs `BIGINT`, `VARCHAR(100)` vs `TEXT`, `TIMESTAMP` vs `TIMESTAMPTZ`.
- **Índices:** qué columnas indexar, con qué tipo de índice, en qué orden.
- **Particionamiento:** dividir una tabla grande en partes manejables.
- **Clustering físico:** si las filas están almacenadas en un orden físico que favorece ciertos accesos.
- **Fill factor:** qué porcentaje de cada página se llena inicialmente.
- **TOAST:** cómo el motor almacena valores grandes.

### 1.2 La independencia física como objetivo y sus límites

El ideal de la arquitectura ANSI/SPARC es que el modelo físico sea completamente transparente: se puede cambiar sin afectar el modelo lógico ni las aplicaciones. En la práctica, esta independencia es parcial:

- Agregar un índice no cambia el modelo lógico ni el comportamiento de las queries (seguirán siendo correctas), pero puede cambiar drásticamente su rendimiento.
- Cambiar un tipo de dato de `INT` a `BIGINT` puede requerir modificar código de aplicación que asumía rangos específicos.
- El particionamiento puede cambiar el comportamiento de ciertas queries si el planner no hace partition pruning correctamente.

La independencia física es real para los cambios que solo afectan el rendimiento (agregar/eliminar índices, ajustar fill factor). Es parcial para los cambios que afectan capacidades (cambiar tipos, agregar particionamiento).

---

## 2. Tipos de datos: costos de almacenamiento y alineación

### 2.1 Por qué el tipo de dato importa más allá de la semántica

La elección del tipo de dato afecta:
- **El espacio en disco:** directamente proporcional al número de filas y páginas que el motor debe leer.
- **La alineación en memoria:** los procesadores modernos acceden a datos alineados más eficientemente.
- **La capacidad de usar ciertos índices:** algunos tipos de índice solo funcionan con tipos específicos.
- **El rendimiento de las comparaciones:** comparar dos `INT` es una instrucción de CPU; comparar dos `VARCHAR` de longitud variable requiere recorrer caracteres.
- **Los casts implícitos:** un predicado `WHERE col = 123` sobre una columna `VARCHAR` requiere un cast que puede invalidar el uso del índice.

### 2.2 Tipos numéricos

| Tipo | Bytes | Rango | Cuándo usar |
|---|---|---|---|
| `SMALLINT` | 2 | -32,768 a 32,767 | Contadores pequeños, códigos de estado |
| `INTEGER` / `INT` | 4 | ~-2.1B a ~2.1B | El entero de propósito general |
| `BIGINT` | 8 | ~-9.2 × 10¹⁸ a ~9.2 × 10¹⁸ | IDs de tablas muy grandes, timestamps en ms |
| `NUMERIC(p,s)` | Variable | Exacto, cualquier precisión | Moneda, cálculos financieros |
| `REAL` | 4 | ~6 dígitos de precisión | Valores aproximados, científicos |
| `DOUBLE PRECISION` | 8 | ~15 dígitos de precisión | Cálculos más precisos, coordenadas |
| `SERIAL` | 4 | Alias de INT + secuencia | Claves primarias autoincrementales (legado) |
| `BIGSERIAL` | 8 | Alias de BIGINT + secuencia | Claves primarias de tablas grandes (legado) |

**El problema de `NUMERIC` para moneda:**

`NUMERIC(12,2)` almacena exactamente dos decimales. Es correcto para valores en unidades de moneda. Pero `NUMERIC` tiene overhead de almacenamiento variable y las operaciones aritméticas son más lentas que con enteros. Una alternativa común es almacenar montos en centavos como `BIGINT`:

```sql
-- Con NUMERIC: 1234.56 se almacena como 1234.56
monto NUMERIC(12,2)

-- Con BIGINT en centavos: 1234.56 se almacena como 123456
monto_centavos BIGINT
-- Requiere dividir por 100 en la capa de presentación, pero es más eficiente
```

**El problema de `FLOAT` para moneda:**

```sql
-- NUNCA usar FLOAT/REAL/DOUBLE para moneda:
SELECT 0.1 + 0.2;
-- Resultado: 0.30000000000000004  ← representación binaria imprecisa
-- 0.1 no es representable exactamente en IEEE 754 binario
```

Los tipos de punto flotante (`REAL`, `DOUBLE PRECISION`) usan representación IEEE 754 binaria, que no puede representar exactamente fracciones decimales como 0.1 o 0.3. Para moneda, usar siempre `NUMERIC` o enteros.

### 2.3 Tipos de texto

| Tipo | Almacenamiento | Cuándo usar |
|---|---|---|
| `CHAR(n)` | Exactamente n bytes (padding con espacios) | Prácticamente nunca. Legado. |
| `VARCHAR(n)` | Variable, máximo n caracteres | Cuando el límite tiene significado de negocio |
| `TEXT` | Variable, sin límite | El tipo de texto de propósito general |

**La trampa de `CHAR(n)`:**

`CHAR(n)` rellena con espacios hasta n caracteres. `CHAR(10)` almacenando 'Ana' ocupa exactamente 10 bytes (`Ana       `). Además, las comparaciones ignoran los espacios finales de forma inconsistente entre motores. No hay razón para usar `CHAR(n)` en PostgreSQL.

**`VARCHAR(n)` vs `TEXT`:**

En PostgreSQL, `VARCHAR(n)` y `TEXT` tienen el mismo rendimiento de almacenamiento. La única diferencia es que `VARCHAR(n)` incluye una restricción de longitud máxima. Preferir `TEXT` cuando no hay un límite semánticamente significativo, o cuando el límite podría cambiar (cambiar el límite de `VARCHAR(100)` a `VARCHAR(200)` en una tabla con millones de filas requiere un `ALTER TABLE` que puede ser costoso).

```sql
-- Preferible para nombres genéricos:
nombre TEXT NOT NULL

-- Cuando el límite tiene significado real (código postal siempre 5 dígitos):
codigo_postal VARCHAR(5) NOT NULL
```

**TOAST (The Oversized-Attribute Storage Technique):**

Cuando un valor de texto (u otro tipo variable) supera ~2KB, PostgreSQL lo almacena fuera de la página principal en una tabla TOAST separada. Esto es transparente para las queries, pero tiene implicaciones de rendimiento:

- Las queries que leen solo columnas no-TOAST no tocan la tabla TOAST (eficiente).
- Las queries que leen columnas TOAST para filas específicas hacen un acceso adicional a la tabla TOAST (un I/O extra por fila).
- `SELECT COUNT(*) FROM tabla` sobre una tabla con columnas TOAST grandes no es más lento que sobre una sin ellas, porque COUNT no necesita leer los valores TOAST.

### 2.4 Tipos de fecha y tiempo

| Tipo | Bytes | Precisión | Cuándo usar |
|---|---|---|---|
| `DATE` | 4 | Día | Fechas sin hora (nacimientos, vencimientos) |
| `TIME` | 8 | Microsegundo | Hora del día sin fecha (horarios) |
| `TIMESTAMP` | 8 | Microsegundo | Fecha + hora sin zona horaria |
| `TIMESTAMPTZ` | 8 | Microsegundo | Fecha + hora con zona horaria |
| `INTERVAL` | 16 | Microsegundo | Duraciones, diferencias de tiempo |

**`TIMESTAMP` vs `TIMESTAMPTZ`:**

Esta es una de las decisiones más importantes y más frecuentemente mal tomada. `TIMESTAMPTZ` almacena el timestamp en UTC internamente y lo convierte a la zona horaria de la sesión al leer. `TIMESTAMP` almacena exactamente lo que recibe, sin conversión.

```sql
-- Problema con TIMESTAMP en un sistema con usuarios en múltiples zonas horarias:
-- Un usuario en NYC inserta '2024-03-15 10:00:00' (hora local EST = UTC-5)
-- Un usuario en Madrid ve '2024-03-15 10:00:00' (interpreta como hora local CET = UTC+1)
-- Ambos ven la misma cadena pero representan momentos distintos

-- Con TIMESTAMPTZ:
-- El motor convierte a UTC al insertar: 2024-03-15 15:00:00 UTC
-- Al leer, convierte a la zona local de cada usuario
-- Todos ven el mismo momento en su hora local
```

**Regla práctica:** Usar `TIMESTAMPTZ` siempre que el timestamp represente un momento específico en el tiempo (cuándo ocurrió un evento). Usar `TIMESTAMP` solo cuando el valor representa "hora del reloj en un contexto local" sin importar la zona horaria (p.ej., horario de atención de una tienda local).

### 2.5 Tipos especiales de PostgreSQL

**`UUID`:** Identificador único universal de 128 bits.

```sql
-- UUID como clave primaria
CREATE TABLE pedidos (
    pedido_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ...
);
```

**Ventajas:** globalmente único sin coordinación, no revela información secuencial sobre el negocio, útil en sistemas distribuidos.
**Desventajas:** 16 bytes vs 4-8 bytes para INT/BIGINT; los UUIDs v4 son aleatorios, lo que causa fragmentación severa en índices B-tree y menor localidad de caché que los IDs secuenciales. UUIDs v7 (ordenados por tiempo) mitigan este problema.

**`JSONB`:** JSON almacenado en formato binario indexable.

```sql
-- JSONB para atributos heterogéneos (patrón del Módulo 5)
CREATE TABLE productos (
    producto_id  INT PRIMARY KEY,
    tipo         VARCHAR(50) NOT NULL,
    atributos    JSONB
);

-- Índice GIN sobre JSONB para búsquedas eficientes
CREATE INDEX idx_prod_atributos ON productos USING GIN (atributos);

-- Query sobre JSONB con índice
SELECT * FROM productos WHERE atributos @> '{"voltaje": "110V"}';
```

**`ARRAY`:** Arreglos multidimensionales de cualquier tipo base.

```sql
CREATE TABLE etiquetas (
    articulo_id  INT PRIMARY KEY,
    tags         TEXT[] NOT NULL DEFAULT '{}'
);

-- Índice GIN para búsquedas en arrays
CREATE INDEX idx_tags ON etiquetas USING GIN (tags);

-- Buscar artículos con la etiqueta 'postgresql'
SELECT * FROM etiquetas WHERE tags @> ARRAY['postgresql'];
```

### 2.6 Alineación y padding en la fila

Los procesadores leen datos de memoria en alineaciones naturales: un `INT` de 4 bytes debe estar en una dirección múltiplo de 4; un `BIGINT` de 8 bytes en una dirección múltiplo de 8. Si los datos no están alineados, el procesador necesita múltiples lecturas.

PostgreSQL almacena los campos de una fila en el orden en que están declarados, con padding entre campos para respetar la alineación de cada tipo. Esto significa que **el orden de las columnas en el `CREATE TABLE` afecta el tamaño de cada fila**.

```sql
-- Diseño ineficiente: alternancia de tipos de distintos tamaños
-- genera padding entre columnas
CREATE TABLE mal_alineada (
    a  SMALLINT,    -- 2 bytes
    b  BIGINT,      -- 8 bytes (necesita padding de 6 bytes después de a)
    c  SMALLINT,    -- 2 bytes
    d  BIGINT,      -- 8 bytes (necesita padding de 6 bytes después de c)
    e  INT          -- 4 bytes
);
-- Tamaño por fila: 2 + 6(pad) + 8 + 2 + 6(pad) + 8 + 4 = 36 bytes

-- Diseño eficiente: agrupar por tamaño (mayor a menor)
CREATE TABLE bien_alineada (
    b  BIGINT,      -- 8 bytes
    d  BIGINT,      -- 8 bytes
    e  INT,         -- 4 bytes
    a  SMALLINT,    -- 2 bytes
    c  SMALLINT     -- 2 bytes (puede empaquetarse junto con a)
);
-- Tamaño por fila: 8 + 8 + 4 + 2 + 2 = 24 bytes
```

En la práctica, la diferencia de alineación raramente justifica reorganizar columnas si ya hay datos (requiere `ALTER TABLE`, que puede ser costoso). Pero en diseños nuevos, ordenar las columnas de mayor a menor alineación es una optimización sencilla y sin costo.

---

## 3. Organización física del heap

### 3.1 Páginas: la unidad fundamental de almacenamiento

PostgreSQL organiza los datos en **páginas** (también llamadas bloques) de tamaño fijo. El tamaño por defecto es **8 KB**. Todas las operaciones de I/O se realizan en unidades de páginas: leer una sola fila de una tabla implica leer toda la página que la contiene.

Esta es la razón por la que el número de páginas que una query debe leer es el factor más importante para su rendimiento: más páginas = más I/O = más tiempo.

```
┌─────────────────────────────────────────────┐
│ Page Header (24 bytes)                       │
│ - LSN: posición en el WAL                    │
│ - checksum, flags                            │
├─────────────────────────────────────────────┤
│ Item Pointers (array de 4 bytes cada uno)   │
│ ItemPtr[1] → offset de la fila 1 en la pág  │
│ ItemPtr[2] → offset de la fila 2 en la pág  │
│ ...                                          │
├─────────────────────────────────────────────┤
│ Free Space (espacio libre)                  │
│                                              │
│ (nuevo espacio libre para inserciones)       │
├─────────────────────────────────────────────┤
│ Tuple 3 (fila 3, incluye xmin, xmax, datos) │
│ Tuple 2 (fila 2)                            │
│ Tuple 1 (fila 1)                            │
│ (las filas se almacenan de abajo hacia arr.) │
└─────────────────────────────────────────────┘
```

### 3.2 El heap como archivo no ordenado

El **heap** es el archivo principal donde se almacenan las filas de una tabla. Las filas se insertan donde haya espacio libre, sin ningún orden predeterminado. Un `SELECT * FROM tabla` sin `ORDER BY` puede retornar las filas en cualquier orden (que generalmente corresponde al orden físico, pero no está garantizado).

Esta ausencia de orden es la razón por la que los índices son necesarios: sin un índice, la única forma de encontrar una fila específica es leer todo el heap (sequential scan).

### 3.3 El ctid: dirección física de una fila

Cada fila tiene un `ctid` (tuple ID) que es su dirección física: `(bloque, offset)`.

```sql
SELECT ctid, empleado_id, nombre FROM empleados LIMIT 5;
-- (0,1) → bloque 0, posición 1 dentro del bloque
-- (0,2) → bloque 0, posición 2
-- (0,3) → bloque 0, posición 3
-- (1,1) → bloque 1, posición 1  (la página 0 se llenó)
-- (1,2) → bloque 1, posición 2
```

El `ctid` cambia cuando se hace `VACUUM FULL` (reescribe la tabla) o `CLUSTER` (reordena físicamente). Por eso, **nunca debe usarse `ctid` como referencia persistente**: es solo para diagnóstico.

---

## 4. Índices B-tree: estructura interna y comportamiento

### 4.1 La estructura del árbol B

El **árbol B** (y su variante árbol B+, que es lo que PostgreSQL implementa) es la estructura de índice más universal y la que se usa por defecto al crear cualquier índice sin especificar tipo.

Un árbol B+ tiene la siguiente estructura:

```
                    ┌─────────────────┐
                    │  Nodo raíz      │
                    │  [50 | 150]     │
                    └────┬──────┬─────┘
              ┌──────────┘      └──────────┐
    ┌─────────────────┐        ┌─────────────────┐
    │  Nodo interno   │        │  Nodo interno   │
    │  [20 | 35]      │        │  [80 | 120]     │
    └──┬──────┬───────┘        └──┬──────┬───────┘
       │      │                   │      │
  ┌────▼──┐ ┌──▼────┐        ┌────▼──┐ ┌──▼────┐
  │ Hoja  │ │ Hoja  │  ...   │ Hoja  │ │ Hoja  │
  │ 10,15 │ │ 20,25 │        │ 80,90 │ │120,130│
  │ 18,19 │ │ 30,35 │        │ 95,100│ │140,148│
  └───────┘ └───────┘        └───────┘ └───────┘
      ↑ Las hojas están enlazadas entre sí (lista doble enlazada)
```

**Características clave del árbol B+ de PostgreSQL:**
- Los nodos internos solo contienen claves de navegación, no datos.
- Las hojas contienen las claves reales junto con punteros (`ctid`) a las filas del heap.
- Las hojas están enlazadas en una lista doblemente enlazada, lo que permite recorridos eficientes en rango sin volver al nodo raíz.
- El árbol está balanceado: todas las hojas están a la misma profundidad. La búsqueda de cualquier valor siempre recorre exactamente la misma cantidad de niveles.
- La altura del árbol crece logarítmicamente con el número de entradas. Un índice B-tree en una columna con 10 millones de filas tiene una altura de apenas 4-5 niveles.

### 4.2 Operaciones en el árbol B

**Búsqueda de un valor (O(log n)):**

1. Comenzar en el nodo raíz.
2. Comparar el valor buscado con las claves del nodo para decidir qué hijo visitar.
3. Descender hasta la hoja que podría contener el valor.
4. En la hoja, buscar el valor exacto y obtener el `ctid`.
5. Con el `ctid`, acceder a la fila en el heap.

Para una tabla con 10 millones de filas: ~4-5 comparaciones en el índice + 1 acceso al heap = ~5-6 I/Os en total. Versus un sequential scan que podría requerir miles de I/Os.

**Búsqueda de un rango (O(log n + k), donde k = filas en el rango):**

1. Buscar el límite inferior del rango (igual que una búsqueda exacta).
2. Desde la hoja encontrada, avanzar por la lista enlazada de hojas hasta alcanzar el límite superior.
3. Para cada entrada en el rango, acceder a la fila en el heap.

Esto es muy eficiente para rangos: después de encontrar el inicio, recorrer el rango es O(k).

**Inserción:**

1. Encontrar la hoja donde debe ir el nuevo valor.
2. Insertar en la hoja.
3. Si la hoja se llenó: **split** (dividir la hoja en dos, promover la clave mediana al nodo padre).
4. Si el nodo padre también se llenó: split recursivo hacia arriba.
5. Si la raíz se divide: crear una nueva raíz, aumentando la altura del árbol en 1.

**Implicación de las inserciones:** La fragmentación ocurre gradualmente. Si los valores se insertan en orden aleatorio (como UUIDs v4), casi todas las inserciones aterrizan en nodos que ya están llenos o casi llenos, causando splits frecuentes y dejando páginas medio vacías. Este es el origen del problema de fragmentación con UUIDs aleatorios como claves primarias.

### 4.3 Operadores que puede usar un B-tree

Un índice B-tree solo puede ser usado por el motor cuando el predicado usa un operador que el B-tree soporta. Para tipos estándar, estos son:

- `=` (igualdad)
- `<`, `<=`, `>`, `>=` (comparaciones de rango)
- `BETWEEN ... AND ...`
- `IN (...)` (el planner puede expandirlo como múltiples búsquedas `=`)
- `IS NULL` / `IS NOT NULL`
- `LIKE 'prefijo%'` (solo si el patrón no empieza con `%`)

**El B-tree NO puede usarse para:**
- `LIKE '%sufijo'` o `LIKE '%subcadena%'` (patrones que no son prefijos)
- Operadores de arrays: `@>`, `<@`, `&&`
- Búsqueda en JSONB
- Búsqueda por distancia geográfica

### 4.4 Fragmentación y el comando REINDEX

Con el tiempo, las inserciones, updates y deletes dejan páginas parcialmente vacías en el índice (fragmentación interna). La ratio de páginas usadas vs páginas totales se llama **densidad** del índice.

```sql
-- Ver el tamaño y estadísticas de un índice (requiere extensión pgstattuple)
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT *
FROM pgstatindex('idx_empleados_apellido');
-- leaf_density: porcentaje de espacio útil en las hojas (100% = perfectamente compacto)
-- avg_leaf_density: promedio
-- index_size: tamaño total en bytes

-- Reconstruir un índice fragmentado (bloquea la tabla)
REINDEX INDEX idx_empleados_apellido;

-- Reconstruir sin bloquear (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_empleados_apellido;
```

---

## 5. Índices hash

### 5.1 Qué son y cuándo usarlos

Un índice hash aplica una función de hash a la clave de cada fila y almacena los pares (hash, ctid) en una tabla hash. Son extremadamente eficientes para búsquedas de igualdad exacta: O(1) en teoría.

```sql
-- Crear un índice hash
CREATE INDEX idx_empleados_email_hash ON empleados USING HASH (email);

-- Solo es útil para predicados de igualdad
SELECT * FROM empleados WHERE email = 'ana@empresa.com';
-- El índice hash es O(1) para esta query
```

**Limitaciones importantes:**
- Solo sirve para predicados de **igualdad exacta** (`=`). Inútil para rangos, prefijos, o NULL.
- No puede ser usado en índices compuestos.
- No puede ser usado para ordenamiento.
- Históricamente (antes de PostgreSQL 10) no eran write-ahead logged y se corrompían en crashes. Desde PostgreSQL 10 son completamente seguros.

**¿Cuándo usar hash sobre B-tree para igualdad?**

En teoría, el hash es O(1) vs O(log n) para el B-tree. En la práctica:
- Para tablas con menos de millones de filas, la diferencia es insignificante (ambos son muy rápidos).
- El índice hash usa significativamente menos espacio que el B-tree para claves largas (el hash es de tamaño fijo).
- El índice hash no puede ser usado para ordernamiento ni rangos, así que es menos flexible.

**Regla práctica:** El B-tree es la elección correcta en el 95% de los casos. El hash es una optimización de nicho para tablas muy grandes con búsquedas de igualdad exclusivamente en claves muy largas.

---

## 6. Índices bitmap

### 6.1 El concepto del índice bitmap

Un índice bitmap es diferente conceptualmente de los anteriores: para cada valor distinto de la columna indexada, almacena un bitmap (un vector de bits) donde el bit N está encendido si la fila N tiene ese valor.

```
Columna "departamento" con valores: RRHH, IT, Ventas, Operaciones

Bitmap para RRHH:     1 0 0 1 0 0 1 0 0 1 ...
Bitmap para IT:       0 1 0 0 1 0 0 1 0 0 ...
Bitmap para Ventas:   0 0 1 0 0 1 0 0 1 0 ...
Bitmap para Operac:   0 0 0 0 0 0 0 0 0 0 ...

WHERE departamento = 'IT' → bitmap IT
WHERE departamento IN ('RRHH', 'IT') → bitmap RRHH OR bitmap IT → 1 1 0 1 1 0 1 1 0 1
```

### 6.2 Bitmap en Oracle vs el Bitmap Index Scan de PostgreSQL

**Importante:** PostgreSQL **no tiene un tipo de índice bitmap permanente**. Lo que PostgreSQL llama "Bitmap Index Scan" es una **operación temporal en memoria** durante la ejecución de una query, no una estructura de índice almacenada. Es el operador que vimos en el Módulo 7: usa un B-tree para construir un bitmap en memoria y luego accede al heap de forma semi-secuencial.

Oracle y otros motores sí tienen índices bitmap persistentes en disco.

**¿Por qué PostgreSQL no tiene índices bitmap persistentes?**

Los índices bitmap persistentes son excelentes para columnas de baja cardinalidad (pocos valores distintos) en tablas de solo lectura o append-only. Pero en tablas OLTP con muchas escrituras concurrentes, actualizar el bitmap para una fila implica modificar el bit correspondiente en todos los bitmaps de todos los valores. Esto crea contención severa de locks entre transacciones concurrentes. PostgreSQL prioriza la concurrencia OLTP, donde los índices bitmap persistentes serían un cuello de botella.

---

## 7. Índices GIN: para búsquedas en arrays y texto completo

### 7.1 Qué problema resuelve GIN

GIN (Generalized Inverted Index) es el tipo de índice correcto cuando el valor de una columna **contiene múltiples elementos** y las queries buscan por los elementos individuales, no por el valor completo.

Casos de uso típicos:
- Arrays: `WHERE tags @> ARRAY['postgresql', 'performance']`
- JSONB: `WHERE atributos @> '{"color": "rojo"}'`
- Búsqueda de texto completo (full-text search): `WHERE to_tsvector(descripcion) @@ to_tsquery('motor')`
- Tipos de rango: consultas de solapamiento

### 7.2 Estructura interna de GIN

GIN construye un **índice invertido**: en lugar de mapear filas a valores, mapea valores (o elementos) a las filas que los contienen.

```
Tabla:
  fila 1: tags = {postgresql, performance, index}
  fila 2: tags = {postgresql, tutorial}
  fila 3: tags = {mysql, performance}

Índice GIN:
  "index"       → {fila 1}
  "mysql"       → {fila 3}
  "performance" → {fila 1, fila 3}
  "postgresql"  → {fila 1, fila 2}
  "tutorial"    → {fila 2}

WHERE tags @> ARRAY['postgresql', 'performance']:
  postgresql ∩ performance = {fila 1}
  → Solo leer fila 1 del heap
```

```sql
-- Índice GIN para búsquedas en arrays
CREATE INDEX idx_articulos_tags ON articulos USING GIN (tags);

-- Operadores soportados:
WHERE tags @> ARRAY['postgresql']     -- contiene todos estos
WHERE tags <@ ARRAY['a','b','c']      -- está contenido en estos
WHERE tags && ARRAY['postgresql']     -- tiene alguno de estos

-- Índice GIN para JSONB
CREATE INDEX idx_prod_attrs ON productos USING GIN (atributos);
WHERE atributos @> '{"voltaje": "110V"}'   -- contiene esta estructura

-- Índice GIN para texto completo
CREATE INDEX idx_articulos_fts ON articulos USING GIN (to_tsvector('spanish', contenido));
WHERE to_tsvector('spanish', contenido) @@ to_tsquery('spanish', 'motor & eficiente')
```

### 7.3 GIN vs GiST para texto completo

Para full-text search, tanto GIN como GiST son posibles. GIN es generalmente más rápido en las búsquedas pero más lento en las actualizaciones. GiST es más rápido en actualizaciones pero más lento en búsquedas. Para cargas de trabajo con más lecturas que escrituras (típico para búsqueda de texto), GIN es la elección correcta.

### 7.4 El parámetro `gin_pending_list_limit`

GIN tiene una optimización para inserciones: en lugar de actualizar el índice completamente en cada escritura, acumula los cambios en una "pending list" y los aplica en lote durante el VACUUM o cuando la lista supera `gin_pending_list_limit` (64MB por defecto). Esto mejora el rendimiento de escritura a costa de que las queries sobre datos recién insertados deben revisar también la pending list.

```sql
-- Crear índice GIN con fast update desactivado (para queries más predecibles)
CREATE INDEX idx_tags ON articulos USING GIN (tags) WITH (fastupdate = off);
```

---

## 8. Índices GiST: para geometría, rangos y tipos complejos

### 8.1 Qué es GiST

GiST (Generalized Search Tree) es un framework extensible para construir índices balanceados sobre tipos de datos complejos. No es un algoritmo de índice específico, sino una infraestructura que permite a los tipos de dato definir sus propios predicados de búsqueda.

Casos de uso principales:
- **Tipos geométricos:** puntos, líneas, polígonos, círculos. Búsquedas por proximidad, intersección, contención.
- **Tipos de rango:** `DATERANGE`, `TSTZRANGE`, `NUMRANGE`. Búsquedas de solapamiento, contención.
- **PostGIS:** extensión geoespacial que usa GiST extensamente.

### 8.2 Índices GiST para rangos

Como vimos en el Módulo 6, los tipos de rango de PostgreSQL se benefician enormemente de índices GiST:

```sql
-- Índice GiST para búsquedas temporales
CREATE INDEX idx_contratos_vigencia ON contratos USING GIST (vigencia);

-- Queries que usan el índice:
WHERE vigencia @> '2024-06-15'::date          -- contiene la fecha
WHERE vigencia && '[2024-01-01, 2024-12-31)'  -- se solapa con el rango

-- Índice GiST para EXCLUDE constraints (visto en Módulo 6)
EXCLUDE USING GIST (empleado_id WITH =, periodo WITH &&)
-- Esta restricción requiere un índice GiST internamente
```

### 8.3 GiST vs GIN: cuándo usar cada uno

| Característica | GiST | GIN |
|---|---|---|
| Tipos soportados | Tipos complejos, geometría, rangos | Arrays, JSONB, texto completo |
| Velocidad de búsqueda | Moderada | Muy alta |
| Velocidad de actualización | Alta | Moderada (fastupdate) |
| Pérdidas (lossiness) | Puede requerir recheck | Sin pérdidas |
| Uso principal | Geometría, rangos, PostGIS | Búsqueda en contenido, FTS |

GiST puede ser "lossy": el índice puede retornar falsos positivos que el motor luego filtra accediendo al heap (recheck). GIN no tiene falsos positivos.

---

## 9. Índices compuestos y el orden de columnas

### 9.1 Qué es un índice compuesto

Un **índice compuesto** (o índice multi-columna) indexa dos o más columnas juntas. Las entradas del índice son la concatenación de los valores de todas las columnas indexadas, ordenadas primero por la primera columna, luego por la segunda dentro de cada grupo de la primera, etc.

```sql
-- Índice compuesto sobre (departamento, salario)
CREATE INDEX idx_emp_dep_sal ON empleados (departamento, salario);

-- El índice ordena:
-- Primero por departamento (A-Z)
-- Dentro de cada departamento, por salario (ascendente)

-- Datos indexados (conceptualmente):
-- (Finanzas, 5000) → ctid de fila X
-- (Finanzas, 6500) → ctid de fila Y
-- (Finanzas, 8000) → ctid de fila Z
-- (RRHH,    4500) → ctid de fila A
-- (RRHH,    5500) → ctid de fila B
-- (Ventas,  4000) → ctid de fila C
-- ...
```

### 9.2 La regla del prefijo izquierdo

Un índice compuesto `(A, B, C)` puede ser usado por el motor solo si el predicado de la query incluye las columnas **en orden desde la izquierda**, formando un prefijo. Puede usarse para:
- Queries sobre `A`
- Queries sobre `A, B`
- Queries sobre `A, B, C`

No puede ser usado (de forma eficiente) para:
- Queries sobre solo `B`
- Queries sobre solo `C`
- Queries sobre `B, C`

```sql
CREATE INDEX idx_emp_dep_sal ON empleados (departamento, salario);

-- USA el índice (predicado sobre el prefijo izquierdo):
WHERE departamento = 'Ventas'
WHERE departamento = 'Ventas' AND salario > 5000
WHERE departamento = 'Ventas' AND salario BETWEEN 4000 AND 8000

-- NO usa el índice completo (predicado no incluye el prefijo):
WHERE salario > 5000  -- solo puede hacer un sequential scan del índice completo,
                      -- lo cual es similar a un sequential scan del heap
```

**La excepción del rango:**

Cuando una columna del prefijo tiene un predicado de rango (no de igualdad), el índice solo puede usarse eficientemente para esa columna y las anteriores. Las columnas posteriores no pueden ser filtradas por el índice.

```sql
-- Índice: (departamento, salario, fecha_ingreso)

-- Puede filtrar por índice las 3 columnas (departamento es igualdad):
WHERE departamento = 'Ventas' AND salario > 5000 AND fecha_ingreso > '2020-01-01'
-- ✓ El índice filtra por departamento (igualdad) y salario (rango)
--   fecha_ingreso solo puede ser filtrada con un recheck en el heap

-- Solo filtra por departamento (salario es rango → fecha_ingreso no puede usarse):
WHERE departamento = 'Ventas' AND salario > 5000
-- ✓ Usa el índice hasta el rango de salario. Correcto.
```

### 9.3 Diseño del orden de columnas en índices compuestos

La decisión del orden de columnas en un índice compuesto depende de cómo se usan en las queries:

**Regla 1: La columna de mayor selectividad primero (para igualdad).**
Si hay predicados de igualdad en ambas columnas, poner primero la columna con mayor cardinalidad (más valores distintos) reduce más el espacio de búsqueda.

```sql
-- Empleados: 50 departamentos, 10,000 empleados distintos por nombre
-- ¿Cuál va primero?

-- Para: WHERE departamento = 'Ventas' AND nombre = 'García'
-- Opción A: (departamento, nombre) → Reduce a ~200 empleados, luego busca García
-- Opción B: (nombre, departamento) → Reduce a ~1-2 García, luego verifica Ventas
-- Opción B es más eficiente para igualdad en ambas columnas (nombre más selectivo)
```

**Regla 2: La columna de igualdad antes que la columna de rango.**

```sql
-- Para: WHERE departamento = 'Ventas' AND salario > 5000
-- Correcto: (departamento, salario) → filtra departamento exacto, luego rango de salario
-- Incorrecto: (salario, departamento) → rango en primera columna → pierde el prefijo para departamento
```

**Regla 3: Considerar el patrón de las queries más frecuentes.**

Si la columna A aparece sola en el 80% de las queries y junto con B en el 20%, el índice `(A, B)` cubre ambos casos. Si B aparece sola frecuentemente, necesitarás un índice separado en B.

---

## 10. Índices parciales

### 10.1 Qué es un índice parcial

Un **índice parcial** indexa solo las filas que satisfacen un predicado `WHERE`. Solo el subconjunto de filas que pasa el predicado está en el índice.

```sql
-- Solo indexar los pedidos en estado 'pendiente'
CREATE INDEX idx_pedidos_pendientes ON pedidos (fecha_pedido)
WHERE estado = 'pendiente';
```

### 10.2 Ventajas de los índices parciales

**Menor tamaño:** Si el predicado filtra la mayoría de las filas, el índice parcial es mucho más pequeño que el índice completo. Un índice parcial en `estado = 'pendiente'` de una tabla donde el 95% de los pedidos están 'completados' será enorme si es un índice completo, pero pequeño si es parcial.

**Mejor rendimiento:** Un índice más pequeño tiene menos niveles en el árbol, cabe más fácilmente en el buffer cache, y los scans son más rápidos.

**Menor overhead de escritura:** Las inserciones y updates que no afectan las filas del predicado no tocan el índice.

### 10.3 Cuándo son más útiles

**Patrón 1: Valores de estado activos/inactivos**

```sql
-- Tabla usuarios: 10M de filas, solo 50K activos
CREATE INDEX idx_usuarios_activos ON usuarios (email)
WHERE activo = TRUE;

-- Esta query usa el índice parcial (y es muy rápida porque el índice es pequeño):
SELECT * FROM usuarios WHERE activo = TRUE AND email = 'usuario@ejemplo.com';
```

**Patrón 2: Columnas con muchos NULLs**

```sql
-- Si el 90% de las filas tienen codigo_promo = NULL,
-- un índice normal incluiría el 90% de filas con NULL (que raramente se buscan)
CREATE INDEX idx_pedidos_con_promo ON pedidos (codigo_promo)
WHERE codigo_promo IS NOT NULL;
```

**Patrón 3: Datos recientes o de alta prioridad**

```sql
-- Solo indexar facturas del año actual (las más consultadas)
CREATE INDEX idx_facturas_2024 ON facturas (cliente_id, fecha)
WHERE fecha >= '2024-01-01';
```

---

## 11. Índices de expresión

### 11.1 Qué son y para qué sirven

Un **índice de expresión** (o índice funcional) indexa el resultado de una expresión o función aplicada a una o más columnas, en lugar de los valores crudos de la columna.

Son necesarios cuando las queries frecuentemente aplican una función o transformación a una columna en sus predicados, lo que impide el uso de índices normales.

```sql
-- Problema: búsquedas case-insensitive no pueden usar índices normales
SELECT * FROM usuarios WHERE LOWER(email) = 'ana@empresa.com';
-- Un índice en (email) no ayuda porque el predicado es sobre LOWER(email)

-- Solución: índice en la expresión LOWER(email)
CREATE INDEX idx_usuarios_email_lower ON usuarios (LOWER(email));

-- Ahora esta query usa el índice:
SELECT * FROM usuarios WHERE LOWER(email) = 'ana@empresa.com';
```

### 11.2 Casos de uso comunes

**Búsquedas case-insensitive:**

```sql
CREATE INDEX idx_clientes_nombre_ci ON clientes (LOWER(nombre));
SELECT * FROM clientes WHERE LOWER(nombre) = 'ana garcía';
```

**Extracción de parte de una fecha:**

```sql
-- Búsquedas por año sin usar rangos
CREATE INDEX idx_pedidos_anio ON pedidos (EXTRACT(YEAR FROM fecha_pedido));
SELECT * FROM pedidos WHERE EXTRACT(YEAR FROM fecha_pedido) = 2024;
```

**Expresiones sobre JSONB:**

```sql
-- Índice sobre un campo específico dentro de un JSONB
CREATE INDEX idx_productos_voltaje ON productos ((atributos->>'voltaje'));
SELECT * FROM productos WHERE atributos->>'voltaje' = '110V';
```

**Valores calculados:**

```sql
-- Búsquedas sobre el precio con descuento
CREATE INDEX idx_precio_con_descuento ON productos ((precio * (1 - descuento)));
SELECT * FROM productos WHERE precio * (1 - descuento) < 100;
```

### 11.3 La condición de inmutabilidad

Las funciones usadas en índices de expresión deben ser **inmutables** (IMMUTABLE): deben retornar siempre el mismo resultado para los mismos argumentos, sin depender de nada externo (configuración de sesión, zona horaria, etc.).

```sql
-- LOWER() es IMMUTABLE: LOWER('ANA') siempre retorna 'ana'
-- NOW() es VOLATILE: retorna el tiempo actual → NO puede usarse en índices
-- CURRENT_DATE es STABLE: puede cambiar entre transacciones → NO puede usarse

-- Esto falla porque NOW() no es inmutable:
CREATE INDEX idx_mal ON pedidos (NOW() - fecha_pedido);  -- ERROR
```

---

## 12. Index-only scans y covering indexes

### 12.1 El problema del acceso al heap

Como vimos en el Módulo 7, un index scan normal requiere dos accesos: uno al índice (para encontrar el `ctid`) y uno al heap (para leer la fila completa). El acceso al heap es un acceso random y puede ser costoso si las filas están dispersas.

Si la query solo necesita columnas que ya están en el índice, el motor puede evitar el acceso al heap completamente: el **index-only scan**.

### 12.2 Cobertura de columnas con INCLUDE

PostgreSQL 11+ permite crear **covering indexes**: índices que incluyen columnas adicionales (que no son parte de la clave de búsqueda) para que estén disponibles en el índice sin ir al heap.

```sql
-- Query frecuente: buscar empleados por departamento y retornar nombre y salario
SELECT nombre, salario FROM empleados WHERE departamento = 'Ventas';

-- Índice normal: busca por departamento, luego va al heap para nombre y salario
CREATE INDEX idx_emp_dep ON empleados (departamento);

-- Covering index: nombre y salario están en el índice, no hay que ir al heap
CREATE INDEX idx_emp_dep_covering ON empleados (departamento)
    INCLUDE (nombre, salario);
-- INCLUDE: estas columnas se almacenan en las hojas del índice pero
-- no afectan el orden ni pueden ser usadas como criterio de búsqueda
```

```sql
-- Verificar si se usa index-only scan:
EXPLAIN SELECT nombre, salario FROM empleados WHERE departamento = 'Ventas';
-- Con el covering index:
-- Index Only Scan using idx_emp_dep_covering on empleados
--   Index Cond: (departamento = 'Ventas')
--   Heap Fetches: 0   ← sin accesos al heap
```

### 12.3 El rol del visibility map

Para que un index-only scan sea verdaderamente "solo del índice", el motor necesita saber que las filas del índice son visibles para la transacción actual sin ir al heap. Esta información está en el **visibility map**: un archivo por tabla donde un bit encendido para una página indica que todas las filas de esa página son visibles para todas las transacciones.

VACUUM actualiza el visibility map. En tablas con alta actividad de escritura, el visibility map puede estar frecuentemente desactualizado, forzando al motor a ir al heap para verificar visibilidad incluso en un "index-only scan".

```sql
-- Ver cuántas páginas están marcadas como all-visible
SELECT relname, all_visible
FROM pg_class
WHERE relkind = 'r'
ORDER BY all_visible ASC;

-- Ejecutar VACUUM para actualizar el visibility map
VACUUM empleados;
```

---

## 13. Cuándo un índice destruye el rendimiento de escritura

### 13.1 El costo de mantener un índice

Cada índice en una tabla tiene un costo en cada operación de escritura:

- **INSERT:** Para cada fila insertada, se inserta una entrada en cada índice. Una tabla con 10 índices requiere 11 inserciones (1 en el heap + 10 en los índices) por cada fila.
- **DELETE:** Similar al INSERT en inverso. Cada entrada del índice debe marcarse como eliminada (o eliminarse físicamente en el siguiente vacuum).
- **UPDATE:** Equivale a un DELETE + INSERT en todos los índices cuyas columnas cambiaron. Para columnas indexadas que cambian frecuentemente, cada UPDATE toca todos los índices relevantes.

### 13.2 Cuándo el overhead de índices es inaceptable

**Carga masiva de datos (bulk load):**

Si vas a insertar millones de filas en una tabla vacía, es mucho más eficiente:
1. Eliminar todos los índices (excepto la PK si es necesaria).
2. Hacer la carga masiva.
3. Recrear los índices.

Esto evita el overhead de mantener el índice durante la carga y permite que PostgreSQL construya el índice de forma eficiente en un solo barrido.

```sql
-- Carga masiva eficiente
DROP INDEX CONCURRENTLY idx_pedidos_cliente;
DROP INDEX CONCURRENTLY idx_pedidos_fecha;

-- Cargar datos:
COPY pedidos FROM '/ruta/archivo.csv' WITH (FORMAT CSV);
-- O muchos INSERT en lotes

-- Recrear índices después de la carga
CREATE INDEX CONCURRENTLY idx_pedidos_cliente ON pedidos (cliente_id);
CREATE INDEX CONCURRENTLY idx_pedidos_fecha ON pedidos (fecha_pedido);
```

**Tablas de log o append-only con alta tasa de inserción:**

Una tabla donde se insertan miles de filas por segundo y raramente se consulta por columnas específicas puede ser mejor sin índices, usando periodic batch queries para el análisis.

**Columnas que se actualizan frecuentemente:**

```sql
-- Si 'ultima_actividad' se actualiza en cada acción del usuario (cientos de veces por día):
CREATE INDEX idx_usuarios_ultima_actividad ON usuarios (ultima_actividad);
-- Cada UPDATE a ultima_actividad también actualiza el índice
-- Para millones de usuarios activos, esto genera millones de actualizaciones de índice por día
-- ¿Realmente necesitas consultar usuarios por ultima_actividad con un índice?
```

### 13.3 `CREATE INDEX CONCURRENTLY`: índices sin bloquear la tabla

Un `CREATE INDEX` normal bloquea la tabla para escrituras durante toda la construcción. Para tablas grandes en producción, esto puede tomar minutos o horas.

`CREATE INDEX CONCURRENTLY` construye el índice en múltiples fases sin bloquear las escrituras:
1. Registra el índice en los catálogos (lock breve).
2. Hace un primer scan del heap para construir el índice base.
3. Espera a que todas las transacciones que vieron el paso 1 terminen.
4. Hace un segundo scan para incorporar los cambios ocurridos durante el paso 2.
5. Declara el índice como valid.

```sql
-- Crear índice sin bloquear escrituras
CREATE INDEX CONCURRENTLY idx_pedidos_cliente ON pedidos (cliente_id);
-- Puede tardar más que CREATE INDEX normal, pero no bloquea la aplicación
```

**Caveats de `CONCURRENTLY`:**
- No puede hacerse dentro de una transacción explícita.
- Si falla a mitad, deja un índice "invalid" que debe eliminarse y recrearse.
- Tarda más que la versión no-concurrente.

---

## 14. Particionamiento de tablas

### 14.1 Por qué particionar

Las tablas muy grandes (decenas o cientos de millones de filas, terabytes de datos) generan problemas que el particionamiento resuelve:

- **Sequential scans** de toda la tabla son costosos; con particionamiento, las queries que filtran por la columna de partición solo leen las particiones relevantes.
- **VACUUM** de tablas enormes tarda mucho; con particionamiento, cada partición se vacuumea independientemente.
- **Mantenimiento:** archivar o eliminar datos históricos es un `DROP TABLE` de una partición, instantáneo, vs un `DELETE` masivo que genera bloat y tarda horas.

### 14.2 Particionamiento por rango

```sql
-- Tabla particionada por rango de fecha
CREATE TABLE ventas (
    venta_id     BIGSERIAL,
    fecha        DATE NOT NULL,
    cliente_id   INT NOT NULL,
    monto        DECIMAL(12,2) NOT NULL
) PARTITION BY RANGE (fecha);

-- Crear las particiones
CREATE TABLE ventas_2023 PARTITION OF ventas
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE ventas_2024 PARTITION OF ventas
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE ventas_2025 PARTITION OF ventas
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

-- Partition pruning automático:
EXPLAIN SELECT * FROM ventas WHERE fecha >= '2024-06-01' AND fecha < '2024-07-01';
-- El planner solo leerá ventas_2024, no ventas_2023 ni ventas_2025
```

### 14.3 Particionamiento por lista

```sql
-- Tabla particionada por lista de valores
CREATE TABLE pedidos (
    pedido_id  BIGSERIAL,
    region     VARCHAR(20) NOT NULL,
    monto      DECIMAL(12,2) NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE pedidos_norte PARTITION OF pedidos
    FOR VALUES IN ('Cochabamba', 'Oruro', 'La Paz');

CREATE TABLE pedidos_sur PARTITION OF pedidos
    FOR VALUES IN ('Potosí', 'Chuquisaca', 'Tarija');

CREATE TABLE pedidos_este PARTITION OF pedidos
    FOR VALUES IN ('Santa Cruz', 'Beni', 'Pando');
```

### 14.4 Particionamiento por hash

```sql
-- Distribución uniforme cuando no hay criterio de rango o lista natural
CREATE TABLE sesiones (
    sesion_id  UUID NOT NULL,
    usuario_id INT NOT NULL,
    datos      JSONB
) PARTITION BY HASH (usuario_id);

-- 4 particiones de igual tamaño
CREATE TABLE sesiones_0 PARTITION OF sesiones FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sesiones_1 PARTITION OF sesiones FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sesiones_2 PARTITION OF sesiones FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sesiones_3 PARTITION OF sesiones FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### 14.5 Índices en tablas particionadas

En PostgreSQL 11+, los índices creados en la tabla padre se propagan automáticamente a todas las particiones:

```sql
-- Índice en la tabla particionada: se crea en todas las particiones
CREATE INDEX idx_ventas_cliente ON ventas (cliente_id);
-- Equivale a crear idx en ventas_2023, ventas_2024, ventas_2025

-- También la PK y restricciones UNIQUE se propagan:
ALTER TABLE ventas ADD PRIMARY KEY (venta_id, fecha);
-- La clave debe incluir la columna de partición para tablas particionadas
```

### 14.6 Partition pruning: qué es y cuándo funciona

Partition pruning es la eliminación de particiones que no pueden contener filas que satisfagan el predicado de la query. Requiere que el predicado incluya la columna de partición con un valor o rango que el planner pueda evaluar en tiempo de planificación.

```sql
-- Partition pruning funciona (constante en tiempo de planificación):
SELECT * FROM ventas WHERE fecha = '2024-06-15';
-- El planner sabe que solo ventas_2024 puede tener esta fecha

-- Partition pruning NO funciona (el valor se conoce en tiempo de ejecución):
PREPARE buscar_ventas(DATE) AS
    SELECT * FROM ventas WHERE fecha = $1;
-- $1 es un parámetro desconocido en tiempo de planificación
-- El planner no puede hacer partition pruning estático
-- (aunque PostgreSQL hace "runtime pruning" que evalúa el parámetro al ejecutar)
```

---

## 15. Clustering físico y fill factor

### 15.1 CLUSTER: reordenar físicamente el heap

El comando `CLUSTER` reescribe la tabla ordenando las filas físicamente según un índice existente. Después del `CLUSTER`, las filas están en el heap en el mismo orden que en el índice, maximizando la correlación física (vimos este concepto en el Módulo 7).

```sql
-- Reordenar empleados por dep_id físicamente (según el índice)
CLUSTER empleados USING idx_empleados_dep;

-- Verificar la correlación después del CLUSTER
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'empleados' AND attname = 'dep_id';
-- correlation ≈ 1.0  ← ahora el index scan sobre dep_id será muy eficiente
```

**Caveats del CLUSTER:**
- Requiere un lock exclusivo sobre la tabla durante la reescritura (inaceptable en producción para tablas grandes).
- La correlación se degrada con el tiempo a medida que las inserciones nuevas se hacen en orden de inserción, no en orden del índice.
- No es una operación de mantenimiento regular; es una optimización puntual.

**Alternativa moderna:** En PostgreSQL 16+, `pg_repack` es una extensión que puede reordenar tablas online sin lock exclusivo, siendo la alternativa práctica para producción.

### 15.2 Fill factor: controlar el espacio libre en las páginas

El **fill factor** (factor de llenado) controla qué porcentaje de cada página de índice (o heap) se llena al crear el índice. El espacio restante se reserva para inserciones futuras que "aterricen" en esa página sin causar splits.

```sql
-- Crear índice con fill factor del 70%
-- Cada página del índice se llenará solo al 70%; 30% queda libre para futuras inserciones
CREATE INDEX idx_pedidos_fecha ON pedidos (fecha_pedido) WITH (fillfactor = 70);

-- Fill factor para el heap de una tabla
CREATE TABLE pedidos (...) WITH (fillfactor = 80);
-- O modificar una tabla existente:
ALTER TABLE pedidos SET (fillfactor = 80);
-- La tabla solo adoptará el nuevo fill factor al hacer VACUUM FULL o CLUSTER
```

**Cuándo ajustar el fill factor:**

- **Fill factor bajo (60-80%) para índices en columnas que se insertan en orden no secuencial:** Reduce la fragmentación y los page splits, a costa de un índice más grande.
- **Fill factor bajo para tablas con alta tasa de UPDATE en las mismas filas:** Más espacio libre en cada página facilita los HOT updates (la nueva versión de la fila puede quedar en la misma página que la anterior).
- **Fill factor 100% (default) para tablas append-only o de solo lectura:** No hay razón para dejar espacio libre si nunca habrá updates en esas páginas.

```sql
-- Para una tabla de log (solo inserciones, nunca updates)
CREATE TABLE log_eventos (
    evento_id  BIGSERIAL PRIMARY KEY,
    timestamp  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    tipo       VARCHAR(50),
    payload    JSONB
) WITH (fillfactor = 100);  -- Sin espacio desperdiciado
```

---

## 16. El proceso de diseño del modelo físico

### 16.1 El flujo de trabajo correcto

El modelo físico no debe diseñarse en el vacío. El flujo correcto es:

```
1. Diseñar el modelo lógico correcto (sin pensar en físico)
2. Identificar los patrones de acceso: ¿qué queries se ejecutarán?
   ¿Con qué frecuencia? ¿Qué proporciones lectura/escritura?
3. Estimar volúmenes: ¿cuántas filas? ¿Cuánto crecimiento?
4. Basado en 2 y 3, diseñar el modelo físico:
   - Tipos de datos apropiados
   - Índices para los patrones de acceso identificados
   - Particionamiento si el volumen lo justifica
   - Fill factor si los patrones de update lo justifican
5. Validar con EXPLAIN sobre queries representativas
6. Medir en producción y ajustar
```

### 16.2 La regla del índice mínimo necesario

No se trata de indexar todo lo que podría necesitarse. La regla es: **crear solo los índices que las queries actuales o planificadas necesitan, y eliminar los que no se usen**.

```sql
-- Ver índices que no se están usando (desde el último reinicio del servidor)
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexname NOT LIKE '%_pkey'  -- excluir claves primarias
ORDER BY schemaname, tablename;

-- Un índice con idx_scan = 0 que lleva semanas en producción
-- probablemente no sea necesario y solo genera overhead de escritura
```

### 16.3 Identificar candidatos a indexación

```sql
-- Queries lentas: la fuente primaria de candidatos a indexación
-- pg_stat_statements requiere configuración en postgresql.conf
SELECT query,
       calls,
       total_exec_time / calls AS avg_ms,
       rows / calls AS avg_rows
FROM pg_stat_statements
WHERE total_exec_time / calls > 100  -- queries que promedian más de 100ms
ORDER BY total_exec_time DESC
LIMIT 20;

-- Tablas con muchos sequential scans que podrían beneficiarse de índices
SELECT relname,
       seq_scan,
       seq_tup_read,
       idx_scan,
       idx_tup_fetch,
       seq_scan / NULLIF(idx_scan + seq_scan, 0)::float AS pct_seq
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC;
-- seq_tup_read muy alto con pct_seq alto → tabla con muchos scans completos
-- Candidata a índices si las queries son selectivas
```

### 16.4 El índice correcto para la query correcta

Dado un patrón de query, ¿qué tipo de índice diseñar?

**Query de igualdad sobre columna de alta cardinalidad:**
```sql
WHERE email = 'usuario@ejemplo.com'
→ CREATE INDEX ON tabla (email);  -- B-tree simple
```

**Query de rango sobre timestamp:**
```sql
WHERE fecha_pedido >= '2024-01-01' AND fecha_pedido < '2024-02-01'
→ CREATE INDEX ON pedidos (fecha_pedido);  -- B-tree simple
```

**Query de igualdad + rango (patrón más común):**
```sql
WHERE cliente_id = 42 AND fecha_pedido >= '2024-01-01'
→ CREATE INDEX ON pedidos (cliente_id, fecha_pedido);
-- cliente_id (igualdad) primero, fecha_pedido (rango) segundo
```

**Query con ORDER BY y LIMIT:**
```sql
SELECT * FROM pedidos WHERE cliente_id = 42 ORDER BY fecha_pedido DESC LIMIT 10
→ CREATE INDEX ON pedidos (cliente_id, fecha_pedido DESC);
-- Si el índice está en el orden correcto, el sort se evita completamente
```

**Query case-insensitive:**
```sql
WHERE LOWER(nombre) LIKE 'garcía%'
→ CREATE INDEX ON clientes (LOWER(nombre));  -- índice de expresión
```

**Búsqueda en contenido de JSONB o array:**
```sql
WHERE atributos @> '{"color": "rojo"}'
WHERE tags @> ARRAY['postgresql']
→ CREATE INDEX ON tabla USING GIN (columna);
```

**Query sobre columnas correlacionadas en EXCLUDE:**
```sql
EXCLUDE USING GIST (empleado_id WITH =, periodo WITH &&)
→ El EXCLUDE constraint crea automáticamente un índice GiST
```

---

## 17. Resumen y conexión con el resto del curso

### Los conceptos clave y cuándo aplicarlos

| Decisión | Regla de oro |
|---|---|
| **Tipos numéricos** | INT para la mayoría; BIGINT para tablas con >2B filas; NUMERIC para moneda |
| **Tipos de texto** | TEXT para propósito general; VARCHAR(n) solo si el límite tiene significado |
| **Timestamps** | TIMESTAMPTZ siempre, excepto "hora del reloj local" |
| **B-tree** | El índice por defecto; para igualdad, rango, orden, prefijo LIKE |
| **GIN** | Arrays, JSONB, full-text search |
| **GiST** | Geometría, tipos de rango, EXCLUDE constraints |
| **Índice compuesto** | Columnas de igualdad antes que columnas de rango; prefijo izquierdo |
| **Índice parcial** | Cuando solo se consulta un subconjunto de filas (activos, pendientes, recientes) |
| **Índice de expresión** | Cuando el predicado aplica una función a la columna |
| **Covering index** | Cuando la query solo necesita columnas que pueden estar en el índice |
| **Particionamiento** | Tablas con decenas de millones de filas y patrones de acceso por rango o lista |
| **Fill factor** | Tablas con alta tasa de UPDATE en las mismas filas |

### La meta-lección del módulo

El modelo físico es la capa de negociación entre el modelo lógico correcto y el hardware real. No se diseña de una vez: evoluciona a medida que los patrones de acceso se clarifican y el volumen de datos crece.

Los errores más comunes en el modelo físico son:
1. **Sobre-indexar:** Crear índices "por si acaso" que nunca se usan pero degradan el rendimiento de escritura.
2. **Sub-indexar:** No tener índices en columnas frecuentemente filtradas en tablas grandes.
3. **Índices compuestos con el orden incorrecto:** Columnas de rango antes que columnas de igualdad.
4. **Tipos de datos incorrectos:** `FLOAT` para moneda, `VARCHAR(n)` con límites arbitrarios, `TIMESTAMP` donde se necesita `TIMESTAMPTZ`.
5. **Ignorar los índices de expresión:** Queries con funciones en predicados que no pueden usar los índices existentes.

El proceso correcto es siempre el mismo: entender los patrones de acceso reales, diseñar índices para esos patrones, y medir con `EXPLAIN ANALYZE` y `pg_stat_statements` para validar y ajustar.

### Conexión con módulos futuros

- **Módulo 10 (OLTP vs OLAP):** Los sistemas OLAP tienen un modelo físico radicalmente distinto: almacenamiento columnar, compresión agresiva, índices bitmap persistentes. El contraste con el modelo físico OLTP que hemos descrito en este módulo explica por qué los dos tipos de sistema tienen arquitecturas tan diferentes.

- **Módulo 13 (Sistemas distribuidos):** En sistemas distribuidos, el particionamiento de este módulo tiene un equivalente más complejo: el sharding. Las decisiones de clave de partición tienen consecuencias mucho más graves cuando los datos están en múltiples nodos que cuando están en una sola máquina.

- **Módulo 15 (ORMs):** Los ORMs como SQLAlchemy o Hibernate generan queries automáticamente. Un ORM mal configurado puede generar queries que ignoran todos los índices cuidadosamente diseñados. Entender el modelo físico permite reconocer cuándo el ORM está generando queries subóptimas.

---

## 18. Ejercicios de comprensión

**Ejercicio 1.** Para el siguiente esquema de una plataforma de comercio electrónico, identifica todos los problemas en los tipos de datos y propón las correcciones:

```sql
CREATE TABLE productos (
    id           FLOAT,          -- identificador
    nombre       CHAR(200),      -- nombre del producto
    precio       DOUBLE PRECISION,  -- precio en dólares
    stock        SMALLINT,
    descripcion  VARCHAR(10000),
    creado_en    TIMESTAMP,
    activo       INTEGER         -- 0 o 1
);
```

---

**Ejercicio 2.** Dado el siguiente conjunto de queries frecuentes sobre una tabla `pedidos` con 50 millones de filas:

```sql
-- Query A (ejecutada 10,000 veces/día):
SELECT * FROM pedidos WHERE cliente_id = $1 ORDER BY fecha_pedido DESC LIMIT 20;

-- Query B (ejecutada 500 veces/día):
SELECT SUM(monto) FROM pedidos WHERE estado = 'completado' AND fecha_pedido >= $1;

-- Query C (ejecutada 100 veces/día):
SELECT * FROM pedidos WHERE numero_referencia = $1;

-- Query D (ejecutada 50 veces/día):
SELECT cliente_id, COUNT(*), SUM(monto)
FROM pedidos
WHERE fecha_pedido >= '2024-01-01'
GROUP BY cliente_id;
```

Para cada query:
a) ¿Qué tipo de índice diseñarías? Escribe el DDL completo.
b) Justifica el tipo de índice y el orden de columnas elegido.
c) ¿Hay alguna query para la que no tiene sentido crear un índice? ¿Por qué?

---

**Ejercicio 3.** Evalúa los siguientes índices e identifica cuáles son incorrectos, redundantes o ineficientes para los casos de uso descritos. Propón el diseño correcto para cada caso:

```sql
-- Caso 1: Buscar usuarios activos por email
CREATE INDEX idx_1 ON usuarios (activo, email);
-- Query típica: WHERE activo = TRUE AND email = $1

-- Caso 2: Buscar artículos por categoría y precio
CREATE INDEX idx_2 ON articulos (precio, categoria_id);
-- Query típica: WHERE categoria_id = $1 AND precio < $2

-- Caso 3: Buscar logs de error recientes
CREATE INDEX idx_3 ON logs (nivel, fecha_hora);
-- Query típica: WHERE nivel = 'ERROR' AND fecha_hora > NOW() - INTERVAL '1 hour'
-- El 99% de los logs son de nivel INFO o DEBUG

-- Caso 4: Búsqueda de productos por nombre (case-insensitive)
CREATE INDEX idx_4 ON productos (nombre);
-- Query típica: WHERE LOWER(nombre) LIKE $1 || '%'
```

---

**Ejercicio 4.** Una tabla `transacciones` con 200 millones de filas tiene este patrón de acceso:
- El 90% de las queries buscan transacciones de los últimos 30 días.
- Las transacciones históricas se consultan raramente pero nunca se eliminan.
- El volumen crece 500,000 filas por día.
- VACUUM tarda más de 2 horas en completarse.
- Las queries de los últimos 30 días se vuelven más lentas con el tiempo.

a) Diseña la estrategia de particionamiento apropiada, incluyendo el DDL completo.
b) ¿Cómo cambiaría el procedimiento de mantenimiento (VACUUM, archivado)?
c) ¿Qué índices crearías sobre la tabla particionada?

---

**Ejercicio 5.** El siguiente `EXPLAIN ANALYZE` muestra un problema relacionado con el modelo físico. Analiza el plan, identifica la causa y propón la solución:

```
Seq Scan on pedidos  (cost=0.00..845230.00 rows=1240 width=120)
                    (actual time=0.042..12450.230 rows=1240 loops=1)
  Filter: ((EXTRACT(year FROM fecha_pedido) = 2024) AND (cliente_id = 42))
  Rows Removed by Filter: 48997580
```

La tabla tiene estos índices:
```sql
CREATE INDEX idx_pedidos_fecha ON pedidos (fecha_pedido);
CREATE INDEX idx_pedidos_cliente ON pedidos (cliente_id);
```

a) ¿Por qué no se usan los índices existentes?
b) ¿Cuántas páginas aproximadamente está leyendo esta query?
c) Propón la solución mínima que resuelva el problema, incluyendo si hay que cambiar el índice, la query, o ambos.

---

**Ejercicio 6.** Diseña el modelo físico completo para el siguiente sistema: una plataforma de análisis de redes sociales que almacena posts de usuarios. Los requisitos son:
- 500 millones de posts, creciendo 5 millones por día.
- Cada post tiene: usuario_id, contenido (texto), lista de hashtags (array), fecha_publicacion, likes, retweets.
- Queries típicas: buscar posts por hashtag, por usuario, por rango de fecha, por contenido (full-text), y combinaciones.
- La lectura de posts recientes (últimos 7 días) representa el 80% del tráfico.
- Los likes y retweets se actualizan muy frecuentemente.

Define: tipos de datos, índices (tipo, columnas, parciales si aplica), estrategia de particionamiento, y fill factor para las columnas de alta actualización.

---

*Próximo módulo: OLTP vs OLAP — donde veremos por qué los sistemas transaccionales y los sistemas analíticos tienen requerimientos tan opuestos que justifican arquitecturas de datos completamente distintas, y cómo los datos viajan de uno al otro.*
