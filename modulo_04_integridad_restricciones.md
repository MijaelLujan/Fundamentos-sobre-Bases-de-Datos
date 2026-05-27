# Módulo 4 — Integridad y restricciones como primera clase

> *"La integridad no es un feature del DBMS, es responsabilidad del modelo. Una restricción que vive en el código de aplicación no es una restricción: es una esperanza."*

---

## Tabla de contenidos

1. [El costo de la integridad ausente](#1-el-costo-de-la-integridad-ausente)
2. [Los cuatro tipos de integridad](#2-los-cuatro-tipos-de-integridad)
3. [Integridad de entidad: claves primarias](#3-integridad-de-entidad-claves-primarias)
4. [Integridad referencial: claves foráneas en profundidad](#4-integridad-referencial-claves-foráneas-en-profundidad)
5. [Integridad de dominio](#5-integridad-de-dominio)
6. [Restricciones declarativas vs procedurales](#6-restricciones-declarativas-vs-procedurales)
7. [Restricciones CHECK complejas](#7-restricciones-check-complejas)
8. [El problema de las restricciones entre múltiples tablas](#8-el-problema-de-las-restricciones-entre-múltiples-tablas)
9. [Claves foráneas diferidas vs inmediatas](#9-claves-foráneas-diferidas-vs-inmediatas)
10. [Aserciones: por qué los DBMS las implementan mal](#10-aserciones-por-qué-los-dbms-las-implementan-mal)
11. [Triggers como mecanismo de integridad](#11-triggers-como-mecanismo-de-integridad)
12. [La restricción que el modelo hace imposible vs la que el código debe hacer cumplir](#12-la-restricción-que-el-modelo-hace-imposible-vs-la-que-el-código-debe-hacer-cumplir)
13. [Estrategia integral de integridad](#13-estrategia-integral-de-integridad)
14. [Resumen y conexión con el resto del curso](#14-resumen-y-conexión-con-el-resto-del-curso)

---

## 1. El costo de la integridad ausente

### 1.1 Los datos corruptos son peores que la ausencia de datos

Un sistema que no tiene datos sobre una entidad simplemente no puede responder preguntas sobre ella. Eso es un problema conocido, manejable. Un sistema que tiene datos **incorrectos** sobre una entidad responde preguntas con información falsa, y nadie sabe que la información es falsa. Ese es un problema mucho peor porque es invisible hasta que produce una consecuencia real: una decisión de negocio incorrecta, una factura mal emitida, un diagnóstico basado en el historial del paciente equivocado.

La integridad de datos es la garantía de que los datos en la base de datos son **consistentes con las reglas del dominio** en todo momento. No solo cuando se insertan, sino durante todo su ciclo de vida, ante operaciones concurrentes, ante fallos parciales, ante migraciones, ante cualquier operación que los modifique.

### 1.2 Escenarios reales de integridad rota

**Escenario 1: Pedido sin cliente**

```sql
-- Sin restricción de integridad referencial:
DELETE FROM clientes WHERE cliente_id = 42;
-- Resultado: existen 380 pedidos cuyo cliente_id apunta a un cliente que ya no existe.
-- Las queries que hacen JOIN pedidos-clientes producen resultados silenciosamente incompletos.
-- Los reportes de ventas por cliente omiten esos 380 pedidos.
-- Nadie lo nota durante meses.
```

**Escenario 2: Saldo negativo imposible**

```sql
-- Sin restricción de dominio:
UPDATE cuentas SET saldo = saldo - 1500 WHERE cuenta_id = 7;
-- Si el saldo era 1200, ahora es -300.
-- El negocio dice que los saldos no pueden ser negativos.
-- La restricción está en el código de la aplicación, que alguien en algún momento
-- llamó un endpoint diferente que no tenía esa validación.
```

**Escenario 3: Dos empleados como gerente del mismo departamento**

```sql
-- Sin restricción UNIQUE en la relación 1:1:
INSERT INTO departamentos (dept_id, nombre, gerente_id) VALUES (5, 'Ventas', 23);
-- Más tarde, por error en un script de migración:
UPDATE departamentos SET gerente_id = 31 WHERE dept_id = 5;
-- Pero el registro anterior también quedó con gerente_id = 23 en otra tabla
-- que guardaba el historial. Ahora hay dos "gerentes actuales" de Ventas.
```

Cada uno de estos escenarios tiene una solución técnica disponible en todos los DBMS modernos. Lo que falla no es la herramienta, es el hábito de considerar la integridad como opcional o como "algo que maneja la aplicación".

### 1.3 La pirámide de confianza en la integridad

La integridad de datos puede defenderse en diferentes capas. La confiabilidad de cada capa, de mayor a menor, es:

```
Capa 1 — Modelo físico: tipos de datos, NOT NULL, PRIMARY KEY, UNIQUE, FK
         → El DBMS lo garantiza matemáticamente. No hay excepciones.

Capa 2 — Restricciones CHECK y triggers en el DBMS
         → El DBMS lo garantiza, pero puede ser deshabilitado por administradores.

Capa 3 — Lógica de la capa de servicio (backend)
         → Solo se garantiza si TODA operación pasa por esa capa.
           Un script directo en la BD, un proceso batch, otra aplicación → se salta.

Capa 4 — Validación en el frontend/cliente
         → Se salta con cualquier llamada directa a la API.
           Útil para UX, no para integridad.
```

La integridad real solo existe en las capas 1 y 2. Las capas 3 y 4 son importantes pero insuficientes si no están respaldadas por las capas inferiores.

---

## 2. Los cuatro tipos de integridad

El modelo relacional define cuatro tipos de integridad que deben garantizarse en cualquier base de datos bien diseñada:

### 2.1 Integridad de entidad

**Definición:** Toda relación debe tener una clave primaria. Ningún componente de la clave primaria puede ser NULL.

**Fundamento:** Como vimos en el Módulo 1, una relación es un conjunto de tuplas. Para que ese conjunto sea significativo, cada tupla debe ser identificable unívocamente. Si la clave primaria pudiera ser NULL, tendríamos tuplas sin identidad — objetos que existen pero no pueden ser señalados, referenciados, ni distinguidos entre sí.

**Implementación:** `PRIMARY KEY` en SQL. La restricción `PRIMARY KEY` implica automáticamente `NOT NULL` y `UNIQUE`.

### 2.2 Integridad referencial

**Definición:** Si una tupla contiene una clave foránea, el valor de esa clave foránea debe existir como clave primaria en la tabla referenciada, o ser NULL (si la participación es parcial).

**Fundamento:** Una clave foránea es una referencia a otra entidad. Una referencia que apunta a algo inexistente es una referencia rota — un puntero colgante. El modelo relacional garantiza que las referencias son siempre válidas.

**Implementación:** `FOREIGN KEY ... REFERENCES` en SQL.

### 2.3 Integridad de dominio

**Definición:** Todos los valores de un atributo deben pertenecer al dominio de ese atributo. El dominio define el tipo de dato, el rango de valores válidos, y las restricciones semánticas adicionales.

**Fundamento:** Los dominios son el fundamento matemático del modelo relacional (Módulo 1). Un valor fuera del dominio no es un dato válido para ese atributo.

**Implementación:** Tipos de dato (`INT`, `DATE`, `DECIMAL`), `NOT NULL`, `CHECK` constraints, tipos enumerados, dominios (`CREATE DOMAIN`).

### 2.4 Integridad definida por el usuario (integridad semántica)

**Definición:** Restricciones adicionales que reflejan las reglas de negocio específicas del dominio y que van más allá de los tres tipos anteriores.

**Ejemplos:**
- "El precio de un producto no puede ser negativo."
- "La fecha de fin de un contrato debe ser posterior a la fecha de inicio."
- "Un empleado no puede ser su propio gerente."
- "El saldo total de las cuentas de un cliente no puede exceder su límite de crédito."

**Implementación:** `CHECK` constraints, triggers, aserciones (en teoría), procedimientos almacenados.

---

## 3. Integridad de entidad: claves primarias

### 3.1 Claves naturales vs claves sustitutas

Una **clave natural** es un atributo (o conjunto de atributos) del dominio que identifica unívocamente a cada instancia. Tiene significado semántico en el mundo real.

Ejemplos: número de cédula, código ISBN, número de matrícula de un vehículo, IATA code de un aeropuerto.

Una **clave sustituta** (surrogate key) es un identificador artificialmente generado, sin significado semántico en el dominio. Su único propósito es identificar filas. Típicamente es un entero secuencial o un UUID.

### 3.2 El debate: naturales vs sustitutas

Este debate es antiguo y genuinamente no tiene una respuesta universalmente correcta. Los argumentos son:

**A favor de claves naturales:**
- Tienen significado: `isbn = '978-0-13-468599-1'` es más legible que `libro_id = 38291`.
- Evitan joins innecesarios: no necesitas una tabla extra para "ver" el identificador real.
- Las relaciones entre tablas son más legibles.
- Garantizan que no existan dos registros que representen la misma entidad del mundo real.

**A favor de claves sustitutas:**
- **Estabilidad:** Las claves naturales cambian. Un número de cédula puede corregirse. Un código de producto puede cambiar por rebranding. Un email como identificador de usuario puede cambiar. Una clave sustituta nunca cambia porque no tiene significado externo.
- **Tamaño:** Una clave entera de 4 bytes es mucho más eficiente en índices y claves foráneas que una clave de texto de 20 caracteres. Multiplica eso por millones de filas y decenas de claves foráneas.
- **Uniformidad:** Todas las tablas tienen la misma estructura de clave primaria, lo que simplifica código genérico (ORMs, APIs REST).
- **Independencia:** No estás atado a la unicidad garantizada por un sistema externo. Si el sistema externo cambia sus reglas de unicidad, tu modelo no se rompe.

**La posición pragmática:**

Usar claves sustitutas como clave primaria Y mantener las claves naturales como claves candidatas con restricción `UNIQUE NOT NULL`. Así tienes la estabilidad de la clave sustituta y la integridad semántica de la clave natural.

```sql
CREATE TABLE productos (
    producto_id   SERIAL        PRIMARY KEY,          -- sustituta, estable
    sku           VARCHAR(50)   NOT NULL UNIQUE,       -- natural, también única
    ean           VARCHAR(13)   UNIQUE,                -- otra natural (puede ser null si no aplica)
    nombre        VARCHAR(200)  NOT NULL,
    precio        DECIMAL(10,2) NOT NULL
);
```

### 3.3 SERIAL, IDENTITY, y secuencias

Diferentes DBMS tienen diferentes formas de generar claves sustitutas:

**PostgreSQL:**
```sql
-- Forma antigua (SERIAL es syntax sugar sobre SEQUENCE):
id SERIAL PRIMARY KEY
-- Equivale a:
id INT NOT NULL DEFAULT nextval('tabla_id_seq') PRIMARY KEY

-- Forma moderna (SQL estándar desde PostgreSQL 10):
id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
-- o:
id INT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY
-- La diferencia: ALWAYS no permite insertar valores manuales; BY DEFAULT sí.
```

**SQL Server:**
```sql
id INT IDENTITY(1,1) PRIMARY KEY
-- (valor_inicial, incremento)
```

**MySQL/MariaDB:**
```sql
id INT AUTO_INCREMENT PRIMARY KEY
```

### 3.4 UUIDs como claves primarias: ventajas y costos

Los UUIDs (Universally Unique Identifiers) son identificadores de 128 bits generados de forma que la probabilidad de colisión es prácticamente cero, incluso generados en sistemas distribuidos sin coordinación.

```sql
-- PostgreSQL con extensión uuid-ossp:
id UUID DEFAULT gen_random_uuid() PRIMARY KEY

-- O con pgcrypto:
id UUID DEFAULT gen_random_uuid() PRIMARY KEY
```

**Ventajas:**
- Pueden generarse en el cliente sin coordinación con la base de datos.
- Seguros en sistemas distribuidos: no hay conflictos entre nodos.
- No exponen información sobre el volumen de datos (un `id=38291` revela que hay al menos 38291 registros).

**Desventajas (serias en tablas grandes):**
- **Fragmentación de índice:** Los UUIDs v4 son aleatorios. Cada INSERT inserta en una posición aleatoria del índice B-tree, causando fragmentación severa y degradación del rendimiento en tablas grandes.
- **Tamaño:** 16 bytes vs 4 bytes de un entero. Pequeño por fila, pero se multiplica en cada clave foránea, cada índice que lo incluya, y cada página del índice de la clave primaria.
- **Ilegibilidad:** `3f2504e0-4f89-11d3-9a0c-0305e82c3301` es difícil de usar en depuración.

**La solución:** UUIDs v7 (ordenados por tiempo) o ULID resuelven el problema de fragmentación manteniendo las ventajas de distribución. PostgreSQL 17+ incluye soporte nativo para `gen_random_uuid()` que produce UUIDs v4; para UUIDs v7 ordenados se puede usar extensiones como `pg_uuidv7`.

```sql
-- ULID-style: ordenado por tiempo, evita fragmentación
-- En PostgreSQL con la extensión pgulid:
id TEXT DEFAULT gen_ulid() PRIMARY KEY
```

---

## 4. Integridad referencial: claves foráneas en profundidad

### 4.1 La anatomía de una restricción FOREIGN KEY

```sql
ALTER TABLE pedidos
    ADD CONSTRAINT fk_pedidos_clientes
    FOREIGN KEY (cliente_id)
    REFERENCES clientes (cliente_id)
    ON DELETE RESTRICT
    ON UPDATE CASCADE
    DEFERRABLE INITIALLY IMMEDIATE;
```

Cada parte tiene significado específico:

- **`CONSTRAINT fk_pedidos_clientes`:** Nombre explícito. Siempre nombra tus constraints. Sin nombre, el DBMS genera uno críptico que es imposible de identificar en mensajes de error.
- **`FOREIGN KEY (cliente_id)`:** El atributo (o atributos) en la tabla actual que forma la FK.
- **`REFERENCES clientes (cliente_id)`:** La tabla y el atributo referenciados. El atributo referenciado debe tener una restricción `UNIQUE` o ser `PRIMARY KEY`.
- **`ON DELETE RESTRICT`:** Comportamiento al borrar una fila referenciada.
- **`ON UPDATE CASCADE`:** Comportamiento al actualizar la clave referenciada.
- **`DEFERRABLE INITIALLY IMMEDIATE`:** Cuándo se verifica la restricción.

### 4.2 Acciones referenciales: ON DELETE y ON UPDATE

Cuando se intenta borrar o actualizar una fila referenciada por una clave foránea, el DBMS puede reaccionar de cinco formas:

---

**RESTRICT (o NO ACTION):**

Impide la operación si existe alguna fila que referencia la fila que se intenta borrar/actualizar. La operación falla con un error.

```sql
-- Con ON DELETE RESTRICT en pedidos(cliente_id → clientes):
DELETE FROM clientes WHERE cliente_id = 42;
-- ERROR: update or delete on table "clientes" violates foreign key constraint
-- "fk_pedidos_clientes" on table "pedidos"
-- DETAIL: Key (cliente_id)=(42) is still referenced from table "pedidos".
```

`RESTRICT` y `NO ACTION` son casi idénticos en comportamiento pero difieren en un detalle sutil: `NO ACTION` permite que la violación temporal exista dentro de una transacción (mientras otras operaciones en la misma transacción puedan resolverla), mientras `RESTRICT` verifica inmediatamente. En la práctica, con restricciones `INITIALLY IMMEDIATE`, son equivalentes.

---

**CASCADE:**

Propaga la operación a las filas dependientes.

- `ON DELETE CASCADE`: Al borrar una fila, borra automáticamente todas las filas que la referencian.
- `ON UPDATE CASCADE`: Al actualizar la clave de una fila, actualiza automáticamente todos los valores de clave foránea que la referenciaban.

```sql
-- Con ON DELETE CASCADE en lineas_pedido(pedido_id → pedidos):
DELETE FROM pedidos WHERE pedido_id = 100;
-- Resultado: se borra el pedido Y automáticamente todas sus líneas de pedido.
```

`CASCADE` en `ON DELETE` es correcto para entidades débiles (las líneas de pedido no tienen sentido sin el pedido). Es peligroso para entidades independientes: borrar un cliente y que automáticamente desaparezcan todos sus pedidos, facturas e historial probablemente no es lo que quieres.

`CASCADE` en `ON UPDATE` es generalmente seguro si usas claves sustitutas (que no cambian), y muy útil si usas claves naturales que pueden cambiar (actualizar el código de un aeropuerto propaga automáticamente a todos los vuelos que lo referencian).

---

**SET NULL:**

Cuando se borra/actualiza la fila referenciada, establece la clave foránea en las filas dependientes a NULL.

```sql
-- Con ON DELETE SET NULL en empleados(gerente_id → empleados):
DELETE FROM empleados WHERE empleado_id = 15;  -- era el gerente de varios empleados
-- Resultado: los empleados que tenían gerente_id = 15 ahora tienen gerente_id = NULL.
-- Los empleados no se borran; quedan "sin gerente" hasta reasignación.
```

Solo es aplicable si la columna de clave foránea permite NULL (participación parcial).

---

**SET DEFAULT:**

Establece la clave foránea al valor por defecto de la columna.

```sql
-- Con ON DELETE SET DEFAULT en empleados(departamento_id → departamentos):
-- departamento_id tiene DEFAULT 999 (departamento "Sin Asignar")
DELETE FROM departamentos WHERE departamento_id = 5;
-- Resultado: los empleados del departamento 5 quedan con departamento_id = 999.
```

Poco usado. Requiere que el valor por defecto sea una clave válida en la tabla referenciada.

---

### 4.3 Diseñando acciones referenciales correctas

La elección de la acción referencial no es técnica, es **semántica del dominio**. Las preguntas correctas son:

| Pregunta | Implicación |
|---|---|
| ¿Las filas dependientes tienen sentido sin la fila referenciada? | No → CASCADE o RESTRICT |
| ¿Es una entidad débil? | Sí → CASCADE |
| ¿El borrado debe propagarse o impedirse? | Propagar → CASCADE, Impedir → RESTRICT |
| ¿Las filas dependientes deben "quedar huérfanas" temporalmente? | Sí → SET NULL |
| ¿Hay un valor por defecto semánticamente correcto? | Sí → SET DEFAULT |

**Ejemplo de decisiones correctas:**

```sql
-- Las líneas de pedido SÍ dependen del pedido: CASCADE
FOREIGN KEY (pedido_id) REFERENCES pedidos ON DELETE CASCADE

-- Un pedido NO debería desaparecer si el cliente se borra: RESTRICT
FOREIGN KEY (cliente_id) REFERENCES clientes ON DELETE RESTRICT

-- Un empleado puede quedar sin gerente si el gerente se va: SET NULL
FOREIGN KEY (gerente_id) REFERENCES empleados ON DELETE SET NULL

-- Si se actualiza la PK de un producto (raro), propagar a todos sus registros: CASCADE
FOREIGN KEY (producto_id) REFERENCES productos ON UPDATE CASCADE
```

### 4.4 Claves foráneas compuestas

Cuando la clave primaria referenciada es compuesta, la clave foránea debe tener exactamente las mismas columnas en el mismo orden.

```sql
CREATE TABLE supervisa (
    supervisor_id   INT NOT NULL,
    empleado_id     INT NOT NULL,
    proyecto_id     INT NOT NULL,
    desde           DATE NOT NULL,

    -- FK a la relación de asignación (clave compuesta)
    FOREIGN KEY (empleado_id, proyecto_id)
        REFERENCES asignaciones (empleado_id, proyecto_id)
        ON DELETE CASCADE,

    -- FK simple al supervisor
    FOREIGN KEY (supervisor_id)
        REFERENCES empleados (empleado_id),

    PRIMARY KEY (supervisor_id, empleado_id, proyecto_id)
);
```

### 4.5 El costo de rendimiento de las claves foráneas

Las FK tienen un costo en operaciones de escritura:

- **INSERT en la tabla dependiente:** El DBMS verifica que el valor de la FK exista en la tabla referenciada. Requiere una lectura del índice de la tabla referenciada.
- **DELETE en la tabla referenciada:** El DBMS verifica que no haya filas dependientes (o aplica la acción configurada). Requiere una lectura del índice de la tabla dependiente.
- **UPDATE en la clave referenciada:** Similar al DELETE + INSERT.

Para tablas con escrituras masivas (bulk inserts de millones de filas), a veces se deshabilitan temporalmente las FKs, se hace el insert, y se re-habilitan. Esto es aceptable en procesos ETL controlados, nunca en sistemas OLTP.

---

## 5. Integridad de dominio

### 5.1 Tipos de dato como primera línea de defensa

El tipo de dato de una columna es la restricción de dominio más básica. Definirlo correctamente no es trivial:

**Enteros:**
```sql
SMALLINT    -- 2 bytes, -32768 a 32767
INT         -- 4 bytes, -2,147,483,648 a 2,147,483,647
BIGINT      -- 8 bytes, hasta ~9.2 quintillones

-- No usar INT para IDs de tablas que podrían crecer a más de 2 mil millones de filas.
-- Es un error común que requiere una migración costosa cuando la tabla crece.
-- Usar BIGINT (o BIGSERIAL) para tablas con alta tasa de inserción desde el inicio.
```

**Decimales y dinero:**
```sql
-- MAL: usar FLOAT o DOUBLE para dinero
precio FLOAT   -- los flotantes no pueden representar exactamente 0.10 en binario
               -- 0.1 + 0.2 = 0.30000000000000004 en punto flotante

-- BIEN: usar DECIMAL/NUMERIC para valores monetarios
precio DECIMAL(10, 2)  -- hasta 99,999,999.99 con exactamente 2 decimales
-- (precisión_total, dígitos_después_del_punto)

-- O el tipo MONEY en PostgreSQL (aunque tiene sus propias limitaciones)
```

**Fechas y tiempos:**
```sql
DATE          -- solo fecha (año, mes, día)
TIME          -- solo hora (sin zona horaria)
TIMESTAMP     -- fecha y hora (sin zona horaria)
TIMESTAMPTZ   -- fecha y hora CON zona horaria (internamente en UTC)

-- Para sistemas con usuarios en múltiples zonas horarias: SIEMPRE TIMESTAMPTZ
-- Para fechas de nacimiento, fechas de eventos "del mundo real": DATE
-- Para horas de apertura de un negocio local: TIME
```

**Texto:**
```sql
CHAR(n)       -- longitud fija, rellena con espacios. Rara vez correcto.
VARCHAR(n)    -- longitud variable hasta n caracteres. El más común.
TEXT          -- longitud ilimitada. Sin overhead vs VARCHAR en PostgreSQL.

-- En PostgreSQL, VARCHAR sin límite y TEXT son equivalentes en rendimiento.
-- Poner un límite (VARCHAR(100)) solo si el negocio tiene una restricción real.
-- Poner CHAR(2) para códigos de país ISO es correcto. Poner CHAR(50) para nombres es incorrecto.
```

### 5.2 NOT NULL: el constraint más importante y más ignorado

`NOT NULL` expresa que un atributo tiene **siempre** un valor. Como discutimos en el Módulo 1, NULL tiene consecuencias teóricas y prácticas graves. La regla debería ser:

> **Todas las columnas son `NOT NULL` por defecto. Solo se permite NULL cuando existe una razón semántica real y documentada.**

```sql
-- MAL: NOT NULL solo cuando es "obvio"
CREATE TABLE empleados (
    empleado_id INT PRIMARY KEY,
    nombre      VARCHAR(100),       -- debería ser NOT NULL: ¿puede existir sin nombre?
    email       VARCHAR(100),       -- depende: ¿es obligatorio tener email?
    gerente_id  INT,                -- OK ser nullable: un empleado puede no tener gerente
    salario     DECIMAL(10,2)       -- debería ser NOT NULL para empleados activos
);

-- BIEN: explícito sobre cada decisión
CREATE TABLE empleados (
    empleado_id INT          PRIMARY KEY,
    nombre      VARCHAR(100) NOT NULL,    -- siempre requerido
    email       VARCHAR(100) NOT NULL,    -- política de empresa: email obligatorio
    gerente_id  INT,                      -- NULL para el CEO u otros sin gerente
    salario     DECIMAL(10,2) NOT NULL    -- siempre tiene un salario si es empleado activo
);
```

### 5.3 Restricciones UNIQUE

`UNIQUE` garantiza que no haya dos filas con el mismo valor en esa columna (o combinación de columnas). Implementa las claves candidatas que no son la clave primaria.

```sql
CREATE TABLE usuarios (
    usuario_id  SERIAL        PRIMARY KEY,
    username    VARCHAR(50)   NOT NULL UNIQUE,        -- clave candidata
    email       VARCHAR(100)  NOT NULL UNIQUE,        -- otra clave candidata
    -- ...
);

-- UNIQUE compuesto: la combinación debe ser única
CREATE TABLE asignaciones_rol (
    usuario_id  INT NOT NULL REFERENCES usuarios,
    rol_id      INT NOT NULL REFERENCES roles,
    -- Un usuario solo puede tener un rol específico una vez
    UNIQUE (usuario_id, rol_id)
);
```

**UNIQUE y NULL:** Como discutimos en el Módulo 1, en SQL estándar múltiples NULLs en una columna UNIQUE son permitidos porque `NULL ≠ NULL`. PostgreSQL permite múltiples NULLs. SQL Server solo permite uno. Esto es una fuente de comportamiento inconsistente entre DBMS.

Si necesitas "único cuando no es nulo" y solo un NULL permitido, hay soluciones específicas por motor:

```sql
-- PostgreSQL: índice parcial (más flexible que UNIQUE constraint)
CREATE UNIQUE INDEX idx_empleados_numero_ext
    ON empleados (numero_extension)
    WHERE numero_extension IS NOT NULL;
-- Resultado: múltiples NULL permitidos, pero todos los no-NULL deben ser únicos
```

---

## 6. Restricciones declarativas vs procedurales

### 6.1 La jerarquía de restricciones

Antes de ver `CHECK` constraints complejos y triggers, es importante entender la jerarquía en el poder expresivo y el costo de cada mecanismo:

```
DECLARATIVAS (en el esquema, verificadas por el DBMS automáticamente):
  PRIMARY KEY → garantiza unicidad e identidad de tupla
  NOT NULL    → garantiza presencia de valor
  UNIQUE      → garantiza unicidad de valor
  FOREIGN KEY → garantiza integridad referencial
  CHECK       → garantiza condición arbitraria sobre la fila

PROCEDURALES (código ejecutado por el DBMS en respuesta a eventos):
  Triggers    → lógica arbitraria ante INSERT/UPDATE/DELETE
  Procedimientos almacenados → lógica arbitraria invocada explícitamente

EXTERNAS (en la capa de aplicación):
  Validaciones en la capa de servicio
  Validaciones en el cliente/frontend
```

### 6.2 Por qué las declarativas son superiores

Las restricciones declarativas son superiores a las procedurales por varias razones fundamentales:

**1. Son imposibles de eludir:**
Una restricción `CHECK` siempre se verifica, sin importar quién hace la operación (aplicación, script, herramienta de administración, migración, bulk insert). Un trigger puede deshabilitarse. Una validación en la aplicación se puede bypasear.

**2. Son autodocumentadas:**
El esquema describe las restricciones del dominio. Cualquier persona que lea el DDL entiende las reglas sin necesidad de leer el código de la aplicación.

**3. Son optimizables:**
El query planner puede usar las restricciones declarativas para optimizar queries (p.ej., saber que una columna nunca es NULL permite usar joins más eficientes, o que un valor siempre está en cierto rango permite usar particiones).

**4. Son atómicas respecto a la transacción:**
Se verifican como parte de la transacción. Si fallan, la transacción completa se revierte. No hay ventanas de inconsistencia.

**5. No tienen overhead de lógica compleja:**
Una restricción `NOT NULL` es verificada en microsegundos. Un trigger que ejecuta lógica arbitraria tiene el overhead de ejecutar código en el motor.

---

## 7. Restricciones CHECK complejas

### 7.1 La anatomía de un CHECK constraint

```sql
-- CHECK a nivel de columna (solo puede referenciar esa columna)
CREATE TABLE productos (
    precio DECIMAL(10,2) NOT NULL CHECK (precio > 0)
);

-- CHECK a nivel de tabla (puede referenciar múltiples columnas de la misma fila)
CREATE TABLE contratos (
    fecha_inicio DATE NOT NULL,
    fecha_fin    DATE,
    CONSTRAINT chk_fechas_contrato
        CHECK (fecha_fin IS NULL OR fecha_fin > fecha_inicio)
);
```

**Siempre nombra tus CHECK constraints.** El nombre aparece en el mensaje de error. `check_contratos_chk_fechas_contrato` es infinitamente más útil para depurar que `contratos_check1`.

### 7.2 Expresiones válidas en CHECK

Un CHECK puede contener cualquier expresión que devuelva `TRUE`, `FALSE`, o `UNKNOWN`. La restricción se viola solo cuando la expresión devuelve explícitamente `FALSE`. Si devuelve `UNKNOWN` (p.ej., comparaciones con NULL), la restricción **no se viola**.

```sql
-- Este CHECK permite NULL en descuento (devuelve UNKNOWN, no FALSE):
CHECK (descuento >= 0 AND descuento <= 100)
-- Si descuento es NULL: NULL >= 0 → UNKNOWN, no FALSE → el CHECK pasa.
-- Si quieres que NULL también sea rechazado:
CHECK (descuento IS NOT NULL AND descuento >= 0 AND descuento <= 100)
-- O simplemente: columna DECIMAL NOT NULL CHECK (descuento >= 0 AND descuento <= 100)
```

### 7.3 Casos prácticos de CHECK constraints

**Rangos de valores:**
```sql
-- Porcentaje entre 0 y 100
tasa_impuesto DECIMAL(5,2) NOT NULL CHECK (tasa_impuesto >= 0 AND tasa_impuesto <= 100),

-- Calificación de 1 a 10
calificacion INT CHECK (calificacion BETWEEN 1 AND 10),

-- Temperatura en grados Celsius razonables para un sistema de climatización
temperatura_objetivo DECIMAL(4,1) NOT NULL CHECK (temperatura_objetivo BETWEEN -10 AND 50)
```

**Enumeraciones (cuando no usas un tipo ENUM):**
```sql
estado_pedido VARCHAR(20) NOT NULL
    CHECK (estado_pedido IN ('PENDIENTE', 'CONFIRMADO', 'EN_PROCESO', 'ENVIADO', 'ENTREGADO', 'CANCELADO')),

tipo_cuenta VARCHAR(20) NOT NULL
    CHECK (tipo_cuenta IN ('CORRIENTE', 'AHORRO', 'INVERSION'))
```

**Consistencia entre columnas de la misma fila:**
```sql
CREATE TABLE empleados (
    empleado_id     INT          PRIMARY KEY,
    fecha_contrato  DATE         NOT NULL,
    fecha_termino   DATE,
    tipo_contrato   VARCHAR(20)  NOT NULL CHECK (tipo_contrato IN ('PLANTA', 'TEMPORAL', 'PRACTICANTE')),

    CONSTRAINT chk_termino_posterior
        CHECK (fecha_termino IS NULL OR fecha_termino > fecha_contrato),

    CONSTRAINT chk_temporales_tienen_termino
        CHECK (tipo_contrato = 'PLANTA' OR fecha_termino IS NOT NULL)
        -- Los empleados de planta no tienen fecha_termino, los temporales sí la deben tener
);
```

**Dependencias condicionales entre columnas:**
```sql
CREATE TABLE vehiculos (
    vehiculo_id     INT          PRIMARY KEY,
    tipo            VARCHAR(20)  NOT NULL CHECK (tipo IN ('CAR', 'TRUCK', 'MOTORCYCLE')),
    num_ejes        INT,
    capacidad_carga DECIMAL(8,2),

    CONSTRAINT chk_camion_requiere_ejes
        CHECK (tipo != 'TRUCK' OR (num_ejes IS NOT NULL AND num_ejes >= 2)),

    CONSTRAINT chk_solo_camiones_tienen_carga
        CHECK (tipo = 'TRUCK' OR capacidad_carga IS NULL)
);
```

**Formato de datos (con expresiones regulares en PostgreSQL):**
```sql
-- Código postal colombiano (6 dígitos)
codigo_postal VARCHAR(10) CHECK (codigo_postal ~ '^\d{6}$'),

-- Email básico
email VARCHAR(100) CHECK (email ~ '^[^@]+@[^@]+\.[^@]+$'),

-- Placa colombiana (formato ABC-123 o ABC-12D)
placa VARCHAR(10) CHECK (placa ~ '^[A-Z]{3}-[0-9]{3}$' OR placa ~ '^[A-Z]{3}-[0-9]{2}[A-Z]$')
```

### 7.4 Limitaciones de CHECK

Los CHECK constraints tienen limitaciones importantes que determinan cuándo son suficientes y cuándo necesitas algo más poderoso:

**No pueden referenciar otras tablas:**
```sql
-- INVÁLIDO en todos los DBMS:
CHECK (categoria_id IN (SELECT categoria_id FROM categorias))
-- Los CHECK solo pueden ver los valores de la fila actual, no hacer subconsultas.
-- Para esto se necesita una FK o un trigger.
```

**No pueden referenciar otras filas de la misma tabla:**
```sql
-- INVÁLIDO: no puedes comparar con otras filas de la misma tabla
CHECK (salario <= (SELECT MAX(salario) FROM empleados WHERE cargo = 'SENIOR'))
-- Para esto se necesita un trigger o una aserción.
```

**No tienen acceso al estado previo de la fila:**
```sql
-- INVÁLIDO: no puedes comparar el valor nuevo con el valor anterior
CHECK (nuevo_estado != estado_anterior)  -- no funciona así
-- Para esto se necesita un trigger con OLD y NEW.
```

---

## 8. El problema de las restricciones entre múltiples tablas

### 8.1 El núcleo del problema

Una gran clase de restricciones de integridad del mundo real involucra **múltiples tablas**. Los CHECK constraints no pueden alcanzarlas. Las claves foráneas solo garantizan existencia, no condiciones complejas. Esto deja un vacío que los DBMS modernos resuelven de forma imperfecta.

### 8.2 Categorías de restricciones multi-tabla

**Tipo 1: Restricciones de cardinalidad máxima**

"Un empleado puede estar asignado a un máximo de 3 proyectos simultáneamente."

```sql
-- No hay constraint declarativo para esto. Requiere trigger:
CREATE OR REPLACE FUNCTION verificar_max_proyectos()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT COUNT(*) FROM asignaciones
        WHERE empleado_id = NEW.empleado_id
          AND activa = TRUE) >= 3 THEN
        RAISE EXCEPTION 'El empleado % ya tiene 3 proyectos activos', NEW.empleado_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_max_proyectos
    BEFORE INSERT ON asignaciones
    FOR EACH ROW EXECUTE FUNCTION verificar_max_proyectos();
```

**Tipo 2: Restricciones de suma o agregación**

"El total de horas asignadas a todos los empleados en un proyecto no puede exceder el presupuesto de horas del proyecto."

```sql
CREATE OR REPLACE FUNCTION verificar_horas_proyecto()
RETURNS TRIGGER AS $$
DECLARE
    total_horas INT;
    max_horas   INT;
BEGIN
    SELECT SUM(horas_semanales * 52) INTO total_horas  -- horas anuales aprox.
    FROM asignaciones
    WHERE proyecto_id = NEW.proyecto_id;

    SELECT presupuesto_horas INTO max_horas
    FROM proyectos WHERE proyecto_id = NEW.proyecto_id;

    IF total_horas > max_horas THEN
        RAISE EXCEPTION 'El proyecto % excede su presupuesto de horas (% > %)',
            NEW.proyecto_id, total_horas, max_horas;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Tipo 3: Restricciones de jerarquía o grafo**

"Un empleado no puede ser su propio gerente, ni gerente de su gerente (sin ciclos en la jerarquía)."

```sql
-- La verificación de acíclico es especialmente compleja:
CREATE OR REPLACE FUNCTION verificar_sin_ciclo_jerarquia()
RETURNS TRIGGER AS $$
DECLARE
    current_id INT;
BEGIN
    current_id := NEW.gerente_id;

    WHILE current_id IS NOT NULL LOOP
        IF current_id = NEW.empleado_id THEN
            RAISE EXCEPTION 'Ciclo detectado en jerarquía: empleado % no puede ser '
                'antecesor de sí mismo', NEW.empleado_id;
        END IF;
        SELECT gerente_id INTO current_id
        FROM empleados WHERE empleado_id = current_id;
    END LOOP;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_sin_ciclo
    BEFORE INSERT OR UPDATE OF gerente_id ON empleados
    FOR EACH ROW EXECUTE FUNCTION verificar_sin_ciclo_jerarquia();
```

**Tipo 4: Restricciones de exclusión mutua**

"Una persona puede ser cliente o proveedor, pero no ambas cosas en la misma empresa."

Esto es especialmente difícil de expresar declarativamente. En PostgreSQL, los índices de exclusión (`EXCLUDE`) permiten algunas de estas restricciones:

```sql
-- Usando EXCLUDE para rangos temporales sin solapamiento:
-- "Un mismo empleado no puede estar asignado a dos proyectos en el mismo período"
CREATE TABLE asignaciones_tiempo (
    empleado_id INT NOT NULL,
    proyecto_id INT NOT NULL,
    periodo     TSRANGE NOT NULL,  -- rango de timestamp
    EXCLUDE USING GIST (empleado_id WITH =, periodo WITH &&)
    -- && significa "se superpone"
);
```

### 8.3 El patrón de tabla de control para restricciones complejas

Para ciertas restricciones multi-tabla, una solución elegante es agregar una tabla de "estado de control" que centraliza la restricción:

```sql
-- Restricción: "Solo puede haber un director activo por departamento en cualquier momento"
-- En lugar de un trigger complejo, usar una tabla de control con UNIQUE:

CREATE TABLE directores_actuales (
    departamento_id INT  PRIMARY KEY REFERENCES departamentos,
    empleado_id     INT  NOT NULL REFERENCES empleados,
    desde           DATE NOT NULL
);
-- La PK en departamento_id garantiza que solo hay un registro por departamento.
-- Para cambiar de director: DELETE el registro anterior, INSERT el nuevo.
-- La restricción es declarativa (PK), no procedimental.
```

---

## 9. Claves foráneas diferidas vs inmediatas

### 9.1 El problema del bootstrapping circular

Algunas estructuras de datos tienen dependencias circulares que hacen imposible insertar datos en el "orden correcto". El caso más común es una jerarquía autorefenciada:

```sql
CREATE TABLE categorias (
    categoria_id INT PRIMARY KEY,
    nombre       VARCHAR(100) NOT NULL,
    padre_id     INT REFERENCES categorias(categoria_id)  -- autoreferencia
);

-- Para insertar la categoría raíz (sin padre), no hay problema:
INSERT INTO categorias VALUES (1, 'Electrónica', NULL);

-- Para insertar una subcategoría, el padre ya existe:
INSERT INTO categorias VALUES (2, 'Smartphones', 1);  -- OK, 1 existe

-- Pero ¿qué pasa si necesitas insertar dos categorías que se referencian mutuamente?
-- (Raro pero posible en ciertos modelos de grafo o en datos de migración)
```

Un problema más común es entre dos tablas con dependencias mutuas:

```sql
-- Empleados referencian departamentos (su departamento asignado)
-- Departamentos referencian empleados (su director)

CREATE TABLE departamentos (
    dept_id    INT PRIMARY KEY,
    nombre     VARCHAR(100),
    director_id INT REFERENCES empleados(empleado_id)  -- FK a empleados
);

CREATE TABLE empleados (
    empleado_id  INT PRIMARY KEY,
    nombre       VARCHAR(100),
    dept_id      INT REFERENCES departamentos(dept_id)  -- FK a departamentos
);

-- ¿Cómo insertar el primer departamento si necesita un director,
-- y el director necesita un departamento?
```

### 9.2 La solución: restricciones diferidas (DEFERRABLE)

Una restricción **diferida** no se verifica en el momento de la operación individual, sino al **final de la transacción** (cuando se hace `COMMIT`). Esto permite que el estado de la base de datos sea transitoriamente inconsistente dentro de una transacción, siempre que sea consistente al final.

```sql
-- Declarar la FK como diferible (puede diferirse, pero por defecto es inmediata)
ALTER TABLE departamentos
    ADD CONSTRAINT fk_dept_director
    FOREIGN KEY (director_id) REFERENCES empleados(empleado_id)
    DEFERRABLE INITIALLY IMMEDIATE;

-- O declarar que siempre se difiere:
ALTER TABLE departamentos
    ADD CONSTRAINT fk_dept_director
    FOREIGN KEY (director_id) REFERENCES empleados(empleado_id)
    DEFERRABLE INITIALLY DEFERRED;
```

Las opciones son:

| Opción | Comportamiento |
|---|---|
| `NOT DEFERRABLE` | Siempre se verifica inmediatamente. No puede cambiarse. |
| `DEFERRABLE INITIALLY IMMEDIATE` | Por defecto inmediata, pero puede diferirse con `SET CONSTRAINTS DEFERRED` dentro de una transacción. |
| `DEFERRABLE INITIALLY DEFERRED` | Por defecto diferida: se verifica solo al hacer COMMIT. |

### 9.3 Usando restricciones diferidas para el bootstrapping circular

```sql
-- Ambas FKs declaradas como diferibles:
ALTER TABLE departamentos
    ADD CONSTRAINT fk_dept_director
    FOREIGN KEY (director_id) REFERENCES empleados(empleado_id)
    DEFERRABLE INITIALLY DEFERRED;

ALTER TABLE empleados
    ADD CONSTRAINT fk_emp_departamento
    FOREIGN KEY (dept_id) REFERENCES departamentos(dept_id)
    DEFERRABLE INITIALLY DEFERRED;

-- Ahora podemos insertar con un orden que temporalmente viola las restricciones:
BEGIN;
    -- Insertar departamento sin director todavía (director_id = NULL temporalmente)
    INSERT INTO departamentos VALUES (1, 'Ingeniería', NULL);

    -- Insertar el empleado que será director (ya existe el departamento)
    INSERT INTO empleados VALUES (10, 'Ana García', 1);

    -- Actualizar el director del departamento
    UPDATE departamentos SET director_id = 10 WHERE dept_id = 1;

COMMIT;
-- En el COMMIT se verifican las FKs diferidas. Todo es consistente → OK.
```

### 9.4 Cuándo diferir y cuándo no

Las restricciones diferidas tienen un costo: el DBMS debe mantener una lista de verificaciones pendientes y ejecutarlas todas al hacer COMMIT, lo que puede hacer más lentos los commits de transacciones largas. Además, los errores de integridad aparecen al final, no en la operación que los causó, lo que dificulta el diagnóstico.

**Usar `DEFERRABLE INITIALLY DEFERRED` cuando:**
- Hay dependencias circulares inevitables.
- Se hacen migraciones de datos donde el orden de inserción no puede garantizarse.
- Se trabaja con datos que se autoreferencian (árboles, grafos).

**Usar `DEFERRABLE INITIALLY IMMEDIATE` cuando:**
- La restricción normalmente debe ser inmediata pero puede ser necesario diferirla excepcionalmente en ciertos procesos.

**Mantener `NOT DEFERRABLE` para:**
- La mayoría de las restricciones en sistemas OLTP. La verificación inmediata hace los errores más fáciles de diagnosticar.

---

## 10. Aserciones: por qué los DBMS las implementan mal

### 10.1 Qué es una aserción

El estándar SQL define las **aserciones** (`CREATE ASSERTION`) como restricciones de integridad que pueden abarcar **múltiples tablas** y **múltiples filas**, expresadas de forma completamente declarativa:

```sql
-- Sintaxis estándar SQL (teórica):
CREATE ASSERTION max_empleados_por_proyecto
    CHECK (
        NOT EXISTS (
            SELECT proyecto_id
            FROM asignaciones
            GROUP BY proyecto_id
            HAVING COUNT(*) > 20
        )
    );
```

Una aserción se verifica automáticamente ante cualquier operación que pueda afectar su resultado. Si la aserción falla, la operación se revierte.

### 10.2 Por qué los DBMS no las implementan

Las aserciones son computacionalmente costosas porque el DBMS debe:

1. Determinar qué operaciones pueden afectar el resultado de cada aserción.
2. Ejecutar la subquery de verificación después de cada una de esas operaciones.
3. Hacerlo de forma eficiente incluso con millones de filas.

El problema es que la verificación de una aserción general puede requerir escanear tablas enteras. Una aserción como "el saldo total de todas las cuentas de un banco debe ser positivo" requiere sumar todos los saldos de todas las cuentas después de cada transacción. Con millones de cuentas y miles de transacciones por segundo, eso es inviable.

**Estado actual de implementación:**

| DBMS | Soporte de aserciones |
|---|---|
| PostgreSQL | No implementado |
| MySQL/MariaDB | No implementado |
| SQL Server | No implementado |
| Oracle | No implementado |
| DB2 (IBM) | Implementado parcialmente |
| SQLite | No implementado |

**Las alternativas en la práctica:**

Las aserciones se implementan mediante:
1. **Triggers:** Más flexibles, pero procedurales y no declarativos.
2. **Vistas materializadas con constraints:** En algunos casos se puede materializar el resultado de la aserción y ponerle un CHECK.
3. **Procedimientos de verificación periódica:** Aceptable para sistemas donde la consistencia eventual es tolerable.

```sql
-- Alternativa para "ningún proyecto puede tener más de 20 empleados asignados":
-- Trigger que se dispara en INSERT sobre asignaciones:

CREATE OR REPLACE FUNCTION check_max_asignaciones()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT COUNT(*) FROM asignaciones WHERE proyecto_id = NEW.proyecto_id) > 20 THEN
        RAISE EXCEPTION 'El proyecto % ya tiene 20 empleados asignados', NEW.proyecto_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_check_max_asignaciones
    BEFORE INSERT ON asignaciones
    FOR EACH ROW EXECUTE FUNCTION check_max_asignaciones();
```

---

## 11. Triggers como mecanismo de integridad

### 11.1 Cuándo los triggers son la herramienta correcta

Los triggers son código que el DBMS ejecuta automáticamente en respuesta a un evento (`INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`). Son la herramienta correcta para integridad cuando:

- La restricción involucra múltiples tablas o múltiples filas.
- La restricción requiere comparar el estado anterior y el nuevo de una fila.
- La restricción requiere lógica condicional compleja.
- Se necesita una acción compensatoria (no solo rechazar la operación, sino modificar datos relacionados).

### 11.2 Tipos de triggers

**Por momento de ejecución:**
- `BEFORE`: Se ejecuta antes de la operación. Puede modificar los valores que se van a insertar/actualizar, o cancelar la operación con un error.
- `AFTER`: Se ejecuta después de la operación (los cambios ya están en la tabla, pero la transacción no se ha confirmado). Puede acceder a los valores finales.
- `INSTEAD OF`: Solo para vistas. Reemplaza la operación por otra lógica.

**Por nivel:**
- `FOR EACH ROW`: Se ejecuta una vez por cada fila afectada.
- `FOR EACH STATEMENT`: Se ejecuta una vez por sentencia SQL, sin importar cuántas filas afecta.

### 11.3 Un ejemplo completo: auditoría de cambios

```sql
-- Tabla de auditoría
CREATE TABLE auditoria_salarios (
    auditoria_id    SERIAL       PRIMARY KEY,
    empleado_id     INT          NOT NULL,
    salario_anterior DECIMAL(10,2),
    salario_nuevo    DECIMAL(10,2),
    modificado_por  VARCHAR(100) DEFAULT CURRENT_USER,
    modificado_en   TIMESTAMPTZ  DEFAULT NOW()
);

-- Función del trigger
CREATE OR REPLACE FUNCTION registrar_cambio_salario()
RETURNS TRIGGER AS $$
BEGIN
    -- Solo actuar si el salario realmente cambió
    IF OLD.salario IS DISTINCT FROM NEW.salario THEN
        INSERT INTO auditoria_salarios
            (empleado_id, salario_anterior, salario_nuevo)
        VALUES
            (NEW.empleado_id, OLD.salario, NEW.salario);
    END IF;
    RETURN NEW;  -- En BEFORE triggers, RETURN NEW continúa con la operación
END;
$$ LANGUAGE plpgsql;

-- El trigger
CREATE TRIGGER tg_auditoria_salario
    AFTER UPDATE OF salario ON empleados
    FOR EACH ROW
    EXECUTE FUNCTION registrar_cambio_salario();
```

### 11.4 Un ejemplo completo: restricción de integridad compleja

```sql
-- Restricción: "El stock de un producto no puede quedar en negativo"
-- El stock se calcula como stock_inicial + entradas - salidas
-- Esta restricción cruza tres tablas: productos, entradas_inventario, salidas_inventario

CREATE OR REPLACE FUNCTION verificar_stock_disponible()
RETURNS TRIGGER AS $$
DECLARE
    stock_actual INT;
BEGIN
    SELECT
        p.stock_inicial
        + COALESCE(SUM(e.cantidad) FILTER (WHERE e.tipo = 'ENTRADA'), 0)
        - COALESCE(SUM(e.cantidad) FILTER (WHERE e.tipo = 'SALIDA'), 0)
    INTO stock_actual
    FROM productos p
    LEFT JOIN movimientos_inventario e ON p.producto_id = e.producto_id
    WHERE p.producto_id = NEW.producto_id;

    IF NEW.tipo = 'SALIDA' AND stock_actual - NEW.cantidad < 0 THEN
        RAISE EXCEPTION
            'Stock insuficiente para el producto %. Stock actual: %, Solicitado: %',
            NEW.producto_id, stock_actual, NEW.cantidad;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_verificar_stock
    BEFORE INSERT ON movimientos_inventario
    FOR EACH ROW
    WHEN (NEW.tipo = 'SALIDA')
    EXECUTE FUNCTION verificar_stock_disponible();
```

### 11.5 Los peligros de los triggers

Los triggers son poderosos pero tienen riesgos serios que justifican usarlos con precaución:

**Invisibilidad:** Un trigger no aparece en la query. Quien escribe `INSERT INTO pedidos ...` puede no saber que ese insert dispara 3 triggers que modifican otras 4 tablas. El comportamiento del sistema es difícil de razonar.

**Cascadas inesperadas:** Un trigger puede disparar operaciones que disparan otros triggers (triggers en cadena). En sistemas complejos, esto puede ser difícil de seguir y puede causar problemas de rendimiento o ciclos infinitos.

**Problemas de rendimiento:** Un trigger `FOR EACH ROW` que ejecuta una query compleja se ejecuta una vez por fila afectada. Un `UPDATE` que modifica 100,000 filas dispara el trigger 100,000 veces.

**Dificultad de testing:** Los triggers son lógica en la base de datos, no en el código de aplicación. Las suites de prueba típicas de la aplicación pueden no cubrirlos.

**Regla práctica:** Usar triggers solo cuando:
1. No hay solución declarativa equivalente.
2. La restricción es fundamentalmente del dominio, no de la aplicación.
3. La lógica es simple y bien documentada.
4. El equipo es consciente de su existencia y está en el proceso de revisión de código.

---

## 12. La restricción que el modelo hace imposible vs la que el código debe hacer cumplir

### 12.1 La diferencia fundamental

Esta es la distinción de diseño más importante del módulo. Hay dos categorías de restricciones:

**Restricciones que el modelo hace imposibles de violar:**
El estado de la base de datos nunca puede estar en violación de esta restricción, por construcción. El DBMS garantiza esto sin importar quién, cómo, o cuándo opere sobre los datos.

**Restricciones que el código de aplicación debe hacer cumplir:**
El estado de la base de datos puede violar esta restricción si alguien opera directamente sobre ella, si un proceso batch incorrecto corre, si hay un bug en la aplicación, o si alguien usa una herramienta de administración. La restricción es una "esperanza" que depende de que todo el código que toca la base de datos sea correcto.

### 12.2 El espectro de garantías

```
GARANTÍA MÁXIMA ←────────────────────────────────────────────────────────→ GARANTÍA MÍNIMA

PRIMARY KEY   FOREIGN KEY   NOT NULL   UNIQUE   CHECK   Trigger   Código de aplicación   Documentación
```

Cuanto más a la izquierda está una restricción, más fuerte es la garantía. Cuanto más a la derecha, mayor es el riesgo de que la restricción sea violada.

### 12.3 El análisis correcto para cada restricción

Para cada regla de negocio identificada, hay que hacerse estas preguntas:

**¿Puede esta restricción expresarse como PRIMARY KEY, UNIQUE, NOT NULL, o FOREIGN KEY?**
Si sí → implementarla así. Es la garantía más fuerte posible.

**¿Puede expresarse como CHECK constraint?**
Si sí → implementarla así. Es declarativa y el DBMS la garantiza.

**¿Involucra múltiples tablas o múltiples filas?**
Si sí → trigger en el DBMS. Procedural pero al menos está en la capa de datos.

**¿Es demasiado compleja para un trigger o tiene implicaciones de rendimiento severas?**
Si sí → validación en la capa de servicio + tests de integridad periódicos. Documentar explícitamente que es una restricción no garantizada por el modelo.

### 12.4 Ejemplos clasificados

```
Regla: "Todo pedido debe tener un cliente."
→ FK NOT NULL: pedidos.cliente_id NOT NULL REFERENCES clientes
→ GARANTÍA MÁXIMA. El modelo hace imposible un pedido sin cliente.

Regla: "El precio de un producto no puede ser negativo."
→ CHECK: precio DECIMAL NOT NULL CHECK (precio >= 0)
→ Garantía fuerte. El DBMS lo rechaza en cualquier operación.

Regla: "La fecha de entrega debe ser posterior a la fecha del pedido."
→ CHECK: CHECK (fecha_entrega IS NULL OR fecha_entrega >= fecha_pedido)
→ Garantía fuerte para la misma fila.

Regla: "Un cliente no puede tener más de 5 pedidos activos simultáneamente."
→ Trigger BEFORE INSERT en pedidos.
→ Garantía media. El trigger puede deshabilitarse, tiene overhead.

Regla: "El descuento total de una factura no puede superar el límite de descuento
        aprobado para el vendedor que la emitió."
→ Trigger + posiblemente validación en servicio.
→ La lógica es compleja; el trigger puede tener problemas de rendimiento.

Regla: "Los reportes de ventas deben mostrar solo datos validados por el gerente."
→ Lógica de aplicación (filtro en queries).
→ No es una restricción de integridad del modelo: es lógica de negocio.
   No necesita estar en el DBMS.
```

### 12.5 El costo de poner restricciones en el código de aplicación

Cuando una restricción de integridad vive solo en el código de aplicación, el riesgo es proporcional a:

- **El número de puntos de entrada a los datos:** 1 aplicación = riesgo bajo. 1 aplicación + API pública + scripts de migración + acceso directo de DBA = riesgo muy alto.
- **La criticidad de los datos:** Una restricción violada en un sistema de preferencias de usuario es molesta. Una restricción violada en un sistema financiero o médico puede tener consecuencias legales o de seguridad.
- **La vida del sistema:** En un sistema nuevo con un equipo pequeño, las restricciones en la aplicación son manejables. En un sistema de 10 años con 5 equipos distintos que lo han tocado, el código de restricción está disperso, parcialmente olvidado, y probablemente inconsistente.

---

## 13. Estrategia integral de integridad

### 13.1 El proceso de análisis de restricciones

Cuando diseñas un esquema, el análisis de integridad debe ser parte del proceso, no un paso opcional al final. La secuencia correcta es:

```
1. Para cada entidad identificada en el modelo EER:
   a. ¿Tiene clave natural? → UNIQUE NOT NULL en la columna
   b. ¿Qué columnas son obligatorias? → NOT NULL en todas ellas
   c. ¿Qué restricciones de dominio tiene cada columna? → Tipos de dato + CHECK

2. Para cada relación identificada en el modelo EER:
   a. ¿Cuál es la cardinalidad máxima? → FK (1:N) o tabla de unión (M:N)
   b. ¿Cuál es la cardinalidad mínima (participación)? → NOT NULL en FK si es total
   c. ¿Qué pasa si se borra la entidad referenciada? → ON DELETE correcto
   d. ¿Hay restricciones adicionales sobre la relación? → CHECK o trigger

3. Para cada regla de negocio identificada:
   a. ¿Es expresable con constraints declarativos? → PRIMARY KEY, UNIQUE, NOT NULL, FK, CHECK
   b. ¿Involucra múltiples tablas/filas? → Trigger
   c. ¿Es demasiado compleja para el DBMS? → Capa de servicio + documentación explícita
   d. Documentar cada restricción con su justificación de negocio
```

### 13.2 Caso práctico: esquema de sistema de préstamos

```sql
-- ===========================================================
-- Sistema de préstamos bibliotecarios con integridad completa
-- ===========================================================

CREATE TABLE socios (
    socio_id        SERIAL         PRIMARY KEY,
    numero_socio    VARCHAR(20)    NOT NULL UNIQUE,   -- clave candidata natural
    nombre          VARCHAR(100)   NOT NULL,
    email           VARCHAR(100)   NOT NULL UNIQUE,
    telefono        VARCHAR(20),
    fecha_alta      DATE           NOT NULL DEFAULT CURRENT_DATE,
    activo          BOOLEAN        NOT NULL DEFAULT TRUE,
    CONSTRAINT chk_email_formato CHECK (email ~ '^[^@]+@[^@]+\.[^@]+$')
);

CREATE TABLE libros (
    libro_id        SERIAL         PRIMARY KEY,
    isbn            VARCHAR(13)    NOT NULL UNIQUE
        CONSTRAINT chk_isbn_formato CHECK (isbn ~ '^\d{13}$'),
    titulo          VARCHAR(300)   NOT NULL,
    autor           VARCHAR(200)   NOT NULL,
    año_publicacion INT            NOT NULL
        CHECK (año_publicacion BETWEEN 1000 AND EXTRACT(YEAR FROM CURRENT_DATE)),
    total_ejemplares INT           NOT NULL CHECK (total_ejemplares > 0)
);

CREATE TABLE prestamos (
    prestamo_id     SERIAL         PRIMARY KEY,
    socio_id        INT            NOT NULL
        REFERENCES socios(socio_id) ON DELETE RESTRICT,
    libro_id        INT            NOT NULL
        REFERENCES libros(libro_id) ON DELETE RESTRICT,
    fecha_prestamo  DATE           NOT NULL DEFAULT CURRENT_DATE,
    fecha_devolucion_esperada DATE NOT NULL,
    fecha_devolucion_real DATE,
    renovaciones    INT            NOT NULL DEFAULT 0
        CHECK (renovaciones BETWEEN 0 AND 3),

    CONSTRAINT chk_devolucion_posterior
        CHECK (fecha_devolucion_esperada > fecha_prestamo),
    CONSTRAINT chk_devolucion_real_valida
        CHECK (fecha_devolucion_real IS NULL
               OR fecha_devolucion_real >= fecha_prestamo)
);

-- Trigger: solo socios activos pueden pedir préstamos
CREATE OR REPLACE FUNCTION verificar_socio_activo()
RETURNS TRIGGER AS $$
BEGIN
    IF NOT (SELECT activo FROM socios WHERE socio_id = NEW.socio_id) THEN
        RAISE EXCEPTION 'El socio % no está activo', NEW.socio_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_socio_activo
    BEFORE INSERT ON prestamos
    FOR EACH ROW EXECUTE FUNCTION verificar_socio_activo();

-- Trigger: un socio no puede tener más de 3 préstamos activos
CREATE OR REPLACE FUNCTION verificar_limite_prestamos()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT COUNT(*) FROM prestamos
        WHERE socio_id = NEW.socio_id
          AND fecha_devolucion_real IS NULL) >= 3 THEN
        RAISE EXCEPTION 'El socio % ya tiene 3 préstamos activos', NEW.socio_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_limite_prestamos
    BEFORE INSERT ON prestamos
    FOR EACH ROW EXECUTE FUNCTION verificar_limite_prestamos();

-- Trigger: no prestar un libro que no tiene ejemplares disponibles
CREATE OR REPLACE FUNCTION verificar_disponibilidad_libro()
RETURNS TRIGGER AS $$
DECLARE
    en_prestamo INT;
    total_ejemplares INT;
BEGIN
    SELECT COUNT(*) INTO en_prestamo
    FROM prestamos
    WHERE libro_id = NEW.libro_id
      AND fecha_devolucion_real IS NULL;

    SELECT l.total_ejemplares INTO total_ejemplares
    FROM libros l WHERE l.libro_id = NEW.libro_id;

    IF en_prestamo >= total_ejemplares THEN
        RAISE EXCEPTION 'No hay ejemplares disponibles del libro %', NEW.libro_id;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_disponibilidad_libro
    BEFORE INSERT ON prestamos
    FOR EACH ROW EXECUTE FUNCTION verificar_disponibilidad_libro();
```

### 13.3 Documentación de restricciones

Documentar las restricciones es tan importante como implementarlas. Una restricción sin documentación es opaca: nadie sabe por qué existe, si sigue siendo válida, o qué pasa con los datos que la violaron antes de que se creara.

```sql
-- Usando comentarios en el esquema (PostgreSQL):
COMMENT ON CONSTRAINT chk_email_formato ON socios IS
    'Valida formato básico de email. Restricción de dominio mínima; '
    'la validación completa de email se hace en la capa de aplicación.';

COMMENT ON TRIGGER tg_limite_prestamos ON prestamos IS
    'Regla de negocio: máximo 3 préstamos activos por socio. '
    'Definida en política de préstamos v2.1 (2024-03-15). '
    'Revisar si cambia la política de membresía premium.';
```

---

## 14. Resumen y conexión con el resto del curso

### Lo que construimos en este módulo

La integridad de datos no es opcional ni secundaria. Es la garantía fundamental que hace que los datos sean confiables. Los mecanismos se organizan en una jerarquía de garantías:

**Nivel más fuerte (absoluto):**
- `PRIMARY KEY`: identidad de tupla, no NULL, único.
- `FOREIGN KEY`: referencias siempre válidas. Con acciones referenciales correctas (CASCADE, RESTRICT, SET NULL) que reflejan la semántica del dominio.
- `NOT NULL`: ausencia de ambigüedad por valores desconocidos.
- `UNIQUE`: unicidad de claves candidatas.

**Nivel fuerte (declarativo, garantizado por el DBMS):**
- `CHECK` constraints: condiciones arbitrarias sobre valores de la misma fila. Limitadas a la fila actual.
- Tipos de dato correctos: primera línea de defensa de dominio.

**Nivel medio (procedimental, robusto pero bypasseable):**
- Triggers: restricciones multi-tabla, cardinalidades máximas, lógica condicional compleja.
- Restricciones diferidas: para dependencias circulares y bootstrapping.

**Nivel débil (depende de todo el código siendo correcto):**
- Validaciones en capa de servicio.
- Validaciones en cliente/frontend.

### El principio rector

> **Cada restricción del dominio debe estar implementada en la capa de garantía más fuerte posible que pueda expresarla.**

No hay razón para poner en código de aplicación algo que puede ser un `NOT NULL`. No hay razón para poner en un trigger algo que puede ser un `CHECK`. No hay razón para dejar sin declarar una FK porque "la aplicación lo maneja".

### Conexión con módulos futuros

- **Módulo 5 (Patrones de modelado):** Los patrones como Party y roles tienen restricciones de integridad complejas que requieren exactamente las técnicas de este módulo: restricciones diferidas para dependencias circulares, triggers para restricciones multi-tabla, CHECK para condiciones dentro del patrón.

- **Módulo 7 (Internals y query planner):** El query planner usa las restricciones declarativas (`NOT NULL`, `CHECK`, `UNIQUE`) como información estadística y semántica para optimizar queries. Una columna `NOT NULL` permite joins más eficientes. Un `CHECK` que limita un rango permite el uso de particiones. Las restricciones no son solo para integridad; son también información para el optimizador.

- **Módulo 8 (Transacciones y aislamiento):** Las restricciones diferidas (`DEFERRABLE`) tienen interacción directa con el modelo de transacciones. Entender cuándo se verifican las restricciones (al final del statement vs al final de la transacción) es esencial para entender el comportamiento bajo concurrencia.

- **Módulo 16 (El oficio):** La seguridad a nivel de datos (row-level security, column masking) es una extensión del concepto de integridad: no solo garantizar que los datos sean correctos, sino que solo los actores autorizados puedan verlos y modificarlos.

---

## Ejercicios de comprensión

**Ejercicio 1.** Dado el siguiente esquema incompleto de un sistema de turnos médicos:
```sql
CREATE TABLE medicos (medico_id INT PRIMARY KEY, nombre TEXT, especialidad TEXT);
CREATE TABLE pacientes (paciente_id INT PRIMARY KEY, nombre TEXT, fecha_nac DATE);
CREATE TABLE turnos (
    turno_id INT PRIMARY KEY,
    medico_id INT,
    paciente_id INT,
    fecha_hora TIMESTAMP,
    duracion_minutos INT,
    estado TEXT,
    notas TEXT
);
```
a) Identifica todas las restricciones de integridad faltantes.
b) Agrega `NOT NULL`, `UNIQUE`, `FOREIGN KEY`, y `CHECK` constraints donde corresponda.
c) Define la acción referencial correcta para cada FK y justifica.
d) Identifica qué reglas de negocio NO pueden expresarse con constraints declarativos y propón el trigger correspondiente para al menos una de ellas.

**Ejercicio 2.** Explica el comportamiento diferente de `NO ACTION` vs `RESTRICT` en el contexto de restricciones diferibles. ¿En qué escenario concreto producen resultados distintos?

**Ejercicio 3.** Tienes una tabla de `cuentas_bancarias(cuenta_id, cliente_id, tipo, saldo)`. Escribe las restricciones necesarias para garantizar:
- El saldo de cuentas de ahorro nunca puede ser negativo.
- Las cuentas de inversión pueden tener saldo negativo hasta -1000.
- Las cuentas corrientes pueden tener saldo negativo hasta el límite de sobregiro (campo `limite_sobregiro` en la misma tabla).

**Ejercicio 4.** Diseña el mecanismo de integridad más fuerte posible para esta restricción: *"En ningún momento puede haber más de un médico de guardia por especialidad en cada turno (mañana/tarde/noche)."*

**Ejercicio 5.** Un colega argumenta: *"No vale la pena poner FKs en producción porque afectan el rendimiento de los inserts y en nuestra aplicación ya validamos las referencias."* Formula la respuesta técnica completa a este argumento.

---

*Próximo módulo: Patrones de modelado para dominios reales — donde aplicaremos integridad y diseño a problemas que se repiten en casi todos los sistemas empresariales: personas, organizaciones, roles, jerarquías recursivas y clasificaciones flexibles.*
