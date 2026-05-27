# Fundamentos sobre Bases de Datos

Curso de 16 módulos para mejorar el nivel de comprensión de las bases de datos, organizado en cinco ejes: teoría formal, diseño, motor, analítica e integración. Cada módulo construye sobre el anterior con explicaciones en profundidad, ejemplos en SQL (PostgreSQL) y ejercicios de comprensión.

---

## Estructura del curso

### Eje 1 — Teoría

**Módulo 01 · El modelo relacional como matemática**
Qué es formalmente una relación, la diferencia entre el modelo lógico y la implementación física, las 12 reglas de Codd, el problema del valor nulo y su lógica trivaluada, y el álgebra relacional completa: selección, proyección, producto cartesiano, join natural, división, unión y diferencia como operaciones matemáticas con propiedades formales demostrables.

**Módulo 02 · Dependencias funcionales y normalización real**
Las formas normales como consecuencias de entender dependencias, no como recetas. Dependencias funcionales, axiomas de Armstrong, cierre de atributos, cobertura mínima, claves candidatas. 1FN, 2FN, 3FN, FNBC, 4FN, 5FN y la Forma Normal de Domain/Key como techo teórico.

---

### Eje 2 — Diseño

**Módulo 03 · Modelado conceptual con profundidad**
El modelo EER completo: entidades débiles, cardinalidades y participación total/parcial, relaciones ternarias y cuándo no son equivalentes a binarias, agregación, jerarquías de generalización y especialización. El proceso sistemático de traducir un diagrama a esquema relacional. Errores de modelado que se vuelven invisibles una vez en tablas.

**Módulo 04 · Integridad y restricciones como primera clase**
La integridad no es un feature del DBMS, es responsabilidad del modelo. Restricciones declarativas vs. procedurales, claves foráneas en profundidad con todas sus acciones referenciales, integridad de dominio, `CHECK` constraints complejos, el problema de las restricciones multi-tabla, claves foráneas diferidas, aserciones y por qué casi ningún motor las implementa bien, triggers como mecanismo de integridad.

**Módulo 05 · Patrones de modelado para dominios reales**
Patrón Party para personas y organizaciones, patrón de roles, jerarquías recursivas y bill of materials con los cuatro modelos (adjacency list, nested set, materialized path, closure table), productos configurables, precios con vigencia temporal, clasificaciones flexibles, y el antipatrón EAV: cuándo es un error y cuándo es la única solución pragmática.

**Módulo 06 · Diseño para el tiempo**
Tiempo válido vs. tiempo de transacción y por qué la distinción importa. Tablas bitemporales. Slowly Changing Dimensions tipo 1, 2, 3, 4 y 6. Modelado de eventos vs. modelado de estado. Event sourcing como patrón de modelado.

---

### Eje 3 — Motor

**Módulo 07 · Internals: cómo ejecuta realmente una query**
El pipeline completo: parsing, reescritura, planificación y ejecución. El query planner, generación de planes candidatos, estadísticas internas, estimación de cardinalidad. Cómo leer un plan de ejecución completo. Operadores físicos: nested loop join, hash join, merge join. Por qué una query idéntica puede tener planes distintos según el volumen o las estadísticas.

**Módulo 08 · Transacciones, aislamiento y MVCC**
ACID sin simplificaciones. Los cuatro niveles de aislamiento y los fenómenos que cada uno permite: dirty reads, non-repeatable reads, phantom reads, lost updates, write skew. MVCC como mecanismo concreto: snapshots, versiones de filas, xmin/xmax. Serializable Snapshot Isolation. Deadlocks: detección y prevención. El costo de las transacciones largas en sistemas MVCC.

**Módulo 09 · Modelado físico e índices**
Tipos de datos reales y su costo. Índices B-tree: estructura interna, búsqueda, inserción, fragmentación. Índices hash, bitmap, GIN y GiST con sus casos de uso. Índices compuestos y el orden de columnas. Index-only scans y covering indexes. Cuándo un índice destruye el rendimiento de escritura. Particionamiento por rango, lista y hash. Fill factor.

