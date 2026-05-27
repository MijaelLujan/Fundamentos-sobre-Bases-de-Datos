# Módulo 6 — Diseño para el tiempo

> *"El tiempo es la dimensión que los modelos de datos más frecuentemente ignoran. No porque sea difícil de entender conceptualmente, sino porque sus implicaciones de diseño son incómodas: obliga a repensar lo que significa 'el estado actual' de un dato, lo que significa 'corregir un error', y lo que significa 'saber lo que sabíamos en un momento dado'. Ignorar el tiempo no hace que los problemas desaparezcan; los convierte en bugs que aparecen meses después, cuando ya es difícil reconstruir lo que pasó."*

---

## Tabla de contenidos

1. [El problema fundamental: los datos cambian](#1-el-problema-fundamental-los-datos-cambian)
2. [Tiempo válido y tiempo de transacción: la distinción central](#2-tiempo-válido-y-tiempo-de-transacción-la-distinción-central)
3. [Tablas con tiempo válido (uni-temporales)](#3-tablas-con-tiempo-válido-uni-temporales)
4. [Tablas con tiempo de transacción (auditoría)](#4-tablas-con-tiempo-de-transacción-auditoría)
5. [Tablas bitemporales](#5-tablas-bitemporales)
6. [Soporte nativo en SQL:2011 y PostgreSQL](#6-soporte-nativo-en-sql2011-y-postgresql)
7. [Slowly Changing Dimensions (SCD)](#7-slowly-changing-dimensions-scd)
8. [Modelado de eventos vs modelado de estado](#8-modelado-de-eventos-vs-modelado-de-estado)
9. [Event Sourcing como patrón de modelado](#9-event-sourcing-como-patrón-de-modelado)
10. [Patrones de consulta temporal y sus trade-offs](#10-patrones-de-consulta-temporal-y-sus-trade-offs)
11. [Resumen y conexión con el resto del curso](#11-resumen-y-conexión-con-el-resto-del-curso)
12. [Ejercicios de comprensión](#12-ejercicios-de-comprensión)

---

## 1. El problema fundamental: los datos cambian

### 1.1 Qué pasa cuando un modelo ignora el tiempo

Imagina el modelo de base de datos más simple: una tabla de empleados.

```sql
CREATE TABLE empleados (
    empleado_id   INT PRIMARY KEY,
    nombre        VARCHAR(100) NOT NULL,
    departamento  VARCHAR(100) NOT NULL,
    salario       DECIMAL(12,2) NOT NULL
);
```

Este modelo responde perfectamente a preguntas presentes: "¿Cuánto gana Ana ahora?", "¿En qué departamento está Juan?". Pero en un sistema real, los datos cambian. Ana recibe un aumento. Juan cambia de departamento. El CEO pide un informe de la nómina a diciembre del año pasado. El departamento de recursos humanos quiere saber cuánto tiempo lleva cada empleado en su posición actual.

El modelo anterior no puede responder ninguna pregunta histórica porque **cada `UPDATE` destruye irrecuperablemente el estado anterior**. Una vez que Ana recibe su aumento, es imposible saber cuánto ganaba antes, a menos que ese dato esté en algún log externo.

Ahora imagina una situación más grave: en febrero, un contador descubre que el salario de Ana debería haber sido 5,500 (no 5,000) desde octubre. En el modelo sin tiempo, la "corrección" consiste en un `UPDATE`. Pero eso oculta el hecho de que hubo un error durante cuatro meses. Si en esos cuatro meses se calculó una participación de utilidades, un seguro, o un reporte regulatorio usando el salario incorrecto, esos documentos son incorrectos. No hay forma de saber, mirando solo la base de datos, qué se "sabía" en esos momentos.

Este es el problema que el diseño temporal resuelve.

### 1.2 Las preguntas que requieren diseño temporal

Un modelo necesita soporte temporal cuando el negocio necesita responder preguntas de cualquiera de estas familias:

**Preguntas históricas sobre el mundo real:**
- ¿Cuál era el salario de Ana el 15 de noviembre del año pasado?
- ¿En qué dirección vivía este cliente cuando hizo su pedido #12345?
- ¿Cuál era el precio de este producto cuando se emitió esta factura?

**Preguntas sobre la evolución de los datos:**
- ¿Cuándo cambió Juan de departamento?
- ¿Cuántas veces ha cambiado de dirección este cliente en los últimos dos años?

**Preguntas sobre lo que el sistema "sabía" en un momento dado:**
- ¿Qué información teníamos sobre este cliente cuando aprobamos su crédito?
- Si el auditor nos pregunta qué decía la base de datos el 31 de diciembre, ¿qué le respondemos?

**Preguntas para corregir errores manteniendo auditoría:**
- Descubrimos que el sueldo de Ana estaba mal desde octubre. Queremos corregirlo, pero necesitamos un rastro de que hubo un error y una corrección.

### 1.3 La solución intuitiva incorrecta

La primera solución que se le ocurre a la mayoría es agregar columnas `fecha_creacion` y `fecha_modificacion`:

```sql
-- Solución intuitiva incorrecta
CREATE TABLE empleados (
    empleado_id    INT PRIMARY KEY,
    nombre         VARCHAR(100) NOT NULL,
    departamento   VARCHAR(100) NOT NULL,
    salario        DECIMAL(12,2) NOT NULL,
    fecha_creacion TIMESTAMP NOT NULL DEFAULT NOW(),
    fecha_modif    TIMESTAMP NOT NULL DEFAULT NOW()
);
```

Esto es útil para saber *cuándo se modificó por última vez* un registro, pero no resuelve nada:

- Todavía no sabes **qué era el salario antes** del cambio.
- No sabes si el cambio fue una corrección de un error o un cambio legítimo.
- No puedes reconstruir el estado de la tabla en ningún momento pasado.

La solución correcta requiere un cambio de mentalidad: en lugar de actualizar registros, los modelos temporales **agregan nuevas filas** que representan los períodos de vigencia de cada versión del dato.

---

## 2. Tiempo válido y tiempo de transacción: la distinción central

Esta distinción fue formalizada por C.J. Date, Hugh Darwen y Nikos Lorentzos en los años 80-90 y es la piedra angular del diseño temporal. SQL:2011 la adoptó formalmente en el estándar.

### 2.1 Tiempo válido (Valid Time / Application Time)

El **tiempo válido** es el período durante el cual un hecho es verdadero **en el mundo real**, independientemente de cuándo el sistema lo registró.

Ejemplos:
- El contrato de alquiler de Ana es válido del 1 de enero de 2024 al 31 de diciembre de 2024. Ese es el período de **tiempo válido** del contrato, aunque el sistema lo haya registrado el 5 de enero o el 15 de diciembre.
- La tasa de impuesto al consumo fue del 13% desde el 1 de abril de 2020 hasta el 31 de marzo de 2022. Ese es el tiempo válido de esa tasa.
- El precio del producto fue 99.90 del 1 de octubre al 31 de octubre. Ese es el tiempo válido de ese precio.

**Característica clave:** El tiempo válido puede referirse al pasado, al presente, o al futuro. Puedo registrar hoy que un contrato será válido desde el año próximo. El tiempo válido lo *controla el negocio o el dominio*, no el sistema de base de datos.

### 2.2 Tiempo de transacción (Transaction Time / System Time)

El **tiempo de transacción** es el período durante el cual el sistema *sabía* o *tenía registrado* un hecho. Es la historia de los cambios en la base de datos misma.

Ejemplos:
- El sistema registró el salario de Ana como 5,000 desde el 1 de octubre hasta que se hizo la corrección el 14 de febrero. Ese es el período de tiempo de transacción de ese registro de salario.
- La dirección incorrecta del cliente estuvo en el sistema desde el 3 de marzo hasta el 8 de marzo, cuando se corrigió.

**Característica clave:** El tiempo de transacción es **siempre en el pasado o en el presente**. No se puede saber de antemano cuándo terminará un registro de sistema (termina cuando se modifica o elimina). **El sistema controla el tiempo de transacción**, no el usuario. Y crucialmente, **el tiempo de transacción nunca se corrige retroactivamente**: es literalmente la historia de lo que pasó en la base de datos.

### 2.3 Un ejemplo concreto que muestra la diferencia

Tomemos este escenario:

> El 1 de octubre de 2024, el departamento de recursos humanos registra en el sistema que Ana recibe un aumento a 5,000 vigente a partir del 1 de octubre. Pero en realidad, por un error tipográfico, debería ser 5,500. Este error es descubierto el 14 de febrero de 2025 y se corrige.

| Evento | Tiempo válido | Tiempo de transacción |
|---|---|---|
| Se registra el salario de 5,000 vigente desde oct. | 2024-10-01 hasta ??? | 2024-10-01 hasta 2025-02-14 |
| Se corrige a 5,500 con vigencia desde oct. | 2024-10-01 hasta ??? | 2025-02-14 hasta ??? |

Ahora podemos responder dos preguntas distintas:

- "¿Cuánto ganaba Ana el 15 de noviembre de 2024?" → Tiempo válido: 5,500 (porque ese es el salario correcto en esa fecha del mundo real).
- "¿Qué decía el sistema sobre el salario de Ana el 15 de noviembre de 2024?" → Tiempo de transacción: 5,000 (porque el error no se había corregido todavía).

Sin la distinción entre ambas dimensiones temporales, es imposible responder ambas preguntas simultáneamente. Un modelo con solo tiempo válido puede responder la primera pero no la segunda. Un modelo con solo tiempo de transacción puede responder la segunda pero no la primera de forma "correcta" (porque registra la versión incorrecta como la verdad).

### 2.4 ¿Cuándo necesitas cada dimensión?

| Necesidad | Dimensión requerida |
|---|---|
| Historial de cambios en el mundo real (precios, salarios, contratos) | Tiempo válido |
| Poder preguntar "¿qué era esto en una fecha pasada?" | Tiempo válido |
| Registrar fechas futuras de vigencia | Tiempo válido |
| Auditoría: saber qué decía el sistema en un momento dado | Tiempo de transacción |
| Corrección de errores con rastro completo | Tiempo de transacción |
| Cumplimiento regulatorio ("¿qué sabían en esa fecha?") | Tiempo de transacción |
| Todo lo anterior | Tabla bitemporal (ambas dimensiones) |

---

## 3. Tablas con tiempo válido (uni-temporales)

### 3.1 La representación de períodos

Para representar tiempo válido, se agregan dos columnas al esquema que forman un **intervalo semiabierto**: `[inicio, fin)`. El intervalo es cerrado por la izquierda (incluye el inicio) y abierto por la derecha (excluye el fin). Esta convención es casi universal en diseño temporal porque simplifica la aritmética:

- Un período `[2024-01-01, 2024-04-01)` abarca enero, febrero y marzo exactamente.
- Dos períodos son adyacentes si `fin_1 = inicio_2`. No hay gap ni overlap.
- Un período indefinido (actualmente vigente sin fecha de fin conocida) usa un valor especial como `9999-12-31` o `infinity` (en PostgreSQL).

La elección de `infinity` vs una fecha máxima convencional como `9999-12-31` es una decisión de equipo, pero `infinity` es más correcto semánticamente y PostgreSQL lo soporta nativamente.

### 3.2 Esquema básico con tiempo válido

```sql
-- Historial de salarios con tiempo válido
CREATE TABLE salarios (
    salario_id    SERIAL PRIMARY KEY,
    empleado_id   INT    NOT NULL REFERENCES empleados(empleado_id),
    salario       DECIMAL(12,2) NOT NULL,
    valido_desde  DATE   NOT NULL,
    valido_hasta  DATE   NOT NULL DEFAULT 'infinity',
    
    -- Garantiza que no haya gaps ni overlaps para el mismo empleado
    -- (explicado en detalle más abajo)
    CONSTRAINT chk_periodo_valido CHECK (valido_desde < valido_hasta)
);

-- Índice para las queries más comunes: "estado en una fecha dada"
CREATE INDEX idx_salarios_empleado_periodo 
    ON salarios (empleado_id, valido_desde, valido_hasta);
```

### 3.3 El problema de la integridad: no gaps, no overlaps

El mayor desafío del diseño con tiempo válido es mantener la **integridad temporal**: para cada empleado, los períodos deben ser:

1. **No solapados:** Un empleado no puede tener dos salarios vigentes al mismo tiempo.
2. **Sin brechas** (si el negocio lo requiere): Puede o no requerirse que siempre haya un registro vigente. Para salarios, probablemente sí; para contratos de alquiler, puede haber períodos sin contrato.

Desafortunadamente, garantizar estas propiedades con restricciones declarativas en SQL estándar es complicado. PostgreSQL ofrece una solución elegante con el tipo `tsrange` y restricciones de exclusión:

```sql
-- Solución con rangos nativos de PostgreSQL
CREATE TABLE salarios (
    salario_id   SERIAL PRIMARY KEY,
    empleado_id  INT    NOT NULL REFERENCES empleados(empleado_id),
    salario      DECIMAL(12,2) NOT NULL,
    periodo      DATERANGE NOT NULL,  -- tipo nativo de rango de PostgreSQL
    
    -- Garantiza no-overlap por empleado: si dos filas tienen el mismo
    -- empleado_id y sus periodos se solapan, se rechaza la inserción
    EXCLUDE USING GIST (
        empleado_id WITH =,
        periodo WITH &&   -- && significa "se solapa con"
    )
);

-- Insertar el salario inicial de Ana
INSERT INTO salarios (empleado_id, salario, periodo)
VALUES (1, 4500.00, '[2023-01-01, infinity)');

-- Ana recibe un aumento el 1 de octubre de 2024:
-- Paso 1: Cerrar el período anterior
UPDATE salarios
SET periodo = '[2023-01-01, 2024-10-01)'
WHERE empleado_id = 1
  AND upper(periodo) = 'infinity';

-- Paso 2: Insertar el nuevo período
INSERT INTO salarios (empleado_id, salario, periodo)
VALUES (1, 5000.00, '[2024-10-01, infinity)');
```

El tipo `DATERANGE` con `EXCLUDE USING GIST` hace que la base de datos **rechace automáticamente** cualquier inserción que genere solapamiento. Esto es un ejemplo de integridad declarativa superior a cualquier validación en código de aplicación.

### 3.4 Consultas con tiempo válido

Una vez que el modelo tiene tiempo válido, las consultas históricas son directas:

```sql
-- ¿Cuánto ganaba Ana (empleado_id = 1) el 15 de noviembre de 2024?
SELECT salario
FROM salarios
WHERE empleado_id = 1
  AND periodo @> '2024-11-15'::date;  -- @> significa "contiene el valor"

-- ¿Cuáles empleados tenían salario mayor a 5,000 el 1 de enero de 2024?
SELECT e.nombre, s.salario
FROM salarios s
JOIN empleados e ON e.empleado_id = s.empleado_id
WHERE s.periodo @> '2024-01-01'::date
  AND s.salario > 5000;

-- ¿Cuántas veces cambió de salario Ana en 2024?
SELECT COUNT(*) - 1 AS cambios
FROM salarios
WHERE empleado_id = 1
  AND periodo && '[2024-01-01, 2025-01-01)'::daterange;
-- Nota: contamos filas que se solapan con el año 2024, y restamos 1
-- porque si hay N períodos hay N-1 cambios

-- Salario actual de todos los empleados (el período que incluye hoy)
SELECT e.nombre, s.salario
FROM salarios s
JOIN empleados e ON e.empleado_id = s.empleado_id
WHERE s.periodo @> CURRENT_DATE;
```

### 3.5 El patrón de actualización temporal

Al trabajar con tiempo válido, el patrón de operaciones cambia completamente. **No se usa `UPDATE` para cambiar un valor; se usa `UPDATE` para cerrar el período actual y `INSERT` para abrir el nuevo.** Este patrón se llama a veces "insert-only" o "append-only".

```sql
-- Procedimiento para cambiar el salario de un empleado
-- (usando una función en PostgreSQL para atomicidad)
CREATE OR REPLACE FUNCTION cambiar_salario(
    p_empleado_id INT,
    p_nuevo_salario DECIMAL(12,2),
    p_fecha_vigencia DATE
) RETURNS VOID AS $$
DECLARE
    v_salario_actual DECIMAL(12,2);
BEGIN
    -- 1. Verificar que el empleado tiene un salario actual
    SELECT salario INTO v_salario_actual
    FROM salarios
    WHERE empleado_id = p_empleado_id
      AND periodo @> p_fecha_vigencia;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'No hay salario vigente para el empleado % en la fecha %',
            p_empleado_id, p_fecha_vigencia;
    END IF;
    
    -- 2. Cerrar el período actual
    UPDATE salarios
    SET periodo = daterange(lower(periodo), p_fecha_vigencia)
    WHERE empleado_id = p_empleado_id
      AND periodo @> p_fecha_vigencia;
    
    -- 3. Abrir el nuevo período
    INSERT INTO salarios (empleado_id, salario, periodo)
    VALUES (p_empleado_id, p_nuevo_salario, 
            daterange(p_fecha_vigencia, 'infinity'));
END;
$$ LANGUAGE plpgsql;

-- Uso:
SELECT cambiar_salario(1, 5000.00, '2024-10-01');
```

---

## 4. Tablas con tiempo de transacción (auditoría)

### 4.1 Qué registra el tiempo de transacción

El tiempo de transacción registra la historia de **lo que el sistema tenía registrado**, no lo que era verdad en el mundo. Es la herramienta para la auditoría: si un auditor pregunta "¿qué decía la base de datos el 31 de diciembre?", el tiempo de transacción permite responder exactamente eso.

A diferencia del tiempo válido, el tiempo de transacción **nunca se modifica retroactivamente**. Una vez que un registro tiene una fecha de inicio de transacción, esa fecha no cambia. Y una vez que un registro es "eliminado" del punto de vista del tiempo de transacción, simplemente se marca como expirado, pero la fila sigue existiendo.

### 4.2 Implementación mediante tabla de historial (patrón shadow table)

La implementación más común es tener una tabla "espejo" que acumula el historial de cambios. Esta tabla recibe el nombre de la tabla principal con un sufijo como `_historial` o `_audit`.

```sql
-- Tabla principal (el estado actual)
CREATE TABLE empleados (
    empleado_id   INT PRIMARY KEY,
    nombre        VARCHAR(100) NOT NULL,
    departamento  VARCHAR(100) NOT NULL,
    salario       DECIMAL(12,2) NOT NULL
);

-- Tabla de historial (la historia de todos los estados)
CREATE TABLE empleados_historial (
    historial_id      SERIAL PRIMARY KEY,
    empleado_id       INT           NOT NULL,
    nombre            VARCHAR(100)  NOT NULL,
    departamento      VARCHAR(100)  NOT NULL,
    salario           DECIMAL(12,2) NOT NULL,
    
    -- Tiempo de transacción: cuándo el sistema tenía estos datos
    tx_desde          TIMESTAMP     NOT NULL,
    tx_hasta          TIMESTAMP     NOT NULL DEFAULT 'infinity',
    
    -- Metadatos de auditoría
    operacion         CHAR(1)       NOT NULL,  -- 'I' insert, 'U' update, 'D' delete
    usuario_bd        VARCHAR(100)  NOT NULL DEFAULT CURRENT_USER,
    
    CONSTRAINT chk_tx_periodo CHECK (tx_desde < tx_hasta)
);

CREATE INDEX idx_emp_hist_id_tx ON empleados_historial (empleado_id, tx_desde, tx_hasta);
```

El mantenimiento de esta tabla de historial se automatiza con **triggers**:

```sql
CREATE OR REPLACE FUNCTION fn_audit_empleados() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO empleados_historial 
            (empleado_id, nombre, departamento, salario, tx_desde, operacion)
        VALUES 
            (NEW.empleado_id, NEW.nombre, NEW.departamento, NEW.salario, 
             NOW(), 'I');
        RETURN NEW;
        
    ELSIF TG_OP = 'UPDATE' THEN
        -- Cerrar el período del registro anterior
        UPDATE empleados_historial
        SET tx_hasta = NOW()
        WHERE empleado_id = OLD.empleado_id
          AND tx_hasta = 'infinity';
        
        -- Abrir un nuevo registro
        INSERT INTO empleados_historial 
            (empleado_id, nombre, departamento, salario, tx_desde, operacion)
        VALUES 
            (NEW.empleado_id, NEW.nombre, NEW.departamento, NEW.salario,
             NOW(), 'U');
        RETURN NEW;
        
    ELSIF TG_OP = 'DELETE' THEN
        -- Cerrar el período del registro actual
        UPDATE empleados_historial
        SET tx_hasta = NOW(), operacion = 'D'
        WHERE empleado_id = OLD.empleado_id
          AND tx_hasta = 'infinity';
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_audit_empleados
AFTER INSERT OR UPDATE OR DELETE ON empleados
FOR EACH ROW EXECUTE FUNCTION fn_audit_empleados();
```

### 4.3 Consultas con tiempo de transacción

```sql
-- ¿Qué decía el sistema sobre Ana (empleado_id = 1) el 31 de diciembre de 2024?
SELECT nombre, departamento, salario
FROM empleados_historial
WHERE empleado_id = 1
  AND tx_desde <= '2024-12-31 23:59:59'
  AND tx_hasta > '2024-12-31 23:59:59';

-- Historia completa de cambios en el registro de Ana
SELECT operacion, nombre, departamento, salario, tx_desde, tx_hasta
FROM empleados_historial
WHERE empleado_id = 1
ORDER BY tx_desde;

-- ¿Quién modificó el registro de Ana en los últimos 30 días?
SELECT operacion, usuario_bd, tx_desde, salario
FROM empleados_historial
WHERE empleado_id = 1
  AND tx_desde >= NOW() - INTERVAL '30 days'
ORDER BY tx_desde;
```

### 4.4 Tiempo de transacción nativo con tablas de sistema en PostgreSQL

PostgreSQL 14+ ofrece soporte nativo para tiempo de transacción a través de **tablas con columna de sistema `xmin`**. Sin embargo, para la mayoría de los casos de uso prácticos, el patrón de tabla de historial con trigger sigue siendo más flexible y legible.

Una alternativa moderna en PostgreSQL es usar la extensión **`temporal_tables`** o el soporte de `PERIOD FOR SYSTEM_TIME` que llegó en versiones más recientes. Sin embargo, el patrón manual con triggers sigue siendo el más portable y comprensible.

---

## 5. Tablas bitemporales

### 5.1 El concepto: dos líneas de tiempo simultáneas

Una tabla **bitemporal** combina ambas dimensiones: tiempo válido (cuándo el hecho fue verdad en el mundo) y tiempo de transacción (cuándo el sistema lo sabía). Cada fila tiene cuatro columnas temporales que forman dos intervalos:

```
[ valido_desde,  valido_hasta )   ← tiempo válido
[ tx_desde,      tx_hasta     )   ← tiempo de transacción
```

Juntas, estas cuatro columnas permiten responder la pregunta más completa posible:

> "¿Qué sabía el sistema el día X sobre lo que era verdad el día Y?"

### 5.2 Esquema bitemporal

Tomemos el ejemplo del salario de Ana con la corrección del error:

```sql
CREATE TABLE salarios_bitemporal (
    salario_id   SERIAL PRIMARY KEY,
    empleado_id  INT           NOT NULL REFERENCES empleados(empleado_id),
    salario      DECIMAL(12,2) NOT NULL,
    
    -- Tiempo válido: cuándo es verdad en el mundo real
    valido_desde DATE NOT NULL,
    valido_hasta DATE NOT NULL DEFAULT 'infinity',
    
    -- Tiempo de transacción: cuándo el sistema lo registró
    tx_desde     TIMESTAMP NOT NULL DEFAULT NOW(),
    tx_hasta     TIMESTAMP NOT NULL DEFAULT 'infinity',
    
    CONSTRAINT chk_valido CHECK (valido_desde < valido_hasta),
    CONSTRAINT chk_tx     CHECK (tx_desde < tx_hasta)
);
```

### 5.3 Simulación del escenario completo

Vamos a simular todo el escenario del error y la corrección de Ana paso a paso:

**Paso 1: El 1 de octubre de 2024, RR.HH. registra un salario de 5,000 (que es un error; debería ser 5,500).**

```sql
-- 2024-10-01: Se registra el "aumento" de Ana
INSERT INTO salarios_bitemporal 
    (empleado_id, salario, valido_desde, valido_hasta, tx_desde, tx_hasta)
VALUES 
    (1, 5000.00, '2024-10-01', 'infinity', '2024-10-01 09:00:00', 'infinity');
```

Estado de la tabla a esta altura:

| salario_id | empleado_id | salario | valido_desde | valido_hasta | tx_desde | tx_hasta |
|---|---|---|---|---|---|---|
| 1 | 1 | 5000.00 | 2024-10-01 | infinity | 2024-10-01 09:00 | **infinity** |

**Paso 2: El 14 de febrero de 2025, se descubre el error. La corrección es que el salario debe ser 5,500 desde el 1 de octubre.**

La corrección bitemporal tiene dos pasos:
1. Cerrar el tiempo de transacción del registro incorrecto (ya no es "lo que el sistema dice").
2. Insertar la corrección con su tiempo válido correcto y el tiempo de transacción actual.

```sql
-- 2025-02-14: Corrección del error

-- Paso 1: "Retirar" el registro incorrecto del sistema
-- (cerramos su tiempo de transacción; el registro sigue existiendo para auditoría)
UPDATE salarios_bitemporal
SET tx_hasta = '2025-02-14 10:30:00'
WHERE empleado_id = 1
  AND tx_hasta = 'infinity';

-- Paso 2: Insertar el dato correcto
-- (el tiempo válido es desde oct-2024, el tiempo de transacción es desde hoy)
INSERT INTO salarios_bitemporal
    (empleado_id, salario, valido_desde, valido_hasta, tx_desde, tx_hasta)
VALUES
    (1, 5500.00, '2024-10-01', 'infinity', '2025-02-14 10:30:00', 'infinity');
```

Estado de la tabla después de la corrección:

| salario_id | empleado_id | salario | valido_desde | valido_hasta | tx_desde | tx_hasta |
|---|---|---|---|---|---|---|
| 1 | 1 | 5000.00 | 2024-10-01 | infinity | 2024-10-01 09:00 | **2025-02-14 10:30** |
| 2 | 1 | 5500.00 | 2024-10-01 | infinity | **2025-02-14 10:30** | infinity |

### 5.4 Consultas bitemporales: el poder del modelo

Ahora podemos responder ambas preguntas simultáneamente:

```sql
-- Pregunta 1: ¿Cuánto DEBERÍA ganar Ana el 15 de noviembre de 2024?
-- (el salario correcto en el mundo real, usando la información más actualizada)
SELECT salario
FROM salarios_bitemporal
WHERE empleado_id = 1
  AND valido_desde <= '2024-11-15'
  AND valido_hasta > '2024-11-15'
  AND tx_hasta = 'infinity';  -- versión actual del sistema
-- Resultado: 5500.00 (el salario correcto)

-- Pregunta 2: ¿Qué decía el sistema el 15 de noviembre de 2024 sobre el salario de Ana?
-- (lo que se usó para cálculos en esa fecha)
SELECT salario
FROM salarios_bitemporal
WHERE empleado_id = 1
  AND valido_desde <= '2024-11-15'
  AND valido_hasta > '2024-11-15'
  AND tx_desde <= '2024-11-15 23:59:59'
  AND tx_hasta > '2024-11-15 23:59:59';  -- lo que el sistema sabía en esa fecha
-- Resultado: 5000.00 (el valor incorrecto que tenía en ese momento)

-- Pregunta 3: Historia completa de correcciones para Ana en octubre de 2024
SELECT salario, valido_desde, valido_hasta, tx_desde, tx_hasta
FROM salarios_bitemporal
WHERE empleado_id = 1
  AND valido_desde <= '2024-10-31'
  AND valido_hasta > '2024-10-01'
ORDER BY tx_desde;
-- Resultado: ambas filas, mostrando el error original y la corrección
```

Este es el poder del modelo bitemporal: la corrección del error **no borra la evidencia de que el error existió**. El registro incorrecto permanece en la tabla con su `tx_hasta` sellado, y el registro correcto se inserta con su propia historia de transacción. La tabla es un registro inmutable de la realidad.

### 5.5 Cuándo usar el modelo bitemporal completo

El modelo bitemporal tiene un costo: más complejidad en las queries, más filas por dato, y la necesidad de siempre especificar qué "vista" del tiempo quieres. Es la solución correcta cuando:

- **Hay regulación o auditoría:** Sistemas financieros, médicos, legales donde se puede preguntar "¿qué sabían en tal fecha?".
- **Se registran datos que describen el pasado o el futuro:** Contratos, tasas impositivas, precios con fechas de vigencia futuras.
- **Los errores son frecuentes y hay correcciones retroactivas:** RR.HH., contabilidad, catálogos de productos.
- **Hay múltiples fuentes de datos que llegan con latencia:** Datos que se registran días después de que ocurrieron.

Para muchos sistemas simples, el tiempo válido solo es suficiente. Para sistemas con alta exigencia de auditoría, se necesita la dimensión de transacción también.

---

## 6. Soporte nativo en SQL:2011 y PostgreSQL

### 6.1 SQL:2011 y los períodos temporales

El estándar SQL:2011 formalizó el soporte temporal con nueva sintaxis. Los conceptos clave introducidos son:

- **`PERIOD FOR`:** Declara formalmente que dos columnas forman un período temporal.
- **`APPLICATION TIME`:** El período de tiempo válido (controlado por la aplicación).
- **`SYSTEM_TIME`:** El período de tiempo de transacción (mantenido por el sistema).
- **`FOR SYSTEM_TIME AS OF`:** Sintaxis para consultas en tiempo de transacción.
- **`FOR APPLICATION TIME AS OF`:** Sintaxis para consultas en tiempo válido.

```sql
-- Sintaxis SQL:2011 (soporte varía por motor)
CREATE TABLE salarios (
    empleado_id  INT,
    salario      DECIMAL(12,2),
    valido_desde DATE,
    valido_hasta DATE,
    
    -- Declarar el período de tiempo válido
    PERIOD FOR valido (valido_desde, valido_hasta),
    
    -- Restricción de unicidad temporal: no puede haber dos salarios
    -- para el mismo empleado en el mismo período válido
    UNIQUE (empleado_id, PERIOD valido)
);

-- Consulta temporal nativa SQL:2011
SELECT salario
FROM salarios
FOR APPLICATION_TIME AS OF DATE '2024-11-15'
WHERE empleado_id = 1;
```

### 6.2 Soporte en PostgreSQL con rangos nativos

PostgreSQL no implementa la sintaxis SQL:2011 completa, pero ofrece algo igualmente potente: tipos de rango nativos (`daterange`, `tsrange`, `tstzrange`) con operadores especializados y soporte de índice GiST.

```sql
-- Ejemplo práctico en PostgreSQL con rangos nativos

CREATE TABLE contratos (
    contrato_id  SERIAL PRIMARY KEY,
    cliente_id   INT NOT NULL,
    tipo         VARCHAR(50) NOT NULL,
    monto        DECIMAL(12,2) NOT NULL,
    
    -- Un rango de fechas para el tiempo válido
    vigencia     DATERANGE NOT NULL,
    
    -- Tiempo de transacción con tstzrange (timestamp con zona horaria)
    tx_periodo   TSTZRANGE NOT NULL DEFAULT tstzrange(NOW(), 'infinity'),
    
    -- No puede haber dos contratos del mismo tipo para el mismo cliente
    -- en períodos de tiempo válido que se solapen
    EXCLUDE USING GIST (
        cliente_id WITH =,
        tipo WITH =,
        vigencia WITH &&
    )
    -- Solo aplica a los registros activos (no cerrados por tiempo de transacción)
    WHERE (upper(tx_periodo) = 'infinity')
);

-- Crear índice para queries por tiempo válido
CREATE INDEX idx_contratos_vigencia ON contratos USING GIST (cliente_id, vigencia);
CREATE INDEX idx_contratos_tx ON contratos USING GIST (tx_periodo);

-- Insertar un contrato activo
INSERT INTO contratos (cliente_id, tipo, monto, vigencia)
VALUES (42, 'servicio_premium', 1200.00, '[2024-01-01, 2024-12-31)');

-- Consulta: ¿Qué contratos activos tenía el cliente 42 el 15 de junio de 2024?
SELECT contrato_id, tipo, monto
FROM contratos
WHERE cliente_id = 42
  AND vigencia @> '2024-06-15'::date    -- incluye esa fecha en el tiempo válido
  AND tx_periodo @> NOW();              -- activo en el sistema ahora
```

### 6.3 Operadores de rango más importantes

| Operador | Significado | Ejemplo |
|---|---|---|
| `@>` | El rango contiene un valor | `periodo @> '2024-06-15'` |
| `<@` | Un valor está contenido en el rango | `'2024-06-15' <@ periodo` |
| `&&` | Los rangos se solapan | `periodo && '[2024-01-01, 2024-07-01)'` |
| `=` | Los rangos son iguales | `periodo = '[2024-01-01, 2025-01-01)'` |
| `<<` | El rango está estrictamente antes | `periodo << '[2025-01-01, 2026-01-01)'` |
| `>>` | El rango está estrictamente después | `periodo >> '[2023-01-01, 2024-01-01)'` |
| `-|-` | Los rangos son adyacentes | `periodo1 -\|- periodo2` |
| `lower()` | Extremo inferior del rango | `lower(periodo)` → `'2024-01-01'` |
| `upper()` | Extremo superior del rango | `upper(periodo)` → `'2024-12-31'` |
| `isempty()` | El rango está vacío | `isempty(periodo)` |

---

## 7. Slowly Changing Dimensions (SCD)

### 7.1 El contexto: dimensiones en data warehouses

Las **Slowly Changing Dimensions** (Dimensiones de Cambio Lento, SCD) son un conjunto de técnicas formalizadas por Ralph Kimball para manejar el cambio temporal de los atributos de las dimensiones en un **data warehouse**. Aunque tienen origen en el contexto analítico (que veremos en los Módulos 10 y 11), sus principios son tan fundamentales que vale la pena estudiarlos aquí como técnicas de manejo temporal.

El problema que resuelven: en un data warehouse, una dimensión como "cliente" o "producto" tiene atributos que cambian con el tiempo (la dirección del cliente, la categoría del producto, la región del vendedor). Cuando un cliente cambia de ciudad, ¿qué haces?:

- ¿Actualizas la fila y pierdes el valor anterior?
- ¿Guardas la versión anterior en alguna forma?
- ¿Añades una columna con el valor anterior?

Cada respuesta corresponde a un "tipo" de SCD distinto.

### 7.2 SCD Tipo 1: Sobrescribir

**La filosofía:** El pasado no importa. Solo nos interesa el valor actual. Si un atributo cambia, simplemente se actualiza la fila.

```sql
-- Tabla de dimensión cliente (SCD Tipo 1)
CREATE TABLE dim_cliente (
    cliente_sk   INT PRIMARY KEY,  -- surrogate key (clave sustituta del DW)
    cliente_id   INT NOT NULL,     -- natural key (clave del sistema fuente)
    nombre       VARCHAR(100) NOT NULL,
    ciudad       VARCHAR(100) NOT NULL,
    segmento     VARCHAR(50)  NOT NULL
);

-- El cliente 1001 se muda de "La Paz" a "Cochabamba"
UPDATE dim_cliente
SET ciudad = 'Cochabamba'
WHERE cliente_id = 1001;
-- El valor anterior "La Paz" se pierde para siempre
```

**Cuándo usar SCD Tipo 1:**
- Cuando el historial del atributo genuinamente no importa para el análisis.
- Cuando el cambio es una corrección de un error (p.ej., el nombre estaba mal escrito).
- Cuando el atributo es un derivado calculado que siempre se recalcula.

**El peligro del Tipo 1:** Si una transacción de ventas de hace dos años estaba asociada a un cliente en "La Paz", y ese cliente ahora vive en "Cochabamba", los reportes históricos mostrarán esa venta como si hubiera sido en "Cochabamba". Esto es una **pérdida de coherencia histórica**.

**Regla de oro:** Solo usa Tipo 1 si puedes afirmar con seguridad que el historial del atributo no cambia el significado de ninguna transacción pasada.

### 7.3 SCD Tipo 2: Filas de historial

**La filosofía:** El historial completo importa. Cuando un atributo cambia, se crea una nueva fila en la tabla de dimensión y se marca la anterior como expirada. Es la técnica más poderosa y más usada.

```sql
-- Tabla de dimensión cliente (SCD Tipo 2)
CREATE TABLE dim_cliente (
    cliente_sk        INT PRIMARY KEY,   -- surrogate key: nueva por cada versión
    cliente_id        INT NOT NULL,      -- natural key: el mismo para todas las versiones
    nombre            VARCHAR(100) NOT NULL,
    ciudad            VARCHAR(100) NOT NULL,
    segmento          VARCHAR(50)  NOT NULL,
    
    -- Período de vigencia de esta versión del registro
    fecha_inicio      DATE NOT NULL,
    fecha_fin         DATE NOT NULL DEFAULT '9999-12-31',  -- convención de DW
    
    -- Flag de conveniencia para encontrar la versión actual rápidamente
    es_actual         BOOLEAN NOT NULL DEFAULT TRUE
);

-- Estado inicial: el cliente 1001 vive en La Paz desde 2020-01-01
INSERT INTO dim_cliente VALUES 
    (1001, 1001, 'Pedro Alvarado', 'La Paz', 'Premium', '2020-01-01', '9999-12-31', TRUE);

-- En 2024-06-15, Pedro se muda a Cochabamba

-- Paso 1: Expirar la versión anterior
UPDATE dim_cliente
SET fecha_fin = '2024-06-14',
    es_actual = FALSE
WHERE cliente_id = 1001
  AND es_actual = TRUE;

-- Paso 2: Insertar la nueva versión
INSERT INTO dim_cliente VALUES
    (9999, 1001, 'Pedro Alvarado', 'Cochabamba', 'Premium', '2024-06-15', '9999-12-31', TRUE);
```

Estado de la tabla:

| cliente_sk | cliente_id | nombre | ciudad | fecha_inicio | fecha_fin | es_actual |
|---|---|---|---|---|---|---|
| 1001 | 1001 | Pedro Alvarado | La Paz | 2020-01-01 | 2024-06-14 | FALSE |
| 9999 | 1001 | Pedro Alvarado | Cochabamba | 2024-06-15 | 9999-12-31 | TRUE |

Ahora, **si las tablas de hechos referencian la `cliente_sk` (la surrogate key)**, las transacciones anteriores a 2024-06-15 siguen apuntando a `cliente_sk = 1001` (La Paz), y las posteriores apuntan a `cliente_sk = 9999` (Cochabamba). El historial es perfectamente coherente.

```sql
-- ¿Cuánto vendió Pedro en La Paz vs Cochabamba?
SELECT dc.ciudad, SUM(vh.monto_venta) AS total_ventas
FROM ventas_hechos vh
JOIN dim_cliente dc ON dc.cliente_sk = vh.cliente_sk  -- JOIN por surrogate key
WHERE dc.cliente_id = 1001  -- todas las versiones de Pedro
GROUP BY dc.ciudad;
-- Resultado: La Paz: X, Cochabamba: Y
-- Cada venta conserva la ciudad que correspondía cuando se hizo
```

**Cuándo usar SCD Tipo 2:**
- Cuando necesitas análisis histórico correcto ("ventas por región en el período que el cliente vivía allí").
- Para atributos que cambian con frecuencia moderada y cuyo historial importa.
- Cuando las tablas de hechos referencian la dimensión y esas referencias deben ser estables.

**El costo del Tipo 2:** La tabla de dimensión puede crecer mucho si los atributos cambian con frecuencia. Un cliente que cambia de ciudad 10 veces tendrá 10 filas. Las queries que necesitan "el valor actual" deben filtrar por `es_actual = TRUE` o `fecha_fin = '9999-12-31'`.

### 7.4 SCD Tipo 3: Columna del valor anterior

**La filosofía:** Solo importa el valor actual y el valor inmediatamente anterior. Se agrega una columna `_anterior` para cada atributo que se quiere rastrear.

```sql
-- Tabla de dimensión cliente (SCD Tipo 3)
CREATE TABLE dim_cliente (
    cliente_sk        INT PRIMARY KEY,
    cliente_id        INT NOT NULL,
    nombre            VARCHAR(100) NOT NULL,
    ciudad_actual     VARCHAR(100) NOT NULL,
    ciudad_anterior   VARCHAR(100),           -- solo la versión previa
    fecha_cambio      DATE                    -- cuándo fue el último cambio
);

-- Estado inicial: Pedro en La Paz
INSERT INTO dim_cliente VALUES
    (1001, 1001, 'Pedro Alvarado', 'La Paz', NULL, NULL);

-- Pedro se muda a Cochabamba
UPDATE dim_cliente
SET ciudad_anterior = ciudad_actual,
    ciudad_actual = 'Cochabamba',
    fecha_cambio = '2024-06-15'
WHERE cliente_id = 1001;
```

| cliente_sk | nombre | ciudad_actual | ciudad_anterior | fecha_cambio |
|---|---|---|---|---|
| 1001 | Pedro Alvarado | Cochabamba | La Paz | 2024-06-15 |

**Cuándo usar SCD Tipo 3:**
- Cuando solo necesitas la "perspectiva anterior" para comparaciones directas ("clientes que se mudaron: de dónde venían y a dónde fueron").
- Cuando el atributo cambia con muy poca frecuencia y rastrear más de dos versiones es innecesario.

**El problema del Tipo 3:** Si Pedro cambia de ciudad por segunda vez, la segunda ciudad anterior sobrescribe la primera. Solo se puede rastrear un nivel de historia. Si necesitas más, Tipo 2 es la respuesta.

### 7.5 SCD Tipo 4: Mini-dimensión

**La filosofía:** Los atributos que cambian con mucha frecuencia se separan de los atributos estables en una tabla aparte (la "mini-dimensión"), evitando que la tabla principal explote en filas.

```sql
-- Tabla principal: atributos estables del cliente
CREATE TABLE dim_cliente (
    cliente_sk    INT PRIMARY KEY,
    cliente_id    INT NOT NULL UNIQUE,
    nombre        VARCHAR(100) NOT NULL,
    fecha_nac     DATE
    -- Sin ciudad, segmento, etc. (atributos que cambian)
);

-- Mini-dimensión: atributos que cambian frecuentemente
CREATE TABLE dim_perfil_cliente (
    perfil_sk    INT PRIMARY KEY,
    ciudad       VARCHAR(100) NOT NULL,
    segmento     VARCHAR(50)  NOT NULL,
    rango_ingreso VARCHAR(20) NOT NULL,
    -- Este registro es reutilizable: si dos clientes tienen el mismo perfil,
    -- ambos referencian el mismo perfil_sk
    UNIQUE (ciudad, segmento, rango_ingreso)  -- perfil único por combinación
);

-- La tabla de hechos referencia ambas dimensiones
CREATE TABLE ventas_hechos (
    venta_id     BIGINT PRIMARY KEY,
    fecha_sk     INT NOT NULL,
    cliente_sk   INT NOT NULL REFERENCES dim_cliente(cliente_sk),
    perfil_sk    INT NOT NULL REFERENCES dim_perfil_cliente(perfil_sk),  -- snapshotted al momento de la venta
    monto        DECIMAL(12,2) NOT NULL
);
```

**Cuándo usar SCD Tipo 4:**
- Cuando un conjunto de atributos cambia muy frecuentemente (diariamente o semanalmente) y el Tipo 2 generaría millones de filas.
- Cuando se quiere compartir "perfiles" entre muchos clientes con las mismas características.

### 7.6 SCD Tipo 6: Híbrido (Tipo 1 + 2 + 3)

**La filosofía:** Combina los tres primeros tipos para obtener lo mejor de cada uno. Es la técnica más completa pero también más compleja.

```sql
-- SCD Tipo 6 (híbrido 1+2+3)
CREATE TABLE dim_cliente (
    cliente_sk         INT PRIMARY KEY,
    cliente_id         INT NOT NULL,
    nombre             VARCHAR(100) NOT NULL,
    
    -- El valor como era cuando esta versión era actual (Tipo 2)
    ciudad_version     VARCHAR(100) NOT NULL,
    
    -- La ciudad actual del cliente (se actualiza en TODAS las versiones, Tipo 1)
    ciudad_actual      VARCHAR(100) NOT NULL,
    
    -- La ciudad anterior (Tipo 3)
    ciudad_anterior    VARCHAR(100),
    
    -- Período de vigencia de esta versión (Tipo 2)
    fecha_inicio       DATE NOT NULL,
    fecha_fin          DATE NOT NULL DEFAULT '9999-12-31',
    es_actual          BOOLEAN NOT NULL DEFAULT TRUE
);
```

Con este esquema puedes responder:
- "¿En qué ciudad vivía Pedro cuando hizo la compra X?" → usa `ciudad_version` a través de la `cliente_sk`.
- "¿En qué ciudad vive Pedro ahora?" → usa `ciudad_actual` en cualquier versión.
- "¿De dónde vino Pedro?" → usa `ciudad_anterior` en la versión actual.

---

## 8. Modelado de eventos vs modelado de estado

### 8.1 Dos filosofías para representar la realidad

Hasta ahora hemos visto principalmente el **modelado de estado**: el esquema almacena cómo están las cosas en un momento dado, y el tiempo válido/transaccional agrega las dimensiones temporales. Pero existe una filosofía radicalmente diferente: el **modelado de eventos**.

La distinción fundamental:

- **Modelado de estado:** La tabla almacena el *estado actual* (o el estado en cada período). `UPDATE` cambia el estado.
- **Modelado de eventos:** La tabla almacena únicamente los *eventos* que ocurrieron. El estado se deriva calculando la secuencia de eventos. Nunca hay `UPDATE`, solo `INSERT`.

### 8.2 Un ejemplo concreto: cuenta bancaria

**Enfoque de estado:**

```sql
-- Modelado de estado: la tabla almacena el saldo actual
CREATE TABLE cuentas (
    cuenta_id  INT PRIMARY KEY,
    cliente_id INT NOT NULL,
    saldo      DECIMAL(12,2) NOT NULL DEFAULT 0
);

-- Depositar 1,000: actualizar el saldo
UPDATE cuentas SET saldo = saldo + 1000 WHERE cuenta_id = 1;

-- Retirar 300: actualizar el saldo
UPDATE cuentas SET saldo = saldo - 300 WHERE cuenta_id = 1;
```

**Enfoque de eventos:**

```sql
-- Modelado de eventos: la tabla almacena cada movimiento
CREATE TABLE movimientos (
    movimiento_id  SERIAL PRIMARY KEY,
    cuenta_id      INT           NOT NULL,
    tipo           VARCHAR(20)   NOT NULL,  -- 'deposito', 'retiro', 'transferencia'
    monto          DECIMAL(12,2) NOT NULL,  -- positivo = ingreso, negativo = egreso
    timestamp      TIMESTAMP     NOT NULL DEFAULT NOW(),
    descripcion    VARCHAR(200)
);

-- Depositar 1,000: insertar un evento
INSERT INTO movimientos (cuenta_id, tipo, monto) VALUES (1, 'deposito', 1000);

-- Retirar 300: insertar un evento
INSERT INTO movimientos (cuenta_id, tipo, monto) VALUES (1, 'retiro', -300);

-- El saldo actual se CALCULA sumando todos los movimientos
SELECT SUM(monto) AS saldo_actual
FROM movimientos
WHERE cuenta_id = 1;
-- Resultado: 700

-- El saldo en cualquier fecha pasada también se puede calcular
SELECT SUM(monto) AS saldo_al_1_de_enero
FROM movimientos
WHERE cuenta_id = 1
  AND timestamp <= '2024-01-01 23:59:59';
```

### 8.3 Comparación profunda de ambos enfoques

| Dimensión | Modelado de estado | Modelado de eventos |
|---|---|---|
| **Consulta del estado actual** | O(1): leer una fila | O(n): sumar todos los eventos |
| **Consulta histórica** | Requiere diseño temporal explícito | Natural: filtrar por fecha |
| **Reconstrucción de historial** | Imposible sin diseño temporal | Siempre posible |
| **Corrección de errores** | UPDATE destruye información | INSERT de un evento "corrector" |
| **Escalabilidad de escritura** | Contención en la fila del estado | Append-only, sin contención |
| **Auditoría** | Requiere infraestructura adicional | Inherente al modelo |
| **Complejidad** | Simple de entender | Más complejo conceptualmente |
| **Rendimiento de lectura** | Muy alto (directo) | Puede degradarse con muchos eventos |

### 8.4 El patrón de snapshot para optimizar eventos

Cuando el volumen de eventos es alto, recalcular el estado desde el principio en cada consulta se vuelve costoso. El patrón de **snapshot** resuelve esto: periódicamente se materializan snapshots del estado calculado, y las queries calculan desde el snapshot más reciente más los eventos posteriores.

```sql
-- Tabla de snapshots para cuentas bancarias
CREATE TABLE saldos_snapshot (
    snapshot_id  SERIAL PRIMARY KEY,
    cuenta_id    INT           NOT NULL,
    saldo        DECIMAL(12,2) NOT NULL,
    hasta_movimiento_id BIGINT NOT NULL,  -- hasta qué evento fue calculado
    calculado_en TIMESTAMP     NOT NULL DEFAULT NOW()
);

-- Query eficiente: saldo actual usando el snapshot más reciente + eventos posteriores
SELECT 
    ss.saldo + COALESCE(SUM(m.monto), 0) AS saldo_actual
FROM saldos_snapshot ss
LEFT JOIN movimientos m ON m.cuenta_id = ss.cuenta_id
    AND m.movimiento_id > ss.hasta_movimiento_id
WHERE ss.cuenta_id = 1
  AND ss.snapshot_id = (
      SELECT MAX(snapshot_id) FROM saldos_snapshot WHERE cuenta_id = 1
  )
GROUP BY ss.saldo;
```

### 8.5 Cuándo elegir eventos vs estado

Elige **modelado de eventos** cuando:
- Los eventos en sí son el objeto de interés (transacciones, movimientos, cambios).
- La auditoría completa es un requisito no negociable.
- Necesitas reproducir el estado en cualquier momento pasado.
- El dominio es naturalmente secuencial (finanzas, inventario, workflows).
- Múltiples subsistemas necesitan reaccionar a los mismos cambios.

Elige **modelado de estado** cuando:
- Solo importa el estado actual.
- Las consultas de estado son de alta frecuencia y latencia crítica.
- El dominio no tiene historial significativo.
- El equipo no está familiarizado con el paradigma de eventos y la complejidad adicional no es justificable.

**La regla práctica:** Si tu dominio tiene la palabra "historial", "movimiento", "registro", "bitácora", o "log" en sus requerimientos funcionales, considera seriamente el modelado de eventos.

---

## 9. Event Sourcing como patrón de modelado

### 9.1 Qué es Event Sourcing

**Event Sourcing** lleva el modelado de eventos a su conclusión lógica: **el estado de cualquier entidad es completamente derivable de la secuencia de eventos que la afectaron**. No hay una tabla de "estado actual"; el estado se computa siempre desde los eventos.

Este patrón es popularizado en el contexto de arquitecturas DDD (Domain-Driven Design) y CQRS (Command Query Responsibility Segregation), pero sus implicaciones de modelado de datos son independientes de esas arquitecturas.

### 9.2 La estructura de un event store relacional

```sql
-- El event store: la única fuente de verdad
CREATE TABLE eventos (
    evento_id        BIGSERIAL    PRIMARY KEY,
    stream_id        VARCHAR(100) NOT NULL,  -- identifica la entidad (p.ej. 'pedido-12345')
    tipo_evento      VARCHAR(100) NOT NULL,  -- nombre del evento (p.ej. 'PedidoCreado')
    version          INT          NOT NULL,  -- versión dentro del stream (para control de concurrencia)
    payload          JSONB        NOT NULL,  -- los datos del evento
    metadata         JSONB        NOT NULL DEFAULT '{}',  -- datos de infraestructura
    timestamp        TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    
    -- Garantía de orden y unicidad: no puede haber dos eventos con la misma versión
    -- para el mismo stream (previene conflictos de concurrencia optimista)
    UNIQUE (stream_id, version)
);

CREATE INDEX idx_eventos_stream ON eventos (stream_id, version);
CREATE INDEX idx_eventos_tipo ON eventos (tipo_evento);
CREATE INDEX idx_eventos_timestamp ON eventos (timestamp);
```

### 9.3 Un ejemplo completo: ciclo de vida de un pedido

```sql
-- Los eventos que ocurrieron para el pedido #1234
INSERT INTO eventos (stream_id, tipo_evento, version, payload) VALUES
(
    'pedido-1234', 'PedidoCreado', 1,
    '{
        "cliente_id": 42,
        "items": [
            {"producto_id": 100, "cantidad": 2, "precio_unitario": 49.90},
            {"producto_id": 205, "cantidad": 1, "precio_unitario": 120.00}
        ],
        "total": 219.80
    }'
),
(
    'pedido-1234', 'DireccionConfirmada', 2,
    '{
        "calle": "Av. Heroínas 1250",
        "ciudad": "Cochabamba",
        "codigo_postal": "0000"
    }'
),
(
    'pedido-1234', 'PagoAcreditado', 3,
    '{
        "metodo": "tarjeta",
        "monto": 219.80,
        "referencia_pago": "TXN-ABC-12345"
    }'
),
(
    'pedido-1234', 'PedidoEnviado', 4,
    '{
        "numero_guia": "GUIA-999-XYZ",
        "transportista": "Correos Bolivia",
        "fecha_estimada_entrega": "2024-12-20"
    }'
);
```

Para reconstruir el estado actual del pedido, la aplicación lee todos los eventos en orden y los "aplica" uno a uno:

```sql
-- Reconstruir el estado del pedido 1234
SELECT tipo_evento, version, payload, timestamp
FROM eventos
WHERE stream_id = 'pedido-1234'
ORDER BY version;
```

La aplicación procesa esta secuencia y construye el estado: "el pedido 1234 fue creado, la dirección fue confirmada, el pago fue acreditado, y fue enviado con la guía XYZ".

### 9.4 Proyecciones: el estado materializado

Reconstruir el estado desde los eventos en cada consulta es costoso para lecturas frecuentes. La solución son las **proyecciones**: tablas denormalizadas que materializan el estado actual, construidas aplicando eventos a medida que llegan.

```sql
-- Proyección: estado actual de los pedidos (tabla de lectura)
CREATE TABLE proyeccion_pedidos (
    pedido_id    VARCHAR(100) PRIMARY KEY,
    cliente_id   INT          NOT NULL,
    estado       VARCHAR(50)  NOT NULL,  -- 'creado', 'confirmado', 'pagado', 'enviado', 'entregado'
    total        DECIMAL(12,2) NOT NULL,
    numero_guia  VARCHAR(100),
    ultima_actualizacion TIMESTAMPTZ NOT NULL
);

-- Esta proyección se actualiza aplicando los eventos a medida que llegan
-- (generalmente via un proceso de background o trigger en el event store)
```

La arquitectura de Event Sourcing con proyecciones se parece a esto:

```
[Escrituras] → Event Store (eventos) ← [Proyector] → Proyecciones (lecturas)
```

Las escrituras solo agregan eventos. Las lecturas solo consultan las proyecciones. Ambas son independientes y pueden escalarse por separado. Esto es la esencia de CQRS.

### 9.5 Ventajas y costos reales de Event Sourcing

**Ventajas:**
- **Auditoría perfecta:** El event store *es* el log de auditoría.
- **Time travel:** Puedes reconstruir el estado en cualquier momento pasado.
- **Debugging:** Puedes reproducir exactamente la secuencia de eventos que llevó a un estado inesperado.
- **Integraciones:** Otros sistemas pueden suscribirse a los eventos y construir sus propias proyecciones.
- **Correcciones sin pérdida:** Un error se corrige con un nuevo evento compensador, no borrando el error.

**Costos:**
- **Complejidad:** El equipo necesita un cambio de mentalidad significativo.
- **Latencia en proyecciones:** Las proyecciones son eventualmente consistentes con el event store.
- **Evolución del esquema de eventos:** Cambiar la estructura de un tipo de evento es difícil si hay miles de eventos del tipo antiguo.
- **Consultas ad-hoc:** Es difícil hacer queries que no sean sobre el estado proyectado.
- **No aplica a todos los dominios:** Para datos de referencia simples (catálogos, configuración), ES es overkill.

### 9.6 Event Sourcing en el modelo relacional puro

Es importante notar que Event Sourcing **no requiere infraestructura especial**. Un event store puede ser una tabla en PostgreSQL, como la que vimos arriba. Las proyecciones pueden ser tablas normales. El "proyector" puede ser un trigger, un proceso de background, o una función de aplicación.

La complejidad de ES viene del paradigma, no de la tecnología. Se puede implementar completamente dentro de un RDBMS convencional.

---

## 10. Patrones de consulta temporal y sus trade-offs

### 10.1 Las cuatro queries fundamentales de los modelos temporales

Cualquier sistema con datos temporales eventualmente necesita responder estas cuatro queries:

```sql
-- Dado un esquema con tiempo válido
-- CREATE TABLE precios (
--     producto_id  INT,
--     precio       DECIMAL(12,2),
--     valido_desde DATE,
--     valido_hasta DATE
-- );

-- Query 1: Estado en una fecha específica ("AS OF")
-- ¿Cuánto costaba el producto 100 el 15 de noviembre?
SELECT precio
FROM precios
WHERE producto_id = 100
  AND valido_desde <= '2024-11-15'
  AND valido_hasta > '2024-11-15';

-- Query 2: Estado actual
-- ¿Cuánto cuesta el producto 100 ahora?
SELECT precio
FROM precios
WHERE producto_id = 100
  AND valido_hasta = 'infinity';  -- o AND valido_hasta > CURRENT_DATE

-- Query 3: Historia completa
-- ¿Cuál fue la historia de precios del producto 100?
SELECT precio, valido_desde, valido_hasta
FROM precios
WHERE producto_id = 100
ORDER BY valido_desde;

-- Query 4: ¿Qué era válido durante un rango de tiempo?
-- ¿Qué precios tuvo el producto 100 durante 2024?
SELECT precio, valido_desde, valido_hasta
FROM precios
WHERE producto_id = 100
  AND valido_desde < '2025-01-01'
  AND valido_hasta > '2024-01-01';
-- Condición: el período del precio y el año 2024 se solapan
```

### 10.2 El problema del "current" query y la performance

La query más común en sistemas temporales es "dame el valor actual". Si se usa `valido_hasta = 'infinity'`, el índice B-tree en `valido_hasta` es poco efectivo porque hay muchas filas con ese valor.

Una estrategia mejor:

```sql
-- Opción 1: Índice parcial (solo las filas vigentes)
CREATE INDEX idx_precios_actuales ON precios (producto_id)
WHERE valido_hasta = 'infinity';

-- La query aprovecha el índice parcial
SELECT precio FROM precios
WHERE producto_id = 100
  AND valido_hasta = 'infinity';

-- Opción 2: Tabla separada para el estado actual (desnormalización)
CREATE TABLE precios_actuales (
    producto_id INT PRIMARY KEY REFERENCES productos,
    precio      DECIMAL(12,2) NOT NULL,
    desde       DATE NOT NULL
);
-- Esta tabla se actualiza con un trigger cuando cambia precios.
-- Las queries de "precio actual" van aquí (O(1), sin filtros temporales).
-- Las queries históricas van a la tabla precios.
```

### 10.3 Generación de series temporales con `generate_series`

Para análisis temporales, frecuentemente se necesita "rellenar" períodos sin datos (p.ej., para un gráfico diario). PostgreSQL's `generate_series` es invaluable:

```sql
-- ¿Cuál era el precio del producto 100 para cada día de diciembre 2024?
SELECT 
    gs.fecha,
    p.precio
FROM generate_series(
    '2024-12-01'::date, 
    '2024-12-31'::date, 
    '1 day'::interval
) AS gs(fecha)
LEFT JOIN precios p ON p.producto_id = 100
    AND p.valido_desde <= gs.fecha
    AND p.valido_hasta > gs.fecha
ORDER BY gs.fecha;
-- Resultado: una fila por día, con el precio vigente ese día
-- Si algún día no tiene precio (gap), aparece NULL en precio
```

### 10.4 Detección de gaps y overlaps

En sistemas con tiempo válido, es crítico poder detectar violaciones de integridad que no fueron capturadas por restricciones:

```sql
-- Detectar overlaps en la tabla de precios (sin usar rangos nativos)
SELECT p1.precio AS precio_1, p1.valido_desde AS inicio_1, p1.valido_hasta AS fin_1,
       p2.precio AS precio_2, p2.valido_desde AS inicio_2, p2.valido_hasta AS fin_2
FROM precios p1
JOIN precios p2 ON p1.producto_id = p2.producto_id
    AND p1.valido_desde < p2.valido_hasta
    AND p1.valido_hasta > p2.valido_desde
    AND p1.precio <> p2.precio  -- Excluir la misma fila comparada consigo misma
WHERE p1.producto_id = 100;

-- Detectar gaps: períodos sin precio entre fechas conocidas
WITH periodos AS (
    SELECT valido_desde, valido_hasta,
           LAG(valido_hasta) OVER (ORDER BY valido_desde) AS fin_anterior
    FROM precios
    WHERE producto_id = 100
)
SELECT fin_anterior AS inicio_gap, valido_desde AS fin_gap
FROM periodos
WHERE fin_anterior < valido_desde;  -- el fin anterior es menor que el inicio: hay un gap
```

### 10.5 Normalización temporal: el principio de cohesión temporal

Un principio de diseño que suele ignorarse: **los atributos que tienen la misma historia temporal deben estar en la misma tabla; los que tienen historias distintas deben estar en tablas separadas.**

Si tienes una tabla `empleados` con atributos `nombre`, `departamento`, `salario`, y `fecha_ingreso`, los cuatro tienen comportamientos temporales distintos:

- `nombre`: raramente cambia (tal vez una vez en la vida).
- `departamento`: cambia con frecuencia moderada.
- `salario`: cambia con frecuencia moderada.
- `fecha_ingreso`: nunca cambia.

Si aplicas un modelo de tiempo válido a toda la tabla, cada cambio de salario crea una nueva versión de la fila completa, incluyendo el nombre y la fecha de ingreso que no cambiaron. Esto es redundante y puede generar confusión.

La solución más limpia es **separar los atributos por sus características temporales**:

```sql
-- Datos estables del empleado (nunca o raramente cambian)
CREATE TABLE empleados (
    empleado_id   INT PRIMARY KEY,
    nombre        VARCHAR(100) NOT NULL,
    fecha_ingreso DATE NOT NULL
);

-- Historial de departamento (cambia por sí solo)
CREATE TABLE historial_departamento (
    empleado_id  INT  NOT NULL REFERENCES empleados,
    departamento VARCHAR(100) NOT NULL,
    periodo      DATERANGE NOT NULL,
    EXCLUDE USING GIST (empleado_id WITH =, periodo WITH &&)
);

-- Historial de salario (cambia por sí solo, independiente del departamento)
CREATE TABLE historial_salario (
    empleado_id  INT  NOT NULL REFERENCES empleados,
    salario      DECIMAL(12,2) NOT NULL,
    periodo      DATERANGE NOT NULL,
    EXCLUDE USING GIST (empleado_id WITH =, periodo WITH &&)
);
```

Este diseño es más complejo en las queries (requiere más JOINs), pero es más preciso: una fila en `historial_salario` representa exactamente un período de un salario, sin mezclar otros cambios.

---

## 11. Resumen y conexión con el resto del curso

### Los conceptos clave y cuándo aplicarlos

| Técnica | Problema que resuelve | Señal de que la necesitas |
|---|---|---|
| **Tiempo válido** | Registrar cuándo algo fue verdad en el mundo | "Necesitamos saber el precio cuando se emitió la factura" |
| **Tiempo de transacción** | Auditoría: qué decía el sistema en un momento dado | "El auditor pregunta qué teníamos registrado el 31 de dic." |
| **Tabla bitemporal** | Ambos: historial del mundo + historial del sistema | "Hay correcciones retroactivas y necesitamos rastro de ellas" |
| **SCD Tipo 1** | El pasado no importa, solo el valor actual | "El nombre estaba mal escrito, simplemente corrígelo" |
| **SCD Tipo 2** | Historial completo con coherencia en las transacciones | "Las ventas deben reportarse según la región del cliente cuando ocurrieron" |
| **SCD Tipo 3** | Solo importa el valor actual y el anterior | "¿De dónde a dónde se mudaron nuestros clientes?" |
| **Modelado de eventos** | Los eventos son el objeto de análisis | "Necesitamos el historial completo de movimientos" |
| **Event Sourcing** | El estado es completamente derivable de los eventos | "Necesitamos poder reproducir cualquier estado pasado" |

### La meta-lección del módulo

El tiempo es la dimensión más ignorada en el diseño de bases de datos porque su tratamiento correcto añade complejidad: más columnas, más restricciones, más lógica en las queries. La tentación es ignorarla hasta que el negocio pregunta algo histórico.

El costo de añadir dimensiones temporales después es mucho mayor que diseñarlas desde el principio: los datos históricos que no se capturaron están perdidos, y las migraciones para agregar columnas temporales a tablas con millones de filas son costosas y riesgosas.

La pregunta correcta no es "¿necesitamos historial?" sino "¿qué preguntas históricas se podrían hacer sobre estos datos?". Si la respuesta no es definitivamente "ninguna", el modelo necesita al menos tiempo válido.

### Conexión con módulos futuros

- **Módulo 7 (Internals y query planner):** Las queries temporales con rangos de fechas, CTEs para historial, y filtros en columnas de período tienen características de rendimiento muy específicas. Los índices en columnas de rango, los índices parciales para "registros actuales", y el comportamiento del query planner con filtros `<= fecha AND > fecha` son temas que se cubren allí.

- **Módulo 9 (Modelado físico e índices):** La decisión de usar `DATERANGE` con índices GiST vs columnas separadas con índices B-tree tiene implicaciones de rendimiento y tamaño de índice que dependen del volumen de datos y el patrón de acceso.

- **Módulo 10 y 11 (OLAP y Dimensional Modeling):** Las Slowly Changing Dimensions de Kimball son la aplicación directa de los conceptos de tiempo válido en el contexto de data warehouses. El Tipo 2 es exactamente un modelo de tiempo válido aplicado a dimensiones analíticas.

- **Módulo 13 (Sistemas distribuidos):** En sistemas distribuidos, la noción de "cuándo ocurrió algo" es filosóficamente compleja porque los relojes de distintos nodos no están sincronizados. Los timestamps de transacción en un sistema distribuido requieren vectores de reloj o timestamps híbridos.

---

## 12. Ejercicios de comprensión

**Ejercicio 1.** Un sistema de gestión de personal necesita responder las siguientes preguntas:

a) "¿En qué departamento estaba María el 15 de septiembre del año pasado?"
b) "¿Qué decía el sistema sobre el departamento de María el 15 de septiembre, antes de que se corrigiera un error de asignación?"
c) "¿Cuántas veces ha cambiado de departamento cada empleado en los últimos 3 años?"

Para cada pregunta, identifica: ¿requiere tiempo válido, tiempo de transacción, o ambos? Diseña el esquema que permite responder las tres.

---

**Ejercicio 2.** Tienes la siguiente tabla de precios sin soporte temporal:

```sql
CREATE TABLE precios (
    producto_id  INT PRIMARY KEY REFERENCES productos,
    precio       DECIMAL(12,2) NOT NULL
);
```

El negocio ahora requiere:
- Registrar precios futuros con anticipación.
- Saber cuánto costaba un producto cuando se emitió cualquier factura histórica.
- No se requiere auditoría de quién cambió qué.

a) Migra el esquema al modelo de tiempo válido. Incluye las restricciones de integridad necesarias.
b) Escribe la query que devuelve el precio correcto para una factura dada su fecha de emisión.
c) Escribe el procedimiento (o las sentencias SQL) para registrar un nuevo precio a partir de una fecha futura.

---

**Ejercicio 3.** Un data warehouse tiene una tabla de dimensión `dim_vendedor` con los atributos: nombre, región, canal (online/presencial), y nivel (junior/senior/master). Los vendedores cambian de región ocasionalmente (1-2 veces por año), de canal raramente, y de nivel aproximadamente una vez cada 2-3 años.

a) ¿Qué tipo de SCD aplicarías para cada atributo? Justifica considerando la frecuencia de cambio y las necesidades analíticas.
b) Diseña el esquema final de `dim_vendedor` con la estrategia elegida.
c) Escribe la query para responder: "¿Cuánto vendió cada vendedor en su región actual vs en la región donde estaba hace 2 años?"

---

**Ejercicio 4.** Considera el siguiente escenario de un sistema de seguros: Una póliza tiene una cobertura de $100,000 desde el 1 de enero. El 1 de julio se incrementa a $150,000. El 15 de agosto ocurre un siniestro. El 20 de agosto el ajustador descubre que, por un error administrativo, el incremento de julio nunca debería haberse aplicado: la cobertura correcta desde enero era $80,000.

a) Modela la tabla `coberturas_poliza` como una tabla bitemporal.
b) Muestra el estado de la tabla después de cada uno de los cuatro eventos (registro inicial, incremento, siniestro, corrección).
c) Escribe la query para responder: "¿Cuál era la cobertura correcta el día del siniestro (15 de agosto)?" y "¿Qué decía el sistema sobre la cobertura el día del siniestro?"

