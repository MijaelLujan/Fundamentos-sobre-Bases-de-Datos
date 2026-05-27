# Módulo 12 — Modelos de datos alternativos

> *"El modelo relacional puede expresar cualquier dominio. Eso no significa que siempre deba hacerlo. Un árbol de decisión expresado como filas en una tabla de adyacencia funciona correctamente; pero un motor de grafos lo traversa en un orden de magnitud menos de tiempo. Un perfil de producto con 200 atributos opcionales expresado en 3FN requiere 200 columnas nullable o un esquema EAV; en un documento JSON simplemente existe. La pregunta no es si el relacional puede representar el dato, sino si es la herramienta más adecuada para ese dominio específico."*

---

## Tabla de contenidos

1. [Por qué existen los modelos alternativos](#1-por-qué-existen-los-modelos-alternativos)
2. [Modelo de documentos](#2-modelo-de-documentos)
3. [Modelo de grafos](#3-modelo-de-grafos)
4. [Modelo columnar](#4-modelo-columnar)
5. [Series de tiempo](#5-series-de-tiempo)
6. [RDF y triplestores](#6-rdf-y-triplestores)
7. [Bases de datos vectoriales](#7-bases-de-datos-vectoriales)
8. [El modelo relacional como base de comparación](#8-el-modelo-relacional-como-base-de-comparación)
9. [Políglota persistence: usar múltiples modelos](#9-políglota-persistence-usar-múltiples-modelos)
10. [Resumen y conexión con el resto del curso](#10-resumen-y-conexión-con-el-resto-del-curso)
11. [Ejercicios de comprensión](#11-ejercicios-de-comprensión)

---

## 1. Por qué existen los modelos alternativos

### 1.1 El modelo relacional y sus límites naturales

El modelo relacional tiene una virtud que también es su límite: está construido sobre un principio de uniformidad. Todas las filas de una relación tienen exactamente los mismos atributos. Todas las relaciones tienen una estructura fija conocida en tiempo de diseño. La integridad referencial asume que las relaciones entre entidades son predefinidas y estables.

Esta uniformidad es perfecta para dominios donde los datos son homogéneos, las relaciones son bien conocidas, y la estructura cambia raramente. Es el modelo correcto para la mayoría de los sistemas de negocio transaccionales.

Pero existen dominios donde esta uniformidad es exactamente el problema:

- **Datos heterogéneos:** Un catálogo de e-commerce donde cada categoría de producto tiene atributos completamente distintos (un libro tiene ISBN, autor, páginas; un televisor tiene pulgadas, resolución, tecnología de panel; un alimento tiene calorías, alérgenos, fecha de vencimiento).
- **Relaciones arbitrariamente profundas:** Una red social donde la query "encuentra todos los amigos de amigos de amigos de X a distancia 4" requiere un JOIN a sí mismo cuatro veces, con rendimiento que se degrada exponencialmente.
- **Datos que son intrínsecamente una secuencia de tiempo:** Métricas de servidor donde se registran millones de puntos por segundo y las queries más comunes son "dame los valores de la métrica X durante el intervalo [t1, t2]".
- **Significado semántico entre conceptos:** Una base de conocimiento donde "Madrid está en España" y "España es un país europeo" implica que "Madrid está en Europa", y el motor debe poder razonar sobre esas implicaciones.

Los modelos alternativos surgieron como respuesta a estos dominios específicos donde el relacional, aunque técnicamente capaz, produce soluciones engorrosas, lentas o difíciles de mantener.

### 1.2 El movimiento NoSQL y su legado

En los años 2000-2010, el movimiento NoSQL (Not Only SQL) produjo una explosión de sistemas de almacenamiento con distintos modelos: MongoDB (documentos), Cassandra (columnas anchas), Neo4j (grafos), Redis (clave-valor), InfluxDB (series de tiempo). La promesa era: elige el modelo que más naturalmente represente tu dominio.

El legado de ese movimiento es mixto. Algunos sistemas NoSQL se consolidaron como las mejores herramientas para sus dominios específicos. Otros generaron deuda técnica al ser adoptados por razones de moda más que de adecuación técnica. Y los RDBMS respondieron incorporando capacidades: PostgreSQL añadió JSONB, tipos de rango, búsqueda de texto completo, y extensiones como PostGIS y pgvector.

La lección consolidada: el modelo relacional sigue siendo la elección por defecto para la mayoría de los sistemas. Los modelos alternativos son la elección correcta cuando el dominio lo justifica genuinamente, no por novedad.

---

## 2. Modelo de documentos

### 2.1 El concepto central: el documento como unidad

En el **modelo de documentos**, la unidad fundamental de almacenamiento es el **documento**: un objeto semi-estructurado (típicamente JSON o BSON) que puede tener estructura anidada y campos variables. Cada documento existe de forma autónoma; no hay un esquema rígido que todos los documentos de una colección deban respetar.

```json
// Documento 1: un libro
{
  "_id": "prod-001",
  "tipo": "libro",
  "nombre": "Cien años de soledad",
  "precio": 45.90,
  "isbn": "978-0-06-088328-7",
  "autor": "Gabriel García Márquez",
  "editorial": "Harper Perennial",
  "paginas": 417,
  "genero": ["realismo mágico", "ficción latinoamericana"]
}

// Documento 2: un televisor (misma colección, estructura completamente distinta)
{
  "_id": "prod-002",
  "tipo": "televisor",
  "nombre": "Samsung QLED 55\"",
  "precio": 899.99,
  "pulgadas": 55,
  "resolucion": "4K",
  "tecnologia_panel": "QLED",
  "tasa_refresco_hz": 120,
  "puertos_hdmi": 4,
  "smart_tv": true,
  "sistema_operativo": "Tizen"
}
```

Ambos documentos coexisten en la misma colección de "productos" sin ninguna restricción de que deban tener los mismos campos.

### 2.2 Embedding vs referencing: la decisión de modelado central

En el modelo de documentos, la decisión más importante es si modelar las relaciones entre entidades usando **embedding** (incrustar los datos relacionados dentro del documento padre) o **referencing** (almacenar una referencia al documento relacionado, similar a una FK).

**Embedding:**

```json
// Pedido con los ítems incrustados
{
  "_id": "ped-001",
  "cliente_id": "cli-042",
  "fecha": "2024-12-15",
  "estado": "completado",
  "items": [
    {"producto_id": "prod-001", "cantidad": 2, "precio_unit": 45.90},
    {"producto_id": "prod-002", "cantidad": 1, "precio_unit": 899.99}
  ],
  "total": 991.79,
  "direccion_entrega": {
    "calle": "Av. Heroínas 1250",
    "ciudad": "Cochabamba",
    "codigo_postal": "0000"
  }
}
```

**Ventajas del embedding:**
- Una sola lectura retorna el pedido completo con todos sus ítems.
- Sin JOINs: el documento es autónomo.
- Coherencia: los datos del pedido y sus ítems se actualizan atómicamente.

**Desventajas del embedding:**
- Si los datos del producto cambian (nombre, precio), cada pedido que lo contiene tiene su propia copia. No hay normalización.
- Los documentos pueden crecer indefinidamente si el array de ítems no tiene límite.
- Buscar "todos los pedidos que contienen el producto X" requiere escanear todos los documentos.

**Referencing:**

```json
// Pedido con referencias a productos
{
  "_id": "ped-001",
  "cliente_id": "cli-042",
  "items": [
    {"producto_ref": "prod-001", "cantidad": 2, "precio_unit_snapshot": 45.90},
    {"producto_ref": "prod-002", "cantidad": 1, "precio_unit_snapshot": 899.99}
  ]
}
```

**Ventajas del referencing:**
- Los datos del producto están en un solo lugar (sin duplicación).
- Los cambios en el producto se reflejan para todos los pedidos que lo referencian.

**Desventajas del referencing:**
- Para leer un pedido completo con datos del producto, se necesitan múltiples lecturas (el pedido + cada producto referenciado).
- Los motores de documentos no tienen JOINs nativos tan eficientes como los RDBMS.

**Las reglas de Fowler para la decisión embedding vs referencing:**

1. Embebe si los datos relacionados siempre se acceden juntos con el padre.
2. Embebe si los datos relacionados no crecen de forma ilimitada (arrays de tamaño acotado).
3. Referencia si los datos relacionados se acceden de forma independiente frecuentemente.
4. Referencia si los datos relacionados son compartidos por múltiples documentos padres.
5. Referencia si el sub-documento puede crecer indefinidamente.

### 2.3 Cuándo el modelo de documentos es superior al relacional

**Caso 1: Catálogos de productos con atributos heterogéneos**

Como vimos en el Módulo 5, el anti-patrón EAV es la solución relacional para atributos heterogéneos. El modelo de documentos es la alternativa natural:

```json
// En lugar de EAV (engorroso y sin tipos), un documento con atributos naturales
{
  "producto_id": "prod-123",
  "tipo": "laptop",
  "atributos": {
    "procesador": "Intel Core i7-12th",
    "ram_gb": 16,
    "almacenamiento_gb": 512,
    "pantalla_pulgadas": 15.6,
    "peso_kg": 1.8,
    "sistema_operativo": "Windows 11",
    "color": "plata"
  }
}
```

Ventaja sobre EAV: los tipos son correctos (enteros, floats, strings), las queries son directas, y no se pierde la validación de tipos.

**Caso 2: Configuraciones y documentos de usuario**

Cuando cada usuario tiene una configuración completamente personalizada sin estructura predefinida:

```json
{
  "usuario_id": "usr-042",
  "preferencias": {
    "tema": "oscuro",
    "idioma": "es",
    "notificaciones": {
      "email": true,
      "push": false,
      "frecuencia": "semanal"
    },
    "dashboard": {
      "widgets": ["ventas", "clientes", "inventario"],
      "layout": "3-columnas"
    }
  }
}
```

Una tabla relacional para esto requeriría columnas para cada posible preferencia, o un esquema EAV, o JSONB en PostgreSQL. El modelo de documentos es la representación natural.

**Caso 3: Datos con estructura que evoluciona rápidamente**

En etapas tempranas de un producto, el esquema de los datos cambia semana a semana. El modelo de documentos permite añadir campos sin migraciones de esquema. En un RDBMS, cada cambio de esquema requiere un `ALTER TABLE` que puede ser costoso en tablas grandes.

### 2.4 Las limitaciones reales del modelo de documentos

**Ausencia de joins eficientes:**

MongoDB introdujo el operador `$lookup` en la versión 3.2 para hacer joins entre colecciones. Pero los joins en bases de datos de documentos son significativamente menos eficientes que en los RDBMS, porque están diseñados para datos que no necesitan joins. Si un dominio requiere muchos joins, el modelo de documentos no es la elección correcta.

**Sin integridad referencial garantizada:**

Si un documento referencia a otro por ID, la base de datos no garantiza que ese otro documento exista. Un documento de pedido puede referenciar un `cliente_id` que nunca existió o fue eliminado. La integridad referencial es responsabilidad del código de aplicación.

**Transacciones limitadas (históricamente):**

MongoDB añadió soporte para transacciones multi-documento ACID en la versión 4.0 (2018). Antes, las transacciones solo eran atómicas a nivel de un documento individual. Incluso hoy, las transacciones multi-documento en MongoDB tienen más overhead que en un RDBMS.

**Queries ad-hoc complejas:**

Los motores de documentos están optimizados para acceder a documentos por su ID o por campos indexados. Queries analíticas complejas con múltiples agrupaciones, filtros sobre campos anidados, o joins son más difíciles de expresar y más lentas que en SQL.

### 2.5 El modelo de documentos dentro del modelo relacional: JSONB en PostgreSQL

PostgreSQL ofrece lo mejor de ambos mundos mediante el tipo `JSONB`: almacenamiento de documentos JSON dentro de columnas de una tabla relacional, con índices GIN para búsquedas eficientes dentro del JSON.

```sql
-- Tabla híbrida: columnas relacionales para los campos comunes + JSONB para los específicos
CREATE TABLE productos (
    producto_id   SERIAL PRIMARY KEY,
    tipo          VARCHAR(50) NOT NULL,
    nombre        TEXT NOT NULL,
    precio        NUMERIC(10,2) NOT NULL,
    activo        BOOLEAN NOT NULL DEFAULT TRUE,
    atributos     JSONB  -- los atributos específicos de cada tipo
);

-- Índice GIN para búsquedas dentro de los atributos
CREATE INDEX idx_productos_attrs ON productos USING GIN (atributos);

-- Insertar productos de tipos distintos
INSERT INTO productos (tipo, nombre, precio, atributos) VALUES
('libro', 'Cien años de soledad', 45.90,
 '{"isbn": "978-0-06-088328-7", "autor": "García Márquez", "paginas": 417}'),
('televisor', 'Samsung QLED 55"', 899.99,
 '{"pulgadas": 55, "resolucion": "4K", "smart_tv": true, "puertos_hdmi": 4}');

-- Queries sobre el JSONB con índice
SELECT nombre, precio
FROM productos
WHERE atributos @> '{"autor": "García Márquez"}';  -- usa el índice GIN

-- Extracción de campos específicos
SELECT nombre, atributos->>'isbn' AS isbn
FROM productos
WHERE tipo = 'libro';

-- Búsqueda en arrays dentro del JSONB
SELECT nombre FROM productos
WHERE atributos->'generos' ? 'ficción';  -- el campo generos contiene 'ficción'

-- Joins entre la tabla relacional y su JSONB
SELECT p.nombre, p.precio, p.atributos->>'autor' AS autor,
       c.nombre AS nombre_categoria
FROM productos p
JOIN categorias c ON c.categoria_id = p.categoria_id
WHERE p.tipo = 'libro';
```

Esta aproximación elimina la necesidad de una base de datos de documentos separada para muchos casos de uso, manteniendo las ventajas del modelo relacional (integridad, transacciones ACID, JOINs eficientes).

---

## 3. Modelo de grafos

### 3.1 La estructura del modelo de grafos

En el **modelo de grafos**, los datos se representan como nodos (entidades) y aristas (relaciones entre entidades). Tanto los nodos como las aristas pueden tener propiedades (pares clave-valor).

```
Nodo: {id: 1, tipo: "Persona", nombre: "Ana", edad: 32}
Nodo: {id: 2, tipo: "Persona", nombre: "Luis", edad: 28}
Nodo: {id: 3, tipo: "Empresa", nombre: "TechCorp", sector: "IT"}

Arista: (Ana) --[CONOCE, desde: 2020]--> (Luis)
Arista: (Ana) --[TRABAJA_EN, cargo: "CTO"]--> (TechCorp)
Arista: (Luis) --[TRABAJA_EN, cargo: "Dev"]--> (TechCorp)
```

La característica definitoria de los grafos es que las relaciones son ciudadanos de primera clase del modelo, no consecuencias derivadas de claves foráneas. Traversar una relación (seguir una arista) es O(1): una operación directa, no un JOIN.

### 3.2 El problema que los grafos resuelven: traversal de relaciones profundas

El caso de uso paradigmático de los grafos es la traversal de relaciones a profundidades variables. En una red social:

- **Profundidad 1:** Los amigos de Ana. En SQL: `SELECT * FROM amistades WHERE usuario_id = 'Ana'`.
- **Profundidad 2:** Los amigos de los amigos de Ana. En SQL: un self-join.
- **Profundidad 3:** Amigos de amigos de amigos. En SQL: otro self-join.
- **Profundidad variable:** "Encuentra todos los usuarios a distancia ≤ 4 de Ana". En SQL: una CTE recursiva o múltiples JOINs. El rendimiento se degrada exponencialmente.

En un motor de grafos como Neo4j, esta query es trivial y eficiente:

```cypher
// Cypher (lenguaje de query de Neo4j)
// Amigos hasta profundidad 4
MATCH (ana:Persona {nombre: 'Ana'})-[:AMIGO*1..4]-(conocido:Persona)
RETURN DISTINCT conocido.nombre
```

La razón del rendimiento superior: en un motor de grafos, cada nodo almacena punteros directos a sus aristas adyacentes. Traversar una relación es seguir un puntero, no hacer un JOIN con un índice. Para grafos densos con millones de relaciones, esta diferencia es de órdenes de magnitud.

### 3.3 Casos de uso donde los grafos son la elección correcta

**Redes sociales:**
Amigos de amigos, sugerencias de conexiones, detección de comunidades, camino más corto entre dos personas.

**Motores de recomendación:**
"Usuarios que compraron X también compraron Y" se modela naturalmente como un grafo bipartito (usuarios ↔ productos) con relaciones de compra. Las recomendaciones son traversals del grafo.

**Detección de fraude:**
Las redes de fraude tienen patrones topológicos específicos (anillos de transacciones, nodos compartidos entre múltiples cuentas). Detectarlos en SQL requiere queries complejas y lentas; en grafos es traversal de patrones.

**Motores de conocimiento y ontologías:**
"Si A trabaja para B, y B es subsidiaria de C, entonces A trabaja indirectamente para C." Esta inferencia transitiva es natural en grafos pero compleja en SQL.

**Cadenas de suministro:**
"¿Qué productos están afectados si el proveedor X falla?" requiere traversar la cadena completa hacia adelante.

**Detección de dependencias en sistemas de software:**
"¿Qué módulos dependen (directa o indirectamente) del módulo que voy a cambiar?" Es una traversal de grafo de dependencias.

### 3.4 Modelar grafos en el modelo relacional

Es completamente posible modelar un grafo en SQL. La representación clásica es la tabla de adyacencia (Módulo 5):

```sql
-- Grafo de relaciones sociales en SQL
CREATE TABLE personas (
    persona_id INT PRIMARY KEY,
    nombre     TEXT NOT NULL
);

CREATE TABLE relaciones (
    origen_id  INT NOT NULL REFERENCES personas,
    destino_id INT NOT NULL REFERENCES personas,
    tipo       VARCHAR(50) NOT NULL,
    desde      DATE,
    PRIMARY KEY (origen_id, destino_id, tipo)
);

-- Amigos de amigos de Ana (profundidad 2) con CTE recursiva
WITH RECURSIVE red AS (
    -- Nivel 0: Ana misma
    SELECT persona_id, nombre, 0 AS profundidad
    FROM personas WHERE nombre = 'Ana'

    UNION ALL

    -- Expandir un nivel en cada iteración
    SELECT p.persona_id, p.nombre, r.profundidad + 1
    FROM red r
    JOIN relaciones rel ON rel.origen_id = r.persona_id AND rel.tipo = 'AMIGO'
    JOIN personas p ON p.persona_id = rel.destino_id
    WHERE r.profundidad < 4  -- límite de profundidad
)
SELECT DISTINCT nombre, profundidad FROM red WHERE profundidad > 0;
```

**El problema de rendimiento:** Para una red con millones de nodos y cientos de millones de relaciones, esta query CTE hace múltiples scans de la tabla `relaciones` con JOINs en cada nivel de profundidad. El rendimiento se degrada gravemente a partir de profundidad 3-4.

**La regla práctica:** Si las traversals de grafo son el caso de uso dominante, con relaciones profundas y grafos densos, un motor de grafos dedicado es la elección correcta. Si el grafo es relativamente pequeño o las traversals son superficiales (profundidad 1-2), SQL con CTE recursiva puede ser suficiente.

### 3.5 El lenguaje Cypher y la diferencia con SQL

Cypher es el lenguaje de query de Neo4j. Su sintaxis usa ASCII art para expresar patrones de grafo intuitivamente:

```cypher
// ¿Quién conoce a alguien que trabaja en TechCorp?
MATCH (persona:Persona)-[:CONOCE]->(:Persona)-[:TRABAJA_EN]->(empresa:Empresa {nombre: 'TechCorp'})
RETURN DISTINCT persona.nombre

// Camino más corto entre Ana y Carlos
MATCH p = shortestPath((ana:Persona {nombre: 'Ana'})-[*]-(carlos:Persona {nombre: 'Carlos'}))
RETURN p

// Detectar ciclos de transacciones (patrón de fraude)
MATCH (cuenta:Cuenta)-[:TRANSFIERE*3..5]->(cuenta)
RETURN cuenta.numero

// PageRank sobre el grafo (algoritmo integrado en Neo4j)
CALL gds.pageRank.stream('miGrafo')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).nombre AS nombre, score
ORDER BY score DESC LIMIT 10
```

La diferencia conceptual con SQL es profunda: Cypher trabaja con **patrones estructurales** del grafo, no con predicados sobre conjuntos de filas. La expresividad para relaciones complejas es incomparablemente mayor que SQL para dominios de grafo.

---

## 4. Modelo columnar

### 4.1 Revisitando el almacenamiento columnar

En el Módulo 10 introdujimos el almacenamiento columnar como el núcleo del rendimiento OLAP. Aquí lo profundizamos desde la perspectiva del modelo de datos.

El almacenamiento columnar no es solo una optimización de rendimiento: es un modelo de almacenamiento distinto con implicaciones para qué operaciones son eficientes y cuáles no.

**El modelo relacional por filas (row store):** Diseñado para acceder a todas las columnas de pocas filas. Excelente para OLTP.

**El modelo columnar:** Diseñado para acceder a pocas columnas de muchas filas. Excelente para OLAP.

### 4.2 Apache Parquet: el formato estándar

**Apache Parquet** es el formato de archivo columnar de facto en el ecosistema de datos modernos. No es un motor de base de datos, sino un formato de archivo que cualquier herramienta puede leer (Spark, Trino, Athena, DuckDB, Pandas, Arrow).

La estructura interna de Parquet:

```
Archivo Parquet
├── Row Group 1 (e.g., filas 1-100,000)
│   ├── Column Chunk: columna "nombre_producto"
│   │   ├── Page 1: valores 1-10,000 (comprimidos con dictionary encoding)
│   │   ├── Page 2: valores 10,001-20,000
│   │   └── ...
│   ├── Column Chunk: columna "precio"
│   │   ├── Page 1: valores 1-10,000 (comprimidos con delta encoding)
│   │   └── ...
│   └── Column Chunk: columna "fecha_venta"
│       └── ...
├── Row Group 2 (filas 100,001-200,000)
│   └── ...
└── Footer (metadatos: esquema, estadísticas por columna por row group)
    ├── min/max de cada columna por row group (para predicate pushdown)
    └── null counts, distinct counts
```

**El footer y el predicate pushdown:**

El footer de cada archivo Parquet contiene estadísticas de min/max para cada columna en cada row group. Esto permite que los motores descarten row groups enteros sin leerlos:

```
Query: WHERE precio > 1000

Motor lee el footer: Row Group 1 tiene precio max = 850 → SKIP
                     Row Group 2 tiene precio max = 1200 → LEER
                     Row Group 3 tiene precio max = 750 → SKIP
                     Row Group 4 tiene precio max = 5000 → LEER

Solo se leen 2 de 4 row groups. 50% menos I/O.
```

Esta técnica se llama **predicate pushdown** al nivel del archivo y es una de las razones por las que los data lakes sobre Parquet pueden tener rendimiento comparable a los data warehouses.

### 4.3 DuckDB: el SQLite del OLAP

**DuckDB** es un motor OLAP embebido que ejecuta directamente sobre archivos Parquet, CSV, o tablas en memoria. Es el "SQLite del análisis": no requiere servidor, se instala como una librería, y puede ejecutar SQL analítico de alto rendimiento sobre datasets de gigabytes a terabytes en una sola máquina.

```python
import duckdb

# DuckDB puede leer directamente archivos Parquet
conn = duckdb.connect()

result = conn.execute("""
    SELECT
        nombre_categoria,
        EXTRACT(YEAR FROM fecha_venta) AS anio,
        SUM(monto_neto) AS total_ventas,
        COUNT(*) AS num_ventas
    FROM 'ventas/*.parquet'       -- Lee todos los Parquet de un directorio
    WHERE fecha_venta >= '2024-01-01'
    GROUP BY nombre_categoria, anio
    ORDER BY total_ventas DESC
""").fetchdf()  # retorna un DataFrame de Pandas
```

DuckDB usa almacenamiento columnar en memoria, vectorización de operaciones, y puede aprovechar múltiples cores. Para análisis en notebooks o pipelines de datos que no requieren un servidor dedicado, DuckDB ha simplificado enormemente el trabajo analítico.

---

## 5. Series de tiempo

### 5.1 El problema con las series de tiempo en RDBMS convencionales

Una **serie de tiempo** es una secuencia de observaciones indexadas por tiempo: métricas de servidor (CPU, memoria, latencia), precios financieros, datos de sensores IoT, logs de aplicación.

Las series de tiempo tienen características que las hacen problemáticas en RDBMS convencionales:

**Volumen extremo:** Un sistema de monitoreo puede generar 1,000,000 de métricas por segundo. En un año, eso es ~31 billones de puntos. Almacenar esto en una tabla PostgreSQL con índice B-tree se vuelve impracticable.

**Patrón de acceso específico:** Las queries más comunes son siempre por rango de tiempo: "dame todos los valores de la métrica X entre t1 y t2". Los índices B-tree en timestamps son eficientes para esto, pero el problema es el volumen.

**Compresión especializada:** Los valores consecutivos en una serie de tiempo suelen ser muy similares (la CPU no salta de 23% a 87% en un segundo, generalmente). Las técnicas de compresión especializadas (delta-of-delta encoding, XOR floating point) pueden comprimir series de tiempo a 10x-100x mejor que compresión genérica.

**Retención y expiración:** Los datos de monitoreo raramente necesitan más de 6-12 meses de retención. Los sistemas de series de tiempo tienen retención configurable con eliminación automática de datos antiguos.

**Downsampling:** Para visualizar un año de datos en un gráfico, no se necesitan todos los puntos; se puede reducir la resolución (calcular el promedio por hora en lugar de por segundo). Los motores de series de tiempo tienen downsampling automático.

### 5.2 Modelar series de tiempo en PostgreSQL

Antes de adoptar un motor dedicado, vale conocer cómo se modela en PostgreSQL. La extensión **TimescaleDB** convierte PostgreSQL en un motor de series de tiempo:

```sql
-- Con TimescaleDB
CREATE TABLE metricas (
    tiempo      TIMESTAMPTZ NOT NULL,
    sensor_id   INT NOT NULL,
    metrica     VARCHAR(100) NOT NULL,
    valor       DOUBLE PRECISION NOT NULL
);

-- Convertir en hypertable (particionamiento automático por tiempo)
SELECT create_hypertable('metricas', 'tiempo',
    chunk_time_interval => INTERVAL '1 day');

-- TimescaleDB crea particiones ("chunks") automáticamente por día
-- Las queries por rango de tiempo solo leen los chunks relevantes

-- Insertar millones de puntos eficientemente
INSERT INTO metricas VALUES
    ('2024-12-15 10:00:00+00', 1, 'cpu_uso_pct', 23.5),
    ('2024-12-15 10:00:01+00', 1, 'cpu_uso_pct', 24.1),
    -- ... millones de filas

-- Query de series de tiempo: promedio por minuto (downsampling)
SELECT
    time_bucket('1 minute', tiempo) AS minuto,
    sensor_id,
    AVG(valor) AS promedio_cpu,
    MAX(valor) AS pico_cpu
FROM metricas
WHERE tiempo >= NOW() - INTERVAL '1 hour'
  AND metrica = 'cpu_uso_pct'
  AND sensor_id = 1
GROUP BY minuto, sensor_id
ORDER BY minuto;
```

### 5.3 Motores dedicados de series de tiempo

**InfluxDB:** El motor de series de tiempo más popular. Usa su propio lenguaje de query (Flux) y almacenamiento altamente optimizado para escritura de alta velocidad. Tiene retención automática, downsampling, y dashboards integrados (con Grafana).

**Prometheus:** Motor de series de tiempo diseñado específicamente para métricas de monitoreo. Usa su propio modelo de datos (métricas con etiquetas) y lenguaje (PromQL). Es el estándar de facto para monitoreo en entornos Kubernetes.

**ClickHouse:** Motor columnar de alto rendimiento que también sirve bien para series de tiempo de alto volumen, con SQL estándar y velocidades de inserción e ingesta superiores a la mayoría de los motores.

```
# PromQL (lenguaje de Prometheus)
# Uso de CPU promedio por instancia en los últimos 5 minutos
rate(node_cpu_seconds_total{mode="user"}[5m])

# Tasa de requests HTTP por segundo
rate(http_requests_total[1m])

# Alerta: si el error rate supera 1%
(sum(rate(http_requests_total{status=~"5.."}[5m])) /
 sum(rate(http_requests_total[5m]))) > 0.01
```

### 5.4 Cuándo usar un motor de series de tiempo vs PostgreSQL

| Criterio | PostgreSQL + TimescaleDB | Motor dedicado |
|---|---|---|
| Volumen diario | < 100M puntos | > 100M puntos |
| Velocidad de inserción | < 100K puntos/seg | > 100K puntos/seg |
| Necesidad de SQL completo | Sí | No necesariamente |
| Integración con datos relacionales | Sí (misma DB) | No (sistema separado) |
| Retención y downsampling automático | Con configuración | Nativo |
| Equipo ya usa PostgreSQL | Sí | Indiferente |

---

## 6. RDF y triplestores

### 6.1 El modelo RDF: datos como triples

**RDF** (Resource Description Framework) es un modelo de datos donde toda la información se expresa como triples: **(sujeto, predicado, objeto)**.

```
(Madrid, estáEn, España)
(España, esUnPaís, Europa)
(Madrid, tienePoblación, "3.3 millones")
(García_Márquez, escribió, "Cien_años_de_soledad")
(García_Márquez, esUn, Escritor)
(García_Márquez, nació_en, Colombia)
```

Cada elemento del triple es una URI o un literal. Las URIs son identificadores globalmente únicos, lo que permite combinar datasets de distintas fuentes sin ambigüedad.

Un **triplestore** es una base de datos especializada para almacenar y consultar triples RDF. Los triplestores implementan **razonamiento**: pueden inferir nuevos triples a partir de los existentes usando reglas lógicas (ontologías).

```
Dado:
  (España, esUnPaísDe, Europa)
  (Madrid, estáEn, España)

El motor puede inferir:
  (Madrid, estáEnContinente, Europa)  ← inferencia transitiva
```

### 6.2 SPARQL: el lenguaje de query para RDF

**SPARQL** es el lenguaje estándar para consultar triplestores. Su modelo de query es la coincidencia de patrones de triple:

```sparql
# ¿Qué escritores latinoamericanos ganaron el Premio Nobel?
PREFIX dbpedia: <http://dbpedia.org/resource/>
PREFIX dbo: <http://dbpedia.org/ontology/>

SELECT ?escritor ?nombre
WHERE {
    ?escritor a dbo:Writer ;              -- es un escritor
              dbo:birthPlace ?pais ;      -- nació en algún país
              dbo:award dbpedia:Nobel_Prize_in_Literature .  -- ganó el Nobel
    ?pais dbo:locatedInArea dbpedia:Latin_America .           -- ese país está en LA
    ?escritor rdfs:label ?nombre .
    FILTER(LANG(?nombre) = "es")
}
```

### 6.3 Cuándo usar RDF

RDF y los triplestores son la elección correcta para:

- **Bases de conocimiento semántico:** Wikipedia tiene Wikidata (un grafo de conocimiento RDF con ~100M de ítems). Google Knowledge Graph usa un modelo similar.
- **Integración de datos heterogéneos de múltiples fuentes:** Si cada fuente usa sus propios identificadores, RDF con URIs globales permite integración sin ETL complejo.
- **Dominios donde la inferencia lógica es necesaria:** Ontologías médicas (SNOMED CT), ontologías de dominio científico.
- **Linked Data:** El movimiento de datos abiertos enlazados en la web usa RDF.

Para la mayoría de los sistemas de negocio convencionales, RDF y SPARQL son herramientas excesivas. Los motores de grafos como Neo4j (con propiedades, no con triples puros) son más prácticos para casos de uso de grafos de negocio.

---

## 7. Bases de datos vectoriales

### 7.1 El modelo vectorial

Un **vector embedding** es una representación numérica de alta dimensionalidad de un objeto (texto, imagen, audio, documento) generada por un modelo de Machine Learning. Objetos semánticamente similares tienen vectores cercanos en el espacio de dimensiones.

```python
# Un texto convertido en vector por un modelo de embeddings
texto = "bases de datos relacionales"
vector = modelo.encode(texto)
# vector = [0.234, -0.156, 0.891, 0.023, -0.445, ...]  (1,536 dimensiones típicas)
```

Las bases de datos vectoriales están optimizadas para la operación más común sobre vectores: **búsqueda por similitud** (encontrar los K vectores más cercanos a un vector de consulta). Esta operación se llama k-NN (k-Nearest Neighbors) o ANN (Approximate Nearest Neighbors).

### 7.2 El uso en sistemas de IA

Con el auge de los modelos de lenguaje (LLMs) y los sistemas de búsqueda semántica, las bases de datos vectoriales se han vuelto una pieza central de la arquitectura de muchas aplicaciones de IA:

**RAG (Retrieval-Augmented Generation):**

```
Usuario pregunta: "¿Cuál es la política de devoluciones?"
    │
    ▼
Se genera el embedding de la pregunta: [0.23, -0.15, ...]
    │
    ▼
Se buscan en la base vectorial los K documentos más similares:
- "Política de devoluciones.pdf" (similaridad: 0.95)
- "FAQ de clientes.docx" (similaridad: 0.87)
- "Manual de usuario.pdf" (similaridad: 0.72)
    │
    ▼
Se envía al LLM: "Dado el siguiente contexto [documentos encontrados],
                  responde: ¿Cuál es la política de devoluciones?"
    │
    ▼
El LLM responde con información precisa del contexto
```

**Búsqueda semántica:**

En lugar de buscar por palabras exactas ("laptop barata"), busca por similitud semántica (encontrar productos similares en concepto a la consulta del usuario).

**Detección de duplicados y similitud:**

Encontrar documentos casi duplicados, imágenes similares, o productos con descripción similar.

### 7.3 pgvector: vectores dentro de PostgreSQL

La extensión **pgvector** para PostgreSQL añade un tipo de dato `vector` y operaciones de búsqueda por similitud:

```sql
-- Instalar la extensión
CREATE EXTENSION IF NOT EXISTS vector;

-- Tabla de documentos con embeddings
CREATE TABLE documentos (
    doc_id       SERIAL PRIMARY KEY,
    titulo       TEXT NOT NULL,
    contenido    TEXT NOT NULL,
    embedding    vector(1536)  -- 1536 dimensiones (modelo text-embedding-ada-002)
);

-- Índice para búsqueda aproximada de vecinos cercanos
CREATE INDEX idx_docs_embedding ON documentos
    USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100);

-- Insertar un documento con su embedding
INSERT INTO documentos (titulo, contenido, embedding)
VALUES ('Política de devoluciones', '...', '[0.23, -0.15, ...]'::vector);

-- Buscar los 5 documentos más similares a una consulta
SELECT doc_id, titulo,
       1 - (embedding <=> '[0.31, -0.12, ...]'::vector) AS similitud
FROM documentos
ORDER BY embedding <=> '[0.31, -0.12, ...]'::vector
LIMIT 5;
-- <=> es el operador de distancia coseno (menor = más similar)

-- Búsqueda híbrida: combinar similitud vectorial con filtros relacionales
SELECT doc_id, titulo, similitud
FROM (
    SELECT doc_id, titulo,
           1 - (embedding <=> $1::vector) AS similitud
    FROM documentos
    WHERE categoria = 'soporte'  -- filtro relacional
      AND activo = TRUE
    ORDER BY embedding <=> $1::vector
    LIMIT 20
) sub
WHERE similitud > 0.8  -- umbral de similitud mínima
ORDER BY similitud DESC;
```

La búsqueda híbrida (vectorial + relacional) es el patrón más poderoso: pgvector permite combinar la búsqueda semántica con los filtros relacionales tradicionales en una sola query SQL, dentro del mismo sistema que ya tiene los datos del negocio.

---

## 8. El modelo relacional como base de comparación

### 8.1 La tabla de decisión

Antes de elegir un modelo alternativo, vale hacer la pregunta: ¿el modelo relacional con las extensiones correctas puede resolver el problema de forma aceptable?

| Dominio | Relacional | Extensión relacional | Alternativa dedicada |
|---|---|---|---|
| **Atributos heterogéneos** | EAV (malo) | PostgreSQL JSONB | MongoDB |
| **Traversal de grafos superficial** | CTE recursiva | Mismo | - |
| **Traversal de grafos profundo/denso** | Ineficiente | - | Neo4j, Neptune |
| **Texto completo** | LIKE (malo) | PostgreSQL GIN + tsvector | Elasticsearch |
| **Series de tiempo bajo volumen** | Tabla con índice en ts | TimescaleDB | - |
| **Series de tiempo alto volumen** | Impracticable | TimescaleDB | InfluxDB, Prometheus |
| **Analytics columnar** | Lento en heap | Tablas particionadas | ClickHouse, BigQuery, DuckDB |
| **Embeddings vectoriales** | Sin soporte nativo | pgvector | Pinecone, Weaviate |
| **Conocimiento semántico/inferencia** | Muy complejo | - | Triple stores |

### 8.2 El costo de la fragmentación

Cada modelo alternativo introduce un nuevo sistema que el equipo debe:
- Aprender a operar y mantener.
- Monitorear y escalar.
- Integrar con el resto de la arquitectura.
- Manejar backups, failover, y seguridad.
- Pagar en infraestructura (costos de compute y storage adicionales).

Esto es el **costo de la fragmentación** (polyglot persistence). No es trivial. Un equipo pequeño que gestiona 5 sistemas de almacenamiento distintos (PostgreSQL, MongoDB, Neo4j, InfluxDB, Redis) tiene mucho overhead operacional.

La regla práctica: antes de adoptar un modelo alternativo, preguntarse si la extensión del RDBMS existente (JSONB, pgvector, TimescaleDB, CTE recursiva) es suficiente para el caso de uso. Solo si genuinamente no lo es, considerar un sistema dedicado.

---

## 9. Políglota persistence: usar múltiples modelos

### 9.1 El concepto

**Polyglot persistence** es el patrón arquitectónico de usar distintos modelos de almacenamiento para distintas partes de un sistema, eligiendo el modelo más adecuado para cada dominio de datos.

El término fue popularizado por Martin Fowler y Pramod Sadalage en "NoSQL Distilled" (2012). La premisa es que no existe un modelo de almacenamiento único que sea óptimo para todos los casos de uso.

### 9.2 Un ejemplo de arquitectura políglota

Para una plataforma de e-commerce de escala media:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Plataforma de E-commerce                                             │
│                                                                       │
│  PostgreSQL (relacional)                                              │
│  ├── Usuarios, pedidos, pagos, inventario (transaccional)             │
│  ├── JSONB para atributos heterogéneos de productos                   │
│  └── pgvector para búsqueda semántica de productos                    │
│                                                                       │
│  Redis (clave-valor en memoria)                                       │
│  ├── Sesiones de usuario                                              │
│  ├── Carrito de compras (temporal)                                    │
│  └── Caché de productos más visitados                                 │
│                                                                       │
│  Elasticsearch (búsqueda de texto)                                    │
│  └── Búsqueda de productos por texto libre                            │
│      (si el volumen justifica Elasticsearch sobre PostgreSQL FTS)     │
│                                                                       │
│  Snowflake / BigQuery (columnar OLAP)                                 │
│  └── Analytics histórico, reportes de negocio                        │
│                                                                       │
│  InfluxDB / Prometheus (series de tiempo)                             │
│  └── Métricas de la plataforma (requests/seg, latencia, errores)      │
└──────────────────────────────────────────────────────────────────────┘
```

### 9.3 El riesgo de la sobreingeniería

El políglota persistence puede convertirse en sobreingeniería si se adopta prematuramente:

- Una startup con 100 usuarios activos no necesita Elasticsearch; PostgreSQL FTS es suficiente.
- Un sistema con 10,000 métricas por día no necesita InfluxDB; TimescaleDB o incluso una tabla PostgreSQL es suficiente.
- Un sistema con grafos de 1,000 nodos no necesita Neo4j; una CTE recursiva en PostgreSQL es suficiente.

**El principio de la herramienta mínima suficiente:** Usar el sistema más simple que resuelva el problema de forma aceptable. Agregar complejidad solo cuando la herramienta simple genuinamente no alcanza.

---

## 10. Resumen y conexión con el resto del curso

### Los conceptos clave

| Modelo | Fortaleza | Debilidad | Cuándo elegirlo |
|---|---|---|---|
| **Relacional** | Integridad, transacciones, JOINs, SQL | Heterogeneidad, grafos profundos | Por defecto para la mayoría de los sistemas |
| **Documentos** | Heterogeneidad, flexibilidad de esquema, anidado | Sin JOINs eficientes, sin integridad referencial | Catálogos heterogéneos, configuraciones, documentos del usuario |
| **Grafos** | Traversal de relaciones profundas | Queries agregadas sobre todo el grafo | Redes sociales, motores de recomendación, detección de fraude |
| **Columnar** | Lectura de pocas columnas sobre muchas filas, compresión | Escritura por fila costosa | Analytics, data warehouse, data lake |
| **Series de tiempo** | Ingesta masiva de puntos temporales, compresión temporal | Solo rangos de tiempo, sin JOINs complejos | Métricas, monitoreo, IoT, datos financieros tick-by-tick |
| **Vectorial** | Búsqueda por similitud semántica | Sin soporte para lógica de negocio compleja | Búsqueda semántica, RAG, detección de similitud |
| **RDF/Triplestore** | Inferencia lógica, integración semántica | Complejidad operacional, curva de aprendizaje | Bases de conocimiento, ontologías, linked data |

### La meta-lección del módulo

Ningún modelo de datos es universalmente superior. Cada modelo existe porque existe un dominio donde es la representación más natural y eficiente. La habilidad del diseñador de datos no es conocer en profundidad todos los modelos (aunque ayuda), sino saber reconocer cuándo el modelo relacional empieza a ser la herramienta equivocada para el dominio y cuál alternativa es la correcta.

El punto de inflexión suele ser claro: cuando las queries relacionales para el dominio se vuelven artificialmente complejas, cuando el rendimiento no mejora con más índices u optimizaciones, o cuando la naturaleza del dato (grafo, tiempo, vector) no encaja naturalmente en filas y columnas.

Hasta ese punto, el modelo relacional con las extensiones correctas es casi siempre la elección más pragmática: menos sistemas que mantener, SQL que todo el equipo conoce, transacciones ACID, y un ecosistema de herramientas maduro.

### Conexión con módulos futuros

- **Módulo 13 (Sistemas distribuidos):** Muchos de los modelos alternativos (Cassandra, DynamoDB, Redis Cluster) nacieron del problema de escalar horizontalmente lo que los RDBMS escalaban verticalmente. La distinción entre consistencia fuerte y eventual, y el teorema CAP, son directamente relevantes para entender por qué NoSQL sacrificó ciertas garantías a cambio de escala.

- **Módulo 16 (El oficio):** La decisión de qué modelo usar para qué dominio es una de las más estratégicas que un equipo de ingeniería puede tomar. Deshacerla después es costoso. Evaluar correctamente los trade-offs desde el principio es parte del oficio.

---

## 11. Ejercicios de comprensión

**Ejercicio 1.** Una plataforma de música en streaming quiere construir un sistema de recomendación. Los datos disponibles son: usuarios (demografía, historial de escucha), canciones (género, artista, álbum, BPM, energía), y el historial de escucha completo (qué usuario escuchó qué canción, cuándo, cuántas veces, si la saltó).

a) Identifica al menos tres modelos de datos distintos que podrían ser relevantes para distintas partes de este sistema.
b) Para el motor de recomendación principal ("usuarios similares a ti también escuchan..."), ¿qué modelo elegirías? Justifica comparando con las alternativas.
c) Diseña el modelo de datos elegido con suficiente detalle para ser implementable.

---

**Ejercicio 2.** Una empresa financiera necesita detectar redes de fraude. El patrón típico es: una cuenta A envía dinero a múltiples cuentas B, C, D, que a su vez envían a una cuenta E (que retira el dinero). El ciclo puede tener 3-7 saltos.

a) ¿Por qué este problema es difícil de resolver eficientemente en SQL puro?
b) Diseña la representación en un modelo de grafos (nodos, tipos de nodos, tipos de aristas, propiedades).
c) Escribe la query en Cypher que detecta el patrón descrito (una cuenta que recibe de múltiples intermediarios que a su vez recibieron del mismo origen).
d) ¿En qué condiciones seguiría siendo razonable usar PostgreSQL con CTE recursiva en lugar de Neo4j?

---

**Ejercicio 3.** Una startup de salud digital recopila datos de sensores de pacientes: frecuencia cardíaca (cada segundo), presión arterial (cada 5 minutos), temperatura (cada minuto), y glucosa en sangre (cada 15 minutos). Hay 50,000 pacientes activos.

a) Calcula el volumen estimado de datos por día y por año.
b) Diseña el esquema con TimescaleDB, incluyendo particionamiento y política de retención.
c) ¿En qué punto (volumen, velocidad de inserción) recomendarías migrar a un motor dedicado como InfluxDB?
d) Escribe la query que detecta pacientes con frecuencia cardíaca anormalmente alta (>100 bpm) sostenida por más de 5 minutos en las últimas 24 horas.

---

**Ejercicio 4.** Un sistema de gestión de contenido (CMS) tiene documentos de blog con la siguiente estructura variable:

```json
{
  "tipo": "articulo",
  "titulo": "...",
  "cuerpo": "...",
  "autor_id": 42,
  "etiquetas": ["tecnología", "bases de datos"],
  "seo": {"meta_descripcion": "...", "slug": "..."},
  "imagen_destacada": {"url": "...", "alt": "..."}
}
```

Pero también tiene páginas estáticas, productos, y eventos, cada uno con atributos completamente distintos.

a) Diseña el modelo en PostgreSQL usando JSONB para los campos variables, manteniendo columnas relacionales para los campos comunes y los que necesitan integridad referencial.
b) Diseña los índices necesarios para soportar eficientemente: búsqueda por etiqueta, búsqueda por autor, búsqueda full-text en el cuerpo, y filtro por tipo.
c) ¿En qué punto recomendarías migrar a MongoDB en lugar de JSONB? ¿Cuál sería la señal que indicaría que JSONB ya no es suficiente?

---

**Ejercicio 5.** Una empresa quiere implementar búsqueda semántica sobre su base de conocimiento interna (manuales, procedimientos, FAQs). Tiene ~50,000 documentos en PostgreSQL.

a) Describe la arquitectura completa del sistema de búsqueda semántica usando pgvector, incluyendo cómo se generan y almacenan los embeddings.
b) Diseña la tabla y los índices necesarios.
c) Escribe la query que implementa búsqueda híbrida: combina similitud vectorial con filtros por departamento y fecha de actualización.
d) ¿Cuándo sería necesario migrar de pgvector a un motor vectorial dedicado como Pinecone o Weaviate?

---

*Próximo módulo: Modelado para sistemas distribuidos — donde exploraremos qué le ocurre al modelo de datos cuando los datos dejan de vivir en una sola máquina, cómo el teorema CAP impone compromisos irresolubles, y qué técnicas de modelado permiten construir sistemas distribuidos que sean tanto consistentes como disponibles, dentro de los límites que la física impone.*
