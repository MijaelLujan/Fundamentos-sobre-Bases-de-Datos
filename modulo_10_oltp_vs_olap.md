# Módulo 10 — OLTP vs OLAP: dos filosofías de diseño

> *"OLTP y OLAP no son simplemente nombres para tipos de sistemas. Son filosofías de diseño opuestas que emergen de requerimientos fundamentalmente distintos. Intentar servir ambas cargas de trabajo con el mismo sistema, el mismo esquema, y el mismo motor es uno de los errores de arquitectura más costosos que existe. Entender por qué son incompatibles, y cómo los datos viajan de uno al otro, es la base de cualquier arquitectura de datos seria."*

---

## Tabla de contenidos

1. [Dos cargas de trabajo, dos mundos](#1-dos-cargas-de-trabajo-dos-mundos)
2. [OLTP: características y requerimientos](#2-oltp-características-y-requerimientos)
3. [OLAP: características y requerimientos](#3-olap-características-y-requerimientos)
4. [Por qué el mismo sistema no puede servir a ambos](#4-por-qué-el-mismo-sistema-no-puede-servir-a-ambos)
5. [El data warehouse como solución arquitectónica](#5-el-data-warehouse-como-solución-arquitectónica)
6. [ETL: de OLTP a OLAP](#6-etl-de-oltp-a-olap)
7. [ELT: la inversión del paradigma](#7-elt-la-inversión-del-paradigma)
8. [Almacenamiento columnar: el corazón del rendimiento OLAP](#8-almacenamiento-columnar-el-corazón-del-rendimiento-olap)
9. [HTAP: el intento de unificar ambos mundos](#9-htap-el-intento-de-unificar-ambos-mundos)
10. [Patrones de arquitectura modernos](#10-patrones-de-arquitectura-modernos)
11. [Resumen y conexión con el resto del curso](#11-resumen-y-conexión-con-el-resto-del-curso)
12. [Ejercicios de comprensión](#12-ejercicios-de-comprensión)

---

## 1. Dos cargas de trabajo, dos mundos

### 1.1 El punto de partida: qué queremos hacer con los datos

Toda base de datos existe para servir algún tipo de uso. Pero no todos los usos son iguales. Existe una divisoria fundamental entre dos tipos de trabajo:

**El primer tipo** es el trabajo transaccional: registrar que un cliente acaba de comprar, actualizar el stock de un producto, confirmar un pago, asignar un turno médico. Son operaciones pequeñas, precisas, que afectan pocas filas a la vez, pero que ocurren miles o millones de veces por día. El sistema existe para mantener el estado operativo del negocio.

**El segundo tipo** es el trabajo analítico: calcular las ventas totales del trimestre por región, identificar los productos más vendidos en los últimos seis meses, comparar el ticket promedio de distintos segmentos de clientes, detectar patrones de abandono. Son operaciones que escanean millones o miles de millones de filas, calculan agregaciones complejas, y producen respuestas que los tomadores de decisión usan para entender el pasado y planificar el futuro.

Estas dos clases de trabajo se llaman **OLTP** (Online Transaction Processing) y **OLAP** (Online Analytical Processing), y sus requerimientos son tan opuestos que optimizar un sistema para uno lo hace casi inevitablemente subóptimo para el otro.

### 1.2 Un poco de historia

El término OLTP se popularizó en los años 70 con el auge de las bases de datos relacionales. Los sistemas transaccionales eran la razón de ser de los RDBMS: reemplazar los archivos planos y los sistemas de tarjetas perforadas con un almacenamiento estructurado que garantizara integridad y concurrencia.

El término OLAP fue acuñado por E.F. Codd (el mismo Codd del modelo relacional) en 1993, en un artículo publicado en Computerworld donde describió 12 reglas para los sistemas analíticos, en paralelo a sus famosas 12 reglas para los RDBMS. Codd observó que las bases de datos relacionales, diseñadas para OLTP, no servían bien las necesidades analíticas.

En respuesta a ese artículo, surgieron los primeros data warehouses y motores OLAP dedicados: Oracle Express, Arbor Essbase, Red Brick. La arquitectura de data warehouse fue formalizada por Bill Inmon y Ralph Kimball en los años 90, sentando las bases de lo que hoy conocemos.

---

## 2. OLTP: características y requerimientos

### 2.1 El patrón de acceso OLTP

Un sistema OLTP típico tiene el siguiente perfil de carga:

- **Muchas transacciones pequeñas:** miles a millones de operaciones por hora, cada una afectando pocas filas (1-100).
- **Alta concurrencia:** cientos o miles de usuarios simultáneos ejecutando operaciones independientes.
- **Mezcla de lecturas y escrituras:** INSERT, UPDATE, DELETE y SELECT en proporciones variables, pero generalmente con una fracción significativa de escrituras.
- **Latencia baja:** cada operación debe completarse en milisegundos, no segundos.
- **Corrección ante concurrencia:** las transacciones deben ser ACID para garantizar consistencia.

**Ejemplos de operaciones OLTP:**

```sql
-- Registrar una venta (INSERT + UPDATE)
BEGIN;
INSERT INTO pedidos (cliente_id, fecha, total) VALUES (42, NOW(), 150.00);
UPDATE inventario SET stock = stock - 3 WHERE producto_id = 100;
COMMIT;

-- Consultar el estado de un pedido (SELECT de pocas filas)
SELECT p.*, e.nombre AS estado_desc
FROM pedidos p
JOIN estados e ON e.estado_id = p.estado_id
WHERE p.pedido_id = 12345;

-- Actualizar el saldo de una cuenta
UPDATE cuentas SET saldo = saldo - 500 WHERE cuenta_id = 'A';
```

Cada operación toca pocas filas, usa índices para encontrarlas rápidamente, y debe completarse en tiempo real.

### 2.2 El esquema OLTP: normalización para integridad

El esquema OLTP está altamente normalizado (típicamente 3FN o FNBC) por razones que ya entendemos del Módulo 2:

- **Eliminar redundancia** para prevenir anomalías de actualización.
- **Garantizar integridad** con foreign keys, checks y restricciones.
- **Reducir el tamaño de las filas afectadas** en cada operación de escritura.

Un esquema OLTP típico para un sistema de e-commerce:

```sql
CREATE TABLE clientes (
    cliente_id   SERIAL PRIMARY KEY,
    email        TEXT UNIQUE NOT NULL,
    nombre       TEXT NOT NULL,
    creado_en    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE productos (
    producto_id  SERIAL PRIMARY KEY,
    nombre       TEXT NOT NULL,
    precio       NUMERIC(10,2) NOT NULL,
    categoria_id INT REFERENCES categorias(categoria_id)
);

CREATE TABLE pedidos (
    pedido_id    SERIAL PRIMARY KEY,
    cliente_id   INT NOT NULL REFERENCES clientes(cliente_id),
    fecha        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    estado       VARCHAR(20) NOT NULL DEFAULT 'pendiente'
);

CREATE TABLE items_pedido (
    item_id      SERIAL PRIMARY KEY,
    pedido_id    INT NOT NULL REFERENCES pedidos(pedido_id),
    producto_id  INT NOT NULL REFERENCES productos(producto_id),
    cantidad     INT NOT NULL,
    precio_unit  NUMERIC(10,2) NOT NULL  -- precio al momento de la compra
);
```

Este esquema tiene tablas pequeñas, bien definidas, con relaciones explícitas. Es perfecto para las operaciones OLTP porque:
- Actualizar el precio de un producto toca solo una fila en `productos`.
- Insertar un nuevo pedido toca solo unas pocas filas distribuidas en varias tablas.
- Los índices en las claves primarias y foráneas garantizan accesos rápidos.

### 2.3 Los índices en OLTP

En un sistema OLTP, los índices son esenciales para las lecturas pero tienen un costo en escrituras. El diseño de índices busca el equilibrio mínimo necesario: indexar exactamente lo que se necesita para las queries más frecuentes, sin más.

Los índices típicos en OLTP son:
- **Claves primarias:** siempre indexadas.
- **Claves foráneas:** casi siempre indexadas (para joins y para que los DELETEs en cascada no sean full scans).
- **Columnas de búsqueda frecuente:** `email` de clientes, `numero_pedido`, `sku` de producto.
- **Columnas de filtro en queries de estado:** `WHERE estado = 'pendiente'` (índice parcial).

### 2.4 La prioridad del OLTP: correctitud y latencia

Las dos prioridades absolutas de un sistema OLTP son:

1. **Correctitud:** Las transacciones deben ser ACID. Un sistema bancario que pierde transacciones o genera inconsistencias es inútil, no importa cuán rápido sea.
2. **Latencia baja:** Cada operación individual debe ser rápida. Un usuario no puede esperar 5 segundos para que su compra se registre.

El throughput agregado (cuántas operaciones por segundo puede manejar el sistema) es importante, pero es secundario a la latencia de las operaciones individuales.

---

## 3. OLAP: características y requerimientos

### 3.1 El patrón de acceso OLAP

Un sistema OLAP tiene un perfil de carga radicalmente distinto:

- **Pocas queries muy grandes:** decenas a cientos de queries por hora, pero cada una puede escanear millones o miles de millones de filas.
- **Baja concurrencia:** pocos analistas simultáneos (decenas, no miles).
- **Dominado por lecturas:** las cargas analíticas son casi exclusivamente SELECT. Las escrituras son por lotes periódicos (ETL), no operaciones individuales.
- **Alta latencia aceptable:** una query analítica que tarda 30 segundos o 5 minutos es aceptable si retorna resultados útiles. El usuario está analizando datos, no esperando una respuesta en tiempo real.
- **Sin transacciones concurrentes complejas:** no hay el problema de dos analistas modificando el mismo dato simultáneamente.

**Ejemplos de operaciones OLAP:**

```sql
-- Ventas totales por región y categoría de producto en el Q4
SELECT r.nombre_region, c.nombre_categoria,
       SUM(f.cantidad * f.precio_unit) AS total_ventas,
       COUNT(DISTINCT f.pedido_id) AS num_pedidos,
       AVG(f.cantidad * f.precio_unit) AS ticket_promedio
FROM hechos_ventas f
JOIN dim_tiempo t ON t.tiempo_id = f.tiempo_id
JOIN dim_region r ON r.region_id = f.region_id
JOIN dim_producto p ON p.producto_id = f.producto_id
JOIN dim_categoria c ON c.categoria_id = p.categoria_id
WHERE t.anio = 2024 AND t.trimestre = 4
GROUP BY r.nombre_region, c.nombre_categoria
ORDER BY total_ventas DESC;
-- Esta query puede escanear 500M de filas

-- Cohorte de retención: clientes que compraron en enero y cuántos compraron en meses siguientes
SELECT primer_mes, mes_siguiente,
       COUNT(DISTINCT cliente_id) AS clientes_retenidos
FROM (
    SELECT c.cliente_id,
           DATE_TRUNC('month', primera_compra) AS primer_mes,
           DATE_TRUNC('month', f.fecha) AS mes_siguiente
    FROM clientes_cohorte c
    JOIN hechos_ventas f ON f.cliente_id = c.cliente_id
    WHERE f.fecha >= c.primera_compra
) sub
GROUP BY primer_mes, mes_siguiente
ORDER BY primer_mes, mes_siguiente;
```

### 3.2 El esquema OLAP: desnormalización para rendimiento

Para soportar las queries analíticas, los data warehouses usan esquemas desnormalizados. La razón es contraria a la lógica del OLTP:

En OLTP, los JOINs son baratos porque operan sobre pocas filas (los índices las encuentran rápidamente). En OLAP, los JOINs son costosos porque operan sobre millones de filas: incluso con hash joins eficientes, unir una tabla de hechos de 500M de filas con 5 dimensiones es un trabajo enorme.

La solución es **desnormalizar** las dimensiones: en lugar de hacer JOINs a tablas separadas de región, categoría, o vendedor, toda esa información se almacena directamente en la tabla de hechos o en tablas dimensionales amplias y planas.

```sql
-- El mismo e-commerce, schema OLAP desnormalizado (esquema estrella):

CREATE TABLE hechos_ventas (
    venta_id         BIGINT,
    tiempo_id        INT NOT NULL,    -- FK a dim_tiempo
    producto_id      INT NOT NULL,    -- FK a dim_producto
    cliente_id       INT NOT NULL,    -- FK a dim_cliente
    vendedor_id      INT NOT NULL,    -- FK a dim_vendedor
    cantidad         INT NOT NULL,
    precio_unit      NUMERIC(10,2) NOT NULL,
    descuento        NUMERIC(5,2) NOT NULL DEFAULT 0,
    monto_neto       NUMERIC(12,2) NOT NULL,  -- precio_unit * cantidad * (1-descuento)
    -- Métricas pre-calculadas para eficiencia
    PRIMARY KEY (venta_id)
);

CREATE TABLE dim_tiempo (
    tiempo_id     INT PRIMARY KEY,
    fecha         DATE NOT NULL,
    anio          SMALLINT NOT NULL,
    trimestre     SMALLINT NOT NULL,
    mes           SMALLINT NOT NULL,
    nombre_mes    VARCHAR(20) NOT NULL,
    semana        SMALLINT NOT NULL,
    dia_semana    SMALLINT NOT NULL,
    nombre_dia    VARCHAR(20) NOT NULL,
    es_fin_semana BOOLEAN NOT NULL,
    es_feriado    BOOLEAN NOT NULL
);

CREATE TABLE dim_producto (
    producto_id       INT PRIMARY KEY,
    nombre_producto   TEXT NOT NULL,
    sku               VARCHAR(50) NOT NULL,
    -- Atributos de la jerarquía desnormalizados en la misma tabla:
    subcategoria_id   INT NOT NULL,
    nombre_subcateg   TEXT NOT NULL,  -- desnormalizado de subcategorias
    categoria_id      INT NOT NULL,
    nombre_categoria  TEXT NOT NULL,  -- desnormalizado de categorias
    linea_producto    TEXT NOT NULL,  -- desnormalizado de lineas
    marca             TEXT NOT NULL
);

CREATE TABLE dim_cliente (
    cliente_id        INT PRIMARY KEY,
    nombre_cliente    TEXT NOT NULL,
    email             TEXT NOT NULL,
    ciudad            TEXT NOT NULL,
    region            TEXT NOT NULL,
    pais              TEXT NOT NULL,
    segmento          TEXT NOT NULL,  -- 'premium', 'standard', 'nuevo'
    fecha_primer_comp DATE NOT NULL,
    -- SCD Tipo 2: versiones históricas del cliente
    fecha_inicio      DATE NOT NULL,
    fecha_fin         DATE NOT NULL DEFAULT '9999-12-31',
    es_actual         BOOLEAN NOT NULL DEFAULT TRUE
);
```

La diferencia con el esquema OLTP es dramática:

- `dim_producto` tiene la categoría, subcategoría y línea desnormalizadas directamente, eliminando JOINs a tablas de categorías.
- `dim_tiempo` tiene todos los atributos de la fecha (año, trimestre, mes, semana, día de semana, es_feriado) pre-calculados, eliminando cálculos en cada query.
- `dim_cliente` incorpora SCD Tipo 2 para mantener historial.
- La tabla de hechos tiene métricas pre-calculadas (`monto_neto`) para evitar recalcularlas en cada query analítica.

### 3.3 La prioridad del OLAP: throughput y completitud

Las prioridades del OLAP son opuestas a las del OLTP:

1. **Throughput de I/O:** La capacidad de escanear y procesar millones de filas por segundo es lo que determina si las queries analíticas son viables o no.
2. **Completitud:** Una query que agrega datos de todos los clientes debe procesar realmente todos los clientes, sin atajos.

La latencia de una query individual es secundaria: si una query de cohorte tarda 2 minutos, está bien si retorna el resultado correcto. Lo que no está bien es que bloquee o interfiera con las operaciones transaccionales del negocio.

---

## 4. Por qué el mismo sistema no puede servir a ambos

### 4.1 Los conflictos fundamentales

No es solo que los dos tipos de carga tengan requerimientos distintos: es que satisfacer uno óptimamente hace el otro significativamente peor. Los conflictos son múltiples y profundos:

**Conflicto 1: Normalización vs desnormalización**

El esquema OLTP altamente normalizado es perfecto para escrituras transaccionales pero terrible para queries analíticas: un informe de ventas por región requeriría JOINs a través de 6-8 tablas sobre millones de filas.

El esquema OLAP desnormalizado es perfecto para queries analíticas pero terrible para escrituras: actualizar el nombre de una categoría requeriría actualizar millones de filas en `dim_producto`.

**Conflicto 2: Índices B-tree vs almacenamiento columnar**

Los índices B-tree de los sistemas OLTP son perfectos para encontrar filas individuales rápidamente. Pero para escanear el 100% de una columna sobre 500M de filas (como hace una query analítica típica), los índices B-tree son ineficientes: hay que leer el índice completo, y luego hacer un acceso al heap por cada fila.

El almacenamiento columnar (donde todas las filas de una columna están almacenadas juntas, en lugar de todas las columnas de una fila juntas) es ideal para escanear columnas completas con alta compresión, pero terrible para las operaciones por fila del OLTP.

**Conflicto 3: MVCC y transacciones largas**

Las queries analíticas largas (segundos o minutos) en un sistema OLTP con MVCC mantienen una snapshot antigua durante toda su duración. Como vimos en el Módulo 8, esto impide que VACUUM limpie las versiones antiguas de filas, generando bloat progresivo que degrada el rendimiento de todas las operaciones, incluyendo las transaccionales.

**Conflicto 4: Concurrencia**

Un sistema OLTP está optimizado para muchas transacciones cortas y concurrentes. Una sola query analítica que escanea toda la tabla de pedidos mientras 1,000 usuarios están comprando compite por el mismo I/O, la misma CPU, y la misma memoria buffer.

**Conflicto 5: El ciclo de escritura**

Los datos del OLTP se escriben continuamente, transacción por transacción. Las cargas OLAP típicamente se hacen en lotes (ETL nocturno o por ventanas de tiempo). Si se intenta ejecutar analítica sobre el sistema OLTP, se ve un estado de los datos que está siendo modificado continuamente, lo que complica el análisis y puede producir resultados inconsistentes.

### 4.2 El impacto en producción: un caso real

Considera una empresa de e-commerce con 50 millones de pedidos históricos. El equipo de analítica necesita calcular la retención de clientes y decide ejecutar la query directamente sobre la base de datos de producción:

```sql
-- Esta query se ejecuta sobre el sistema OLTP de producción
SELECT DATE_TRUNC('month', primera_compra) AS cohorte,
       DATE_TRUNC('month', p.fecha) AS mes,
       COUNT(DISTINCT p.cliente_id) AS clientes
FROM pedidos p
JOIN (
    SELECT cliente_id, MIN(fecha) AS primera_compra
    FROM pedidos GROUP BY cliente_id
) primera ON primera.cliente_id = p.cliente_id
GROUP BY 1, 2
ORDER BY 1, 2;
```

Esta query escanea la tabla `pedidos` dos veces (una para la subquery, otra para el join exterior). Con 50 millones de filas, esto puede tardar 15-30 minutos en un sistema OLTP no optimizado para ello. Durante esos minutos:

- El VACUUM no puede limpiar filas muertas que la query está "mirando".
- El buffer cache se llena con las páginas de la tabla `pedidos`, expulsando los datos de las tablas que las operaciones transaccionales necesitan (pedidos recientes, clientes activos).
- La latencia de las operaciones de la aplicación sube porque el buffer cache tiene menos datos útiles.
- El CPU y el I/O están saturados.

El resultado: la aplicación se vuelve lenta para todos los usuarios mientras el analista espera su reporte. Este escenario se repite en miles de empresas que no han separado sus cargas de trabajo.

### 4.3 La solución: separación de sistemas

La respuesta a estos conflictos irresolubles es **separar los sistemas**:

```
┌─────────────────────────┐         ┌─────────────────────────┐
│   Sistema OLTP          │         │   Sistema OLAP          │
│   (Base de datos        │  ETL/   │   (Data Warehouse /     │
│    operacional)         │──ELT──► │    Data Lake)           │
│                         │         │                         │
│ - Esquema normalizado   │         │ - Esquema desnorm.      │
│ - Índices B-tree        │         │ - Almacen. columnar     │
│ - Optimizado para       │         │ - Optimizado para       │
│   escrituras y lecturas │         │   lecturas masivas      │
│   de pocas filas        │         │   y agregaciones        │
│ - ACID, baja latencia   │         │ - Alta latencia OK      │
└─────────────────────────┘         └─────────────────────────┘
```

Los datos fluyen del sistema OLTP al OLAP de forma periódica (cada hora, cada día, en tiempo casi-real). El sistema OLTP sigue siendo la fuente de verdad para los datos operacionales. El sistema OLAP tiene su propia copia de los datos, optimizada para el análisis.

---

## 5. El data warehouse como solución arquitectónica

### 5.1 Qué es un data warehouse

Un **data warehouse** (DW) es un sistema de almacenamiento de datos diseñado específicamente para análisis e informes. Sus características esenciales son:

- **Orientado a temas:** Los datos están organizados alrededor de temas del negocio (ventas, clientes, productos) en lugar de alrededor de los procesos transaccionales que los generaron.
- **Integrado:** Consolida datos de múltiples sistemas fuente (el sistema de ventas, el ERP, el CRM, el sistema de logística) en un esquema unificado y coherente.
- **No volátil:** Los datos del DW no se modifican ni eliminan. Se agregan nuevos datos, pero los históricos permanecen inmutables. Esto garantiza reproducibilidad: el mismo reporte ejecutado hoy y en seis meses debe retornar los mismos datos históricos.
- **Variable en el tiempo:** El DW mantiene el historial completo de los datos. No solo "cómo están las cosas ahora" sino "cómo estaban en cualquier momento del pasado".

Esta definición es de Bill Inmon, quien acuñó el término en su libro "Building the Data Warehouse" (1992).

### 5.2 La arquitectura clásica de tres capas

La arquitectura clásica de un data warehouse tiene tres capas:

```
┌─────────────────────────────────────────────────────────────────┐
│  FUENTES DE DATOS                                                │
│  Sistema ventas │ ERP │ CRM │ Logs web │ APIs externas          │
└────────────────────────────┬────────────────────────────────────┘
                             │ ETL/ELT
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  STAGING AREA                                                    │
│  Datos crudos de las fuentes, sin transformar                   │
│  Zona temporal para validación y limpieza                       │
└────────────────────────────┬────────────────────────────────────┘
                             │ Transformación
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  DATA WAREHOUSE (Core)                                          │
│  Datos limpios, integrados, históricos                          │
│  Esquema normalizado (Inmon) o dimensional (Kimball)            │
└────────────────────────────┬────────────────────────────────────┘
                             │ Agregación / Marts
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  DATA MARTS                                                     │
│  Subconjuntos temáticos del DW optimizados para departamentos   │
│  Data mart de ventas │ Data mart de finanzas │ Data mart de RR  │
└────────────────────────────┬────────────────────────────────────┘
                             │ Consultas
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│  HERRAMIENTAS DE BI Y ANÁLISIS                                  │
│  Tableau │ Power BI │ Looker │ SQL ad-hoc │ Jupyter notebooks   │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 Inmon vs Kimball: dos enfoques para el DW

Existen dos escuelas de pensamiento sobre cómo diseñar el núcleo del data warehouse, asociadas con sus creadores:

**El enfoque de Bill Inmon (Corporate Information Factory):**

El DW central está en 3FN, normalizado como una base de datos relacional tradicional. Los data marts se construyen sobre el DW central para cada área de negocio, desnormalizando los datos para el análisis. El DW es la única fuente de verdad; los data marts son vistas analíticas del mismo.

```
Fuentes → Staging → DW Normalizado (3FN) → Data Marts (estrella)
```

*Ventaja:* El DW central es flexible: puede alimentar cualquier data mart. La normalización facilita la integración de nuevas fuentes.
*Desventaja:* Alto costo inicial de diseño e implementación. Las queries sobre el DW central siguen siendo complejas.

**El enfoque de Ralph Kimball (Dimensional Modeling):**

No existe un DW central separado. Se construyen directamente data marts con esquemas dimensionales (estrella o copo de nieve). La coherencia entre data marts se logra mediante dimensiones conformadas (dimensiones compartidas con la misma estructura y los mismos valores entre todos los marts). La colección de data marts integrados es el DW.

```
Fuentes → Staging → Data Marts dimensionales directamente
```

*Ventaja:* Resultados más rápidos. Los esquemas dimensionales son fáciles de entender para los usuarios de negocio.
*Desventaja:* Riesgo de inconsistencias si las dimensiones no están bien conformadas. Más difícil de integrar nuevas fuentes retroactivamente.

**En la práctica:** El enfoque Kimball dominó la industria durante los años 2000-2010 por su pragmatismo. Los data lakes y las plataformas modernas han cambiado el debate, pero el modelado dimensional de Kimball sigue siendo la base conceptual de la mayoría de los data warehouses en producción.

---

## 6. ETL: de OLTP a OLAP

### 6.1 Qué es ETL

**ETL** (Extract, Transform, Load) es el proceso de mover datos del sistema OLTP al data warehouse. Son tres fases distintas:

**Extract (Extracción):** Leer los datos desde las fuentes. Puede ser completo (full extract: leer todo cada vez) o incremental (delta extract: leer solo lo que cambió desde la última extracción). La extracción incremental es más eficiente pero más compleja.

**Transform (Transformación):** Limpiar, validar, enriquecer, reformatear, y reestructurar los datos para que sean coherentes con el esquema del DW. Esta es la fase más compleja y consume más tiempo de desarrollo.

**Load (Carga):** Insertar los datos transformados en el DW. Puede ser una carga completa (truncate + insert) o una carga incremental (upsert: insert o update si ya existe).

### 6.2 Los desafíos de la extracción

**Detectar cambios (Change Data Capture, CDC):**

Para la extracción incremental, el sistema ETL necesita saber qué datos cambiaron en el sistema fuente desde la última extracción. Las estrategias principales son:

**Timestamp-based CDC:** El sistema fuente tiene una columna `updated_at` que registra cuándo se modificó cada fila. La extracción incremental lee solo filas donde `updated_at > última_extracción`.

```sql
-- Extracción incremental por timestamp
SELECT *
FROM pedidos_fuente
WHERE updated_at > '2024-12-31 23:00:00'  -- desde la última extracción
ORDER BY updated_at;
```

*Problema:* No captura DELETEs (una fila eliminada no tiene `updated_at`). Requiere que la columna `updated_at` esté bien mantenida en el sistema fuente.

**Log-based CDC:** Lee el WAL del sistema OLTP para capturar todos los cambios (INSERT, UPDATE, DELETE) de forma exacta y en orden. Es la técnica más precisa y de menor impacto en el sistema fuente.

PostgreSQL soporta logical replication, que permite a herramientas como **Debezium** leer el WAL y publicar los cambios como eventos:

```
Debezium → Lee el WAL de PostgreSQL → Publica eventos en Kafka
→ Consumidor ETL lee eventos de Kafka → Carga en DW
```

**Full table extract:** Simplemente leer toda la tabla en cada extracción. Solo viable para tablas pequeñas o cuando no hay otra opción.

### 6.3 Los desafíos de la transformación

La transformación es donde vive la mayor parte de la complejidad del ETL. Los problemas más comunes:

**Datos sucios en la fuente:**

```python
# El campo nombre puede tener valores como:
# "GARCÍA, ANA MARÍA"    ← apellido primero, todo mayúsculas
# "ana garcia"            ← minúsculas, sin acento
# "Ana M. García"         ← inicial del segundo nombre
# None                    ← NULL
# ""                      ← cadena vacía

def limpiar_nombre(nombre):
    if not nombre or not nombre.strip():
        return None
    # Normalizar: título case, eliminar espacios extra
    return ' '.join(nombre.strip().title().split())
```

**Resolución de entidades (entity resolution):**

El mismo cliente puede existir en múltiples sistemas con identificadores distintos:
- En el ERP: cliente_id = 10045
- En el CRM: cliente_crm_id = "C-2019-00234"
- En el sistema web: usuario_id = "ana.garcia@email.com"

La transformación debe resolver que estos tres registros son el mismo cliente y asignarle un único identificador en el DW.

**Transformaciones de tipo SCD:**

Cuando un cliente cambia de ciudad, la transformación debe decidir si aplicar SCD Tipo 1 (sobreescribir), Tipo 2 (crear nueva versión), u otro tipo, según las reglas del negocio y el diseño del DW (visto en el Módulo 6).

**Cálculo de métricas derivadas:**

```sql
-- El DW puede pre-calcular métricas que se usan frecuentemente
INSERT INTO hechos_ventas (venta_id, tiempo_id, producto_id, cliente_id,
                           cantidad, precio_unit, descuento, monto_neto,
                           margen_bruto)
SELECT
    v.venta_id,
    t.tiempo_id,
    v.producto_id,
    v.cliente_id,
    v.cantidad,
    v.precio_unit,
    COALESCE(v.descuento, 0),
    v.cantidad * v.precio_unit * (1 - COALESCE(v.descuento, 0)),  -- monto_neto
    v.cantidad * (v.precio_unit - p.costo_unitario)               -- margen_bruto
FROM ventas_staging v
JOIN dim_tiempo t ON t.fecha = DATE(v.fecha_venta)
JOIN productos_staging p ON p.producto_id = v.producto_id;
```

### 6.4 Los desafíos de la carga

**Upsert eficiente:**

Para la carga incremental, necesitamos insertar filas nuevas y actualizar filas existentes sin duplicar:

```sql
-- Upsert en PostgreSQL (INSERT ... ON CONFLICT)
INSERT INTO dim_cliente (cliente_id, nombre, email, ciudad, segmento)
SELECT cliente_id, nombre, email, ciudad, segmento
FROM staging_clientes
ON CONFLICT (cliente_id) DO UPDATE
SET nombre    = EXCLUDED.nombre,
    ciudad    = EXCLUDED.ciudad,
    segmento  = EXCLUDED.segmento
WHERE dim_cliente.nombre    IS DISTINCT FROM EXCLUDED.nombre
   OR dim_cliente.ciudad    IS DISTINCT FROM EXCLUDED.ciudad
   OR dim_cliente.segmento  IS DISTINCT FROM EXCLUDED.segmento;
-- La condición WHERE evita actualizar filas que no cambiaron realmente
```

**Orden de carga:**

Las tablas de hechos referencian las tablas de dimensiones mediante surrogate keys. La carga debe seguir el orden correcto: primero cargar todas las dimensiones, luego cargar los hechos que las referencian.

```
1. Cargar dim_tiempo (raramente cambia)
2. Cargar dim_producto (cambios por SCD)
3. Cargar dim_cliente (cambios por SCD)
4. Cargar dim_vendedor
5. Cargar hechos_ventas (referencia todas las dimensiones anteriores)
```

**Ventanas de carga y disponibilidad:**

Los ETL tradicionales tenían ventanas nocturnas: extraían y cargaban datos entre las 2 AM y las 6 AM, cuando la carga del sistema OLTP era mínima. Con datos de mayor frecuencia requeridos (dashboards en tiempo cuasi-real), estas ventanas se han reducido a horas o incluso minutos, lo que lleva al paradigma de streaming ETL.

---

## 7. ELT: la inversión del paradigma

### 7.1 La diferencia entre ETL y ELT

En ETL, la transformación ocurre **antes** de cargar los datos en el DW. En **ELT** (Extract, Load, Transform), los datos se cargan primero en crudo en el destino, y las transformaciones se realizan **dentro** del destino, usando su capacidad de procesamiento.

```
ETL: Fuente → [Extraer] → [Transformar en servidor ETL] → [Cargar] → DW
ELT: Fuente → [Extraer] → [Cargar en crudo] → DW raw → [Transformar en DW] → DW listo
```

### 7.2 Por qué ELT ganó popularidad

La razón principal es que los data warehouses modernos en la nube (Snowflake, BigQuery, Redshift, Databricks) tienen una capacidad de procesamiento masiva y elástica. Tiene más sentido aprovechar esa capacidad para las transformaciones que hacerlo en un servidor ETL separado.

Ventajas del ELT:
- **Los datos crudos están siempre disponibles** en el DW. Si se descubre un bug en la transformación, se puede re-transformar desde el crudo sin necesidad de re-extraer de las fuentes.
- **Menos movimiento de datos:** los datos se mueven una sola vez (fuente → DW). Las transformaciones son operaciones internas del DW.
- **Escalabilidad:** la capacidad de transformación escala con el DW, no con el servidor ETL.
- **Menor latencia potencial:** cargar datos crudos es más rápido que transformarlos primero; la transformación puede ejecutarse después.

### 7.3 La herramienta dbt: transformaciones como código SQL versionado

**dbt** (data build tool) se ha convertido en el estándar de facto para el paradigma ELT. dbt gestiona las transformaciones dentro del DW como modelos SQL versionados en git, con tests de calidad de datos integrados.

Un modelo dbt es un archivo `.sql` que define una tabla o vista en el DW:

```sql
-- models/marts/ventas/fct_ventas.sql
-- dbt materializa este SELECT como una tabla en el DW

WITH ventas_raw AS (
    SELECT * FROM {{ ref('stg_ventas') }}  -- referencia a otro modelo dbt
),

productos AS (
    SELECT * FROM {{ ref('dim_producto') }}
),

clientes AS (
    SELECT * FROM {{ ref('dim_cliente') }}
    WHERE es_actual = TRUE
)

SELECT
    v.venta_id,
    v.fecha_venta,
    p.nombre_producto,
    p.nombre_categoria,
    c.nombre_cliente,
    c.segmento,
    v.cantidad,
    v.precio_unit,
    v.cantidad * v.precio_unit AS monto_bruto
FROM ventas_raw v
LEFT JOIN productos p ON p.producto_id = v.producto_id
LEFT JOIN clientes c ON c.cliente_id = v.cliente_id
```

dbt genera el grafo de dependencias entre modelos y los ejecuta en el orden correcto. También genera documentación automáticamente y permite añadir tests:

```yaml
# schema.yml - tests de calidad de datos
models:
  - name: fct_ventas
    columns:
      - name: venta_id
        tests:
          - unique
          - not_null
      - name: monto_bruto
        tests:
          - not_null
          - dbt_utils.accepted_range:
              min_value: 0
```

---

## 8. Almacenamiento columnar: el corazón del rendimiento OLAP

### 8.1 Row store vs column store

En un **row store** (almacenamiento por filas), como el heap de PostgreSQL, todos los atributos de una fila están almacenados contiguamente en disco:

```
Página del heap (row store):
[fila1: id=1, nombre='Ana', depto='Ventas', salario=5000]
[fila2: id=2, nombre='Luis', depto='RRHH', salario=4500]
[fila3: id=3, nombre='María', depto='Ventas', salario=5500]
...
```

Para una query OLAP que calcula el salario promedio por departamento, debe leer **todas las páginas** para obtener los valores de `depto` y `salario`, aunque las columnas `id` y `nombre` no sean necesarias.

En un **column store** (almacenamiento columnar), todos los valores de una columna están almacenados contiguamente:

```
Archivo de columna "depto":
['Ventas', 'RRHH', 'Ventas', 'IT', 'Ventas', 'RRHH', ...]

Archivo de columna "salario":
[5000, 4500, 5500, 6000, 4800, 5200, ...]

Archivo de columna "nombre":
['Ana', 'Luis', 'María', 'Carlos', 'Pedro', 'Elena', ...]
```

Para la misma query, el motor columnar lee **solo los archivos de `depto` y `salario`**, ignorando completamente `nombre` e `id`. Si la tabla tiene 20 columnas pero la query solo usa 3, el motor columnar lee 15% de los datos que leería un row store.

### 8.2 Las ventajas del almacenamiento columnar para OLAP

**Ventaja 1: Proyección eficiente**

Las queries analíticas raramente necesitan todas las columnas. Leen 3-5 de las 20-50 columnas de una tabla de hechos. El almacenamiento columnar lee solo las columnas necesarias.

**Ventaja 2: Compresión superior**

Los valores de una misma columna suelen tener alta correlación: el departamento de los empleados tiene pocos valores distintos, los timestamps están cerca entre sí, los montos tienen distribuciones predecibles. Comprimir valores similares juntos es mucho más efectivo que comprimir filas heterogéneas.

Las técnicas de compresión columnares incluyen:
- **Run-Length Encoding (RLE):** Si los mismos valores se repiten en secuencia, almacenar (valor, count) en lugar de repetir el valor.
  ```
  [Ventas, Ventas, Ventas, RRHH, RRHH, IT, IT, IT, IT] → [(Ventas,3), (RRHH,2), (IT,4)]
  ```
- **Dictionary encoding:** Reemplazar strings frecuentes por un código numérico corto. 'Cochabamba' (10 bytes) → 42 (1-2 bytes).
- **Delta encoding:** Almacenar diferencias entre valores consecutivos en lugar de los valores absolutos. Útil para timestamps ordenados.
- **Bit-packing:** Comprimir enteros de baja cardinalidad usando solo los bits necesarios.

Los ratios de compresión típicos en columnar son 5x-20x, vs 2x-4x en row store. Menos datos en disco = menos I/O = queries más rápidas.

**Ventaja 3: SIMD y vectorización**

Los procesadores modernos tienen instrucciones SIMD (Single Instruction, Multiple Data) que pueden operar sobre múltiples valores simultáneamente. Procesar un array de 16 enteros de 32 bits con una sola instrucción SIMD es mucho más eficiente que procesarlos uno por uno. El almacenamiento columnar permite aprovechar SIMD directamente sobre los arrays de valores de cada columna.

### 8.3 Las desventajas del columnar para OLTP

Todo lo que hace el columnar excelente para OLAP lo hace terrible para OLTP:

- **INSERT fila por fila:** Insertar una fila requiere agregar un valor a cada archivo de columna. En row store es una escritura secuencial en el heap; en columnar son N escrituras dispersas (una por columna).
- **UPDATE:** Actualizar un valor de una columna requiere reescribir esa columna (o marcar el valor como eliminado y escribir el nuevo en un delta store).
- **Lectura de filas completas:** Para leer todos los atributos de una fila específica (operación básica del OLTP), el columnar debe leer N archivos de columna y reconstructir la fila, vs una sola lectura en el row store.

### 8.4 Motores columnares en la práctica

**PostgreSQL + cstore_fdw / Citus:** PostgreSQL no tiene almacenamiento columnar nativo, pero extensiones como `cstore_fdw` (ahora parte de Hydra) ofrecen tablas columnares. Citus (ahora Citus de Microsoft) añade capacidades distribuidas y analíticas.

**Amazon Redshift:** Data warehouse columnar en la nube, basado en una versión modificada de PostgreSQL 8. Usa almacenamiento columnar y compresión nativa. Es la arquitectura de referencia del paradigma columnar cloud.

**Google BigQuery:** Motor columnar serverless en la nube de Google. No requiere gestión de clusters; factura por bytes procesados.

**Snowflake:** Almacenamiento columnar cloud con separación de cómputo y almacenamiento. Puede escalar el cómputo sin mover los datos.

**DuckDB:** Motor columnar embebido (como SQLite pero para OLAP). Puede ejecutarse directamente en un proceso Python o dentro de un archivo. Excelente para análisis local o en notebooks.

**Apache Parquet:** No es un motor, sino un formato de archivo columnar para datos en un data lake. Es el formato de facto para almacenar datos analíticos en sistemas como Spark, Presto, Trino, y Athena.

---

## 9. HTAP: el intento de unificar ambos mundos

### 9.1 Qué es HTAP

**HTAP** (Hybrid Transactional/Analytical Processing) es un paradigma que intenta combinar las capacidades OLTP y OLAP en un único sistema, eliminando la necesidad de un pipeline ETL separado.

El objetivo: ejecutar consultas analíticas sobre los datos transaccionales en tiempo real, sin copiar los datos a un DW separado y sin degradar el rendimiento transaccional.

### 9.2 Enfoques técnicos para HTAP

**Tablas in-memory con almacenamiento dual:**

SAP HANA fue pionero en este enfoque: mantiene los datos en memoria en formato columnar para analítica y en formato de fila para transacciones, sincronizados automáticamente.

**Replicación en tiempo real hacia un nodo columnar:**

MySQL/MariaDB con InnoDB + un nodo secundario con el columnar engine Columnstore Index (TiDB hace algo similar). Las escrituras van al nodo de fila; las lecturas analíticas van al nodo columnar.

**Snapshot isolation como separador natural:**

PostgreSQL con una réplica de solo lectura (standby) configurada para consultas analíticas. Las queries analíticas van al standby, las transaccionales al primario. No es exactamente HTAP (los datos en el standby tienen latencia de replicación), pero es el patrón más pragmático en ecosistemas PostgreSQL.

### 9.3 Los compromisos de HTAP

No existe un almuerzo gratis. Los sistemas HTAP tienen compromisos:

- **Complejidad:** Mantener dos representaciones de los mismos datos en sincronía tiene overhead.
- **Latencia de analytics:** Los datos analíticos pueden tener segundos o minutos de latencia respecto al primario (dependiendo del mecanismo de sincronización).
- **Costo de hardware:** Mantener datos en memoria (como SAP HANA) tiene un costo de infraestructura alto.
- **Escala:** Los sistemas HTAP escalan bien hasta cierto punto, pero para análisis histórico de terabytes, un DW dedicado sigue siendo superior.

**La conclusión práctica:** HTAP es una solución valiosa para análisis operacional en tiempo real (monitorear KPIs del día actual, dashboards operacionales en tiempo real). Para análisis histórico profundo, ML sobre grandes volúmenes de datos históricos, o informes regulatorios complejos, el data warehouse dedicado sigue siendo la arquitectura correcta.

---

## 10. Patrones de arquitectura modernos

### 10.1 El Data Lake

Un **data lake** es un repositorio de datos en su forma cruda (sin transformar), generalmente almacenado en un sistema de archivos distribuido (HDFS, S3, GCS, ADLS) en formato Parquet o similares.

```
Fuentes de datos
    │
    ▼
Data Lake (S3/GCS/ADLS)
    ├── raw/           ← datos crudos tal como llegan
    ├── cleaned/       ← datos limpios
    └── curated/       ← datos transformados para análisis
```

**La diferencia entre data lake y data warehouse:**
- El DW tiene esquema fijo, datos estructurados, calidad garantizada.
- El data lake acepta cualquier formato (JSON, CSV, Parquet, logs, imágenes), el esquema se define al leer ("schema on read").

**El problema del data lake:** Sin disciplina, se convierte en un "data swamp": datos sin documentar, sin linaje, sin calidad garantizada, imposibles de usar.

### 10.2 El Lakehouse

El **lakehouse** es la arquitectura moderna que intenta combinar lo mejor del data lake (almacenamiento barato, datos crudos, schema-on-read) con lo mejor del data warehouse (calidad de datos, transacciones ACID, rendimiento):

- Almacenamiento en un object store (S3/GCS) en formato Parquet.
- Una capa de metadatos transaccional (Delta Lake, Apache Iceberg, Apache Hudi) que añade ACID, schema enforcement, y time travel sobre los archivos Parquet.
- Motores de query que pueden leer directamente del object store con rendimiento comparable al DW.

```
Object Store (S3)
    + Delta Lake / Iceberg (ACID, schema, versioning)
    + Motor de query (Spark, Trino, DuckDB, Databricks SQL)
    = Lakehouse
```

### 10.3 El pipeline moderno de datos

Una arquitectura típica de datos moderna:

```
Fuentes OLTP (PostgreSQL, MySQL)
    │
    ├── CDC via Debezium → Kafka (streaming)
    │                          │
    │                          ▼
    │                    Data Lake raw (S3 + Iceberg)
    │                          │
    │                          ▼
    │                    dbt transformations
    │                          │
    │                          ▼
    │                    Data Warehouse (Snowflake/BigQuery)
    │                          │
    │                          ▼
    │                    BI Tools (Tableau/Looker/Metabase)
    │
    └── Réplica de lectura PostgreSQL (para analítica operacional en tiempo real)
```

Esta arquitectura separa claramente las responsabilidades:
- PostgreSQL (OLTP): fuente de verdad operacional.
- Kafka: transporte de eventos en tiempo real.
- Data lake: almacenamiento histórico completo y barato.
- dbt: transformaciones versionadas y testeadas.
- Data warehouse: consultas analíticas de alto rendimiento.
- Réplica de lectura: analítica operacional sin impactar el primario.

---

## 11. Resumen y conexión con el resto del curso

### Los conceptos clave y cuándo aplicarlos

| Concepto | Aplicación práctica |
|---|---|
| **OLTP normalizado** | Base de datos operacional: integridad, baja latencia, escrituras frecuentes |
| **OLAP desnormalizado** | Data warehouse: queries analíticas sobre millones de filas |
| **Separación OLTP/OLAP** | Cualquier sistema con analítica sobre millones de registros |
| **ETL** | Pipeline de datos cuando la transformación es compleja y controlada |
| **ELT + dbt** | Pipeline moderno: transformaciones versionadas dentro del DW |
| **CDC / Debezium** | Extracción incremental de bajo impacto sobre el sistema fuente |
| **Almacenamiento columnar** | Cuando las queries leen pocas columnas sobre muchas filas |
| **HTAP** | Análisis operacional en tiempo real sobre datos recientes |
| **Lakehouse** | Cuando se necesita la flexibilidad del data lake con garantías del DW |

### La meta-lección del módulo

La separación entre OLTP y OLAP no es una preferencia de arquitectura; es una necesidad que emerge de requerimientos fundamentalmente incompatibles. Ignorarla tiene consecuencias predecibles: el sistema OLTP se degrada bajo carga analítica, los análisis son lentos e inconsistentes, y el equipo de datos pierde tiempo optimizando queries que nunca serán eficientes en el modelo equivocado.

El momento correcto para introducir la separación depende del volumen y la frecuencia del análisis. Para una startup con 100,000 registros, un reporting schema en la misma base de datos (con vistas y una réplica de lectura) puede ser suficiente. Para una empresa con 100 millones de registros y analistas que ejecutan queries complejas varias veces por día, la separación es urgente.

### Conexión con módulos futuros

- **Módulo 11 (Dimensional Modeling):** Es la continuación directa de este módulo. El esquema estrella y el modelo dimensional de Kimball son la forma estándar de diseñar el data warehouse. Los conceptos de tablas de hechos, dimensiones, SCD y granularidad se explican en profundidad.

- **Módulo 12 (Modelos alternativos):** El almacenamiento columnar descrito aquí es un modelo de almacenamiento alternativo al relacional por filas. El módulo 12 también cubre series de tiempo, grafos y documentos, que tienen sus propios patrones de almacenamiento con motivaciones similares.

- **Módulo 13 (Sistemas distribuidos):** Los data warehouses modernos en la nube son sistemas distribuidos. Snowflake, BigQuery, y Redshift separan el almacenamiento del cómputo y distribuyen las queries entre múltiples nodos. Los conceptos del módulo 13 sobre consistencia eventual, particionamiento, y coordinación son directamente relevantes.

---

## 12. Ejercicios de comprensión

**Ejercicio 1.** Una empresa de seguros tiene un sistema OLTP en PostgreSQL con las siguientes tablas principales: `polizas` (5M filas), `siniestros` (2M filas), `pagos` (20M filas), `asegurados` (3M filas). El equipo de actuarios ejecuta queries como:

```sql
-- Tasa de siniestralidad por tipo de póliza y región en los últimos 5 años
SELECT tipo_poliza, region, anio,
       SUM(monto_siniestro) / SUM(prima_anual) AS tasa_siniestralidad
FROM polizas p
JOIN siniestros s ON s.poliza_id = p.poliza_id
JOIN asegurados a ON a.asegurado_id = p.asegurado_id
WHERE p.fecha_inicio >= '2019-01-01'
GROUP BY tipo_poliza, region, anio;
```

Esta query tarda 8 minutos en el sistema de producción.

a) Identifica todos los problemas que causa ejecutar esta query en el sistema OLTP.
b) Diseña la arquitectura de separación mínima viable para este caso.
c) Diseña el esquema del DW para soportar eficientemente las queries actuariales, incluyendo al menos 4 tablas.

---

**Ejercicio 2.** Una empresa necesita implementar CDC desde su PostgreSQL OLTP hacia un data warehouse. Las tablas fuente son:

- `transacciones`: 500,000 inserciones/día, nunca se actualiza.
- `clientes`: 1,000 updates/día, 200 inserciones/día.
- `productos`: 50 updates/día, 10 inserciones/día.
- `pedidos`: 100,000 inserciones/día, 50,000 updates/día (cambios de estado).

Para cada tabla:
a) ¿Qué estrategia de CDC elegirías (timestamp-based, log-based, full extract)?
b) Justifica considerando el volumen, la frecuencia de cambios, y si hay DELETEs posibles.
c) ¿Qué columnas adicionales necesitarías agregar a las tablas OLTP para soportar la estrategia elegida (si aplica)?

---

**Ejercicio 3.** Dado el siguiente esquema OLTP:

```sql
CREATE TABLE ventas (
    venta_id     SERIAL PRIMARY KEY,
    fecha        TIMESTAMPTZ NOT NULL,
    cliente_id   INT NOT NULL,
    producto_id  INT NOT NULL,
    vendedor_id  INT NOT NULL,
    sucursal_id  INT NOT NULL,
    cantidad     INT NOT NULL,
    precio_unit  NUMERIC(10,2) NOT NULL,
    descuento    NUMERIC(5,2) DEFAULT 0
);
CREATE TABLE clientes (cliente_id SERIAL PK, nombre TEXT, ciudad TEXT, segmento TEXT);
CREATE TABLE productos (producto_id SERIAL PK, nombre TEXT, categoria TEXT, costo NUMERIC);
CREATE TABLE vendedores (vendedor_id SERIAL PK, nombre TEXT, zona TEXT, antiguedad INT);
CREATE TABLE sucursales (sucursal_id SERIAL PK, nombre TEXT, ciudad TEXT, region TEXT);
```

Diseña el esquema del data warehouse (esquema estrella) para soportar las siguientes necesidades analíticas:
- Análisis de ventas por período de tiempo (año, trimestre, mes, semana, día).
- Rentabilidad por producto y categoría.
- Análisis de clientes por segmento y ciudad.
- Rendimiento de vendedores por zona.
- Tendencias de ventas por sucursal y región.

Incluye: DDL completo, qué atributos van en cada dimensión, qué métricas van en la tabla de hechos, y cómo manejarías los cambios históricos en clientes y vendedores.

---

**Ejercicio 4.** Compara el rendimiento teórico de row store vs column store para las siguientes queries sobre una tabla con 100 millones de filas y 30 columnas (promedio 50 bytes por columna = 1,500 bytes por fila total):

a) `SELECT cliente_id, monto FROM ventas WHERE fecha > '2024-01-01'` (2 columnas de 30)
b) `SELECT * FROM ventas WHERE venta_id = 12345` (1 fila específica)
c) `SELECT AVG(monto), COUNT(*) FROM ventas GROUP BY region` (2 columnas + 1 filtro implícito)

Para cada query: ¿cuántos bytes aproximadamente leería un row store vs un column store? ¿Cuándo es cada uno más eficiente?

---

**Ejercicio 5.** Una empresa tiene el siguiente pipeline ETL que se ejecuta cada noche:

1. A las 2 AM: extrae datos del OLTP (tarda 45 min).
2. A las 2:45 AM: transforma y limpia (tarda 90 min).
3. A las 4:15 AM: carga al DW (tarda 30 min).
4. A las 4:45 AM: los dashboards de negocio se actualizan.
5. Los ejecutivos revisan los dashboards a las 8 AM con datos del día anterior.

La empresa quiere reducir la latencia de datos a menos de 2 horas. Diseña la migración de ETL a ELT en tiempo casi-real usando CDC y dbt. Define:
a) Las herramientas y componentes necesarios.
b) La frecuencia de actualización de cada capa.
c) Qué decisiones de diseño del DW cambiarían con datos en tiempo casi-real (vs datos del día anterior).

---

*Próximo módulo: Dimensional Modeling y Kimball — donde profundizaremos en la metodología formal de Ralph Kimball para diseñar el data warehouse: tablas de hechos, dimensiones, el esquema estrella, y cómo las Slowly Changing Dimensions del Módulo 6 se aplican en este contexto.*