---

**Ejercicio 5.** Diseña un event store relacional para el ciclo de vida de una orden de compra en un sistema de e-procurement. Los eventos posibles son: `OrdenCreada`, `ItemAgregado`, `ItemEliminado`, `OrdenAprobada`, `OrdenRechazada`, `OrdenEnviada`, `OrdenRecibida`, `OrdenCancelada`.

a) Define el esquema del event store con todas las restricciones necesarias.
b) Define el esquema de una proyección que muestre el estado actual de cada orden.
c) Escribe el pseudo-código de la función que aplica un evento al estado actual para actualizar la proyección.
d) ¿Cómo manejarías una `OrdenCancelada` que llega para una orden que ya fue `RecibidaConfirmada`? ¿Qué estrategia usarías en el event store?

---

**Ejercicio 6.** Evalúa el siguiente diseño que intenta manejar el tiempo de forma ad-hoc e identifica todos sus problemas. Propón el diseño correcto:

```sql
CREATE TABLE contratos (
    contrato_id       INT PRIMARY KEY,
    cliente_id        INT NOT NULL,
    monto_mensual     DECIMAL(10,2) NOT NULL,
    fecha_inicio      DATE NOT NULL,
    fecha_fin         DATE,           -- NULL significa activo
    version_anterior  INT REFERENCES contratos,  -- puntero al contrato anterior
    modificado_en     TIMESTAMP,
    modificado_por    VARCHAR(100)
);
```

---

*Próximo módulo: Internals — donde bajaremos al nivel del motor para entender cómo una query transita desde el texto SQL hasta los bytes en disco, y cómo ese viaje determina si un modelo bien diseñado es también un modelo eficiente en producción.*
