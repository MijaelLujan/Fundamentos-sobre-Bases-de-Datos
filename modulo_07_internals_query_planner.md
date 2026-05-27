# Módulo 7 — Internals: cómo ejecuta realmente una query

> *"Diseñar un modelo correcto es necesario pero no suficiente. Un modelo impecable puede comportarse de forma catastrófica en producción si no entiendes lo que ocurre desde el momento en que pulsas Enter hasta que aparece el primer resultado. El query planner es el puente entre tu intención expresada en SQL y la realidad física de los bytes en disco. Ignorarlo es diseñar con los ojos cerrados."*

---

## Tabla de contenidos

1. [El viaje de una query: de texto a resultado](#1-el-viaje-de-una-query-de-texto-a-resultado)
2. [Parsing y el árbol sintáctico abstracto](#2-parsing-y-el-árbol-sintáctico-abstracto)
3. [Reescritura y el árbol de álgebra relacional](#3-reescritura-y-el-árbol-de-álgebra-relacional)
4. [El query planner: generación de planes candidatos](#4-el-query-planner-generación-de-planes-candidatos)
5. [Estadísticas internas: la materia prima del planner](#5-estadísticas-internas-la-materia-prima-del-planner)
6. [Estimación de cardinalidad: el problema central](#6-estimación-de-cardinalidad-el-problema-central)
7. [El modelo de costos](#7-el-modelo-de-costos)
8. [Operadores físicos de acceso a datos](#8-operadores-físicos-de-acceso-a-datos)
9. [Operadores físicos de join](#9-operadores-físicos-de-join)
10. [Operadores físicos de agregación y ordenamiento](#10-operadores-físicos-de-agregación-y-ordenamiento)
11. [Cómo leer un plan de ejecución real](#11-cómo-leer-un-plan-de-ejecución-real)
12. [Cuándo y por qué el planner se equivoca](#12-cuándo-y-por-qué-el-planner-se-equivoca)
13. [Técnicas de diagnóstico y corrección](#13-técnicas-de-diagnóstico-y-corrección)
14. [Resumen y conexión con el resto del curso](#14-resumen-y-conexión-con-el-resto-del-curso)
15. [Ejercicios de comprensión](#15-ejercicios-de-comprensión)

---

## 1. El viaje de una query: de texto a resultado

### 1.1 Una visión general del pipeline

Cuando ejecutas una query SQL, en la base de datos ocurre un proceso de transformación en múltiples etapas. Cada etapa tiene un propósito específico y produce una representación distinta de la misma información:

```
SQL (texto)
    │
    ▼
┌─────────────────────────────────────────────┐
│  PARSER                                      │
│  Texto → Árbol Sintáctico Abstracto (AST)   │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  ANALIZADOR / REESCRITOR                    │
│  AST → Árbol de álgebra relacional          │
│  Resolución de nombres, tipos, vistas       │
│  Aplicación de reglas de reescritura        │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  PLANNER / OPTIMIZADOR                      │
│  Álgebra relacional → Plan de ejecución     │
│  Enumera alternativas, estima costos        │
│  Elige el plan de menor costo estimado      │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│  EXECUTOR                                   │
│  Plan de ejecución → Filas de resultado     │
│  Ejecuta los operadores físicos             │
│  Administra memoria y disco                 │
└─────────────────────────────────────────────┘
    │
    ▼
Resultado (filas devueltas al cliente)
```

Este módulo recorre cada etapa en profundidad, con énfasis en el planner y el executor, que son donde residen la mayoría de los problemas de rendimiento.

### 1.2 Por qué esto importa al diseñador

El planner toma decisiones automáticamente, pero esas decisiones dependen de información que tú controlas:

- **El esquema:** qué índices existen, qué restricciones están declaradas, qué tipos de datos se usan.
- **Las estadísticas:** qué tan precisas son y con qué frecuencia se actualizan.
- **La query misma:** cómo la escribes puede influir en qué planes el planner puede considerar.
- **La configuración:** parámetros del motor que ajustan el modelo de costos.

Un modelo bien diseñado puede rendir mal si las estadísticas están desactualizadas. Una query bien intencionada puede forzar al planner a elegir un plan terrible por cómo está escrita. Entender el pipeline permite diagnosticar y corregir ambos problemas.

---

## 2. Parsing y el árbol sintáctico abstracto

### 2.1 Qué hace el parser

El parser toma el texto SQL y verifica que sea sintácticamente válido según la gramática del lenguaje. Si la query tiene un error de sintaxis (una palabra clave mal escrita, un paréntesis sin cerrar, un operador inválido), el parser lo detecta y rechaza la query en esta etapa.

El resultado del parsing es un **Árbol Sintáctico Abstracto (AST)**: una representación en árbol de la estructura gramatical de la query.

Para la query:
```sql
SELECT e.nombre, d.nombre_dep
FROM empleados e
JOIN departamentos d ON e.dep_id = d.dep_id
WHERE e.salario > 7000;
```

El AST luce aproximadamente así:

```
SelectStmt
├── targetList
│   ├── ColumnRef (e.nombre)
│   └── ColumnRef (d.nombre_dep)
├── fromClause
│   └── JoinExpr (JOIN)
│       ├── RangeVar (empleados, alias=e)
│       ├── RangeVar (departamentos, alias=d)
│       └── quals: OpExpr (=)
│           ├── ColumnRef (e.dep_id)
│           └── ColumnRef (d.dep_id)
└── whereClause
    └── OpExpr (>)
        ├── ColumnRef (e.salario)
        └── Const (7000)
```

El AST es puramente sintáctico: no sabe si `empleados` existe, si `e.salario` es una columna válida, ni si el tipo de dato de `salario` es comparable con `7000`. Eso lo resuelve la siguiente etapa.

### 2.2 El analizador semántico

Inmediatamente después del parsing, el analizador semántico (a veces llamado "binder" o "analyzer") toma el AST y lo valida contra el catálogo del sistema:

- **Resolución de nombres:** ¿Existen las tablas `empleados` y `departamentos`? ¿Tiene permisos el usuario? ¿Existe la columna `salario` en `empleados`?
- **Resolución de tipos:** ¿El tipo de `salario` es compatible con el operador `>` aplicado a `7000`? ¿Necesita una conversión implícita?
- **Expansión de aliases y vistas:** Si `empleados` es en realidad una vista, se sustituye por su definición.
- **Resolución de `*`:** `SELECT *` se expande en la lista explícita de todas las columnas.

Si algo falla en esta etapa, el error que ves es "column X does not exist" o "operator does not exist: text > integer" — mensajes de error semánticos, no sintácticos.

---

## 3. Reescritura y el árbol de álgebra relacional

### 3.1 Del AST al árbol lógico

Una vez validado semánticamente, el AST se transforma en un **árbol de álgebra relacional**: una representación de la query en términos de las operaciones formales del modelo relacional (selección, proyección, join, etc.).

Este árbol lógico es equivalente al AST pero en la representación formal del modelo relacional, lo que permite aplicar transformaciones matemáticamente fundamentadas.

Para la query del ejemplo, el árbol lógico inicial podría ser:

```
π(nombre, nombre_dep)
    └── σ(salario > 7000)
            └── ⋈(e.dep_id = d.dep_id)
                    ├── empleados
                    └── departamentos
```

### 3.2 Transformaciones de reescritura

El reescritor aplica un conjunto de **reglas de reescritura** que preservan el resultado pero producen árboles lógicos más eficientes. Estas reglas están fundamentadas en las equivalencias del álgebra relacional que vimos en el Módulo 1.

**Predicado pushdown (la transformación más importante):**

Mover los predicados de filtro lo más abajo posible en el árbol, de modo que se apliquen antes de los joins y se reduzca el volumen de datos que cada operación superior debe procesar.

```
-- Árbol inicial (filtro después del join)
π(nombre, nombre_dep)
    └── σ(salario > 7000)
            └── ⋈(e.dep_id = d.dep_id)
                    ├── empleados
                    └── departamentos

-- Árbol después del predicado pushdown (filtro antes del join)
π(nombre, nombre_dep)
    └── ⋈(e.dep_id = d.dep_id)
            ├── σ(salario > 7000)
            │       └── empleados      ← solo los empleados con salario > 7000
            └── departamentos
```

En el árbol inicial, se hace el join completo de empleados y departamentos, y luego se filtra. En el árbol transformado, primero se filtran solo los empleados con salario alto, y ese subconjunto (mucho más pequeño) es el que participa en el join.

**Eliminación de subqueries (decorrelación):**

```sql
-- Subquery correlacionada (se ejecuta una vez por fila del exterior)
SELECT nombre FROM empleados e
WHERE salario > (
    SELECT AVG(salario) FROM empleados WHERE dep_id = e.dep_id
);

-- El reescritor puede transformarla en un join:
SELECT e.nombre
FROM empleados e
JOIN (
    SELECT dep_id, AVG(salario) AS avg_sal FROM empleados GROUP BY dep_id
) dep_avg ON dep_avg.dep_id = e.dep_id
WHERE e.salario > dep_avg.avg_sal;
```

La versión con subquery correlacionada ejecuta el `SELECT AVG(...)` una vez por cada empleado. La versión con join precalcula los promedios una sola vez. El reescritor hace esta transformación automáticamente en la mayoría de los motores modernos.

**Expansión de vistas:**

Si la query referencia una vista, su definición se sustituye en el árbol antes de la optimización. Esto permite que el planner optimice la query completa, incluyendo la lógica de la vista, en lugar de optimizarlas por separado.

---

## 4. El query planner: generación de planes candidatos

### 4.1 De árbol lógico a plan físico

El árbol lógico describe *qué* debe hacerse (operaciones relacionales). El planner convierte ese árbol en un **plan físico** que describe *cómo* hacerlo: qué algoritmo específico usar para cada operación, en qué orden hacerlas, y qué índices o estructuras de acceso aprovechar.

El planner es fundamentalmente un **optimizador de búsqueda**: genera un espacio de planes candidatos y elige el que estima que tiene el menor costo.

### 4.2 El espacio de planes es enorme

Para una query con `n` tablas en un join, el número de órdenes posibles de join es `n!` (factorial de n). Además, para cada orden de join, hay múltiples algoritmos de join posibles (nested loop, hash join, merge join) y múltiples formas de acceder a cada tabla (sequential scan, index scan, index-only scan, bitmap scan).

Esto hace que el espacio de búsqueda sea gigantesco:

- Para 2 tablas: hay 2 órdenes × 3 algoritmos de join × 2-3 métodos de acceso ≈ 12-18 planes.
- Para 5 tablas: hay 120 órdenes × múltiples combinaciones ≈ miles de planes posibles.
- Para 10 tablas: hay más de 3.6 millones de órdenes. Con algoritmos de join y acceso, el espacio es astronómico.

### 4.3 Estrategias de búsqueda: exhaustiva vs heurística

**Búsqueda exhaustiva (programación dinámica):**

PostgreSQL usa programación dinámica para joins de hasta un cierto número de tablas (controlado por el parámetro `join_collapse_limit`, por defecto 8). Divide el problema en subproblemas: calcula el costo óptimo de unir cada subconjunto de tablas, y construye la solución completa desde los subconjuntos más pequeños.

La programación dinámica garantiza encontrar el **plan óptimo** dentro del espacio que considera, pero su costo computacional crece rápidamente con `n`.

**Búsqueda genética (GEQO):**

Para queries con muchas tablas (por encima del `join_collapse_limit`), PostgreSQL cambia a un algoritmo genético (GEQO: Genetic Query Optimizer) que explora el espacio de forma heurística. No garantiza el plan óptimo, pero encuentra planes suficientemente buenos en tiempo razonable.

**Implicación práctica:** Las queries con 15-20 joins son un territorio peligroso donde el planner puede no encontrar el plan óptimo. Si tienes queries así, considera reescribirlas, usar CTEs, o usar hints (en motores que los soportan).

### 4.4 La función objetivo: el costo estimado

Para comparar planes, el planner necesita una función que asigne un número a cada plan. Esa función es el **modelo de costos**, que combina:

- El **costo de I/O:** cuántas páginas de disco hay que leer (o escribir).
- El **costo de CPU:** cuántas operaciones de CPU son necesarias (comparaciones, hashes, ordenamientos).
- El **costo de startup:** el costo antes de devolver la primera fila (relevante para LIMIT queries).

El modelo de costos es una **estimación**, no una medición. Usa estadísticas sobre los datos y parámetros de configuración que representan las características del hardware. Un planner puede elegir un plan subóptimo si sus estimaciones son incorrectas.

---

## 5. Estadísticas internas: la materia prima del planner

### 5.1 Qué son las estadísticas del planner

Las estadísticas son un conjunto de información sobre la distribución de valores en las tablas que el DBMS mantiene en su catálogo del sistema. En PostgreSQL, se almacenan principalmente en `pg_statistic` (datos crudos) y `pg_stats` (vista legible por humanos).

Sin estas estadísticas, el planner no tiene más información que el esquema mismo, y solo puede hacer estimaciones muy crudas. Con buenas estadísticas, el planner puede tomar decisiones mucho más informadas.

### 5.2 Las estadísticas más importantes

**1. Conteo de filas (`n_live_tup`):**

El número estimado de filas vivas en la tabla. Es la estadística más fundamental.

```sql
-- Ver el conteo de filas estimado para cada tabla
SELECT schemaname, tablename, n_live_tup
FROM pg_stat_user_tables
ORDER BY n_live_tup DESC;
```

**2. Porcentaje de valores NULL (`null_frac`):**

La fracción de filas donde la columna es NULL. Esto afecta cómo el planner estima la selectividad de predicados y la necesidad de manejar NULLs en joins.

```sql
SELECT attname, null_frac
FROM pg_stats
WHERE tablename = 'empleados';
```

**3. Número de valores distintos (`n_distinct`):**

Cuántos valores únicos tiene la columna. Si `n_distinct` es positivo, es el conteo absoluto. Si es negativo (p.ej., `-0.8`), es la fracción de filas que tienen valores distintos (0.8 = 80% de filas son únicas). Una columna con `-1.0` tiene todos los valores únicos.

```sql
SELECT attname, n_distinct
FROM pg_stats
WHERE tablename = 'empleados';
-- nombre: n_distinct = -0.95  → casi todos los nombres son únicos
-- departamento: n_distinct = 8 → hay exactamente 8 departamentos distintos
```

**4. El array de valores más comunes (`most_common_vals` / `most_common_freqs`):**

Una lista de los valores más frecuentes de la columna y su frecuencia respectiva. Esto permite al planner calcular la selectividad de predicados de igualdad con mucha precisión para los valores comunes.

```sql
SELECT attname, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'pedidos'
  AND attname = 'estado';
-- most_common_vals: {pendiente, completado, cancelado}
-- most_common_freqs: {0.45, 0.40, 0.15}
-- El planner sabe que WHERE estado = 'pendiente' retorna el 45% de las filas
```

**5. El histograma de valores (`histogram_bounds`):**

Para columnas con alta cardinalidad (muchos valores distintos), el planner almacena un histograma que divide el rango de valores en cubetas de igual frecuencia. Esto permite estimar la selectividad de predicados de rango (`BETWEEN`, `>`, `<`).

```sql
SELECT histogram_bounds
FROM pg_stats
WHERE tablename = 'pedidos'
  AND attname = 'monto';
-- histogram_bounds: {10.00, 45.50, 89.20, 156.00, 280.50, 520.00, 1200.00, ...}
-- Cada cubeta contiene aproximadamente 1/n de los valores
-- Para WHERE monto BETWEEN 89.20 AND 156.00, el planner estima ~1/n de las filas
```

**6. La correlación de almacenamiento (`correlation`):**

Un valor entre -1 y 1 que mide cuánto está correlacionado el orden físico de las filas con el orden de los valores de la columna. Un valor de 1 significa que las filas están ordenadas físicamente por ese valor (el primero en disco tiene el valor más pequeño, el último tiene el más grande). Un valor de 0 significa orden completamente aleatorio.

Esta estadística es crucial para decidir si usar un index scan es eficiente:

- **Alta correlación (≈ 1):** Un index scan seguirá leyendo páginas de disco en orden. Muy eficiente.
- **Baja correlación (≈ 0):** Un index scan tendrá que saltar entre páginas de disco aleatoriamente. Muy costoso para grandes resultados (puede ser peor que un sequential scan).

```sql
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'empleados';
-- empleado_id: correlation ≈ 1.0  (las filas se insertaron en orden)
-- nombre: correlation ≈ 0.05      (nombres sin orden físico)
-- salario: correlation ≈ -0.12    (sin correlación)
```

### 5.3 Las estadísticas extendidas: correlación entre columnas

El `ANALYZE` estándar calcula estadísticas columna a columna de forma independiente. Esto crea un problema: si dos columnas están correlacionadas entre sí, las estimaciones que asumen independencia son muy imprecisas.

**Ejemplo del problema:**

Supongamos una tabla de direcciones con columnas `pais`, `ciudad`, y `codigo_postal`. El planner podría estimar:

- `WHERE pais = 'Bolivia'`: retorna el 1% de las filas.
- `WHERE ciudad = 'Cochabamba'`: retorna el 0.2% de las filas.
- `WHERE pais = 'Bolivia' AND ciudad = 'Cochabamba'`: el planner asume independencia y estima 1% × 0.2% = 0.002% de las filas.

Pero estas columnas **no son independientes**: si la ciudad es 'Cochabamba', casi con certeza el país es 'Bolivia'. La estimación correcta es mucho más cercana al 0.2%. Esta brecha entre estimación y realidad puede llevar a un plan de ejecución completamente equivocado.

**La solución: estadísticas extendidas en PostgreSQL 10+:**

```sql
-- Declarar estadísticas extendidas sobre columnas correlacionadas
CREATE STATISTICS stats_pais_ciudad (dependencies, ndistinct, mcv)
ON pais, ciudad
FROM direcciones;

-- Recolectar las estadísticas extendidas
ANALYZE direcciones;

-- Verificar que se crearon
SELECT stxname, stxkeys, stxkind
FROM pg_statistic_ext
WHERE stxrelid = 'direcciones'::regclass;
```

Los tipos de estadísticas extendidas:
- **`dependencies`:** Detecta dependencias funcionales aproximadas entre columnas.
- **`ndistinct`:** Estima el número de valores distintos para combinaciones de columnas.
- **`mcv`:** Lista de valores más comunes para la combinación de columnas.

### 5.4 Cuándo se actualizan las estadísticas: ANALYZE

Las estadísticas **no se actualizan automáticamente** con cada INSERT, UPDATE o DELETE. Se actualizan cuando se ejecuta `ANALYZE`:

- **Manualmente:** `ANALYZE tabla;` o `ANALYZE;` (toda la base de datos).
- **Via autovacuum:** PostgreSQL tiene un daemon (`autovacuum`) que dispara `ANALYZE` automáticamente cuando el número de filas modificadas supera un umbral. Por defecto, el umbral es `50 + 0.2 × n_filas` (50 filas más el 20% del total de filas).

**El problema de las estadísticas desactualizadas:**

Si una tabla tiene 1,000 filas y el planner calculó sus estadísticas con esa información, y luego se insertan 5,000,000 de filas antes del próximo `ANALYZE`, el planner seguirá estimando basado en la tabla de 1,000 filas. Las consecuencias pueden ser muy graves:

- Estimará que un sequential scan es más económico que un index scan (porque cree que la tabla es pequeña).
- Estimará cardinalidades muy bajas para los resultados intermedios de los joins.
- Elegirá algoritmos de join inadecuados para el volumen real.

```sql
-- Ver cuándo fue el último ANALYZE para cada tabla
SELECT schemaname, tablename, last_analyze, last_autoanalyze, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
ORDER BY last_analyze ASC NULLS FIRST;

-- Forzar actualización de estadísticas
ANALYZE VERBOSE empleados;
```

### 5.5 El parámetro `default_statistics_target`

El `statistics_target` controla el nivel de detalle de las estadísticas recolectadas. Su valor por defecto es 100, que determina:

- El número de entradas en `most_common_vals`.
- El número de cubetas en el histograma.

Para columnas con distribuciones muy asimétricas o consultas frecuentes con predicados sobre esa columna, aumentar el `statistics_target` puede mejorar drásticamente las estimaciones:

```sql
-- Aumentar el nivel de estadísticas solo para la columna crítica
ALTER TABLE pedidos ALTER COLUMN estado SET STATISTICS 500;

-- Actualizar estadísticas con el nuevo nivel de detalle
ANALYZE pedidos;
```

Aumentar `statistics_target` tiene un costo: más tiempo para `ANALYZE` y más espacio en `pg_statistic`. No hay que aumentarlo indiscriminadamente, solo para columnas donde las estimaciones actuales son problemáticas.

---

## 6. Estimación de cardinalidad: el problema central

### 6.1 Qué es la cardinalidad en este contexto

La **cardinalidad** de un nodo en el plan de ejecución es el número de filas que ese nodo produce como salida. Estimar correctamente la cardinalidad de cada paso del plan es el problema más difícil que enfrenta el planner, y la fuente principal de malos planes de ejecución.

La cardinalidad importa porque:
- Determina qué algoritmo de join elegir (hash join necesita construir una tabla hash en memoria; si la cardinalidad es alta, puede no caber).
- Determina el orden de los joins (conviene hacer primero los joins que reducen más el conjunto de filas).
- Determina si un index scan es más económico que un sequential scan.

### 6.2 Cómo se estima la cardinalidad de un predicado simple

Para un predicado de igualdad `WHERE columna = valor`:

**Si `valor` está en `most_common_vals`:** La selectividad es exactamente la frecuencia almacenada.

```
-- most_common_vals: {pendiente, completado, cancelado}
-- most_common_freqs: {0.45, 0.40, 0.15}
-- n_filas = 100,000

-- WHERE estado = 'pendiente'
-- selectividad = 0.45
-- cardinalidad estimada = 100,000 × 0.45 = 45,000 filas
```

**Si `valor` no está en `most_common_vals`:** El planner usa la estimación de valores "no comunes":

```
selectividad = (1 - sum(most_common_freqs)) / (n_distinct - count(most_common_vals))
```

Es decir, divide el "espacio restante" entre los valores distintos que no son comunes.

**Para predicados de rango `WHERE columna BETWEEN a AND b`:** Usa el histograma para contar qué fracción del espacio de valores cae dentro del rango.

### 6.3 Propagación de cardinalidad a través del árbol

La cardinalidad de un nodo padre se estima a partir de las cardinalidades estimadas de sus hijos, multiplicadas por la selectividad del predicado del nodo.

**Problema de la multiplicación de estimaciones:**

Si el planner estima la cardinalidad de cada paso con un error del 10%, y hay 5 pasos en secuencia, el error se puede multiplicar:

- Paso 1: estimado 100, real 110 (error: 10%)
- Paso 2: estimado 50, real 66 (error acumulado: ~32%)
- Paso 3: estimado 25, real 45 (error acumulado: ~80%)
- Paso 4: estimado 12, real 38 (error acumulado: ~217%)
- Paso 5: estimado 6, real 30 (error acumulado: ~400%)

Los errores de cardinalidad se propagan y se amplifican a lo largo del árbol. Una query con muchos joins puede terminar con cardinalidades estimadas que son órdenes de magnitud diferentes de las reales. Es la principal causa de planes de ejecución subóptimos.

### 6.4 Cardinalidad estimada vs real en el plan de ejecución

`EXPLAIN ANALYZE` muestra ambas:

```sql
EXPLAIN ANALYZE
SELECT e.nombre, d.nombre_dep
FROM empleados e
JOIN departamentos d ON e.dep_id = d.dep_id
WHERE e.salario > 7000;

-- Salida (simplificada):
-- Hash Join  (cost=15.40..58.20 rows=42 width=40)
--            (actual time=0.815..2.341 rows=847 loops=1)
--   Hash Cond: (e.dep_id = d.dep_id)
--   ->  Seq Scan on empleados e  (cost=0.00..28.75 rows=42 width=24)
--                               (actual time=0.012..0.891 rows=847 loops=1)
--         Filter: (salario > 7000)
--         Rows Removed by Filter: 9153
```

En este ejemplo: el planner estimó `rows=42` pero la realidad fue `rows=847`. El planner subestimó por un factor de 20. Para este caso específico el plan sigue siendo correcto (hash join es eficiente), pero con errores más grandes o en nodos intermedios de joins complejos, este tipo de divergencia puede llevar al planner a elegir el algoritmo equivocado.

---

## 7. El modelo de costos

### 7.1 Las unidades de costo en PostgreSQL

PostgreSQL expresa los costos en unidades abstractas calibradas alrededor del costo de leer una página de 8KB secuencialmente desde disco, definido como `1.0`.

Los parámetros que configuran el modelo de costos:

| Parámetro | Valor por defecto | Significado |
|---|---|---|
| `seq_page_cost` | 1.0 | Costo de leer una página secuencialmente |
| `random_page_cost` | 4.0 | Costo de leer una página aleatoriamente (acceso random) |
| `cpu_tuple_cost` | 0.01 | Costo de CPU de procesar una fila |
| `cpu_index_tuple_cost` | 0.005 | Costo de CPU de procesar una entrada de índice |
| `cpu_operator_cost` | 0.0025 | Costo de CPU de evaluar un operador simple |
| `parallel_tuple_cost` | 0.1 | Costo de transferir una fila entre procesos paralelos |
| `effective_cache_size` | 4GB | Cuánto de los datos estimamos que está en caché (no es memoria asignada) |

El valor de `random_page_cost = 4.0` asume que el acceso aleatorio a disco es 4 veces más costoso que el secuencial. En un disco HDD, este ratio puede ser 10-50. En un SSD, puede ser 1.1-1.5. **Ajustar este parámetro según el hardware es una de las optimizaciones más impactantes:**

```sql
-- Para SSDs (acceso random mucho más barato)
SET random_page_cost = 1.1;

-- Para HDDs clásicos
SET random_page_cost = 4.0;  -- (el default, adecuado)

-- Para datos completamente en memoria (RAM muy grande, todo en caché)
SET random_page_cost = 1.0;
```

Si `random_page_cost` está mal calibrado para el hardware real, el planner sobre- o subestimará el costo de los index scans. Con `random_page_cost = 4.0` en un servidor con SSDs, el planner preferirá sequential scans cuando los index scans serían más rápidos en la realidad.

### 7.2 El costo de startup vs costo total

Cada nodo en el plan tiene dos costos:
- **Costo de startup:** El costo antes de poder devolver la primera fila.
- **Costo total:** El costo hasta devolver todas las filas.

Esta distinción importa para queries con `LIMIT`:

```sql
-- Sin LIMIT: el planner optimiza para costo total
EXPLAIN SELECT * FROM empleados ORDER BY salario DESC;

-- Con LIMIT: el planner optimiza para obtener las primeras N filas barato
EXPLAIN SELECT * FROM empleados ORDER BY salario DESC LIMIT 10;
```

Para un `LIMIT 10`, un sort completo (alto startup, puede ser innecesario) podría ser menos eficiente que un index scan en sentido inverso (bajo startup, devuelve las primeras 10 filas sin procesar el resto).

---

## 8. Operadores físicos de acceso a datos

Antes de ejecutar joins o agregaciones, el motor necesita obtener las filas de las tablas. Hay varios operadores de acceso a datos, cada uno con trade-offs distintos.

### 8.1 Sequential Scan (Seq Scan)

Lee la tabla entera, de principio a fin, página por página, en orden físico. Es el operador más simple y predecible.

```
Seq Scan on empleados  (cost=0.00..28.75 rows=1000 width=40)
```

**Cuándo el planner lo elige:**
- La selectividad del predicado es baja: la query retorna una fracción grande de la tabla.
- La tabla es pequeña (cabe en pocas páginas, la latencia de index lookup no compensa).
- No hay índice en las columnas del predicado.
- La correlación física del índice candidato es baja (leer el índice generaría muchos accesos random).

**Por qué a veces es mejor que un index scan:**

Para leer el 40% de una tabla grande, un sequential scan es más eficiente que un index scan porque:
1. Lee las páginas en orden (I/O secuencial, muy eficiente).
2. Lee cada página exactamente una vez.
3. No tiene overhead del recorrido del árbol B-tree del índice.

Un index scan para el 40% de la tabla requeriría saltar entre muchas páginas aleatoriamente, leyendo algunas varias veces.

**La heurística del 5-10%:** Muy aproximadamente, por debajo del 5-10% de las filas de una tabla grande, un index scan suele ser más eficiente que un sequential scan. Por encima, el sequential scan compite o gana. Pero esto depende enormemente del hardware, la distribución de los datos, y la correlación física.

### 8.2 Index Scan

Recorre el árbol B-tree del índice para encontrar las entradas que satisfacen el predicado, y luego para cada entrada accede a la fila real en el heap (la tabla).

```
Index Scan using idx_emp_salario on empleados
    (cost=0.29..8.31 rows=42 width=40)
  Index Cond: (salario > 7000)
```

**Cuándo el planner lo elige:**
- Alta selectividad: la query retorna pocas filas (bajo porcentaje de la tabla).
- Alta correlación física: las filas están ordenadas en disco de forma similar al índice (acceso secuencial al heap).
- El `random_page_cost` está bien calibrado para el hardware.

**El costo oculto del index scan:**

Cada entrada del índice requiere un acceso al heap para obtener la fila completa (si el índice no cubre todas las columnas necesarias). Si las filas están dispersas aleatoriamente en el heap, cada acceso al heap es un I/O random. Para un resultado de 10,000 filas con baja correlación, eso puede significar 10,000 accesos random, lo que es muy costoso.

### 8.3 Index-Only Scan

Si todas las columnas que la query necesita están en el índice, el motor puede evitar acceder al heap completamente. Solo lee el árbol del índice.

```
Index Only Scan using idx_emp_dep_sal on empleados
    (cost=0.29..4.21 rows=42 width=8)
  Index Cond: (dep_id = 5)
  Heap Fetches: 0  ← sin accesos al heap
```

**Cuándo es posible:**
- El índice incluye todas las columnas del `SELECT` y del `WHERE`.
- La tabla tiene un **visibility map** actualizado (PostgreSQL necesita verificar que las filas son visibles para la transacción actual; si el visibility map está actualizado, puede confiar en el índice sin ir al heap).

**Diseñar para index-only scans (covering indexes):**

```sql
-- Query frecuente:
SELECT nombre, salario FROM empleados WHERE dep_id = 5;

-- Índice que cubre la query (incluye todas las columnas necesarias)
CREATE INDEX idx_emp_dep_cobertura ON empleados (dep_id) INCLUDE (nombre, salario);
-- La cláusula INCLUDE agrega columnas al índice sin incluirlas en la clave
-- (no afectan el orden del índice ni permiten búsquedas por esas columnas)
```

### 8.4 Bitmap Index Scan + Bitmap Heap Scan

Este operador en dos fases es una solución elegante para el problema de los accesos random cuando la selectividad es media (ni muy alta ni muy baja para un index scan directo).

**Fase 1 — Bitmap Index Scan:** Recorre el índice y construye un bitmap en memoria. Cada bit corresponde a una página del heap y está encendido si el índice encontró al menos una fila en esa página que satisface el predicado.

**Fase 2 — Bitmap Heap Scan:** Recorre las páginas del heap, pero **solo las páginas marcadas en el bitmap**, y en **orden físico** (no en orden del índice). Esto convierte los accesos random del índice en accesos semi-secuenciales.

```
Bitmap Heap Scan on empleados  (cost=8.45..142.30 rows=350 width=40)
  Recheck Cond: (salario BETWEEN 5000 AND 8000)
  ->  Bitmap Index Scan on idx_emp_salario
        (cost=0.00..8.36 rows=350 width=0)
        Index Cond: (salario BETWEEN 5000 AND 8000)
```

**Ventaja adicional:** múltiples predicados sobre distintos índices pueden combinarse con operaciones lógicas (AND, OR) sobre los bitmaps antes de acceder al heap:

```sql
-- Predicados sobre dos índices distintos
EXPLAIN SELECT * FROM empleados
WHERE dep_id = 5 AND salario > 6000;

-- El planner puede hacer:
-- BitmapAnd
--   Bitmap Index Scan on idx_emp_dep (dep_id = 5)     → bitmap A
--   Bitmap Index Scan on idx_emp_sal (salario > 6000) → bitmap B
-- Bitmap Heap Scan: solo páginas en A AND B
```

---

## 9. Operadores físicos de join

El join es la operación más costosa y la que más depende de una buena estimación de cardinalidad. Hay tres algoritmos principales, cada uno óptimo en condiciones distintas.

### 9.1 Nested Loop Join

El algoritmo más simple conceptualmente: para cada fila de la tabla "exterior" (outer), busca las filas coincidentes en la tabla "interior" (inner).

```
-- Pseudocódigo
FOR EACH fila_outer IN tabla_outer:
    FOR EACH fila_inner IN tabla_inner:
        IF condicion_join(fila_outer, fila_inner):
            output(fila_outer, fila_inner)
```

Si la tabla inner tiene un índice en la columna de join, la búsqueda interior no es un scan completo sino un index scan:

```
-- Con índice en la tabla inner:
Nested Loop  (cost=0.29..42.50 rows=15 width=80)
  ->  Seq Scan on departamentos  (cost=0.00..1.05 rows=5 width=40)
  ->  Index Scan on empleados using idx_emp_dep
        (cost=0.29..8.24 rows=3 width=40)
        Index Cond: (dep_id = departamentos.dep_id)
```

**Costo:** O(N × M) en el peor caso (sin índice), O(N × log M) con índice en la tabla inner (siendo N = filas outer, M = filas inner).

**Cuándo el planner lo elige:**
- La tabla outer es pequeña (pocas filas que iterar).
- Hay un índice eficiente en la tabla inner para la condición de join.
- La query tiene `LIMIT`: el nested loop puede devolver las primeras filas rápidamente (bajo startup cost).
- El resultado del join es pequeño (pocos matches).

**Cuándo es catastrófico:**
- La tabla outer es grande (muchas iteraciones).
- No hay índice en la tabla inner (convierte en O(N × M)).
- Ambas tablas son grandes.

### 9.2 Hash Join

El hash join tiene dos fases:

**Fase de construcción (build phase):** Recorre la tabla más pequeña (la "inner" o "build side"), aplica una función de hash a la columna de join de cada fila, y almacena el resultado en una **tabla hash en memoria**.

**Fase de sondeo (probe phase):** Recorre la tabla más grande (la "outer" o "probe side"), aplica la misma función de hash a cada fila, y busca en la tabla hash los matches.

```
-- Pseudocódigo
hash_table = {}
FOR EACH fila_inner IN tabla_build:
    clave = hash(fila_inner.columna_join)
    hash_table[clave].append(fila_inner)

FOR EACH fila_outer IN tabla_probe:
    clave = hash(fila_outer.columna_join)
    FOR EACH fila_inner IN hash_table[clave]:
        IF fila_outer.columna_join == fila_inner.columna_join:
            output(fila_outer, fila_inner)
```

```
Hash Join  (cost=12.50..145.20 rows=850 width=80)
  Hash Cond: (e.dep_id = d.dep_id)
  ->  Seq Scan on empleados e  (cost=0.00..89.50 rows=5000 width=40)
  ->  Hash  (cost=8.00..8.00 rows=360 width=40)
        ->  Seq Scan on departamentos d  (cost=0.00..8.00 rows=360 width=40)
```

**Costo:** O(N + M) — lineal en el tamaño de ambas tablas. Es el algoritmo de join más eficiente para datasets grandes sin un orden previo.

**Restricciones:**
- Solo funciona para condiciones de **equi-join** (`=`). No funciona para `>`, `<`, `BETWEEN`, etc.
- Requiere que la tabla hash quepa en `work_mem`. Si no cabe, el motor hace un "batched hash join": divide los datos en lotes que caben en memoria y procesa cada lote por separado (con más I/O a disco).

**Cuándo el planner lo elige:**
- Ambas tablas son grandes.
- No hay índices útiles (o los índices no reducirían suficientemente el costo).
- La condición de join es de igualdad.
- La tabla build (la más pequeña) cabe en `work_mem`.

**Implicación de `work_mem`:**

```sql
-- Ver el work_mem actual
SHOW work_mem;  -- típicamente '4MB' por defecto

-- Aumentar para la sesión actual (no persistente)
SET work_mem = '256MB';
-- Ahora el planner puede considerar hash joins para tablas más grandes
-- sin spill a disco
```

⚠️ `work_mem` se asigna **por operador por proceso**. Si una query compleja tiene 3 hash joins en paralelo, y `work_mem = 256MB`, puede usar hasta 3 × 256MB = 768MB. En un servidor con 100 conexiones concurrentes y queries complejas, aumentar `work_mem` puede agotar la RAM. Hay que ajustarlo con cuidado.

### 9.3 Merge Join

El merge join aprovecha que ambas tablas estén ordenadas por la columna de join. Recorre ambas secuencias en paralelo, como el merge de merge-sort, emitiendo pares que coinciden:

```
-- Pseudocódigo (ambas tablas ya están ordenadas por columna_join)
i = 0; j = 0
WHILE i < len(tabla_A) AND j < len(tabla_B):
    IF tabla_A[i].clave == tabla_B[j].clave:
        output(tabla_A[i], tabla_B[j])
        j++  (y manejar duplicados...)
    ELIF tabla_A[i].clave < tabla_B[j].clave:
        i++
    ELSE:
        j++
```

```
Merge Join  (cost=120.30..185.60 rows=1200 width=80)
  Merge Cond: (e.dep_id = d.dep_id)
  ->  Sort  (cost=75.20..78.00 rows=1120 width=40)
            Sort Key: e.dep_id
        ->  Seq Scan on empleados e  ...
  ->  Sort  (cost=12.10..12.20 rows=40 width=40)
            Sort Key: d.dep_id
        ->  Seq Scan on departamentos d  ...
```

**Costo:** O(N log N + M log M) si hay que ordenar (incluye el costo del sort). O(N + M) si los datos ya están ordenados (p.ej., con índices en columnas de join).

**Restricciones:**
- Solo funciona para condiciones de igualdad (o condiciones donde se puede definir un orden).
- Requiere que los datos estén ordenados. Si no lo están, hay que pagar el costo del sort primero.

**Cuándo el planner lo elige:**
- Ambas tablas ya están ordenadas por la columna de join (p.ej., porque hay índices y el planner puede aprovecharlos).
- El resultado del join se necesita ordenado de todas formas (elimina la necesidad de un sort posterior).
- Las tablas son grandes y el hash table no cabría en `work_mem`.

### 9.4 Tabla comparativa de algoritmos de join

| Algoritmo | Condición | Memoria | Costo | Ideal para |
|---|---|---|---|---|
| **Nested Loop** | Cualquiera (=, <, >, BETWEEN) | Ninguna (streaming) | O(N × log M) con índice | Outer pequeño + índice en inner |
| **Hash Join** | Solo equi-join (`=`) | O(tamaño del build side) | O(N + M) | Tablas grandes, sin índice, sin orden |
| **Merge Join** | Solo equi-join (con orden) | O(N log N) para sort | O(N + M) (sin sort) | Datos ya ordenados o con índice |

---

## 10. Operadores físicos de agregación y ordenamiento

### 10.1 Hash Aggregate

Para `GROUP BY`, el operador más común es el hash aggregate: construye una tabla hash donde cada clave es la combinación de columnas del `GROUP BY`, y para cada clave acumula los valores de las funciones de agregación.

```
HashAggregate  (cost=42.50..47.50 rows=500 width=16)
  Group Key: dep_id
  ->  Seq Scan on empleados  (cost=0.00..28.75 rows=1750 width=8)
```

**Cuándo el planner lo elige:** Cuando el número de grupos es relativamente pequeño y la tabla hash de grupos cabe en `work_mem`.

**Cuándo tiene problemas:** Si el número de grupos es muy alto (p.ej., `GROUP BY usuario_id` en una tabla con 10 millones de usuarios distintos), la tabla hash no cabe en memoria y hay spill a disco.

### 10.2 Group Aggregate

Si los datos ya están ordenados por las columnas del `GROUP BY`, el planner puede usar un group aggregate que simplemente avanza linealmente por los datos ya ordenados, acumulando valores para cada grupo hasta que el grupo cambia.

```
GroupAggregate  (cost=0.29..82.50 rows=500 width=16)
  Group Key: dep_id
  ->  Index Scan using idx_emp_dep on empleados ...
```

Es muy eficiente cuando hay un índice en las columnas del `GROUP BY` y la cardinalidad de los grupos es alta (muchos grupos, pocos elementos por grupo).

### 10.3 Sort

El operador `Sort` materializa todas las filas de su hijo, las ordena, y las devuelve en orden. Tiene **alto startup cost** (no devuelve nada hasta haber procesado todas las filas) y puede spill a disco si los datos no caben en `work_mem`.

```
Sort  (cost=142.50..144.88 rows=950 width=40)
  Sort Key: salario DESC
  ->  Seq Scan on empleados  ...
```

**Alternativas al sort:**
- Un **index scan** en el orden correcto puede reemplazar un sort, evitando el costo completamente.
- Para `LIMIT n ORDER BY col`, un **index scan** devuelve las primeras n filas directamente sin necesidad de ordenar el conjunto completo.

### 10.4 Limit

El operador `Limit` interrumpe la ejecución de su hijo cuando ya tiene suficientes filas. Es el operador que hace que queries con `LIMIT` puedan ser muy eficientes, especialmente cuando se combinan con index scans o nested loops que tienen bajo startup cost.

```
Limit  (cost=0.29..8.54 rows=10 width=40)
  ->  Index Scan Backward using idx_emp_salario on empleados
        (cost=0.29..1678.29 rows=1750 width=40)
```

---

## 11. Cómo leer un plan de ejecución real

### 11.1 EXPLAIN vs EXPLAIN ANALYZE

```sql
-- EXPLAIN: muestra el plan estimado sin ejecutar la query
EXPLAIN SELECT e.nombre, d.nombre_dep
FROM empleados e
JOIN departamentos d ON e.dep_id = d.dep_id
WHERE e.salario > 7000;

-- EXPLAIN ANALYZE: ejecuta la query y muestra plan estimado + estadísticas reales
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT e.nombre, d.nombre_dep
FROM empleados e
JOIN departamentos d ON e.dep_id = d.dep_id
WHERE e.salario > 7000;
```

⚠️ `EXPLAIN ANALYZE` **ejecuta la query realmente**. Para queries `INSERT`, `UPDATE`, `DELETE`, envuelve en una transacción que haces rollback si no quieres los efectos:

```sql
BEGIN;
EXPLAIN ANALYZE UPDATE empleados SET salario = salario * 1.1;
ROLLBACK;
```

### 11.2 Anatomía de un plan completo

```
Hash Join  (cost=22.75..148.20 rows=847 width=40)
           (actual time=1.432..5.891 rows=847 loops=1)
  Hash Cond: (e.dep_id = d.dep_id)
  Buffers: shared hit=42 read=18
  ->  Seq Scan on empleados e  (cost=0.00..120.00 rows=847 width=24)
                               (actual time=0.015..2.451 rows=847 loops=1)
        Filter: (salario > 7000)
        Rows Removed by Filter: 9153
        Buffers: shared hit=35 read=18
  ->  Hash  (cost=8.00..8.00 rows=1180 width=20)
            (actual time=1.112..1.113 rows=1180 loops=1)
        Buckets: 2048  Batches: 1  Memory Usage: 73kB
        Buffers: shared hit=7
        ->  Seq Scan on departamentos d  (cost=0.00..8.00 rows=1180 width=20)
                                        (actual time=0.008..0.351 rows=1180 loops=1)
              Buffers: shared hit=7

Planning Time: 0.312 ms
Execution Time: 6.145 ms
```

**Desglose de cada campo:**

| Campo | Significado |
|---|---|
| `cost=X..Y` | X = startup cost, Y = total cost (unidades abstractas) |
| `rows=N` | Filas estimadas que producirá este nodo |
| `width=N` | Ancho estimado en bytes de cada fila de salida |
| `actual time=X..Y` | X = tiempo hasta primera fila (ms), Y = tiempo total (ms) |
| `actual rows=N` | Filas realmente producidas |
| `loops=N` | Cuántas veces se ejecutó este nodo (en nested loops, > 1) |
| `shared hit=N` | Páginas leídas del buffer cache (en memoria) |
| `shared read=N` | Páginas leídas desde disco |
| `Rows Removed by Filter` | Filas que pasaron el scan pero no cumplieron el predicado |
| `Batches: N` | Para hash joins: > 1 significa spill a disco |

### 11.3 El orden de lectura del plan: de adentro hacia afuera

Los planes se leen de adentro (hojas) hacia afuera (raíz). La indentación indica la jerarquía: los nodos más indentados son los que se ejecutan primero.

```
-- Orden de ejecución:
-- 1. Seq Scan on departamentos (nodo hoja más interno)
-- 2. Hash (construye la tabla hash con los resultados del paso 1)
-- 3. Seq Scan on empleados (otro nodo hoja)
-- 4. Hash Join (combina los resultados de 2 y 3)
```

### 11.4 Las señales de alarma en un plan

**Señal 1: Gran divergencia entre `rows` estimado y `actual rows`**

```
Seq Scan on pedidos  (cost=0.00..45.00 rows=12 width=40)
                    (actual time=0.015..1.230 rows=4521 loops=1)
-- El planner esperaba 12 filas, recibió 4521. Las estadísticas están muy desactualizadas.
```

**Señal 2: `Batches: N` (N > 1) en un Hash Join**

```
Hash  (cost=145.00..145.00 rows=11600 width=40)
      (actual time=85.230..85.230 rows=11600 loops=1)
  Buckets: 1024  Batches: 4  Memory Usage: 2048kB
-- El hash no cabía en work_mem y tuvo que spill a disco 4 veces. Aumentar work_mem.
```

**Señal 3: Nested Loop con muchos loops sobre tabla grande**

```
Nested Loop  (cost=0.29..89000.50 rows=42000 width=80)
  ->  Seq Scan on pedidos (rows=42000)
  ->  Index Scan on clientes using pk_clientes
        (actual loops=42000)
-- 42,000 index scans en clientes. Considera si un hash join sería más eficiente.
```

**Señal 4: Sort con spill a disco**

```
Sort  (cost=245.20..249.45 rows=1700 width=40)
      (actual time=380.200..400.100 rows=1700 loops=1)
  Sort Key: monto DESC
  Sort Method: external merge  Disk: 1842kB
-- "external merge" significa que el sort no cabía en work_mem y fue a disco.
```

**Señal 5: `Rows Removed by Filter` muy alto**

```
Seq Scan on facturas  (cost=0.00..580.00 rows=12 width=40)
                     (actual time=0.025..45.200 rows=12 loops=1)
  Filter: (cliente_id = 42 AND estado = 'pendiente')
  Rows Removed by Filter: 98523
-- Se leyeron 98,535 filas para encontrar 12. Un índice en (cliente_id, estado)
-- convertiría este scan en un index scan que lee solo las 12 filas directamente.
```

### 11.5 Usando `EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)`

Para análisis automatizado o con herramientas externas, el formato JSON es más fácil de procesar:

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT JSON)
SELECT ...;
```

Herramientas como [explain.dalibo.com](https://explain.dalibo.com) o [pgMustard](https://www.pgmustard.com) reciben el JSON y producen visualizaciones interactivas del plan con anotaciones sobre los problemas detectados.

---

## 12. Cuándo y por qué el planner se equivoca

### 12.1 Distribuciones no uniformes no capturadas

El planner asume, en ausencia de estadísticas específicas, que los valores se distribuyen uniformemente. Si la distribución real es muy asimétrica, las estimaciones pueden ser muy malas.

**Ejemplo:** Una tabla de pedidos donde el 90% de las filas tienen `estado = 'completado'` y el 10% están distribuidas entre otros estados. Si el planner no tiene estadísticas `most_common_vals` actualizadas, puede estimar que `WHERE estado = 'pendiente'` retorna 1/N de las filas (distribución uniforme), cuando en realidad retorna el 2%.

**Solución:** `ANALYZE` frecuente y un `statistics_target` mayor para columnas con distribuciones asimétricas.

### 12.2 Correlación entre columnas sin estadísticas extendidas

Como vimos en la sección 5.3, si dos columnas están correlacionadas, el planner multiplicará sus selectividades asumiendo independencia, produciendo estimaciones demasiado bajas.

**Solución:** `CREATE STATISTICS ... (dependencies, mcv)` para los pares o grupos de columnas correlacionados.

### 12.3 Funciones en predicados que impiden el uso de estadísticas

```sql
-- El planner NO puede usar las estadísticas de la columna para este predicado:
WHERE EXTRACT(YEAR FROM fecha_pedido) = 2024

-- El planner SÍ puede usar estadísticas:
WHERE fecha_pedido >= '2024-01-01' AND fecha_pedido < '2025-01-01'
```

Cuando se aplica una función a una columna en un predicado, el planner pierde la capacidad de usar el histograma de esa columna (a menos que haya un índice de expresión y estadísticas sobre él). La selectividad estimada cae a un valor por defecto (`default_selectivity`), que suele ser 0.005 para predicados de igualdad con funciones.

### 12.4 Parámetros en queries preparadas

Cuando se usa una prepared statement con parámetros (`$1`, `$2`), el planner en PostgreSQL tiene dos estrategias:

- **Planificación genérica:** Genera un plan que sea razonablemente bueno para cualquier valor de los parámetros. Se usa después de 5 ejecuciones.
- **Planificación específica:** Genera el plan óptimo para los valores concretos de los parámetros en cada ejecución.

El problema: un plan genérico puede ser muy subóptimo para distribuciones asimétricas. Si `WHERE estado = $1` se planifica genéricamente, el plan elegido para cuando `$1 = 'pendiente'` (2% de filas) será el mismo que cuando `$1 = 'completado'` (90% de filas).

```sql
-- Forzar re-planificación con los valores concretos (desactiva plan genérico)
SET plan_cache_mode = 'force_custom_plan';
```

### 12.5 El problema del join de muchas tablas

Como mencionamos en la sección 4.2, para queries con más de `join_collapse_limit` tablas (por defecto 8), el planner usa GEQO y puede no encontrar el plan óptimo. Además, con muchos joins, los errores de cardinalidad se amplifican.

**Señal:** Un query con 12-15 joins en producción que toma mucho más tiempo del esperado, con un plan que parece irracional.

**Opciones:**
- Dividir la query en CTEs materializadas (le das "puntos de control" al planner).
- Usar `SET join_collapse_limit = 1` y reordenar los joins manualmente (avanzado).
- Revisar si todos los joins son realmente necesarios.

---

## 13. Técnicas de diagnóstico y corrección

### 13.1 El proceso de diagnóstico

Un proceso sistemático para diagnosticar una query lenta:

```
1. Capturar el plan completo con EXPLAIN (ANALYZE, BUFFERS)
2. Identificar el nodo más costoso (el que consume más tiempo real)
3. Verificar: ¿Las filas estimadas se acercan a las reales?
   - Si hay gran divergencia: problema de estadísticas
   - Si no hay divergencia pero sigue siendo lento: el algoritmo elegido puede ser subóptimo
4. Para cada nodo de acceso (Seq Scan, Index Scan):
   - ¿Hay un índice que podría usarse pero no se está usando?
   - ¿Hay muchas filas removidas por filtro?
5. Para cada join:
   - ¿El algoritmo elegido es el correcto para el tamaño real de las tablas?
   - ¿El orden de los joins minimiza el tamaño de los resultados intermedios?
```

### 13.2 Forzar y probar planes alternativos

PostgreSQL permite deshabilitar selectivamente operadores para forzar al planner a explorar alternativas:

```sql
-- Deshabilitar sequential scans para forzar uso de índices
SET enable_seqscan = OFF;

-- Deshabilitar hash joins para forzar nested loop o merge join
SET enable_hashjoin = OFF;

-- Deshabilitar nested loops
SET enable_nestloop = OFF;

-- Deshabilitar merge joins
SET enable_mergejoin = OFF;

-- SIEMPRE restaurar después de las pruebas
RESET enable_seqscan;
RESET enable_hashjoin;
-- etc.
```

Estos `SET` son temporales para la sesión actual. Si al deshabilitar un operador el plan mejora drásticamente, eso es una señal de que el modelo de costos o las estadísticas están mal calibradas, no de que el operador sea genuinamente malo.

### 13.3 Hints: controlar el planner explícitamente

PostgreSQL estándar no tiene hints de join al estilo Oracle o SQL Server. Pero hay extensiones:

- **`pg_hint_plan`:** Extensión que permite añadir hints en comentarios de la query.

```sql
/*+ HashJoin(e d) IndexScan(e idx_emp_salario) */
SELECT e.nombre, d.nombre_dep
FROM empleados e
JOIN departamentos d ON e.dep_id = d.dep_id
WHERE e.salario > 7000;
```

Los hints son poderosos pero peligrosos: si el dato cambia (la tabla crece, la distribución cambia), el hint puede volverse contraproducente. Son un último recurso, no un sustituto de buenas estadísticas y un modelo correcto.

### 13.4 El rol de los CTEs en el control del planner

En PostgreSQL, los CTEs tienen dos comportamientos importantes:

**PostgreSQL 12+:** Por defecto, los CTEs se comportan como "fences" opcionales. El planner puede o no materializarlos.

**CTE materializado (`MATERIALIZED`):** Fuerza que el CTE se ejecute exactamente una vez y su resultado se almacene en memoria. Es un "punto de control" que le da al planner resultados concretos en lugar de álgebra relacional abstracta.

```sql
-- Sin materializar: el planner puede "inlinear" el CTE (tratarlo como una subquery)
WITH empleados_activos AS (
    SELECT * FROM empleados WHERE activo = TRUE
)
SELECT ea.nombre, d.nombre_dep
FROM empleados_activos ea
JOIN departamentos d ON d.dep_id = ea.dep_id;

-- Con materialización forzada: primero ejecuta el CTE, luego el join
WITH empleados_activos AS MATERIALIZED (
    SELECT * FROM empleados WHERE activo = TRUE
)
SELECT ea.nombre, d.nombre_dep
FROM empleados_activos ea
JOIN departamentos d ON d.dep_id = ea.dep_id;
```

La materialización forzada puede ser útil cuando:
- Sabes que la subquery devuelve muchas menos filas que la tabla completa y quieres que el planner lo sepa con certeza (no estimado).
- El CTE se referencia múltiples veces (sin materializar, puede ejecutarse múltiples veces).
- Estás depurando un plan complejo y quieres aislar el comportamiento de una parte.

### 13.5 El impacto del modelo de datos en el planner

Decisiones de modelado del Eje 2 tienen consecuencias directas en el planner:

**Tipos de datos correctos:** Una columna declarada como `INT` permite índices más eficientes que la misma columna como `VARCHAR`. Las comparaciones entre tipos compatibles evitan casts implícitos que rompen el uso del índice.

```sql
-- Problema: columna varchar comparada con entero
-- El planner no puede usar el índice en columna_varchar para esta comparación
WHERE columna_varchar = 12345  -- cast implícito de int a varchar

-- Correcto:
WHERE columna_varchar = '12345'  -- o mejor: usar el tipo correcto (INT)
```

**Restricciones declaradas:** Las restricciones `NOT NULL`, `CHECK`, y `UNIQUE` le dan información semántica al planner:

- `NOT NULL`: el planner sabe que no hay NULLs, simplifica la evaluación de predicados.
- `CHECK (valor IN ('A', 'B', 'C'))`: el planner puede usar esta restricción para excluir particiones o simplificar joins.
- `UNIQUE`: implica un índice; el planner puede usarlo y sabe que cada valor aparece a lo sumo una vez.

**Particionamiento:** En tablas particionadas, el planner puede aplicar **partition pruning**: si el predicado de la query filtra por la columna de partición, el planner elimina las particiones que no pueden contener filas que satisfagan el predicado, reduciendo el trabajo a solo las particiones relevantes.

```sql
-- Tabla particionada por rango de fecha
CREATE TABLE ventas (
    venta_id  BIGINT,
    fecha     DATE NOT NULL,
    monto     DECIMAL(12,2)
) PARTITION BY RANGE (fecha);

-- El planner hará partition pruning: solo leerá la partición de 2024
EXPLAIN SELECT * FROM ventas WHERE fecha >= '2024-01-01' AND fecha < '2025-01-01';
-- Plan incluirá solo la partición ventas_2024, no ventas_2022, ventas_2023, etc.
```

---

## 14. Resumen y conexión con el resto del curso

### Los conceptos clave de este módulo

| Concepto | Por qué importa |
|---|---|
| **Pipeline de ejecución** | Entender que SQL → AST → álgebra → plan físico → filas ayuda a saber en qué etapa diagnosticar un problema |
| **Predicado pushdown** | La reescritura más impactante; escribe queries que permitan pushdown |
| **Estadísticas** | La fuente de verdad del planner; sin estadísticas correctas, todo lo demás falla |
| **Cardinalidad estimada vs real** | La divergencia aquí es la raíz de la mayoría de los malos planes |
| **Sequential scan** | No siempre es el enemigo; para fracciones grandes de la tabla, es el operador correcto |
| **Hash join** | El algoritmo de join más eficiente para tablas grandes sin orden ni índice |
| **Nested loop + índice** | Óptimo cuando la tabla outer es pequeña y hay un índice bueno en la inner |
| **Merge join** | Óptimo cuando los datos ya están ordenados o el resultado necesita estarlo |
| **`EXPLAIN ANALYZE`** | La herramienta fundamental para diagnóstico; hay que saber leerla |
| **`work_mem`** | Controla si los hash joins y sorts se quedan en memoria o spill a disco |

### La meta-lección del módulo

El planner es un sistema de optimización probabilístico. Trabaja con estimaciones, no con certezas. Toma la mejor decisión que puede con la información que tiene. Tu trabajo como diseñador es asegurarte de que esa información sea correcta y completa:

- **Mantén las estadísticas actualizadas** (`ANALYZE` regular, o confiar en autovacuum bien configurado).
- **Declara todas las restricciones** que el modelo impone (NOT NULL, UNIQUE, CHECK, FK).
- **Usa tipos de datos correctos** que permitan comparaciones directas sin casts.
- **Diseña índices que coincidan** con los patrones de acceso reales.
- **Escribe queries que permitan** predicado pushdown y el uso de índices.
- **Calibra el modelo de costos** (`random_page_cost`, `work_mem`) para reflejar el hardware real.

Un modelo bien diseñado, con estadísticas correctas y un modelo de costos bien calibrado, produce planes excelentes automáticamente en la gran mayoría de los casos. Los problemas son la excepción, no la regla.

### Conexión con módulos futuros

- **Módulo 8 (Transacciones y MVCC):** El MVCC de PostgreSQL hace que el planner necesite acceder al heap para verificar la visibilidad de las filas (incluso en index-only scans sin visibility map actualizado). Entender MVCC explica por qué `VACUUM` y el visibility map afectan el rendimiento de los planes.

- **Módulo 9 (Modelado físico e índices):** Este módulo presenta la perspectiva del planner sobre los índices. El módulo 9 profundiza en los índices desde la perspectiva del diseñador: cómo construirlos, qué tipos existen, y cómo su estructura física afecta los operadores que hemos visto aquí.

- **Módulo 11 (Dimensional modeling):** Los data warehouses hacen star joins: una tabla de hechos grande unida con múltiples dimensiones. El planner enfrenta desafíos específicos aquí, y los motores OLAP tienen optimizaciones propias (bitmap index scans, columnar storage) que se explican en ese contexto.

- **Módulo 16 (El oficio):** La capacidad de leer un plan de ejecución y diagnosticar problemas de rendimiento es una de las habilidades prácticas más valiosas en el trabajo real con bases de datos.

---

## 15. Ejercicios de comprensión

**Ejercicio 1.** Para la siguiente query sobre una tabla `pedidos` con 5,000,000 de filas, un campo `estado` (valores: 'pendiente' 5%, 'en_proceso' 3%, 'completado' 90%, 'cancelado' 2%), y un campo `fecha_pedido`:

```sql
SELECT cliente_id, SUM(monto) AS total
FROM pedidos
WHERE estado = 'pendiente'
  AND fecha_pedido >= '2024-01-01'
GROUP BY cliente_id;
```

a) ¿Qué operadores de acceso esperarías ver en el plan para el scan de `pedidos`? Justifica considerando la selectividad del predicado.
b) ¿Qué operador de agregación esperarías para el `GROUP BY cliente_id`? ¿Qué factores determinan si se hace en memoria o con spill a disco?
c) ¿Qué índice diseñarías para optimizar esta query? Escribe el DDL.

---

**Ejercicio 2.** El siguiente plan de ejecución muestra una divergencia preocupante:

```
Hash Join  (cost=85.40..2420.80 rows=47 width=60)
           (actual time=12.340..8942.120 rows=87654 loops=1)
  Hash Cond: (o.cliente_id = c.cliente_id)
  ->  Seq Scan on ordenes o  (cost=0.00..1890.00 rows=47 width=40)
                             (actual time=0.012..3421.230 rows=87654 loops=1)
        Filter: (year = 2024)
        Rows Removed by Filter: 12346
  ->  Hash  (cost=28.00..28.00 rows=1800 width=24)
            (actual time=1.230..1.231 rows=1800 loops=1)
        Buckets: 2048  Batches: 1  Memory Usage: 102kB
        ->  Seq Scan on clientes c ...
```

a) Identifica todos los problemas en este plan.
b) ¿Por qué el planner estimó 47 filas cuando la realidad fue 87,654?
c) ¿Qué correcciones harías? Considera estadísticas, predicados, e índices.

---

**Ejercicio 3.** Tienes una query con 3 tablas y quieres entender qué plan eligió el planner:

```sql
SELECT p.nombre_producto, c.nombre_categoria, SUM(v.cantidad) AS total_vendido
FROM ventas v
JOIN productos p ON p.producto_id = v.producto_id
JOIN categorias c ON c.categoria_id = p.categoria_id
WHERE v.fecha >= '2024-01-01'
  AND c.nombre_categoria = 'Electrónica'
GROUP BY p.nombre_producto, c.nombre_categoria
ORDER BY total_vendido DESC
LIMIT 10;
```

a) Dibuja el árbol lógico **después del predicado pushdown**, indicando qué predicados se moverán hacia las hojas.
b) Considerando que `ventas` tiene 10M filas, `productos` tiene 50K filas, y `categorias` tiene 500 filas, ¿cuál sería el orden de join más eficiente? ¿Qué algoritmo de join esperarías en cada paso?
c) ¿Cómo impactaría el `LIMIT 10` en la elección del algoritmo de ordenamiento final?

---

**Ejercicio 4.** Explica por qué la siguiente query puede ser significativamente más lenta que su equivalente semántico, y cómo la reescribirías:

```sql
-- Versión lenta
SELECT * FROM empleados
WHERE UPPER(apellido) = 'García';

-- Pista: considera qué hace esta función al predicado respecto al índice
-- en la columna apellido.
```

---

**Ejercicio 5.** Un DBA ejecuta `EXPLAIN ANALYZE` sobre una query que une `facturas` (2M filas) con `clientes` (500K filas) y obtiene:

```
Nested Loop  (cost=0.42..8920450.00 rows=500000 width=80)
             (actual time=0.231..45823.120 rows=500000 loops=1)
  ->  Seq Scan on clientes c  (rows=500000)
  ->  Index Scan on facturas using idx_facturas_cliente
        (actual loops=500000)
        Index Cond: (cliente_id = c.cliente_id)
```

a) ¿Por qué el planner eligió nested loop para este join? (Analiza las cardinalidades).
b) ¿Por qué este plan es probablemente ineficiente aquí?
c) ¿Qué harías para que el planner considere un hash join? Escribe las sentencias SQL necesarias y explica el mecanismo.

---

**Ejercicio 6.** Diseña una estrategia de mantenimiento de estadísticas para un sistema OLTP con las siguientes características:
- Tabla `transacciones`: 50M filas, crece 500K filas por día.
- Tabla `productos`: 10K filas, cambia raramente (10-20 updates por día).
- Tabla `usuarios`: 2M filas, 1K inserts/updates por día.
- La columna `transacciones.estado` tiene distribución muy asimétrica (95% 'completada').
- Las columnas `transacciones.pais` y `transacciones.ciudad` están correlacionadas.

Para cada tabla, especifica: frecuencia de ANALYZE, statistics_target recomendado, y si necesita estadísticas extendidas. Justifica cada decisión.

---

*Próximo módulo: Transacciones, aislamiento y MVCC — donde descubriremos que la mayoría de los bugs de producción más difíciles de reproducir no son errores de lógica, sino consecuencias de cómo múltiples transacciones concurrentes interactúan con el modelo de aislamiento del motor.*
