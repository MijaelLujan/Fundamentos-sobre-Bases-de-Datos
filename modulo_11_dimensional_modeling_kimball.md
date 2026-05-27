# Módulo 11 — Dimensional Modeling y Kimball

> *"El modelo dimensional no es solo una técnica de diseño de base de datos. Es un lenguaje que permite que los datos hablen el mismo idioma que los usuarios de negocio. Cuando Ralph Kimball definió los conceptos de hechos, dimensiones y el esquema estrella, no estaba describiendo cómo almacenar datos; estaba describiendo cómo los seres humanos piensan sobre los eventos del negocio: algo ocurrió, en cierto momento, en cierto lugar, involucrando ciertos actores, y produjo cierta medición."*

---

## Tabla de contenidos

1. [El modelo dimensional: por qué existe](#1-el-modelo-dimensional-por-qué-existe)
2. [Hechos y dimensiones: los dos componentes fundamentales](#2-hechos-y-dimensiones-los-dos-componentes-fundamentales)
3. [Tablas de hechos en profundidad](#3-tablas-de-hechos-en-profundidad)
4. [Tablas de dimensiones en profundidad](#4-tablas-de-dimensiones-en-profundidad)
5. [El esquema estrella](#5-el-esquema-estrella)
6. [El esquema copo de nieve](#6-el-esquema-copo-de-nieve)
7. [La granularidad: la decisión más importante](#7-la-granularidad-la-decisión-más-importante)
8. [Slowly Changing Dimensions en contexto dimensional](#8-slowly-changing-dimensions-en-contexto-dimensional)
9. [Dimensiones conformadas](#9-dimensiones-conformadas)
10. [Tablas de hechos sin hechos](#10-tablas-de-hechos-sin-hechos)
11. [Tipos especiales de tablas de hechos](#11-tipos-especiales-de-tablas-de-hechos)
12. [La dimensión de tiempo](#12-la-dimensión-de-tiempo)
13. [El proceso de diseño dimensional de Kimball](#13-el-proceso-de-diseño-dimensional-de-kimball)
14. [Resumen y conexión con el resto del curso](#14-resumen-y-conexión-con-el-resto-del-curso)
15. [Ejercicios de comprensión](#15-ejercicios-de-comprensión)

---

## 1. El modelo dimensional: por qué existe

### 1.1 El problema con el esquema relacional normalizado para análisis

En el Módulo 10 vimos que los esquemas OLTP altamente normalizados son subóptimos para queries analíticas. Pero el problema es más profundo que el rendimiento: la estructura normalizada también es difícil de entender y usar para los analistas y usuarios de negocio.

Considera un analista que quiere saber las ventas por región y categoría de producto durante el último trimestre. En un esquema normalizado típico, la query podría requerir joins a través de 7 u 8 tablas:

```sql
-- En un esquema normalizado, esta query aparentemente simple es compleja
SELECT
    geo.nombre_region,
    cat.nombre_categoria,
    SUM(v.cantidad * v.precio_unit) AS total_ventas
FROM ventas v
JOIN items_pedido ip ON ip.item_id = v.item_id
JOIN pedidos p ON p.pedido_id = ip.pedido_id
JOIN clientes c ON c.cliente_id = p.cliente_id
JOIN direcciones d ON d.direccion_id = c.dir_facturacion_id
JOIN regiones geo ON geo.region_id = d.region_id
JOIN productos pr ON pr.producto_id = ip.producto_id
JOIN subcategorias sc ON sc.subcategoria_id = pr.subcategoria_id
JOIN categorias cat ON cat.categoria_id = sc.categoria_id
WHERE p.fecha >= DATE_TRUNC('quarter', NOW() - INTERVAL '3 months')
  AND p.fecha < DATE_TRUNC('quarter', NOW())
GROUP BY geo.nombre_region, cat.nombre_categoria;
```

Para un analista de negocio, esta query es hostil: requiere conocer el modelo de datos detallado, saber cómo se navega la jerarquía de categorías, y entender la estructura de las direcciones. La barrera de entrada es alta.

### 1.2 La intuición del modelo dimensional

Ralph Kimball propuso una forma distinta de pensar sobre los datos analíticos, basada en cómo los negocios describen sus operaciones naturalmente:

> "Vendimos 3 unidades del producto X al cliente Y el día Z en la tienda W a un precio de P."

Este enunciado tiene dos partes:
- **La medición numérica del evento:** 3 unidades, precio P.
- **El contexto del evento:** qué producto, qué cliente, cuándo, en qué tienda.

En el modelo dimensional:
- Las **mediciones** viven en la **tabla de hechos**.
- El **contexto** vive en las **tablas de dimensiones**.

La query analítica del ejemplo anterior, en un esquema dimensional bien diseñado, se convierte en:

```sql
-- En un esquema dimensional, la misma query es intuitiva
SELECT
    dc.nombre_region,
    dp.nombre_categoria,
    SUM(fv.monto_neto) AS total_ventas
FROM fact_ventas fv
JOIN dim_cliente dc ON dc.cliente_sk = fv.cliente_sk
JOIN dim_producto dp ON dp.producto_sk = fv.producto_sk
JOIN dim_tiempo dt ON dt.tiempo_sk = fv.tiempo_sk
WHERE dt.anio = 2024 AND dt.trimestre = 4
GROUP BY dc.nombre_region, dp.nombre_categoria;
```

Cuatro tablas en lugar de ocho. Las jerarquías (región dentro de dirección, categoría dentro de producto) están desnormalizadas dentro de las dimensiones. Los analistas pueden explorar los datos sin necesidad de conocer cada detalle del modelo OLTP subyacente.

---

## 2. Hechos y dimensiones: los dos componentes fundamentales

### 2.1 Los hechos: mediciones numéricas del negocio

Un **hecho** es una medición numérica de un evento del negocio. Los hechos son lo que el negocio quiere analizar y agregar: cuánto vendió, cuánto costó, cuántas unidades se movieron, cuántos clientes se perdieron.

Los hechos tienen una propiedad esencial: son **aditivos** (o al menos parcialmente aditivos). Tiene sentido sumarlos a lo largo de las dimensiones de análisis. "Total de ventas en diciembre" suma todas las ventas de diciembre. "Promedio de precio" es una agregación válida de los precios individuales.

Ejemplos de hechos:
- Monto de una venta (`monto_venta NUMERIC(12,2)`)
- Cantidad vendida (`cantidad INT`)
- Costo del producto (`costo_unitario NUMERIC(10,2)`)
- Duración de una llamada en segundos (`duracion_seg INT`)
- Número de clicks (`num_clicks INT`)

Lo que no es un hecho (aunque sea numérico): el ID del cliente, el código del producto. Estos son **claves de dimensión**, no mediciones.

### 2.2 Las dimensiones: el contexto de los hechos

Una **dimensión** describe el contexto en el que ocurrió el hecho. Es el "quién, qué, cuándo, dónde, cómo" del evento del negocio.

Las dimensiones contienen atributos **descriptivos y textuales** que los analistas usan para filtrar, agrupar y etiquetar los hechos:

- **Dimensión Tiempo:** año, trimestre, mes, semana, día, nombre del día, es_feriado.
- **Dimensión Producto:** nombre del producto, categoría, subcategoría, marca, precio de lista.
- **Dimensión Cliente:** nombre, ciudad, región, segmento, nivel de fidelidad.
- **Dimensión Vendedor:** nombre, zona, equipo, nivel.
- **Dimensión Tienda/Sucursal:** nombre, ciudad, región, tipo de tienda.

Las dimensiones son ampias y planas (desnormalizadas). Una tabla de dimensión de producto puede tener 50-100 columnas que describen todos los atributos del producto y su jerarquía de categorías, todo en una sola fila.

### 2.3 La surrogate key: la clave del modelo dimensional

Cada tabla de dimensión tiene una **surrogate key** (clave sustituta): un entero generado por el sistema de DW, sin significado de negocio, que identifica de forma única cada versión de un registro de la dimensión.

```sql
CREATE TABLE dim_producto (
    producto_sk      INT PRIMARY KEY,   -- surrogate key: generada por el DW
    producto_bk      VARCHAR(50) NOT NULL,  -- business key: el ID del sistema fuente
    nombre_producto  TEXT NOT NULL,
    sku              VARCHAR(50) NOT NULL,
    -- ... más atributos
);
```

La surrogate key es necesaria por varias razones:

**Razón 1: Soporte para SCD Tipo 2.** Cuando un cliente cambia de ciudad (SCD Tipo 2), se crea una nueva fila con una nueva `cliente_sk`. Todas las ventas históricas siguen apuntando a la `cliente_sk` anterior (la versión del cliente cuando ocurrió la venta). Sin surrogate key, no habría forma de distinguir la versión "antes" y "después".

**Razón 2: Integración de múltiples fuentes.** Si el mismo cliente existe en el ERP con ID 10045 y en el CRM con ID "C-2019-00234", la surrogate key proporciona un identificador único en el DW independiente de los sistemas fuente.

**Razón 3: Protección contra cambios en los sistemas fuente.** Si el sistema fuente cambia su esquema de IDs, el DW no se ve afectado porque usa sus propias surrogate keys.

---

## 3. Tablas de hechos en profundidad

### 3.1 Estructura de una tabla de hechos

Una tabla de hechos tiene:
- **Foreign keys** a cada tabla de dimensión relevante (las claves de dimensión).
- **Mediciones numéricas** del evento (los hechos).
- Opcionalmente, atributos degenerados (ver más abajo).

```sql
CREATE TABLE fact_ventas (
    -- Claves de dimensión (FKs)
    tiempo_sk    INT NOT NULL REFERENCES dim_tiempo(tiempo_sk),
    producto_sk  INT NOT NULL REFERENCES dim_producto(producto_sk),
    cliente_sk   INT NOT NULL REFERENCES dim_cliente(cliente_sk),
    vendedor_sk  INT NOT NULL REFERENCES dim_vendedor(vendedor_sk),
    tienda_sk    INT NOT NULL REFERENCES dim_tienda(tienda_sk),
    
    -- Hechos (mediciones)
    cantidad         INT NOT NULL,
    precio_unit      NUMERIC(10,2) NOT NULL,
    descuento_pct    NUMERIC(5,2) NOT NULL DEFAULT 0,
    monto_bruto      NUMERIC(12,2) NOT NULL,  -- cantidad * precio_unit
    monto_descuento  NUMERIC(12,2) NOT NULL,  -- monto_bruto * descuento_pct
    monto_neto       NUMERIC(12,2) NOT NULL,  -- monto_bruto - monto_descuento
    costo_total      NUMERIC(12,2) NOT NULL,  -- cantidad * costo_unitario
    margen_bruto     NUMERIC(12,2) NOT NULL,  -- monto_neto - costo_total
    
    -- Atributo degenerado (ver sección 3.3)
    numero_factura   VARCHAR(20) NOT NULL
);
```

### 3.2 Los tres tipos de hechos

**Hechos aditivos:**

Pueden sumarse a lo largo de **todas** las dimensiones. Son el tipo más común y el más útil.

```sql
-- monto_neto es aditivo: sumar a través de tiempo, producto, cliente, etc. es válido
SELECT SUM(monto_neto) AS total_ventas FROM fact_ventas;
SELECT SUM(monto_neto) FROM fact_ventas WHERE tiempo_sk IN (...); -- por período
SELECT SUM(monto_neto) FROM fact_ventas WHERE producto_sk IN (...); -- por producto
-- Todos son válidos y significativos
```

**Hechos semi-aditivos:**

Pueden sumarse a lo largo de algunas dimensiones pero no de todas. El ejemplo clásico es el saldo de una cuenta bancaria: sumar saldos de diferentes cuentas en el mismo momento tiene sentido ("saldo total del banco"), pero sumar saldos de la misma cuenta en diferentes momentos no lo tiene ("suma de saldos de enero a diciembre").

```sql
CREATE TABLE fact_saldos_diarios (
    tiempo_sk   INT NOT NULL REFERENCES dim_tiempo,
    cuenta_sk   INT NOT NULL REFERENCES dim_cuenta,
    saldo       NUMERIC(15,2) NOT NULL  -- semi-aditivo: no sumar a través del tiempo
);

-- CORRECTO: saldo total en un momento dado (sumar a través de cuentas)
SELECT SUM(saldo) AS saldo_total_sistema
FROM fact_saldos_diarios
WHERE tiempo_sk = 20241231;  -- fin de año

-- INCORRECTO (pero fácil de cometer): suma a través del tiempo
SELECT SUM(saldo) AS esto_no_tiene_sentido
FROM fact_saldos_diarios
WHERE cuenta_sk = 42;
-- Esto suma el saldo de diciembre 31 veces (una por cada día del mes)
-- No tiene ningún significado de negocio
```

La query correcta para el análisis temporal de saldos usa `AVG` o lee un punto en el tiempo específico:

```sql
-- Saldo promedio durante el mes (puede tener sentido)
SELECT AVG(saldo) AS saldo_promedio_mes
FROM fact_saldos_diarios
WHERE tiempo_sk IN (SELECT tiempo_sk FROM dim_tiempo WHERE anio = 2024 AND mes = 12)
  AND cuenta_sk = 42;
```

**Hechos no aditivos:**

No pueden sumarse a lo largo de ninguna dimensión de forma significativa. Ratios, porcentajes, y tasas son los ejemplos más comunes.

```sql
-- precio_unitario es no aditivo: sumar precios unitarios de distintos productos no tiene sentido
-- tasa_conversion (visitas / ventas) es no aditivo
-- porcentaje_descuento es no aditivo

-- Para analizar ratios en el DW, la práctica correcta es almacenar
-- el numerador y el denominador por separado (que sí son aditivos)
-- y calcular el ratio en la capa de presentación:

CREATE TABLE fact_metricas_web (
    tiempo_sk        INT NOT NULL,
    campana_sk       INT NOT NULL,
    num_visitas      INT NOT NULL,   -- aditivo
    num_conversiones INT NOT NULL,   -- aditivo
    -- tasa_conversion = num_conversiones / num_visitas
    -- se calcula en la query, no se almacena
);

-- Tasa de conversión por campaña (calculada, no almacenada):
SELECT dc.nombre_campana,
       SUM(num_conversiones)::FLOAT / NULLIF(SUM(num_visitas), 0) AS tasa_conversion
FROM fact_metricas_web fmw
JOIN dim_campana dc ON dc.campana_sk = fmw.campana_sk
GROUP BY dc.nombre_campana;
```

### 3.3 Atributos degenerados

Un **atributo degenerado** es un atributo que pertenece lógicamente al hecho (no a ninguna dimensión), que no tiene una tabla de dimensión propia porque no hay atributos adicionales que valga la pena almacenar sobre él.

El ejemplo más común es el número de orden de pedido, número de factura, o número de ticket: estos identificadores son importantes para la auditoría y la trazabilidad, pero no tienen atributos propios que valga la pena almacenar en una dimensión separada.

```sql
-- numero_pedido es un atributo degenerado en la tabla de hechos:
CREATE TABLE fact_items_pedido (
    tiempo_sk      INT NOT NULL REFERENCES dim_tiempo,
    producto_sk    INT NOT NULL REFERENCES dim_producto,
    cliente_sk     INT NOT NULL REFERENCES dim_cliente,
    cantidad       INT NOT NULL,
    monto_neto     NUMERIC(12,2) NOT NULL,
    numero_pedido  VARCHAR(20) NOT NULL  -- atributo degenerado: no hay dim_pedido
);
-- Para drill-down a los ítems de un pedido específico:
SELECT * FROM fact_items_pedido WHERE numero_pedido = 'PED-2024-001234';
```

Si se creara una `dim_pedido` con solo el número de pedido y quizás la fecha (que ya está en `dim_tiempo`), sería una dimensión de un solo atributo que no aporta valor. El atributo degenerado es la solución pragmática.

### 3.4 Las claves primarias en las tablas de hechos

Las tablas de hechos pueden o no tener una clave primaria explícita, dependiendo de la granularidad:

**Opción 1: Clave primaria compuesta por las claves de dimensión.** Si la granularidad es "una fila por venta por producto por cliente por día", la combinación `(tiempo_sk, producto_sk, cliente_sk)` podría ser la PK. Pero esto falla si pueden ocurrir múltiples ventas del mismo producto al mismo cliente en el mismo día.

**Opción 2: Surrogate key en la tabla de hechos.** Una `venta_sk BIGSERIAL` que identifica cada fila de forma única, independientemente de la combinación de dimensiones. Es la opción más simple y la más común.

**Opción 3: Sin clave primaria (para tablas de hechos muy grandes).** En algunos data warehouses columnar en la nube, las tablas de hechos de miles de millones de filas no tienen PK explícita para evitar el overhead de la restricción de unicidad durante las cargas.

---

## 4. Tablas de dimensiones en profundidad

### 4.1 La estructura de una tabla de dimensión

Las tablas de dimensiones son **anchas y planas**: pocas filas (relativo a la tabla de hechos) pero muchas columnas. Una tabla de dimensión de producto bien diseñada puede tener 50-100 columnas que describen todos los atributos relevantes del producto, incluyendo los de su jerarquía de categorías desnormalizados.

```sql
CREATE TABLE dim_producto (
    -- Identificadores
    producto_sk          INT PRIMARY KEY,          -- surrogate key del DW
    producto_bk          VARCHAR(50) NOT NULL,      -- business key del sistema fuente
    sku                  VARCHAR(50) NOT NULL,

    -- Atributos del producto
    nombre_producto      TEXT NOT NULL,
    descripcion          TEXT,
    peso_kg              NUMERIC(8,3),
    color                VARCHAR(50),
    talla                VARCHAR(20),
    unidad_medida        VARCHAR(20) NOT NULL,

    -- Jerarquía de categorías (desnormalizada)
    subcategoria_id      INT NOT NULL,
    nombre_subcategoria  TEXT NOT NULL,
    categoria_id         INT NOT NULL,
    nombre_categoria     TEXT NOT NULL,
    linea_id             INT NOT NULL,
    nombre_linea         TEXT NOT NULL,
    departamento_id      INT NOT NULL,
    nombre_departamento  TEXT NOT NULL,

    -- Atributos de precio y costo
    precio_lista         NUMERIC(10,2) NOT NULL,
    costo_estandar       NUMERIC(10,2) NOT NULL,
    margen_objetivo_pct  NUMERIC(5,2),

    -- Atributos de marca y proveedor
    marca                TEXT,
    fabricante           TEXT,
    pais_origen          VARCHAR(50),
    proveedor_principal  TEXT,

    -- Atributos de ciclo de vida
    fecha_lanzamiento    DATE,
    fecha_descontinuacion DATE,
    estado               VARCHAR(20) NOT NULL DEFAULT 'activo',

    -- SCD Tipo 2
    fecha_inicio_vigencia DATE NOT NULL,
    fecha_fin_vigencia    DATE NOT NULL DEFAULT '9999-12-31',
    es_version_actual     BOOLEAN NOT NULL DEFAULT TRUE
);
```

### 4.2 La desnormalización intencional de jerarquías

Una de las decisiones de diseño más importantes en el modelado dimensional es **desnormalizar las jerarquías** dentro de las tablas de dimensiones. En el esquema OLTP, las categorías de productos están en tablas separadas (`categorias`, `subcategorias`, `lineas`, `departamentos`). En la dimensión de producto, toda esa información está en la misma fila.

**¿Por qué esta desnormalización es correcta en el DW?**

En el DW, la normalización serviría para evitar redundancia. Pero:
- La redundancia en dimensiones es mínima en comparación con la tabla de hechos (hay 50,000 productos vs 500M de ventas).
- Eliminar los JOINs dentro de la dimensión simplifica enormemente las queries analíticas.
- Los datos de dimensiones cambian raramente; la anomalía de actualización que justifica la normalización prácticamente no existe.

**Cómo usar la jerarquía desnormalizada:**

```sql
-- Ventas por categoría (nivel alto de la jerarquía)
SELECT dp.nombre_categoria, SUM(fv.monto_neto)
FROM fact_ventas fv
JOIN dim_producto dp ON dp.producto_sk = fv.producto_sk
GROUP BY dp.nombre_categoria;

-- Ventas por subcategoría dentro de una categoría (drill-down)
SELECT dp.nombre_subcategoria, SUM(fv.monto_neto)
FROM fact_ventas fv
JOIN dim_producto dp ON dp.producto_sk = fv.producto_sk
WHERE dp.nombre_categoria = 'Electrónica'
GROUP BY dp.nombre_subcategoria;

-- Drill-down a producto individual dentro de una subcategoría
SELECT dp.nombre_producto, SUM(fv.monto_neto)
FROM fact_ventas fv
JOIN dim_producto dp ON dp.producto_sk = fv.producto_sk
WHERE dp.nombre_subcategoria = 'Smartphones'
GROUP BY dp.nombre_producto
ORDER BY SUM(fv.monto_neto) DESC;
```

Estas tres queries de drill-down son idénticas en estructura: solo cambia el nivel de la jerarquía. Ninguna requiere JOINs adicionales.

### 4.3 Dimensiones de baja cardinalidad vs alta cardinalidad

**Dimensiones de alta cardinalidad** (muchas filas): Clientes (millones), Productos (decenas de miles), Empleados (miles). Son las dimensiones "anchas" que necesitan diseño cuidadoso.

**Dimensiones de baja cardinalidad** (pocas filas): Tiempo (365 filas por año), Estado de pedido (5-10 filas), Canal de venta (10-20 filas). Son dimensiones pequeñas que raramente presentan problemas de rendimiento.

**El caso especial del campo de baja cardinalidad:**

Algunos atributos de baja cardinalidad no merecen su propia tabla de dimensión. "Estado del pedido" con 5 valores puede vivir directamente en la tabla de hechos como atributo degenerado, o puede ser una "dimensión junk" (dimensión basura).

### 4.4 Dimensiones junk (dimensiones basura)

Una **dimensión junk** (el término es de Kimball, no es peyorativo) consolida múltiples flags y atributos de baja cardinalidad que no pertenecen a ninguna otra dimensión en una sola tabla.

**El problema que resuelve:** Si una venta tiene atributos como `tipo_pago` (5 valores), `canal_venta` (4 valores), `es_primera_compra` (TRUE/FALSE), y `tipo_promocion` (10 valores), agregar estos como columnas directamente en la tabla de hechos la hace ancha e ineficiente. Crear una dimensión separada para cada uno (dim_tipo_pago, dim_canal_venta, etc.) llena el esquema estrella de dimensiones minúsculas.

La solución es combinarlos en una dimensión junk:

```sql
CREATE TABLE dim_clasificacion_venta (
    clasif_sk          INT PRIMARY KEY,   -- surrogate key
    tipo_pago          VARCHAR(30) NOT NULL,   -- efectivo, tarjeta, transferencia...
    canal_venta        VARCHAR(30) NOT NULL,   -- online, tienda, telefono...
    es_primera_compra  BOOLEAN NOT NULL,
    tipo_promocion     VARCHAR(30) NOT NULL    -- sin_promo, descuento, 2x1...
);

-- Pre-cargar todas las combinaciones posibles (5 × 4 × 2 × 10 = 400 combinaciones)
-- O insertar bajo demanda al encontrar una combinación nueva
```

La tabla de hechos entonces tiene una sola FK a `dim_clasificacion_venta` en lugar de 4 columnas separadas o 4 FKs.

---

## 5. El esquema estrella

### 5.1 La estructura del esquema estrella

El **esquema estrella** es el resultado de colocar la tabla de hechos en el centro, rodeada de las tablas de dimensiones. El diagrama resultante se parece a una estrella: la tabla de hechos es el núcleo, y las dimensiones son los rayos.

```
                          dim_tiempo
                              │
                              │ tiempo_sk
                              │
dim_vendedor ────── vendedor_sk ──── fact_ventas ──── cliente_sk ──── dim_cliente
                              │
                              │ producto_sk
                              │
                          dim_producto
                              │
                              │ tienda_sk
                              │
                          dim_tienda
```

Cada tabla de dimensión está conectada **directamente** a la tabla de hechos mediante su surrogate key. No hay JOINs intermedios entre dimensiones.

### 5.2 Por qué el esquema estrella tiene tan buen rendimiento

**Un solo JOIN por dimensión:** Para agregar ventas por región, categoría de producto, y trimestre, se necesitan exactamente 3 JOINs: uno a `dim_cliente` (para la región), uno a `dim_producto` (para la categoría), uno a `dim_tiempo` (para el trimestre). Ningún JOIN intermedio.

**Las dimensiones son pequeñas:** `dim_producto` puede tener 50,000 filas; `dim_tiempo` tiene 3,650 filas por 10 años. El hash join de la tabla de hechos (500M filas) con estas dimensiones pequeñas es muy eficiente: la tabla hash del build side (la dimensión) cabe completamente en memoria.

**Queries intuitivas:** El analista solo necesita conocer la estructura básica (hechos en el centro, dimensiones alrededor) para escribir cualquier query analítica. No necesita conocer la jerarquía completa del modelo OLTP.

### 5.3 Un esquema estrella completo para ventas al detalle

```sql
-- ========== TABLAS DE DIMENSIONES ==========

CREATE TABLE dim_tiempo (
    tiempo_sk       INT PRIMARY KEY,
    fecha_completa  DATE NOT NULL UNIQUE,
    anio            SMALLINT NOT NULL,
    trimestre       SMALLINT NOT NULL,       -- 1-4
    mes             SMALLINT NOT NULL,       -- 1-12
    nombre_mes      VARCHAR(20) NOT NULL,    -- 'Enero', 'Febrero'...
    semana_anio     SMALLINT NOT NULL,       -- 1-53
    dia_mes         SMALLINT NOT NULL,       -- 1-31
    dia_semana      SMALLINT NOT NULL,       -- 1=Lunes, 7=Domingo
    nombre_dia      VARCHAR(20) NOT NULL,
    es_fin_semana   BOOLEAN NOT NULL,
    es_feriado      BOOLEAN NOT NULL,
    nombre_feriado  VARCHAR(100)             -- NULL si no es feriado
);

CREATE TABLE dim_producto (
    producto_sk         INT PRIMARY KEY,
    producto_bk         VARCHAR(50) NOT NULL,
    nombre_producto     TEXT NOT NULL,
    sku                 VARCHAR(50) NOT NULL,
    precio_lista        NUMERIC(10,2) NOT NULL,
    costo_estandar      NUMERIC(10,2) NOT NULL,
    -- Jerarquía desnormalizada
    subcategoria_id     INT NOT NULL,
    nombre_subcategoria TEXT NOT NULL,
    categoria_id        INT NOT NULL,
    nombre_categoria    TEXT NOT NULL,
    departamento_id     INT NOT NULL,
    nombre_departamento TEXT NOT NULL,
    marca               TEXT,
    -- SCD Tipo 2
    fecha_inicio  DATE NOT NULL,
    fecha_fin     DATE NOT NULL DEFAULT '9999-12-31',
    es_actual     BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE dim_cliente (
    cliente_sk      INT PRIMARY KEY,
    cliente_bk      INT NOT NULL,
    nombre          TEXT NOT NULL,
    email           TEXT,
    telefono        TEXT,
    -- Jerarquía geográfica desnormalizada
    ciudad          TEXT NOT NULL,
    provincia       TEXT NOT NULL,
    region          TEXT NOT NULL,
    pais            TEXT NOT NULL,
    -- Segmentación
    segmento        TEXT NOT NULL,      -- 'oro', 'plata', 'bronce'
    canal_adq       TEXT NOT NULL,      -- 'online', 'tienda', 'referido'
    fecha_primera_compra DATE,
    -- SCD Tipo 2
    fecha_inicio    DATE NOT NULL,
    fecha_fin       DATE NOT NULL DEFAULT '9999-12-31',
    es_actual       BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE dim_tienda (
    tienda_sk       INT PRIMARY KEY,
    tienda_bk       INT NOT NULL,
    nombre_tienda   TEXT NOT NULL,
    tipo_tienda     TEXT NOT NULL,      -- 'flagship', 'express', 'outlet'
    ciudad          TEXT NOT NULL,
    region          TEXT NOT NULL,
    pais            TEXT NOT NULL,
    metros_cuadrados INT,
    gerente         TEXT
);

CREATE TABLE dim_vendedor (
    vendedor_sk     INT PRIMARY KEY,
    vendedor_bk     INT NOT NULL,
    nombre          TEXT NOT NULL,
    zona            TEXT NOT NULL,
    equipo          TEXT NOT NULL,
    nivel           TEXT NOT NULL,      -- 'junior', 'senior', 'lead'
    -- SCD Tipo 2
    fecha_inicio    DATE NOT NULL,
    fecha_fin       DATE NOT NULL DEFAULT '9999-12-31',
    es_actual       BOOLEAN NOT NULL DEFAULT TRUE
);

-- ========== TABLA DE HECHOS ==========

CREATE TABLE fact_ventas (
    -- Surrogate key de la tabla de hechos
    venta_sk        BIGSERIAL PRIMARY KEY,

    -- Claves de dimensión
    tiempo_sk       INT NOT NULL REFERENCES dim_tiempo(tiempo_sk),
    producto_sk     INT NOT NULL REFERENCES dim_producto(producto_sk),
    cliente_sk      INT NOT NULL REFERENCES dim_cliente(cliente_sk),
    tienda_sk       INT NOT NULL REFERENCES dim_tienda(tienda_sk),
    vendedor_sk     INT NOT NULL REFERENCES dim_vendedor(vendedor_sk),

    -- Hechos aditivos
    cantidad            INT NOT NULL,
    precio_unit_real    NUMERIC(10,2) NOT NULL,  -- precio cobrado (puede diferir del lista)
    descuento_pct       NUMERIC(5,2) NOT NULL DEFAULT 0,
    monto_bruto         NUMERIC(12,2) NOT NULL,  -- cantidad * precio_unit_real
    monto_descuento     NUMERIC(12,2) NOT NULL,  -- monto_bruto * descuento_pct
    monto_neto          NUMERIC(12,2) NOT NULL,  -- monto_bruto - monto_descuento
    costo_total         NUMERIC(12,2) NOT NULL,  -- cantidad * costo_estandar
    margen_bruto        NUMERIC(12,2) NOT NULL,  -- monto_neto - costo_total

    -- Atributo degenerado
    numero_transaccion  VARCHAR(30) NOT NULL
);

-- Índices para las queries más comunes
CREATE INDEX idx_fv_tiempo    ON fact_ventas (tiempo_sk);
CREATE INDEX idx_fv_producto  ON fact_ventas (producto_sk);
CREATE INDEX idx_fv_cliente   ON fact_ventas (cliente_sk);
CREATE INDEX idx_fv_tienda    ON fact_ventas (tienda_sk);
```

---

## 6. El esquema copo de nieve

### 6.1 La diferencia con el esquema estrella

El **esquema copo de nieve** (snowflake schema) es una variante del esquema estrella donde las tablas de dimensiones están parcial o completamente normalizadas. Las jerarquías que en el esquema estrella están desnormalizadas dentro de la dimensión, en el copo de nieve se separan en tablas adicionales.

```
-- Esquema estrella: una sola tabla dim_producto con todo desnormalizado
fact_ventas → dim_producto (incluye subcategoría, categoría, departamento)

-- Esquema copo de nieve: las jerarquías se separan
fact_ventas → dim_producto → dim_subcategoria → dim_categoria → dim_departamento
```

```sql
-- dim_producto normalizada (copo de nieve)
CREATE TABLE dim_producto (
    producto_sk      INT PRIMARY KEY,
    nombre_producto  TEXT NOT NULL,
    sku              VARCHAR(50) NOT NULL,
    subcategoria_sk  INT NOT NULL REFERENCES dim_subcategoria(subcategoria_sk)
    -- Los atributos de categoría y departamento NO están aquí
);

CREATE TABLE dim_subcategoria (
    subcategoria_sk  INT PRIMARY KEY,
    nombre           TEXT NOT NULL,
    categoria_sk     INT NOT NULL REFERENCES dim_categoria(categoria_sk)
);

CREATE TABLE dim_categoria (
    categoria_sk  INT PRIMARY KEY,
    nombre        TEXT NOT NULL,
    departamento_sk INT NOT NULL REFERENCES dim_departamento(departamento_sk)
);

CREATE TABLE dim_departamento (
    departamento_sk  INT PRIMARY KEY,
    nombre           TEXT NOT NULL
);
```

### 6.2 Cuándo el copo de nieve vale la pena

El copo de nieve es una compensación entre tamaño del DW y complejidad de las queries:

**A favor del copo de nieve:**
- Reduce la redundancia en las tablas de dimensiones: si hay 50,000 productos en 10 categorías, el nombre de la categoría se repite 5,000 veces en la estrella pero solo 10 veces en el copo de nieve.
- Facilita la actualización de atributos de niveles superiores de la jerarquía (cambiar el nombre de una categoría es un UPDATE de una sola fila en lugar de miles).

**En contra del copo de nieve:**
- Cada nivel de la jerarquía agrega un JOIN adicional a cada query analítica.
- Hace las queries más complejas y difíciles de escribir para los analistas.
- El ahorro de espacio en las dimensiones es insignificante comparado con el tamaño de la tabla de hechos (las dimensiones son el 1-5% del tamaño total del DW).
- Los motores columnares modernos comprimen bien la redundancia de las dimensiones.

**La posición de Kimball:** El esquema estrella es casi siempre preferible al copo de nieve para el modelo dimensional del data warehouse. El copo de nieve es aceptable para dimensiones extremadamente grandes o cuando los atributos de nivel alto de la jerarquía cambian frecuentemente.

**La posición pragmática:** En la práctica, muchos DW usan una mezcla: estrella para las dimensiones que se consultan frecuentemente, copo de nieve para jerarquías muy profundas o grandes donde la redundancia es genuinamente costosa.

---

## 7. La granularidad: la decisión más importante

### 7.1 Qué es la granularidad

La **granularidad** de una tabla de hechos define exactamente qué representa una fila. Es la decisión más importante en el diseño del modelo dimensional porque determina qué preguntas se pueden responder con ese modelo.

Ejemplos de distintos niveles de granularidad para ventas:
- **Línea de factura:** Una fila por cada producto en cada transacción. La granularidad más fina disponible.
- **Transacción:** Una fila por cada transacción completa (sin detalle por producto).
- **Resumen diario por producto:** Una fila por cada producto por cada día.
- **Resumen mensual por categoría:** Una fila por cada categoría por cada mes.

### 7.2 La regla de la granularidad más fina

Kimball tiene una regla clara:

> **Siempre diseñar la tabla de hechos con la granularidad más fina disponible.**

La razón: una tabla de hechos de granularidad fina puede responder todas las preguntas que respondería una tabla de granularidad gruesa, más todas las preguntas adicionales que solo la granularidad fina permite. El análisis puede agregar desde lo fino hacia lo grueso, pero nunca puede desagregar de lo grueso hacia lo fino.

```
-- Con granularidad de LÍNEA DE FACTURA puedo calcular:
-- ✓ Ventas totales del mes (agrupar por tiempo)
-- ✓ Ventas por categoría (agrupar por dim_producto.nombre_categoria)
-- ✓ Ticket promedio por transacción (agregar por numero_transaccion, luego promediar)
-- ✓ Los 10 productos más vendidos del trimestre
-- ✓ Ventas de un producto específico en un día específico

-- Con granularidad de RESUMEN DIARIO POR PRODUCTO solo puedo calcular:
-- ✓ Ventas totales del mes
-- ✓ Ventas por categoría
-- ✗ Ticket promedio por transacción (la información de transacción individual se perdió)
-- ✓ Los 10 productos más vendidos del trimestre
-- ✓ Ventas de un producto en un día específico
```

El costo de la granularidad fina es el volumen: más filas, más espacio, queries más lentas. Para DW modernos en la nube con almacenamiento columnar y precios por byte procesado, este costo es cada vez más manejable.

### 7.3 El proceso de declarar la granularidad

El proceso formal de Kimball para declarar la granularidad de una tabla de hechos:

1. **Identificar el proceso de negocio** que se está modelando (la venta, la llamada al call center, la visita web).
2. **Declarar la granularidad** explícitamente: "Una fila representa UN ítem de línea de UNA transacción de venta en UNA tienda específica".
3. **Seleccionar las dimensiones** que aplican a esa granularidad: tiempo, producto, cliente, tienda, vendedor. Una dimensión aplica si tiene exactamente un valor para cada fila de la tabla de hechos.
4. **Identificar los hechos** numéricos que se pueden medir para cada fila a esa granularidad.

**El test de la dimensión:** ¿Tiene la dimensión X exactamente un valor para cada fila de la tabla de hechos? Si sí, es una dimensión válida para esa tabla. Si no (p.ej., una venta puede tener múltiples categorías de producto), la granularidad es incorrecta o se necesita una tabla de hechos más fina.

---

## 8. Slowly Changing Dimensions en contexto dimensional

### 8.1 El problema en el data warehouse

En el Módulo 6 introdujimos las Slowly Changing Dimensions (SCD) como técnicas para manejar cambios temporales en los datos. En el contexto del data warehouse dimensional, el problema es específico:

Cuando un cliente cambia de segmento (de 'plata' a 'oro'), ¿las ventas históricas que ese cliente hizo cuando era 'plata' deben seguir siendo reportadas como 'plata'? ¿O deben ahora reportarse como 'oro'?

La respuesta depende del requerimiento de análisis:
- "¿Cuánto vendimos al segmento 'oro'?" puede querer decir "¿cuánto vendimos a clientes que son ORO HOY?" (SCD Tipo 1) o "¿cuánto vendimos a clientes que eran ORO CUANDO SE REALIZÓ LA VENTA?" (SCD Tipo 2).

Kimball recomienda casi siempre usar **SCD Tipo 2** para preservar la coherencia histórica de los reportes.

### 8.2 SCD Tipo 1 en el DW: sobrescribir

```sql
-- ETL: actualizar la dimensión con el nuevo valor (SCD Tipo 1)
UPDATE dim_cliente
SET segmento = 'oro'
WHERE cliente_bk = 42;
-- Todas las ventas históricas ahora aparecen como "ventas al segmento oro"
-- La historia se reescribe
```

Apropiado cuando el cambio es una **corrección de un error** (el segmento siempre fue oro, se registró mal) o cuando el análisis histórico con el valor anterior no tiene valor de negocio.

### 8.3 SCD Tipo 2 en el DW: versiones históricas

```sql
-- ETL: crear nueva versión del cliente (SCD Tipo 2)

-- Paso 1: expirar la versión actual
UPDATE dim_cliente
SET fecha_fin = '2024-09-30', es_actual = FALSE
WHERE cliente_bk = 42 AND es_actual = TRUE;

-- Paso 2: insertar la nueva versión con la nueva surrogate key
INSERT INTO dim_cliente (
    cliente_sk, cliente_bk, nombre, email,
    ciudad, region, segmento,
    fecha_inicio, fecha_fin, es_actual
)
VALUES (
    nextval('dim_cliente_sk_seq'), 42, 'Ana García', 'ana@email.com',
    'Cochabamba', 'Centro', 'oro',  -- nuevo segmento
    '2024-10-01', '9999-12-31', TRUE
);
```

Ahora la tabla de hechos tiene dos surrogate keys para el cliente 42: la antigua (versión 'plata') y la nueva (versión 'oro'). Las ventas realizadas cuando era 'plata' apuntan a la surrogate key antigua; las nuevas ventas apuntarán a la nueva.

```sql
-- ¿Cuánto vendimos por segmento, preservando el segmento en el momento de la venta?
SELECT dc.segmento, SUM(fv.monto_neto)
FROM fact_ventas fv
JOIN dim_cliente dc ON dc.cliente_sk = fv.cliente_sk  -- JOIN por surrogate key
GROUP BY dc.segmento;
-- Las ventas de Ana cuando era 'plata' aparecen en 'plata'
-- Las ventas post-cambio aparecen en 'oro'
-- La historia es coherente
```

### 8.4 SCD Tipo 3: columna de valor anterior

```sql
-- En la dimensión, agregar la columna del valor anterior
ALTER TABLE dim_cliente
ADD COLUMN segmento_anterior TEXT,
ADD COLUMN fecha_cambio_segmento DATE;

-- Al actualizar:
UPDATE dim_cliente
SET segmento_anterior = segmento,
    segmento = 'oro',
    fecha_cambio_segmento = '2024-10-01'
WHERE cliente_bk = 42 AND es_actual = TRUE;
```

Útil para análisis del tipo "¿de qué segmento vinieron los clientes que subieron a oro?", pero solo mantiene un nivel de historia.

### 8.5 SCD Tipo 6 en la práctica del DW

Kimball popularizó el tipo 6 (combinación de 1+2+3) para casos donde se necesitan múltiples perspectivas simultáneamente:

```sql
CREATE TABLE dim_cliente (
    cliente_sk          INT PRIMARY KEY,
    cliente_bk          INT NOT NULL,
    nombre              TEXT NOT NULL,
    -- Atributo de la versión histórica (cambia con cada versión Tipo 2)
    segmento_historico  TEXT NOT NULL,
    -- Atributo actual (se actualiza en TODAS las versiones, Tipo 1)
    segmento_actual     TEXT NOT NULL,
    -- Atributo anterior (Tipo 3)
    segmento_anterior   TEXT,
    -- Fechas de vigencia (Tipo 2)
    fecha_inicio        DATE NOT NULL,
    fecha_fin           DATE NOT NULL DEFAULT '9999-12-31',
    es_actual           BOOLEAN NOT NULL DEFAULT TRUE
);
```

Con este diseño:
- `segmento_historico` responde "¿en qué segmento estaba el cliente cuando se hizo la venta?"
- `segmento_actual` (disponible en cualquier versión) responde "¿en qué segmento está el cliente hoy?"
- `segmento_anterior` responde "¿de qué segmento vino?"

---

## 9. Dimensiones conformadas

### 9.1 El problema de la integración entre data marts

Una empresa típica tiene múltiples procesos de negocio que quiere analizar: ventas, inventario, compras, recursos humanos. Kimball propone construir un data mart separado para cada proceso. Pero surge un problema: ¿cómo comparar y combinar datos de distintos data marts?

Si el data mart de ventas tiene su propia `dim_cliente` y el data mart de soporte tiene su propia `dim_cliente`, y no son exactamente iguales (mismas claves, mismos atributos, misma definición de "segmento"), no se pueden combinar en una sola query.

### 9.2 Dimensiones conformadas: la solución

Una **dimensión conformada** (conformed dimension) es una tabla de dimensión que se comparte entre múltiples tablas de hechos y/o data marts. Tiene exactamente las mismas columnas, las mismas surrogate keys, y las mismas definiciones de atributos en todos los contextos donde se usa.

```
Data Mart Ventas:
fact_ventas → dim_cliente (compartida)
                         ↑
Data Mart Soporte:       │ misma tabla, mismos datos
fact_incidencias ────────┘

-- Query que cruza ventas e incidencias:
SELECT dc.segmento,
       SUM(fv.monto_neto) AS total_ventas,
       COUNT(fi.incidencia_id) AS total_incidencias,
       SUM(fv.monto_neto) / NULLIF(COUNT(fi.incidencia_id), 0) AS ventas_por_incidencia
FROM dim_cliente dc
LEFT JOIN fact_ventas fv ON fv.cliente_sk = dc.cliente_sk AND dc.es_actual
LEFT JOIN fact_incidencias fi ON fi.cliente_sk = dc.cliente_sk AND dc.es_actual
GROUP BY dc.segmento;
-- Esta query es posible porque dim_cliente es conformada entre ambos data marts
```

### 9.3 La bus matrix de Kimball

Para planificar las dimensiones conformadas, Kimball usa la **enterprise data warehouse bus matrix**: una grilla donde las filas son los procesos de negocio (tablas de hechos) y las columnas son las dimensiones. Un punto de intersección indica que ese proceso usa esa dimensión.

| Proceso de negocio | dim_tiempo | dim_producto | dim_cliente | dim_tienda | dim_vendedor | dim_proveedor |
|---|---|---|---|---|---|---|
| fact_ventas | ✓ | ✓ | ✓ | ✓ | ✓ | |
| fact_inventario | ✓ | ✓ | | ✓ | | ✓ |
| fact_compras | ✓ | ✓ | | | | ✓ |
| fact_incidencias | ✓ | ✓ | ✓ | ✓ | ✓ | |

Las dimensiones marcadas en múltiples filas (dim_tiempo, dim_producto, dim_cliente) son candidatas a ser conformadas. Si el negocio quiere poder cruzar datos de ventas con inventario (p.ej., "ventas vs stock por producto por tienda"), dim_producto y dim_tienda deben ser exactamente las mismas tablas en ambos data marts.

---

## 10. Tablas de hechos sin hechos

### 10.1 El concepto

Una **tabla de hechos sin hechos** (factless fact table) es una tabla de hechos que no tiene mediciones numéricas. Parece una contradicción, pero tiene sentido: registra el hecho de que cierto evento ocurrió, o de que cierta relación existía en un momento dado, sin ninguna medición asociada.

### 10.2 Tipo 1: registrar eventos sin medición

Algunos eventos importantes del negocio no tienen una medición natural. Por ejemplo, los estudiantes que asistieron a clases:

```sql
CREATE TABLE fact_asistencia (
    tiempo_sk    INT NOT NULL REFERENCES dim_tiempo,
    estudiante_sk INT NOT NULL REFERENCES dim_estudiante,
    clase_sk     INT NOT NULL REFERENCES dim_clase,
    profesor_sk  INT NOT NULL REFERENCES dim_profesor
    -- Sin hechos: el hecho ES la asistencia
);

-- ¿Cuántos estudiantes asistieron a cada clase?
SELECT dc.nombre_clase, COUNT(*) AS asistentes
FROM fact_asistencia fa
JOIN dim_clase dc ON dc.clase_sk = fa.clase_sk
WHERE fa.tiempo_sk = 20241015
GROUP BY dc.nombre_clase;
```

Otros ejemplos: clicks en una página (el hecho es el click, no hay una medición), visualizaciones de una publicidad (el hecho es la visualización), el estudiante se inscribió en un curso.

### 10.3 Tipo 2: cobertura y elegibilidad

El segundo uso clásico es registrar "cobertura": qué combinaciones de dimensiones eran elegibles o estaban disponibles, para poder calcular tasas de conversión.

```
Ejemplo: Campaña de marketing
- fact_productos_en_promocion: qué productos estaban en promoción cada día
- fact_ventas: qué productos se vendieron

Sin fact_productos_en_promocion, no se puede responder:
"¿Qué porcentaje de los productos en promoción se vendieron?"
```

```sql
-- Registrar qué productos estaban en promoción cada día
CREATE TABLE fact_productos_en_promocion (
    tiempo_sk    INT NOT NULL REFERENCES dim_tiempo,
    producto_sk  INT NOT NULL REFERENCES dim_producto,
    promocion_sk INT NOT NULL REFERENCES dim_promocion
    -- Sin hechos numéricos
);

-- Tasa de conversión de la promoción:
SELECT
    dp.nombre_promocion,
    COUNT(DISTINCT fpp.producto_sk) AS productos_en_promo,
    COUNT(DISTINCT fv.producto_sk) AS productos_vendidos,
    100.0 * COUNT(DISTINCT fv.producto_sk) /
        NULLIF(COUNT(DISTINCT fpp.producto_sk), 0) AS pct_conversion
FROM fact_productos_en_promocion fpp
JOIN dim_promocion dp ON dp.promocion_sk = fpp.promocion_sk
LEFT JOIN fact_ventas fv ON fv.producto_sk = fpp.producto_sk
    AND fv.tiempo_sk = fpp.tiempo_sk
WHERE fpp.tiempo_sk IN (SELECT tiempo_sk FROM dim_tiempo WHERE mes = 12 AND anio = 2024)
GROUP BY dp.nombre_promocion;
```

---

## 11. Tipos especiales de tablas de hechos

### 11.1 Tablas de hechos de transacción (Transaction Fact Tables)

Son las más comunes. Cada fila representa un evento discreto que ocurrió en un momento específico: una venta, un click, una llamada, un pago. La granularidad es el evento individual.

**Característica:** La fila se inserta una vez y nunca se actualiza.

### 11.2 Tablas de hechos de snapshot periódico (Periodic Snapshot Fact Tables)

Registran el estado de algo a intervalos regulares de tiempo. En lugar de eventos discretos, capturan "cómo estaban las cosas" al final de cada período.

```sql
-- Snapshot del inventario al final de cada día
CREATE TABLE fact_inventario_diario (
    tiempo_sk        INT NOT NULL REFERENCES dim_tiempo,
    producto_sk      INT NOT NULL REFERENCES dim_producto,
    tienda_sk        INT NOT NULL REFERENCES dim_tienda,
    
    -- Hechos del snapshot (semi-aditivos: no sumar a través del tiempo)
    unidades_mano    INT NOT NULL,
    unidades_pedidas INT NOT NULL,
    unidades_vendidas_dia INT NOT NULL,  -- este sí es aditivo (es del período, no acumulado)
    costo_inventario NUMERIC(14,2) NOT NULL
);

-- ¿Cuál es el inventario actual por tienda?
SELECT dt.nombre_tienda, SUM(fi.unidades_mano) AS total_unidades
FROM fact_inventario_diario fi
JOIN dim_tienda dt ON dt.tienda_sk = fi.tienda_sk
WHERE fi.tiempo_sk = (SELECT MAX(tiempo_sk) FROM fact_inventario_diario)
GROUP BY dt.nombre_tienda;
```

### 11.3 Tablas de hechos de snapshot acumulado (Accumulating Snapshot Fact Tables)

Diseñadas para procesos con un ciclo de vida definido y múltiples hitos: el procesamiento de un pedido, el ciclo de una solicitud de crédito, el pipeline de ventas.

Una fila representa **todo el ciclo de vida** de una instancia del proceso. La fila se **actualiza** cada vez que el proceso avanza a un nuevo hito.

```sql
CREATE TABLE fact_ciclo_pedido (
    pedido_sk              INT PRIMARY KEY,  -- una fila por pedido

    -- Dimensiones de contexto
    cliente_sk             INT NOT NULL REFERENCES dim_cliente,
    producto_sk            INT NOT NULL REFERENCES dim_producto,

    -- Múltiples dimensiones de tiempo: una por cada hito
    tiempo_creacion_sk     INT REFERENCES dim_tiempo,   -- cuándo se creó el pedido
    tiempo_pago_sk         INT REFERENCES dim_tiempo,   -- cuándo se pagó
    tiempo_picking_sk      INT REFERENCES dim_tiempo,   -- cuándo se empacó
    tiempo_envio_sk        INT REFERENCES dim_tiempo,   -- cuándo se envió
    tiempo_entrega_sk      INT REFERENCES dim_tiempo,   -- cuándo llegó al cliente

    -- Hechos
    monto_pedido           NUMERIC(12,2) NOT NULL,
    -- Lag metrics: días entre hitos (útiles para análisis de eficiencia)
    dias_creacion_a_pago   INT,     -- tiempo_pago - tiempo_creacion
    dias_pago_a_envio      INT,     -- tiempo_envio - tiempo_pago
    dias_envio_a_entrega   INT      -- tiempo_entrega - tiempo_envio
);
```

Esta tabla de hechos se actualiza a medida que el pedido avanza: cuando se paga, se llena `tiempo_pago_sk` y se calcula `dias_creacion_a_pago`. Cuando se envía, se llena `tiempo_envio_sk`, etc.

```sql
-- Análisis de eficiencia del ciclo de pedidos
SELECT AVG(dias_creacion_a_pago)  AS promedio_dias_pago,
       AVG(dias_pago_a_envio)     AS promedio_dias_envio,
       AVG(dias_envio_a_entrega)  AS promedio_dias_entrega,
       AVG(dias_creacion_a_pago + dias_pago_a_envio + dias_envio_a_entrega) AS ciclo_total
FROM fact_ciclo_pedido
WHERE tiempo_creacion_sk IN (
    SELECT tiempo_sk FROM dim_tiempo WHERE anio = 2024 AND trimestre = 4
)
AND tiempo_entrega_sk IS NOT NULL;  -- pedidos completados
```

---

## 12. La dimensión de tiempo

### 12.1 Por qué la dimensión de tiempo es especial

La dimensión de tiempo es la única dimensión que está presente en **prácticamente todas** las tablas de hechos y que puede calcularse completamente sin datos del sistema fuente. Es la primera tabla que se carga al construir un DW y rara vez necesita ser actualizada.

### 12.2 Diseño de la dimensión de tiempo

```sql
CREATE TABLE dim_tiempo (
    tiempo_sk        INT PRIMARY KEY,       -- e.g., 20240101 para 2024-01-01
    fecha_completa   DATE NOT NULL UNIQUE,
    
    -- Año
    anio             SMALLINT NOT NULL,     -- 2024
    
    -- Trimestre
    trimestre        SMALLINT NOT NULL,     -- 1, 2, 3, 4
    nombre_trimestre VARCHAR(10) NOT NULL,  -- 'T1 2024', 'Q1 2024'
    
    -- Mes
    mes              SMALLINT NOT NULL,     -- 1-12
    nombre_mes       VARCHAR(20) NOT NULL,  -- 'Enero'
    mes_anio         VARCHAR(10) NOT NULL,  -- '2024-01' (para ordenar)
    
    -- Semana
    semana_anio      SMALLINT NOT NULL,     -- 1-53 (ISO)
    inicio_semana    DATE NOT NULL,
    fin_semana       DATE NOT NULL,
    
    -- Día
    dia_mes          SMALLINT NOT NULL,     -- 1-31
    dia_anio         SMALLINT NOT NULL,     -- 1-366
    dia_semana       SMALLINT NOT NULL,     -- 1=Lunes, 7=Domingo (ISO)
    nombre_dia       VARCHAR(20) NOT NULL,  -- 'Lunes', 'Martes'
    
    -- Indicadores
    es_fin_semana    BOOLEAN NOT NULL,
    es_feriado       BOOLEAN NOT NULL,
    nombre_feriado   VARCHAR(100),          -- NULL si no es feriado
    es_dia_laboral   BOOLEAN NOT NULL,      -- !es_fin_semana AND !es_feriado
    
    -- Períodos fiscales (si el año fiscal difiere del calendario)
    mes_fiscal       SMALLINT,
    trimestre_fiscal SMALLINT,
    anio_fiscal      SMALLINT
);
```

### 12.3 Poblar la dimensión de tiempo

La dimensión de tiempo se genera con un script, no se extrae de ninguna fuente:

```sql
-- Poblar dim_tiempo con todas las fechas de un rango
INSERT INTO dim_tiempo (
    tiempo_sk, fecha_completa, anio, trimestre, nombre_trimestre,
    mes, nombre_mes, mes_anio,
    semana_anio, inicio_semana, fin_semana,
    dia_mes, dia_anio, dia_semana, nombre_dia,
    es_fin_semana, es_feriado, es_dia_laboral
)
SELECT
    TO_CHAR(fecha, 'YYYYMMDD')::INT AS tiempo_sk,
    fecha AS fecha_completa,
    EXTRACT(YEAR FROM fecha)::SMALLINT AS anio,
    EXTRACT(QUARTER FROM fecha)::SMALLINT AS trimestre,
    'T' || EXTRACT(QUARTER FROM fecha) || ' ' || EXTRACT(YEAR FROM fecha) AS nombre_trimestre,
    EXTRACT(MONTH FROM fecha)::SMALLINT AS mes,
    TO_CHAR(fecha, 'TMMonth') AS nombre_mes,
    TO_CHAR(fecha, 'YYYY-MM') AS mes_anio,
    EXTRACT(WEEK FROM fecha)::SMALLINT AS semana_anio,
    DATE_TRUNC('week', fecha)::DATE AS inicio_semana,
    (DATE_TRUNC('week', fecha) + INTERVAL '6 days')::DATE AS fin_semana,
    EXTRACT(DAY FROM fecha)::SMALLINT AS dia_mes,
    EXTRACT(DOY FROM fecha)::SMALLINT AS dia_anio,
    EXTRACT(ISODOW FROM fecha)::SMALLINT AS dia_semana,
    TO_CHAR(fecha, 'TMDay') AS nombre_dia,
    EXTRACT(ISODOW FROM fecha) IN (6, 7) AS es_fin_semana,
    FALSE AS es_feriado,    -- Se actualizará con los feriados reales
    EXTRACT(ISODOW FROM fecha) NOT IN (6, 7) AS es_dia_laboral
FROM generate_series('2020-01-01'::date, '2030-12-31'::date, '1 day'::interval) AS t(fecha);

-- Luego, actualizar los feriados de forma separada
UPDATE dim_tiempo SET es_feriado = TRUE, nombre_feriado = 'Año Nuevo', es_dia_laboral = FALSE
WHERE fecha_completa = '2024-01-01';
-- (repetir para cada feriado)
```

### 12.4 La clave del tiempo como entero

La convención de usar `YYYYMMDD` como entero para la surrogate key tiene varias ventajas:
- La clave es legible: `20241225` es obviamente el 25 de diciembre de 2024.
- Se puede filtrar directamente por año o mes: `WHERE tiempo_sk BETWEEN 20240101 AND 20241231`.
- Es más eficiente que buscar por fecha: la clave es el valor de negocio directamente.

---

## 13. El proceso de diseño dimensional de Kimball

### 13.1 Los cuatro pasos de Kimball

Kimball definió un proceso de cuatro pasos para diseñar un modelo dimensional:

**Paso 1: Seleccionar el proceso de negocio.**

Un proceso de negocio es una actividad operativa que la organización realiza: registrar ventas, gestionar inventario, procesar pedidos, atender llamadas. Cada proceso se modela con su propia tabla de hechos.

*Criterio:* Empezar por el proceso que genera más valor analítico para el negocio, o el que tiene más urgencia. Típicamente es el proceso de ventas o de operaciones core.

**Paso 2: Declarar la granularidad.**

Definir exactamente qué representa una fila en la tabla de hechos. Esta declaración debe ser lo más específica posible y debe ser acordada con los usuarios de negocio.

*Ejemplo:* "Una fila representa una línea de ítem de una transacción de venta en una tienda física o virtual, incluyendo el producto, el cliente, el vendedor, y la fecha de la transacción."

**Paso 3: Identificar las dimensiones.**

Listar todos los descriptores que aplican a cada fila de la tabla de hechos. Para cada descriptor, preguntarse si tiene exactamente un valor por cada hecho a la granularidad declarada.

*Prueba:* "¿Tiene [dimensión] exactamente un valor por cada [hecho a la granularidad declarada]?" Si sí, es una dimensión válida.

**Paso 4: Identificar los hechos.**

Listar todas las mediciones numéricas que se pueden capturar para cada fila a la granularidad declarada. Para cada medición, clasificarla como aditiva, semi-aditiva, o no aditiva.

### 13.2 Las decisiones de diseño difíciles

**¿Qué atributos van en la dimensión vs en la tabla de hechos?**

Regla general: los atributos descriptivos/textuales van en las dimensiones; las mediciones numéricas van en los hechos. Pero hay casos ambiguos: ¿el precio unitario de lista (que podría ser consultado como descripción del producto) va en `dim_producto` o como hecho en `fact_ventas`?

La respuesta correcta en este caso: **ambos lugares**. `dim_producto` tiene el precio de lista vigente en ese momento (SCD Tipo 2). `fact_ventas` tiene el precio unitario real al que se vendió (que puede diferir del de lista por descuentos o excepciones).

**¿Cuándo crear una nueva tabla de hechos vs agregar a una existente?**

Si un nuevo proceso de negocio comparte dimensiones con uno existente pero tiene una granularidad diferente o mide cosas distintas, se necesita una nueva tabla de hechos. No se deben mezclar hechos de granularidades distintas en la misma tabla.

**¿Cómo modelar muchos-a-muchos entre hechos y dimensiones?**

Cuando un hecho puede relacionarse con múltiples valores de una dimensión (p.ej., un pedido puede tener múltiples métodos de pago), hay que elegir entre:
- Crear columnas separadas (si son pocos y fijos).
- Crear una dimensión de puente (bridge table).
- Cambiar la granularidad de la tabla de hechos.

---

## 14. Resumen y conexión con el resto del curso

### Los conceptos clave

| Concepto | Aplicación práctica |
|---|---|
| **Tabla de hechos** | Centro del esquema; contiene mediciones numéricas del proceso de negocio |
| **Tabla de dimensión** | Contexto de los hechos; amplia, plana, desnormalizada |
| **Surrogate key** | Identificador del DW; desvincula del sistema fuente, soporta SCD |
| **Granularidad** | La decisión más importante; siempre la más fina disponible |
| **Esquema estrella** | El diseño estándar; join directo de hechos a dimensiones |
| **Dimensión conformada** | Permite análisis cruzado entre data marts; base de la arquitectura bus |
| **SCD Tipo 2** | Preserva la coherencia histórica de los reportes |
| **Tabla de hechos sin hechos** | Para registrar eventos sin medición o cobertura |
| **Snapshot acumulado** | Para procesos con ciclo de vida y múltiples hitos |
| **Bus matrix** | Mapa de qué dimensiones usa cada proceso; identifica dimensiones conformadas |

### La meta-lección del módulo

El modelo dimensional de Kimball no es solo una técnica de base de datos. Es un lenguaje de comunicación entre el equipo técnico y los usuarios de negocio. Cuando los analistas pueden escribir queries intuitivas sobre hechos y dimensiones sin necesitar conocer la complejidad del OLTP subyacente, el DW cumple su propósito.

La disciplina de Kimball es que todo empieza con el negocio, no con la tecnología: primero entender qué proceso se quiere analizar, qué granularidad tienen los eventos, qué contexto describe cada evento. La tecnología (el esquema SQL, el motor columnar, el ETL) viene después.

### Conexión con módulos futuros

- **Módulo 12 (Modelos alternativos):** El modelo dimensional asume datos estructurados y bien definidos. Cuando los datos son semi-estructurados (JSON), en grafos (redes sociales), o en series de tiempo (métricas de monitoreo), otros modelos de almacenamiento son más apropiados.

- **Módulo 16 (El oficio):** El proceso de diseño dimensional de Kimball se aplica en conversaciones reales con usuarios de negocio. La habilidad de extraer hechos y dimensiones de una descripción imprecisa del negocio ("necesitamos saber cómo van las ventas") es una de las habilidades más valiosas del trabajo real.

---

## 15. Ejercicios de comprensión

**Ejercicio 1.** Una empresa de telecomunicaciones quiere analizar el uso de sus servicios. Los eventos que quiere analizar son las llamadas telefónicas. Para cada llamada se tiene: número de origen, número de destino, fecha y hora de inicio, duración en segundos, tipo de llamada (nacional, internacional, local), resultado (completada, fallida, buzón), y costo.

a) Aplica los cuatro pasos de Kimball: proceso, granularidad, dimensiones, hechos.
b) Diseña el esquema estrella completo con DDL.
c) Clasifica cada hecho como aditivo, semi-aditivo, o no aditivo. Justifica.
d) ¿Cuáles dimensiones serían conformadas si la empresa también quisiera analizar el consumo de datos de internet?

---

**Ejercicio 2.** Una clínica médica quiere construir un data warehouse para analizar la atención a pacientes. Las consultas médicas son el proceso central. Los datos disponibles son: paciente (edad, género, ciudad, diagnósticos previos), médico (especialidad, años de experiencia, consultorio), fecha y hora de la consulta, tipo de consulta (primera vez, seguimiento, urgencia), diagnóstico, tratamiento indicado, duración de la consulta en minutos, y si derivó a especialista.

a) ¿Cuál es la granularidad apropiada? Justifica.
b) ¿Es "diagnóstico" un hecho o una dimensión? ¿Por qué?
c) Diseña el esquema estrella completo.
d) ¿Necesitarías una tabla de hechos sin hechos para algún aspecto de este dominio? ¿Cuál?

---

**Ejercicio 3.** El siguiente esquema estrella tiene varios problemas de diseño. Identifícalos todos y propón el diseño correcto:

```sql
CREATE TABLE fact_pedidos (
    pedido_id      INT PRIMARY KEY,     -- ← ¿está bien usar el ID del sistema fuente?
    cliente_id     INT,                 -- ← ¿qué está mal aquí?
    producto_id    INT,                 -- ← ¿y aquí?
    fecha          DATE,                -- ← ¿esto debería ser una FK a dim_tiempo?
    monto          NUMERIC(12,2),
    cantidad       INT,
    precio_prom    NUMERIC(10,2),       -- ← ¿qué tipo de hecho es este?
    nombre_cliente TEXT,                -- ← ¿debería estar en fact?
    nombre_producto TEXT,               -- ← ¿debería estar en fact?
    ciudad_cliente TEXT                 -- ← ¿debería estar en fact?
);
```

---

**Ejercicio 4.** Una empresa de retail tiene los siguientes data marts ya construidos:

- **Data mart de ventas:** fact_ventas con dim_tiempo, dim_producto, dim_cliente, dim_tienda.
- **Data mart de inventario:** fact_inventario con dim_tiempo, dim_producto, dim_tienda.

El negocio quiere responder: "¿Qué productos se agotan más rápido en relación a sus ventas?"

a) ¿Qué dimensiones deben ser conformadas para responder esta pregunta?
b) ¿Qué implica "conformada" en términos concretos del diseño?
c) Escribe la query SQL que responde la pregunta del negocio, asumiendo las dimensiones conformadas correctamente.

---

**Ejercicio 5.** Una empresa de seguros quiere modelar el ciclo de vida de una reclamación (siniestro). El proceso tiene los siguientes hitos: recepción de la reclamación, asignación a un ajustador, inspección del siniestro, aprobación/rechazo, y pago (si fue aprobada).

a) ¿Qué tipo de tabla de hechos es la más apropiada para este caso? Justifica.
b) Diseña la tabla de hechos completa, incluyendo las dimensiones de tiempo para cada hito, los hechos numéricos, y los lag metrics relevantes.
c) ¿Qué columnas quedarán NULL para reclamaciones que no han llegado a todos los hitos? ¿Es esto un problema de diseño?
d) Escribe una query que muestre el tiempo promedio entre cada hito del proceso para reclamaciones del último año.

---

*Próximo módulo: Modelos de datos alternativos — donde exploraremos los casos en que el modelo relacional, por más bien diseñado que esté, no es la solución más natural: grafos para redes de relaciones, documentos para datos heterogéneos, series de tiempo para métricas, y cuándo la elección del modelo de datos importa tanto como el diseño del esquema.*