---

### Eje 4 — Analítico

**Módulo 10 · OLTP vs OLAP: dos filosofías de diseño**
Por qué los requerimientos son opuestos y se reflejan en el modelo. El data warehouse como capa separada. ETL y ELT. HTAP como intento de unificar ambos mundos y sus compromisos.

**Módulo 11 · Dimensional modeling y Kimball**
La metodología estándar para data warehouses. Hechos y dimensiones. Métricas aditivas, semi-aditivas y no aditivas. Esquema estrella y copo de nieve. Granularidad como la decisión más importante. Slowly Changing Dimensions en contexto analítico. Dimensiones conformadas y la bus matrix. Tablas de hechos sin hechos. Snapshots acumulados.

---

### Eje 5 — Integración

**Módulo 12 · Modelos de datos alternativos**
Documentos, grafos, columnar, series de tiempo, RDF y vectores. Para cada modelo: qué problema resuelve, cuándo el modelo relacional es insuficiente, y cuándo las extensiones del RDBMS (JSONB, pgvector, TimescaleDB) son suficientes sin adoptar un sistema dedicado. Polyglot persistence y el costo de la fragmentación.

**Módulo 13 · Modelado para sistemas distribuidos**
El teorema CAP y PACELC. El espectro de consistencia: linearizability, consistencia causal, read-your-writes, eventual consistency. Replicación síncrona y asíncrona. Split-brain y quórum. CRDTs: estructuras que se fusionan sin conflicto por construcción matemática. Sharding por rango vs. hash. El patrón Saga. Joins distribuidos y por qué el modelado los evita o los hace inevitables.

**Módulo 14 · Patrones de Fowler y la capa de acceso a datos**
Table Data Gateway, Row Data Gateway, Active Record, Data Mapper, Repository, Unit of Work, Identity Map y Lazy Load. Cuándo cada patrón es la elección correcta. Cómo los ORMs implementan automáticamente estos patrones. Los patrones en el contexto de sistemas distribuidos.

**Módulo 15 · ORMs, impedancia objeto-relacional y SQL a mano**
La impedancia objeto-relacional en cinco dimensiones. Herencia en tablas: Table Per Hierarchy, Table Per Type, Table Per Concrete Class y cuándo elegir cada una. El problema N+1: cómo los ORMs lo generan silenciosamente, cómo detectarlo y cómo corregirlo. Lazy loading fuera de sesión. Transacciones implícitas. Cuándo el ORM genera SQL eficiente y cuándo escribir SQL a mano es inevitable.

**Módulo 16 · El oficio: del requerimiento al modelo en producción**
Cómo extraer el modelo de una conversación con el cliente. Cómo evaluar un modelo ajeno y detectar problemas antes de producción. Refactoring de esquemas en sistemas vivos: el patrón Expand/Contract, backfill progresivo, migraciones sin downtime. La tensión entre el modelo correcto y el modelo entregable. Documentar el modelo. El modelo como contrato entre equipos. Row-level security, column masking, auditoría. El modelo de datos como decisión estratégica.

---

## Tecnologías y herramientas

Los ejemplos de código usan **PostgreSQL** como motor de referencia. Algunos módulos hacen referencia a herramientas específicas del ecosistema:

- SQL: PostgreSQL 14+, con extensiones `btree_gist`, `pgcrypto`, `uuid-ossp`, `pg_stat_statements`, `pgvector`, `timescaledb`
- Python: SQLAlchemy (ORM), psycopg2 (driver), Alembic (migraciones)
- Java: Hibernate / JPA
- Herramientas de migración: Flyway, Liquibase
- CDC y pipelines: Debezium, dbt
- Formatos y motores analíticos: Apache Parquet, DuckDB, ClickHouse

---

