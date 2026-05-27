# Módulo 16 — El oficio: del requerimiento al modelo en producción

> *"La teoría dice que un modelo debe estar en tercera forma normal, que las constraints deben declararse en la base de datos, que los índices deben cubrir los patrones de acceso. La práctica dice que el cliente no sabe exactamente qué datos va a necesitar mañana, que el plazo es la mitad del tiempo necesario, que el sistema heredado tiene diez años de deuda técnica que nadie quiere tocar, y que la migración tiene que hacerse sin apagar nada. El oficio es la habilidad de hacer las cosas bien dentro de esas restricciones. No es un compromiso con la mediocridad; es la ingeniería real."*

---

## Tabla de contenidos

1. [Del requerimiento al modelo: la conversación](#1-del-requerimiento-al-modelo-la-conversación)
2. [Las preguntas que revelan el modelo](#2-las-preguntas-que-revelan-el-modelo)
3. [Del modelo conceptual al esquema: el proceso completo](#3-del-modelo-conceptual-al-esquema-el-proceso-completo)
4. [Evaluar un modelo ajeno: la auditoría de esquemas](#4-evaluar-un-modelo-ajeno-la-auditoría-de-esquemas)
5. [Refactoring de esquemas en sistemas vivos](#5-refactoring-de-esquemas-en-sistemas-vivos)
6. [El patrón Expand/Contract](#6-el-patrón-expandcontract)
7. [Backfill progresivo](#7-backfill-progresivo)
8. [Migraciones sin downtime: blue/green y técnicas avanzadas](#8-migraciones-sin-downtime-bluegreen-y-técnicas-avanzadas)
9. [La tensión entre el modelo correcto y el modelo entregable](#9-la-tensión-entre-el-modelo-correcto-y-el-modelo-entregable)
10. [Documentar el modelo: la memoria del sistema](#10-documentar-el-modelo-la-memoria-del-sistema)
11. [El modelo como contrato entre equipos](#11-el-modelo-como-contrato-entre-equipos)
12. [Seguridad a nivel de datos](#12-seguridad-a-nivel-de-datos)
13. [El modelo de datos como decisión estratégica](#13-el-modelo-de-datos-como-decisión-estratégica)
14. [Síntesis del curso](#14-síntesis-del-curso)
15. [Ejercicios de comprensión](#15-ejercicios-de-comprensión)

---

## 1. Del requerimiento al modelo: la conversación

### 1.1 Por qué el requerimiento no es el modelo

Cuando un cliente o un product manager describe lo que necesita, no describe un modelo de datos. Describe un problema de negocio, o una funcionalidad deseada, o un proceso que quiere digitalizar. El trabajo del diseñador de datos es traducir esa descripción —que está en el lenguaje del negocio— a un modelo que sea correcto, completo y duradero.

El error más frecuente en este proceso es leer el requerimiento y empezar a dibujar tablas de inmediato. El requerimiento describe el **camino feliz**: lo que ocurre cuando todo va bien. El modelo de datos tiene que manejar el camino feliz y también todos los casos borde, excepciones, cambios futuros probables, y condiciones de error. Esa información no está en el requerimiento; hay que extraerla mediante preguntas.

Considera este requerimiento:

> *"Necesitamos un sistema para gestionar las reservas de nuestros espacios de coworking. Los clientes deben poder reservar salas por horas. Las salas tienen distintas capacidades y precios."*

Un diseñador apresurado produce esto:

```sql
CREATE TABLE clientes (id, nombre, email);
CREATE TABLE salas (id, nombre, capacidad, precio_por_hora);
CREATE TABLE reservas (id, cliente_id, sala_id, inicio, fin, total);
```

Este esquema es correcto para el camino feliz. Pero hay decenas de preguntas sin responder que pueden hacer que sea completamente inadecuado para el sistema real. Las próximas dos secciones exploran sistemáticamente cómo descubrir esas preguntas.

### 1.2 El dominio no existe en la cabeza del desarrollador

Una de las verdades más incómodas del diseño de datos es que el desarrollador no conoce el dominio. El conocimiento del dominio vive en la cabeza del cliente, del experto del negocio, del usuario final. El diseñador solo puede acceder a ese conocimiento mediante conversación, observación y ejemplos concretos.

Esta es la razón por la que los modelos de datos diseñados en aislamiento (sin hablar con quienes conocen el negocio) tienden a ser elegantes pero incorrectos: reflejan la comprensión del desarrollador, no la realidad del dominio.

El proceso correcto no es diseñar y luego validar. Es **diseñar en conversación continua**, alternando preguntas, propuestas de modelo, y validación con los expertos del dominio.

---

## 2. Las preguntas que revelan el modelo

### 2.1 La primera categoría: preguntas sobre los datos en sí

**Preguntas sobre identidad:**
- ¿Qué identifica de forma única a una [entidad]? ¿El email? ¿Un código interno? ¿Puede cambiar?
- ¿Puede existir un [cliente] sin [email]? ¿Qué pasa si el email cambia?
- ¿Hay dos tipos de [clientes] que son fundamentalmente distintos? (señal del patrón Party o jerarquía de tipos)

**Preguntas sobre cardinalidad:**
- ¿Un [cliente] puede tener múltiples [reservas]? ¿Simultáneas?
- ¿Una [sala] puede tener múltiples [reservas] en el mismo día? ¿Solapadas?
- ¿Un [pago] puede cubrir múltiples [reservas]? ¿Una [reserva] puede tener múltiples [pagos] parciales?

Cada respuesta a estas preguntas de cardinalidad define relaciones y constraints que deben aparecer en el modelo.

**Preguntas sobre valores permitidos:**
- ¿Qué estados puede tener una [reserva]? ¿Puede cancelarse? ¿Puede modificarse?
- ¿Hay estados inválidos? ¿Puede una reserva cancelada volverse activa?
- ¿Hay atributos con valores restringidos? ¿Los precios pueden ser negativos? ¿La capacidad cero?

Estas respuestas definen constraints de dominio, CHECK constraints, y el diseño de máquinas de estado.

### 2.2 La segunda categoría: preguntas sobre el tiempo

Uno de los mayores errores de modelado es ignorar la dimensión temporal. Las preguntas sobre el tiempo revelan si el modelo necesita historial, bitemporalidad, o solo el estado actual.

**Preguntas sobre cambio:**
- ¿Qué pasa cuando [el precio de una sala] cambia? ¿Las reservas existentes mantienen el precio antiguo o usan el nuevo?
- ¿Qué pasa cuando un [cliente] cambia su [email]? ¿Necesitamos el email que tenía cuando hizo cada reserva?
- ¿Puede un [cliente] eliminarse del sistema? ¿O se desactiva? ¿Qué pasa con su historial?

**Preguntas sobre historial:**
- ¿Necesitan ver el historial de cambios de [estado de una reserva]? ¿O solo el estado actual?
- ¿Hay reportes que pregunten "¿cómo estaba el sistema en una fecha pasada?"?
- ¿Necesitan auditoría de quién cambió qué y cuándo?

Las respuestas a estas preguntas determinan si se necesitan tablas de historial (módulo 6: tiempo válido, tiempo de transacción, tablas bitemporales, event sourcing).

**Ejemplo concreto de cómo una respuesta cambia todo el modelo:**

Pregunta: *"¿Si el precio de una sala cambia, las reservas futuras ya confirmadas mantienen el precio al que se reservaron?"*

- Respuesta "SÍ": El precio debe guardarse en la reserva al momento de crearla, no solo en la sala. Además, es posible que se necesite un historial de precios por sala con vigencia temporal.
- Respuesta "NO": El precio se calcula al momento de la facturación desde la tabla de precios vigentes. Las reservas no necesitan almacenar precio.

Estas dos respuestas producen modelos completamente distintos, y ninguna es obvia desde el requerimiento original.

### 2.3 La tercera categoría: preguntas sobre los casos borde

El camino feliz no revela el modelo real. Los casos borde sí.

**Preguntas sobre la excepción:**
- ¿Puede un cliente reservar sin haber pagado? ¿Puede haber una reserva sin cliente (walk-in)?
- ¿Puede una sala no tener precio asignado? ¿Qué pasa entonces?
- ¿Pueden dos reservas solaparse en la misma sala? ¿Cómo se previene?
- ¿Qué pasa si el cliente no aparece? ¿Qué pasa con la sala durante ese tiempo?

**Preguntas sobre volumen extremo:**
- ¿Cuántas salas tiene el sistema? ¿Cuántos clientes? ¿Cuántas reservas por día?
- ¿Hay picos de carga? ¿Cuándo? (Esto afecta el diseño físico: índices, particionamiento)
- ¿Qué tan antiguo es el historial que se necesita accesible? (Afecta la política de retención y archivado)

**Preguntas sobre el futuro probable:**
- ¿Habrá múltiples sedes en el futuro? (Señal de que `sede_id` debe entrar en el modelo desde el inicio)
- ¿Habrá diferentes tipos de espacios además de salas? (Señal de generalización)
- ¿Se necesitará integración con sistemas de facturación externos?

Las respuestas a las preguntas sobre el futuro no deben sobrediseñar el sistema (la trampa del YAGNI: "you ain't gonna need it"), pero sí deben revelar si el modelo actual tiene puntos de extensión fáciles o si va a requerir migraciones costosas para cambios probables.

### 2.4 La técnica del ejemplo concreto

La forma más eficiente de extraer el modelo de una conversación es pedir **ejemplos concretos** en lugar de descripciones abstractas.

En lugar de: *"¿Cómo funciona el proceso de reserva?"*

Preguntar: *"Cuéntame step by step qué pasa exactamente desde que un cliente decide reservar hasta que la reserva está confirmada. ¿Qué datos concretos se recopilan en cada paso?"*

En lugar de: *"¿Los precios son fijos o variables?"*

Preguntar: *"Si María quiere reservar la sala 'Azul' el próximo martes de 10am a 12pm, ¿cuánto le cobrarías? ¿Y si fuera el jueves de 9pm a 11pm? ¿Y si fuera el fin de semana?"*

Los ejemplos concretos revelan casos que las descripciones abstractas ocultan. Si al pedir el precio del fin de semana el cliente dice "ah, los fines de semana tienen tarifa especial", eso es una regla de negocio que el modelo tiene que capturar y que no apareció en el requerimiento original.

---

## 3. Del modelo conceptual al esquema: el proceso completo

### 3.1 El proceso iterativo de tres capas

El diseño de un modelo de datos no ocurre de una vez. Es un proceso iterativo que pasa por tres capas, refinando en cada vuelta:

```
┌─────────────────────────────────────────────────────────────────┐
│  CAPA CONCEPTUAL (Módulo 3)                                     │
│  Diagrama EER: entidades, relaciones, cardinalidades            │
│  Lenguaje del negocio; independiente del motor                  │
│  Validar con el cliente: ¿esto representa su realidad?          │
├─────────────────────────────────────────────────────────────────┤
│  CAPA LÓGICA (Módulos 1, 2, 4, 5, 6)                           │
│  Esquema relacional: tablas, columnas, tipos, constraints       │
│  Normalización: dependencias funcionales, formas normales       │
│  Integridad: FKs, CHECKs, UNIQUE, NOT NULL                     │
│  Patrones: Party, roles, jerarquías, temporalidad               │
├─────────────────────────────────────────────────────────────────┤
│  CAPA FÍSICA (Módulos 7, 9)                                     │
│  Índices: B-tree, GIN, compuestos, parciales, de expresión      │
│  Particionamiento: rango, lista, hash                           │
│  Tipos concretos: VARCHAR vs TEXT, TIMESTAMP vs TIMESTAMPTZ     │
│  Afinación: fill factor, clustering, estadísticas               │
└─────────────────────────────────────────────────────────────────┘
```

Cada capa se valida con distintas preguntas:

- **Conceptual:** ¿Refleja correctamente el dominio? ¿Faltan entidades? ¿Las cardinalidades son correctas?
- **Lógica:** ¿Hay anomalías de actualización? ¿Las constraints capturan todas las invariantes del negocio? ¿El modelo soporta las queries que necesita el sistema?
- **Física:** ¿Los índices cubren los patrones de acceso? ¿El tamaño en disco es razonable? ¿Los tipos de datos son los más eficientes?

### 3.2 El ejemplo completo: sistema de coworking

Tomemos el requerimiento del ejemplo anterior y apliquemos el proceso completo, incluyendo las respuestas a las preguntas que haríamos.

**Resultado de la conversación:** Después de las preguntas, sabemos:
- Los clientes son siempre empresas o individuos (patrón Party).
- Las salas tienen tarifas distintas según el horario (diurna/nocturna) y el día de semana/fin de semana.
- Las reservas tienen estados: `pendiente → confirmada → en_uso → completada | cancelada`.
- El precio se fija al momento de confirmar la reserva (no al facturar).
- Los pagos pueden ser parciales: una reserva puede tener múltiples pagos hasta completar el total.
- Se necesita saber quién hizo cada cambio de estado (auditoría).
- En el futuro probablemente habrá múltiples sedes.

**Capa conceptual (simplificada):**

```
PARTY (1,N)──reserva──(1,1) RESERVA
SALA  (1,N)──ocupa────(1,1) RESERVA
RESERVA (1,N)──tiene──(0,N) PAGO
RESERVA (1,N)──estado──(0,N) HISTORIAL_ESTADO
TARIFA (vigencia temporal) pertenece a SALA
```

**Capa lógica:**

```sql
-- Patrón Party para clientes
CREATE TABLE parties (
    party_id    BIGSERIAL PRIMARY KEY,
    party_type  TEXT NOT NULL CHECK (party_type IN ('individual', 'empresa')),
    nombre      TEXT NOT NULL,
    email       TEXT NOT NULL,
    activo      BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE individuals (
    party_id    BIGINT PRIMARY KEY REFERENCES parties(party_id),
    documento   TEXT NOT NULL UNIQUE
);

CREATE TABLE empresas (
    party_id    BIGINT PRIMARY KEY REFERENCES parties(party_id),
    rut         TEXT NOT NULL UNIQUE,
    razon_social TEXT NOT NULL
);

-- Sede: preparado para expansión futura
CREATE TABLE sedes (
    sede_id     BIGSERIAL PRIMARY KEY,
    nombre      TEXT NOT NULL,
    direccion   TEXT NOT NULL,
    ciudad      TEXT NOT NULL
);

-- Sala
CREATE TABLE salas (
    sala_id     BIGSERIAL PRIMARY KEY,
    sede_id     BIGINT NOT NULL REFERENCES sedes(sede_id),
    nombre      TEXT NOT NULL,
    capacidad   INT NOT NULL CHECK (capacidad > 0),
    descripcion TEXT,
    activa      BOOLEAN NOT NULL DEFAULT TRUE,
    UNIQUE (sede_id, nombre)
);

-- Tarifas con vigencia temporal (Módulo 6: tiempo válido)
CREATE TABLE tarifas (
    tarifa_id       BIGSERIAL PRIMARY KEY,
    sala_id         BIGINT NOT NULL REFERENCES salas(sala_id),
    tipo_horario    TEXT NOT NULL CHECK (tipo_horario IN ('diurno', 'nocturno')),
    tipo_dia        TEXT NOT NULL CHECK (tipo_dia IN ('semana', 'finde')),
    precio_por_hora NUMERIC(10,2) NOT NULL CHECK (precio_por_hora >= 0),
    vigente_desde   DATE NOT NULL,
    vigente_hasta   DATE,  -- NULL = vigente indefinidamente
    CHECK (vigente_hasta IS NULL OR vigente_hasta > vigente_desde)
);

-- Máquina de estados de la reserva
CREATE TYPE estado_reserva AS ENUM (
    'pendiente', 'confirmada', 'en_uso', 'completada', 'cancelada'
);

-- Reservas
CREATE TABLE reservas (
    reserva_id      BIGSERIAL PRIMARY KEY,
    party_id        BIGINT NOT NULL REFERENCES parties(party_id),
    sala_id         BIGINT NOT NULL REFERENCES salas(sala_id),
    inicio          TIMESTAMPTZ NOT NULL,
    fin             TIMESTAMPTZ NOT NULL,
    precio_total    NUMERIC(10,2) NOT NULL CHECK (precio_total >= 0),
    estado          estado_reserva NOT NULL DEFAULT 'pendiente',
    creada_en       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (fin > inicio)
);

-- Índice para evitar solapamientos (el constraint más importante del negocio)
-- Un EXCLUDE constraint en PostgreSQL es la herramienta correcta:
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE reservas ADD CONSTRAINT no_solapamiento
    EXCLUDE USING GIST (
        sala_id WITH =,
        TSTZRANGE(inicio, fin, '[)') WITH &&
    )
    WHERE (estado NOT IN ('cancelada'));

-- Historial de cambios de estado (auditoría)
CREATE TABLE historial_estados_reserva (
    id              BIGSERIAL PRIMARY KEY,
    reserva_id      BIGINT NOT NULL REFERENCES reservas(reserva_id),
    estado_anterior estado_reserva,
    estado_nuevo    estado_reserva NOT NULL,
    cambiado_por    BIGINT REFERENCES parties(party_id), -- usuario del sistema
    motivo          TEXT,
    ocurrido_en     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Pagos (parciales posibles)
CREATE TABLE pagos (
    pago_id     BIGSERIAL PRIMARY KEY,
    reserva_id  BIGINT NOT NULL REFERENCES reservas(reserva_id),
    monto       NUMERIC(10,2) NOT NULL CHECK (monto > 0),
    metodo      TEXT NOT NULL,
    referencia  TEXT,
    pagado_en   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

**Capa física (índices críticos):**

```sql
-- Búsqueda de reservas por sala y rango de fechas (la query más frecuente)
CREATE INDEX idx_reservas_sala_tiempo 
    ON reservas USING GIST (sala_id, TSTZRANGE(inicio, fin, '[)'));

-- Búsqueda de reservas por cliente
CREATE INDEX idx_reservas_party ON reservas(party_id, estado, inicio DESC);

-- Índice parcial para reservas activas (las que más se consultan)
CREATE INDEX idx_reservas_activas 
    ON reservas(sala_id, inicio)
    WHERE estado IN ('pendiente', 'confirmada', 'en_uso');

-- Búsqueda de tarifa vigente para una sala y fecha
CREATE INDEX idx_tarifas_sala_vigencia 
    ON tarifas(sala_id, vigente_desde, vigente_hasta);
```

Este esquema, resultado de un proceso iterativo de preguntas y refinamiento, es radicalmente más completo y correcto que el que se produce dibujando tablas directamente desde el requerimiento original.

---

## 4. Evaluar un modelo ajeno: la auditoría de esquemas

### 4.1 Por qué evaluar modelos ajenos es una habilidad crítica

En el trabajo real, la mayoría de los modelos que se encuentran no fueron diseñados por uno mismo. Son herencias de proyectos anteriores, legados de desarrolladores que ya no están, o diseños apurados bajo presión. La capacidad de leer un esquema existente, entender sus intenciones, y detectar sus problemas antes de que lleguen a producción es una habilidad diferenciadora.

### 4.2 El proceso de auditoría: lista de verificación

**1. Verificar integridad referencial**

```sql
-- ¿Hay FKs no declaradas? (columnas que por nombre parecen FKs pero no tienen constraint)
SELECT 
    col.table_name,
    col.column_name
FROM information_schema.columns col
WHERE col.column_name LIKE '%_id'
  AND NOT EXISTS (
      SELECT 1
      FROM information_schema.key_column_usage kcu
      JOIN information_schema.referential_constraints rc 
          ON rc.constraint_name = kcu.constraint_name
      WHERE kcu.table_name = col.table_name
        AND kcu.column_name = col.column_name
  )
  AND col.table_schema = 'public';
```

Las columnas con nombre `xxx_id` sin FK declarada son una señal de alerta: o bien son FKs que se olvidó declarar (riesgo de datos huérfanos), o bien son referencias a entidades en otro sistema que el desarrollador no quiso atar con constraint (decisión válida pero que debe ser explícita).

**2. Detectar columnas nullable sin motivo aparente**

```sql
-- Columnas nullable en tablas importantes
SELECT 
    table_name, 
    column_name, 
    data_type
FROM information_schema.columns
WHERE is_nullable = 'YES'
  AND table_schema = 'public'
  AND column_name NOT LIKE '%descripcion%'
  AND column_name NOT LIKE '%notas%'
  AND column_name NOT LIKE '%comentario%'
ORDER BY table_name, column_name;
```

Para cada columna nullable, la pregunta es: ¿Por qué puede ser NULL? Si la respuesta es "no sé" o "para dejar flexible", eso es una deuda de modelado. El NULL debe tener una semántica precisa (módulo 1).

**3. Detectar ausencia de constraints de dominio**

```sql
-- Tablas con columnas de "tipo" o "estado" sin CHECK constraint
SELECT 
    t.table_name,
    c.column_name,
    c.data_type
FROM information_schema.tables t
JOIN information_schema.columns c ON c.table_name = t.table_name
LEFT JOIN (
    SELECT constraint_name, table_name, column_name
    FROM information_schema.constraint_column_usage
) cc ON cc.table_name = t.table_name AND cc.column_name = c.column_name
WHERE t.table_schema = 'public'
  AND t.table_type = 'BASE TABLE'
  AND (c.column_name LIKE '%tipo%' OR c.column_name LIKE '%estado%' OR c.column_name LIKE '%status%')
  AND cc.constraint_name IS NULL;
```

Una columna llamada `estado` o `tipo` sin un CHECK constraint o un ENUM es una bomba de tiempo: en algún momento habrá un valor inválido ('ACTIV' en lugar de 'ACTIVO', 'cancelado' en lugar de 'cancelada') que nadie detectará hasta que una query falle silenciosamente.

**4. Detectar violaciones de normalización**

Señales visuales de desnormalización no intencional:
- Columnas como `nombre_categoria` y `categoria_id` en la misma tabla (la primera es derivable de la segunda mediante JOIN).
- Columnas numeradas: `telefono_1`, `telefono_2`, `telefono_3` (señal de violación de 1NF: debería ser una tabla de teléfonos).
- Prefijos repetitivos en columnas: `cliente_nombre`, `cliente_email`, `cliente_telefono` en una tabla `pedidos` (señal de que esos datos del cliente deberían estar solo en la tabla `clientes`).

**5. Detectar índices faltantes y superfluos**

```sql
-- Tablas con FKs sin índice (problema frecuente de rendimiento)
SELECT
    tc.table_name,
    kcu.column_name,
    'FK sin índice' AS problema
FROM information_schema.table_constraints tc
JOIN information_schema.key_column_usage kcu 
    ON kcu.constraint_name = tc.constraint_name
LEFT JOIN pg_indexes pi 
    ON pi.tablename = tc.table_name 
    AND pi.indexdef LIKE '%' || kcu.column_name || '%'
WHERE tc.constraint_type = 'FOREIGN KEY'
  AND pi.indexname IS NULL;
```

Una FK sin índice en la columna referenciante significa que cualquier JOIN que use esa FK requiere un Sequential Scan de toda la tabla, sin importar cuántos datos haya.

**6. Buscar antipatrones conocidos**

- **EAV no intencional:** Tablas con columnas `attribute_name`, `attribute_value` mezcladas con datos normales.
- **Columnas de auditoria incompletas:** Solo `created_at` sin `updated_at`, o sin `created_by`.
- **IDs UUID en tablas de alta escritura sin secuencia:** Los UUIDs v4 aleatorios fragmentan los índices B-tree agresivamente (módulo 9).
- **Fechas como TEXT:** Almacenar `'2024-01-15'` como VARCHAR en lugar de usar DATE/TIMESTAMPTZ.
- **Dinero como FLOAT:** FLOAT no representa exactamente los decimales; los montos monetarios deben ser NUMERIC o DECIMAL.

### 4.3 Los cinco problemas más frecuentes en modelos heredados

De la experiencia acumulada en auditorías, los problemas que aparecen con más frecuencia son:

1. **Ausencia de constraints de integridad referencial:** Las relaciones están documentadas en comentarios o en la cabeza del equipo, pero no en la base de datos. Hay datos huérfanos acumulados durante años.

2. **NULL usado como "desconocido", "no aplica" y "vacío" simultáneamente:** La misma columna nullable significa tres cosas distintas según el contexto (violación del principio de módulo 1 sobre el significado del NULL).

3. **Timestamps en UTC no garantizados:** Algunas fechas están en UTC, otras en la zona horaria del servidor donde corría la aplicación hace cinco años. Imposible reconstruir el orden temporal real.

4. **Enumeraciones como TEXT sin restricción:** `status = 'activo'` en algunos registros y `status = 'Activo'` en otros, `status = 'ACTV'` en los más antiguos. La inconsistencia es imposible de detectar sin leer cada registro.

5. **Tablas de 100+ columnas sin documentación:** Nadie sabe para qué sirven las columnas `flag_b`, `dato_extra_3`, `referencia_legacy_codigo`. No se pueden eliminar por miedo, no se pueden usar con confianza.

---

## 5. Refactoring de esquemas en sistemas vivos

### 5.1 El problema fundamental: el sistema no puede detenerse

En el desarrollo de software general, refactorizar significa cambiar la estructura interna del código sin cambiar su comportamiento externo. Se hace con la comodidad de poder ejecutar los tests, verificar que todo sigue funcionando, y desplegar la versión refactorizada.

El refactoring de esquemas de base de datos es fundamentalmente diferente porque:

1. **Los datos son estado persistente.** A diferencia del código, no puedes simplemente reemplazar el esquema viejo por el nuevo. Los datos que viven en el esquema viejo deben migrar al esquema nuevo.

2. **El sistema está en producción mientras se migra.** A menos que el sistema tenga una ventana de mantenimiento aceptable (cada vez más raro), la migración debe ejecutarse mientras el sistema sigue atendiendo requests.

3. **Múltiples versiones del código pueden estar corriendo simultáneamente.** Durante un deploy progresivo (rolling deploy, blue/green), la versión vieja del código y la versión nueva pueden estar corriendo al mismo tiempo, ambas apuntando a la misma base de datos.

4. **Las migraciones de datos pueden tardar horas o días.** Un `ALTER TABLE` que agrega una columna a una tabla de 500 millones de filas puede bloquear la tabla durante horas en algunos motores, haciendo el sistema inaccesible.

### 5.2 Las operaciones peligrosas vs. las seguras

Antes de planificar cualquier migración, es fundamental saber qué operaciones son seguras (no bloquean el sistema) y cuáles son peligrosas.

**Operaciones generalmente seguras (no bloquean escrituras):**

```sql
-- Agregar una columna nullable (PostgreSQL: O(1) en versiones modernas)
ALTER TABLE orders ADD COLUMN notes TEXT;

-- Crear un índice en PostgreSQL con CONCURRENTLY (no bloquea)
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders(created_at);

-- Agregar un constraint NOT VALID (no valida datos existentes)
ALTER TABLE orders ADD CONSTRAINT fk_orders_customer
    FOREIGN KEY (customer_id) REFERENCES customers(id)
    NOT VALID;

-- Renombrar una columna (PostgreSQL: solo actualiza metadatos)
ALTER TABLE orders RENAME COLUMN notes TO internal_notes;
```

**Operaciones potencialmente peligrosas (pueden bloquear o ser lentas):**

```sql
-- Cambiar el tipo de una columna: requiere reescribir toda la tabla
ALTER TABLE orders ALTER COLUMN total TYPE NUMERIC(12,2);
-- ↑ En PostgreSQL, puede ser instantáneo si es compatible (e.g., int → bigint)
--   o requiere reescritura completa (e.g., varchar → int)

-- Agregar una columna NOT NULL sin DEFAULT: bloquea en tablas grandes
ALTER TABLE orders ADD COLUMN source TEXT NOT NULL;
-- ↑ Fallará o bloqueará si la tabla ya tiene filas (no tiene valor para las filas existentes)

-- Crear un índice SIN CONCURRENTLY: bloquea escrituras durante toda la creación
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Validar un constraint NOT VALID: escanea toda la tabla, pero no bloquea escrituras
ALTER TABLE orders VALIDATE CONSTRAINT fk_orders_customer;

-- DROP de una columna: genera un rewrite completo de la tabla en algunos motores
ALTER TABLE orders DROP COLUMN deprecated_field;
-- En PostgreSQL moderno, en realidad solo marca la columna como invisible;
-- la eliminación física ocurre en el próximo VACUUM FULL o pg_repack
```

---

## 6. El patrón Expand/Contract

### 6.1 La idea central

El patrón **Expand/Contract** (también llamado **Parallel Change** en la literatura de Martin Fowler) es la técnica estándar para realizar cambios de esquema sin downtime. La idea es dividir cualquier cambio disruptivo en tres fases:

```
FASE 1: EXPAND (Expansión)
  Agregar lo nuevo al esquema sin eliminar lo viejo.
  El esquema soporta simultáneamente el comportamiento viejo y el nuevo.
  El código viejo sigue funcionando sin cambios.

FASE 2: MIGRATE (Migración)
  El nuevo código empieza a usar el nuevo esquema.
  Los datos existentes se migran progresivamente al nuevo formato.
  Durante esta fase, tanto el código viejo como el nuevo están activos.

FASE 3: CONTRACT (Contracción)
  Una vez que todo el código usa el nuevo esquema, eliminar lo viejo.
  El esquema se "contrae" de vuelta a su forma mínima necesaria.
```

### 6.2 Ejemplo completo: renombrar una columna

Renombrar una columna parece una operación simple, pero en un sistema en producción con el código corriendo mientras se migra, hacerlo directamente rompe el código viejo.

```
PROBLEMA:
  Tabla: orders
  Columna actual: total_amount (usada en todo el código actual)
  Columna deseada: total (nombre más limpio)
```

**FASE 1: EXPAND — Agregar la columna nueva sin eliminar la vieja**

```sql
-- Agregar la nueva columna (nullable inicialmente)
ALTER TABLE orders ADD COLUMN total NUMERIC(12,2);

-- Trigger para mantener las dos columnas sincronizadas durante la transición
CREATE OR REPLACE FUNCTION sync_order_total()
RETURNS TRIGGER AS $$
BEGIN
    -- Cuando se escribe en total_amount, también se escribe en total
    IF NEW.total_amount IS DISTINCT FROM OLD.total_amount THEN
        NEW.total := NEW.total_amount;
    END IF;
    -- Cuando se escribe en total, también se escribe en total_amount
    IF NEW.total IS DISTINCT FROM OLD.total THEN
        NEW.total_amount := NEW.total;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_sync_order_total
BEFORE INSERT OR UPDATE ON orders
FOR EACH ROW EXECUTE FUNCTION sync_order_total();
```

En este punto: el sistema antiguo sigue escribiendo en `total_amount` (el trigger sincroniza a `total`). Se puede deployar sin romper nada.

**Backfill: poblar la nueva columna con datos existentes**

```sql
-- Poblar la nueva columna para los registros existentes
-- (se hace en lotes para no bloquear la tabla: ver sección 7)
UPDATE orders
SET total = total_amount
WHERE total IS NULL
  AND id BETWEEN 1 AND 100000;
-- Repetir para los siguientes 100,000 registros, etc.
```

**FASE 2: MIGRATE — El código nuevo usa la nueva columna**

```python
# Versión nueva del código: usa 'total' en lugar de 'total_amount'
# Se deployea gradualmente mientras el trigger mantiene las dos columnas en sync
SELECT id, total FROM orders WHERE status = 'pending';
```

**FASE 3: CONTRACT — Eliminar la columna vieja**

Solo cuando todo el código usa `total` y no hay versiones del código viejo corriendo:

```sql
-- Eliminar el trigger (ya no necesario)
DROP TRIGGER trg_sync_order_total ON orders;
DROP FUNCTION sync_order_total();

-- Eliminar la columna vieja
ALTER TABLE orders DROP COLUMN total_amount;

-- Opcionalmente: hacer la nueva columna NOT NULL ahora que todos los registros tienen valor
ALTER TABLE orders ALTER COLUMN total SET NOT NULL;
```

### 6.3 Otros casos de uso del patrón Expand/Contract

**Caso: Dividir una columna en dos**

```
Problema: address TEXT → street TEXT + city TEXT + country TEXT
```

```sql
-- FASE 1: Agregar las nuevas columnas
ALTER TABLE customers ADD COLUMN street TEXT;
ALTER TABLE customers ADD COLUMN city TEXT;
ALTER TABLE customers ADD COLUMN country TEXT;

-- Trigger que sincroniza desde address (viejo) hacia los nuevos campos
-- al escribir en address

-- BACKFILL: parsear y poblar las nuevas columnas desde los registros existentes
-- (proceso complejo porque hay que parsear texto libre)

-- FASE 2: El código nuevo escribe en las columnas separadas
-- FASE 3: DROP COLUMN address
```

**Caso: Cambiar el tipo de una columna (e.g., INT → BIGINT)**

```sql
-- FASE 1: Agregar nueva columna con tipo nuevo
ALTER TABLE events ADD COLUMN event_id_new BIGINT;

-- Trigger para sincronizar
-- BACKFILL: copiar valores existentes

-- FASE 2: Código usa event_id_new
-- FASE 3: Renombrar y eliminar la vieja
ALTER TABLE events RENAME COLUMN event_id TO event_id_old;
ALTER TABLE events RENAME COLUMN event_id_new TO event_id;
ALTER TABLE events DROP COLUMN event_id_old;
```

**Caso: Cambiar la estructura de una relación (e.g., one-to-one → one-to-many)**

```
Problema: users tiene una sola dirección (columnas address_street, address_city)
         pero ahora puede tener múltiples direcciones
```

```sql
-- FASE 1: Crear la nueva tabla de addresses
CREATE TABLE user_addresses (
    address_id  BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id),
    street      TEXT NOT NULL,
    city        TEXT NOT NULL,
    is_default  BOOLEAN NOT NULL DEFAULT FALSE
);

-- BACKFILL: copiar las direcciones existentes de users a user_addresses
INSERT INTO user_addresses (user_id, street, city, is_default)
SELECT id, address_street, address_city, TRUE
FROM users
WHERE address_street IS NOT NULL;

-- FASE 2: El código nuevo usa user_addresses
-- FASE 3: DROP de las columnas address_* de users
ALTER TABLE users DROP COLUMN address_street;
ALTER TABLE users DROP COLUMN address_city;
```

---

## 7. Backfill progresivo

### 7.1 El problema del UPDATE masivo

Un backfill es la operación de poblar datos nuevos en un esquema extendido a partir de datos existentes. El error más frecuente es hacer el backfill con un único UPDATE masivo:

```sql
-- ERROR: Bloquea la tabla durante minutos o horas
UPDATE orders
SET new_column = compute_new_value(old_column)
WHERE new_column IS NULL;
-- En una tabla de 50 millones de filas, esto puede tardar 30 minutos
-- y mantener un lock que bloquea todas las escrituras.
```

La solución es el **backfill progresivo**: procesar los datos en lotes pequeños, liberando el lock entre cada lote.

### 7.2 El patrón de backfill en lotes

```sql
-- Función de backfill por lotes (PostgreSQL)
CREATE OR REPLACE FUNCTION backfill_orders_total(batch_size INT DEFAULT 10000)
RETURNS TABLE(procesados BIGINT, pendientes BIGINT) AS $$
DECLARE
    v_min_id BIGINT;
    v_max_id BIGINT;
    v_current_id BIGINT;
    v_procesados BIGINT := 0;
BEGIN
    -- Encontrar el rango de IDs a procesar
    SELECT MIN(id), MAX(id)
    INTO v_min_id, v_max_id
    FROM orders
    WHERE total IS NULL;  -- columna nueva aún no poblada
    
    v_current_id := v_min_id;
    
    WHILE v_current_id <= v_max_id LOOP
        -- Actualizar un lote
        UPDATE orders
        SET total = total_amount
        WHERE id BETWEEN v_current_id AND v_current_id + batch_size - 1
          AND total IS NULL;
        
        v_procesados := v_procesados + ROW_COUNT;
        v_current_id := v_current_id + batch_size;
        
        -- Pausa entre lotes para no sobrecargar el sistema
        PERFORM pg_sleep(0.1);  -- 100ms entre lotes
        
        COMMIT;  -- Liberar el lock entre lotes
    END LOOP;
    
    RETURN QUERY
    SELECT v_procesados,
           (SELECT COUNT(*) FROM orders WHERE total IS NULL);
END;
$$ LANGUAGE plpgsql;
```

En Python, el patrón es similar:

```python
def backfill_orders_total(session, batch_size=10_000):
    """
    Backfill progresivo: poblamos orders.total desde orders.total_amount
    en lotes de batch_size registros para no bloquear la tabla.
    """
    import time
    
    total_processed = 0
    last_id = 0
    
    while True:
        # Obtener el siguiente lote
        with session.begin():
            rows = session.execute(
                text("""
                    SELECT id, total_amount
                    FROM orders
                    WHERE total IS NULL
                      AND id > :last_id
                    ORDER BY id
                    LIMIT :batch_size
                """),
                {'last_id': last_id, 'batch_size': batch_size}
            ).fetchall()
        
        if not rows:
            break  # No hay más registros a procesar
        
        # Actualizar el lote
        ids_amounts = [(row.id, row.total_amount) for row in rows]
        
        with session.begin():
            session.execute(
                text("""
                    UPDATE orders
                    SET total = v.total_amount
                    FROM (VALUES :values) AS v(id, total_amount)
                    WHERE orders.id = v.id
                      AND orders.total IS NULL
                """),
                {'values': ids_amounts}
            )
        
        last_id = rows[-1].id
        total_processed += len(rows)
        
        print(f"Procesados: {total_processed}, último ID: {last_id}")
        time.sleep(0.1)  # Pausa entre lotes
    
    print(f"Backfill completado. Total procesados: {total_processed}")
```

### 7.3 Idempotencia del backfill

El backfill debe ser **idempotente**: ejecutarlo múltiples veces sobre los mismos registros no debe tener efectos diferentes. Si el proceso se interrumpe a la mitad y se reinicia, no debe corromper datos ni procesar dos veces incorrectamente.

La clave es la condición `WHERE total IS NULL`: solo procesa registros que aún no fueron migrados. Si el proceso se interrumpe, al reiniciarlo simplemente continúa desde donde quedó (o reprocesa algunos que ya estaban migrados, pero sin efecto porque el WHERE los excluirá).

---

## 8. Migraciones sin downtime: blue/green y técnicas avanzadas

### 8.1 Blue/Green deployments con bases de datos

El patrón **blue/green deployment** consiste en tener dos entornos de producción idénticos (llamados "blue" y "green"). El tráfico está en uno (blue). Se despliega la nueva versión en el otro (green). Cuando green está listo y verificado, el tráfico se redirige de blue a green.

Con bases de datos, el blue/green es más complejo porque los datos cambian mientras tanto. Las estrategias son:

**Estrategia 1: Base de datos compartida con Expand/Contract**

Ambos entornos (blue y green) apuntan a la misma base de datos. El esquema es compatible con ambas versiones del código simultáneamente (gracias al Expand/Contract). La migración de datos ocurre progresivamente.

```
Blue (código v1) ──┬──▶ Base de datos (esquema compatible con v1 y v2)
                   │
Green (código v2) ─┘

Secuencia:
1. Migrar esquema (EXPAND): agregar nueva columna, trigger de sync
2. Verificar que código v1 funciona con esquema extendido
3. Deploy código v2 en green
4. Enrutar tráfico gradualmente de blue a green (canary release)
5. Cuando 100% del tráfico está en green, parar blue
6. Migrar esquema (CONTRACT): eliminar columna vieja
```

**Estrategia 2: Bases de datos separadas con replicación**

Cada entorno tiene su propia base de datos. La base de datos green es una réplica de blue con el nuevo esquema aplicado.

```
Blue ──▶ DB-Blue (esquema viejo)
          │
          │ replicación + transformación
          ▼
Green ──▶ DB-Green (esquema nuevo)
```

Cuando se redirige el tráfico a green, se corta la replicación y DB-Green se convierte en la fuente de verdad. DB-Blue puede quedarse como backup.

Este enfoque es más limpio pero requiere herramientas de replicación y transformación en tiempo real (Debezium + Kafka, o AWS DMS, o replicación lógica de PostgreSQL).

### 8.2 Herramientas de gestión de migraciones

El proceso de Expand/Contract requiere que las migraciones de esquema estén versionadas y sean reproducibles. Las herramientas estándar son:

**Flyway (Java/multi-lenguaje):**

```
migrations/
  V1__create_orders_table.sql
  V2__add_customer_index.sql
  V3__expand_add_total_column.sql      ← EXPAND
  V4__backfill_total_from_amount.sql   ← MIGRATE (o proceso separado)
  V5__contract_drop_total_amount.sql   ← CONTRACT
```

**Alembic (Python/SQLAlchemy):**

```python
# migrations/versions/abc123_expand_add_total.py
def upgrade():
    op.add_column('orders', sa.Column('total', sa.Numeric(12, 2)))
    # El backfill se hace como proceso separado, no como migración DDL

def downgrade():
    op.drop_column('orders', 'total')
```

**Principios de las migraciones como código:**

- Cada migración es un archivo versionado con timestamp o número de versión.
- Las migraciones son idempotentes o incluyen checks de idempotencia.
- Las migraciones tienen tanto `upgrade()` como `downgrade()` (aunque el downgrade no siempre es posible para cambios destructivos).
- Las migraciones se ejecutan automáticamente en el deploy, antes de que el nuevo código empiece a recibir tráfico.
- Las migraciones de datos pesadas (backfills grandes) se ejecutan como procesos separados, no como parte del deploy.

### 8.3 El error de incluir backfills grandes en el deploy

Un error frecuente es incluir un backfill masivo dentro de la migración de esquema:

```python
# PELIGROSO: migración con backfill masivo
def upgrade():
    op.add_column('orders', sa.Column('total', sa.Numeric(12, 2)))
    
    # ESTO PUEDE TARDAR HORAS y bloquear el deploy:
    op.execute("""
        UPDATE orders SET total = total_amount WHERE total IS NULL
    """)
    
    op.alter_column('orders', 'total', nullable=False)
```

Si el backfill tarda 2 horas, el deploy tarda 2 horas. Durante esas 2 horas, el sistema puede estar en un estado inconsistente. Si algo falla, el rollback es complicado.

La práctica correcta es separar el DDL del backfill:

1. **Migración DDL** (parte del deploy, rápida): solo el `ADD COLUMN`.
2. **Proceso de backfill** (ejecutado aparte, fuera del deploy): el UPDATE masivo en lotes.
3. **Migración DDL** (parte de un deploy posterior, cuando el backfill terminó): el `NOT NULL` y la limpieza.

---

## 9. La tensión entre el modelo correcto y el modelo entregable

### 9.1 El modelo perfecto no existe en producción

Todo diseñador de datos enfrenta la misma tensión: el modelo correcto desde el punto de vista técnico y el modelo que se puede entregar dado el tiempo, los recursos, y las restricciones existentes.

El modelo perfecto tiene todas las constraints, la normalización completa, los índices correctos, la gestión temporal apropiada, y la documentación exhaustiva. El modelo entregable tiene lo que fue posible hacer en el tiempo disponible.

Esta tensión es real y no tiene una respuesta única. Pero hay principios que ayudan a navegar las decisiones:

### 9.2 El principio de la deuda técnica consciente

La deuda técnica en el modelo de datos (columnas sin constraints, desnormalización no intencional, falta de indices) no es inherentemente mala si es **consciente y gestionada**.

Una decisión de deuda técnica consciente suena así:

> *"Vamos a agregar esta columna como nullable porque no tenemos tiempo de hacer el backfill ahora mismo. Creamos el ticket [#1234] para hacerlo la próxima semana. Cuando lo hagamos, agregaremos el NOT NULL constraint."*

Una decisión de deuda técnica inconsciente suena así:

> *"Agregamos la columna como nullable porque era más fácil."* — y nunca se vuelve a mencionar.

La diferencia entre las dos no es el resultado inmediato (el esquema es el mismo), sino la probabilidad de que el problema se corrija eventualmente.

### 9.3 Lo que nunca debe omitirse por rapidez

Hay cosas que es tentador omitir cuando hay presión de tiempo, pero que generan costos desproporcionados después:

**Las FKs nunca deben omitirse.** El costo de agregar una FK después, cuando ya hay datos huérfanos, puede ser enorme: hay que limpiar los datos inválidos antes de poder declarar el constraint.

**El tipo de datos correcto para fechas nunca debe omitirse.** Almacenar timestamps como TEXT, o usar TIMESTAMP sin zona horaria en un sistema que opera en múltiples zonas, genera bugs que son imposibles de corregir retroactivamente sobre datos existentes.

**Los índices de las FKs nunca deben omitirse.** Un JOIN sobre una FK sin índice en la tabla hijo es un Full Sequential Scan. En una tabla de millones de filas, esto destruye el rendimiento. El índice tarda segundos en crear y evita semanas de diagnóstico.

**El discriminador de tipo nunca debe omitirse.** Si hay una columna que indica el "tipo" o "estado" de una entidad, debe ser o un ENUM o tener un CHECK constraint. Un campo de texto libre sin restricción genera inconsistencias imposibles de limpiar.

### 9.4 El modelo mínimo correcto

La pregunta correcta no es "¿qué es lo máximo que puedo diseñar?" sino "¿cuál es el mínimo que es correcto?". El mínimo correcto incluye:

- Claves primarias en todas las tablas.
- FKs declaradas para todas las relaciones de referencia.
- NOT NULL en columnas que no pueden ser nulas semánticamente.
- CHECK constraints o ENUMs para columnas con valores restringidos.
- Índices en las columnas de FK.
- Timestamps en UTC (`TIMESTAMPTZ` en PostgreSQL, no `TIMESTAMP`).
- Nombres de columnas y tablas consistentes y en el idioma elegido para el proyecto.

Todo lo demás —normalización completa, índices de cobertura, particionamiento, gestión temporal— puede agregarse después sin la urgencia de las FKs y los tipos correctos.

---

## 10. Documentar el modelo: la memoria del sistema

### 10.1 Por qué la documentación del modelo es diferente a la documentación del código

El código documenta su propia lógica (o debería). El esquema de base de datos documenta mucho menos de lo que parece. Un `CREATE TABLE` dice qué columnas existen y qué tipos tienen, pero no dice:

- ¿Por qué existe esta columna? ¿Para qué se usa?
- ¿Qué significa NULL en este contexto?
- ¿Cuál es la cardinalidad esperada? ¿Esta tabla tiene 100 filas o 100 millones?
- ¿Qué regla de negocio produce los valores de esta columna?
- ¿Esta tabla es de solo lectura para las aplicaciones y se pobla solo por procesos batch?
- ¿Este campo está deprecado? ¿Cuándo se puede eliminar?

### 10.2 Los tres niveles de documentación del modelo

**Nivel 1: Comentarios en el DDL**

El nivel más básico y más frecuentemente omitido. Los comentarios deben estar en el DDL de producción, no solo en un documento separado que nadie actualiza.

```sql
COMMENT ON TABLE reservas IS 
    'Reservas de espacios de coworking. Una reserva representa el derecho de un cliente 
     a usar una sala durante un período específico. Estado sigue la máquina de estados:
     pendiente → confirmada → en_uso → completada | cancelada.';

COMMENT ON COLUMN reservas.precio_total IS
    'Precio fijado al momento de confirmar la reserva (no al facturar). 
     Incluye todos los cargos aplicables según la tarifa vigente en el momento de la confirmación.
     No cambia aunque la tarifa de la sala cambie después.';

COMMENT ON COLUMN reservas.estado IS
    'Estado actual de la reserva. Ver tabla historial_estados_reserva para 
     el historial completo de transiciones.';
```

```sql
-- PostgreSQL: los comentarios son consultables
SELECT 
    c.table_name,
    c.column_name,
    pgd.description
FROM information_schema.columns c
JOIN pg_class pgc ON pgc.relname = c.table_name
JOIN pg_attribute pga ON pga.attrelid = pgc.oid AND pga.attname = c.column_name
LEFT JOIN pg_description pgd ON pgd.objoid = pgc.oid AND pgd.objsubid = pga.attnum
WHERE c.table_schema = 'public'
ORDER BY c.table_name, c.ordinal_position;
```

**Nivel 2: Diagrama ER actualizado**

Un diagrama ER que refleja el esquema actual, generado automáticamente desde el DDL. Herramientas como DBeaver, pgAdmin, SchemaSpy, o dbdiagram.io pueden generar estos diagramas automáticamente.

El error más común con los diagramas ER es que se crean en el momento del diseño inicial y nunca se actualizan. Un diagrama ER desactualizado es peor que no tener diagrama: da una falsa sensación de conocimiento.

La práctica correcta es generar el diagrama automáticamente en el CI/CD a partir del DDL de producción, y publicarlo en la wiki del equipo. Así siempre refleja el estado real.

**Nivel 3: El data dictionary**

Un data dictionary es un documento (o sistema) que describe cada tabla y columna con información que no puede inferirse del DDL:

| Tabla | Columna | Tipo | Descripción | Rango esperado | Propietario | Fuente |
|---|---|---|---|---|---|---|
| `reservas` | `precio_total` | NUMERIC | Precio fijado al confirmar | 0-10,000 USD | Equipo de producto | Calculado desde tarifas |
| `reservas` | `estado` | ENUM | Estado actual en la máquina de estados | pendiente/confirmada/en_uso/completada/cancelada | Equipo de reservas | Transiciones de negocio |

El data dictionary es especialmente valioso para:
- **Nuevos miembros del equipo:** Pueden entender el modelo sin necesidad de preguntar por cada campo.
- **Equipos de análisis de datos:** Necesitan saber qué significa cada campo antes de construir reportes.
- **Auditorías:** Regulaciones como GDPR requieren saber exactamente qué datos personales existen y dónde.

---

## 11. El modelo como contrato entre equipos

### 11.1 La base de datos como API interna

En organizaciones con múltiples equipos, la base de datos es frecuentemente compartida entre equipos que la consumen de distintas formas: el equipo de backend escribe a través de la aplicación, el equipo de datos lee para análisis, el equipo de integraciones sincroniza con sistemas externos.

En este contexto, el esquema de la base de datos es un **contrato implícito** entre todos estos consumidores. Cuando el equipo de backend renombra una columna, el equipo de datos tiene reportes que se rompen. Cuando el equipo de análisis agrega una vista materializada pesada, impacta el rendimiento del backend.

### 11.2 Principios del modelo como contrato

**Versionado de la API de datos:**

Las columnas y tablas que son consumidas por sistemas externos (pipelines de datos, integraciones, ORMs de otros equipos) no deben cambiar sin notificación y período de migración.

```sql
-- Ejemplo: renombrar una columna usada por el equipo de datos
-- En lugar de renombrar directamente, crear una VIEW de compatibilidad:

-- Paso 1: Renombrar la columna internamente
ALTER TABLE orders RENAME COLUMN total_amount TO total;

-- Paso 2: Crear una view de compatibilidad para los consumidores externos
CREATE VIEW orders_v1 AS
SELECT *, total AS total_amount  -- expone el nombre viejo también
FROM orders;

-- Paso 3: Notificar a los consumidores que tienen 3 sprints para migrar
-- Paso 4: Cuando todos han migrado, eliminar la view de compatibilidad
```

**Separación de esquemas por responsabilidad:**

En PostgreSQL, los **schemas** (namespaces dentro de la base de datos) permiten separar las tablas por responsabilidad:

```sql
-- Schema para datos operacionales (del backend)
CREATE SCHEMA app;
CREATE TABLE app.orders (...);
CREATE TABLE app.customers (...);

-- Schema para datos analíticos (expuesto al equipo de datos)
CREATE SCHEMA reporting;
CREATE VIEW reporting.orders_summary AS
    SELECT DATE_TRUNC('day', created_at) AS date,
           COUNT(*) AS orders,
           SUM(total) AS revenue
    FROM app.orders
    WHERE status = 'completed'
    GROUP BY 1;

-- El equipo de datos tiene permiso en 'reporting', no en 'app'
GRANT SELECT ON ALL TABLES IN SCHEMA reporting TO analyst_role;
```

---

## 12. Seguridad a nivel de datos

### 12.1 Row-Level Security (RLS)

El **Row-Level Security** es una feature de PostgreSQL (y otros DBMS modernos) que permite que las restricciones de acceso a datos se definan a nivel de fila dentro del motor de base de datos, en lugar de depender exclusivamente del código de la aplicación.

Sin RLS, la seguridad de acceso a datos depende de que el código de la aplicación siempre agregue los filtros correctos en cada query. Un bug en el código puede filtrar datos de un usuario a otro. Con RLS, el motor garantiza que las restricciones se cumplan independientemente de lo que haga el código.

```sql
-- Ejemplo: sistema multi-tenant donde cada empresa solo puede ver sus datos

-- Habilitar RLS en la tabla
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Policy: solo ver las órdenes del tenant al que pertenece el usuario
CREATE POLICY orders_tenant_isolation ON orders
    FOR ALL
    TO application_role
    USING (
        tenant_id = current_setting('app.current_tenant_id')::BIGINT
    );

-- La aplicación establece el tenant_id al inicio de cada sesión:
SET app.current_tenant_id = '42';

-- Ahora cualquier SELECT sobre orders solo devuelve filas con tenant_id = 42,
-- incluso si la query no tiene WHERE tenant_id = 42 explícito.
SELECT * FROM orders;  
-- Solo devuelve órdenes del tenant 42, automáticamente.
```

Esto es especialmente valioso en sistemas SaaS multi-tenant: la separación de datos entre tenants está garantizada por el motor, no por la diligencia del código de la aplicación.

```sql
-- Políticas más granulares: distintos accesos para distintos roles
-- Los managers pueden ver todas las órdenes de su tenant
CREATE POLICY orders_manager_access ON orders
    FOR SELECT
    TO manager_role
    USING (
        tenant_id = current_setting('app.current_tenant_id')::BIGINT
    );

-- Los vendedores solo pueden ver las órdenes que ellos crearon
CREATE POLICY orders_seller_access ON orders
    FOR SELECT
    TO seller_role
    USING (
        tenant_id = current_setting('app.current_tenant_id')::BIGINT
        AND created_by = current_setting('app.current_user_id')::BIGINT
    );

-- Para INSERT: un vendedor solo puede crear órdenes a su nombre
CREATE POLICY orders_seller_insert ON orders
    FOR INSERT
    TO seller_role
    WITH CHECK (
        tenant_id = current_setting('app.current_tenant_id')::BIGINT
        AND created_by = current_setting('app.current_user_id')::BIGINT
    );
```

### 12.2 Column-Level Security y enmascaramiento

Para datos sensibles (números de tarjeta, contraseñas, datos médicos), la protección no es solo sobre qué filas se pueden ver, sino sobre qué columnas son visibles.

**Column-Level Security con permisos:**

```sql
-- Dar acceso a una tabla pero solo a ciertas columnas
GRANT SELECT (id, name, email) ON customers TO support_role;
-- El rol de soporte puede ver id, name, email pero NO credit_card_number, SSN, etc.

-- El intento de ver columnas no autorizadas falla:
-- SELECT credit_card_number FROM customers;  -- ERROR: permission denied for column
```

**Enmascaramiento dinámico con vistas:**

```sql
-- Vista que enmascara datos sensibles para el rol de soporte
CREATE VIEW customers_masked AS
SELECT
    id,
    name,
    LEFT(email, 3) || '***@***' AS email,           -- email parcialmente enmascarado
    '****-****-****-' || RIGHT(card_number, 4) AS card_number,  -- solo últimos 4
    CASE 
        WHEN current_setting('app.role') = 'compliance'
        THEN ssn  -- el equipo de compliance ve el SSN completo
        ELSE '***-**-' || RIGHT(ssn, 4)  -- todos los demás ven enmascarado
    END AS ssn
FROM customers;

-- El equipo de soporte usa esta vista, no la tabla directa
GRANT SELECT ON customers_masked TO support_role;
REVOKE SELECT ON customers FROM support_role;
```

### 12.3 Tablas de auditoría

La auditoría a nivel de datos es el registro de quién hizo qué cambio, cuándo, y qué valor había antes. Es un requisito legal en muchos sectores (finanzas, salud, gobierno) y una práctica prudente en cualquier sistema.

```sql
-- Tabla de auditoría genérica
CREATE TABLE audit_log (
    log_id          BIGSERIAL PRIMARY KEY,
    tabla           TEXT NOT NULL,
    operacion       TEXT NOT NULL CHECK (operacion IN ('INSERT', 'UPDATE', 'DELETE')),
    registro_id     TEXT NOT NULL,       -- ID del registro afectado (como TEXT para generalidad)
    datos_antes     JSONB,               -- estado antes del cambio (NULL para INSERT)
    datos_despues   JSONB,               -- estado después del cambio (NULL para DELETE)
    usuario_app     TEXT,                -- usuario de la aplicación (no de la DB)
    ip_origen       INET,
    ocurrido_en     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Función trigger genérica de auditoría
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (tabla, operacion, registro_id, datos_despues, usuario_app)
        VALUES (TG_TABLE_NAME, 'INSERT', NEW.id::TEXT, row_to_json(NEW)::JSONB,
                current_setting('app.current_user', TRUE));
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (tabla, operacion, registro_id, datos_antes, datos_despues, usuario_app)
        VALUES (TG_TABLE_NAME, 'UPDATE', OLD.id::TEXT, 
                row_to_json(OLD)::JSONB, row_to_json(NEW)::JSONB,
                current_setting('app.current_user', TRUE));
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (tabla, operacion, registro_id, datos_antes, usuario_app)
        VALUES (TG_TABLE_NAME, 'DELETE', OLD.id::TEXT, row_to_json(OLD)::JSONB,
                current_setting('app.current_user', TRUE));
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Aplicar el trigger a las tablas críticas
CREATE TRIGGER audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();

CREATE TRIGGER audit_payments
    AFTER INSERT OR UPDATE OR DELETE ON payments
    FOR EACH ROW EXECUTE FUNCTION audit_trigger_function();
```

### 12.4 Temporal Tables y auditoría nativa

PostgreSQL desde la versión 13 tiene soporte mejorado para tablas temporales de sistema, pero la alternativa clásica es el patrón de tabla de historial que cubrimos en el módulo 6:

```sql
-- Tabla con historial completo (pattern temporal)
CREATE TABLE product_prices (
    price_id        BIGSERIAL PRIMARY KEY,
    product_id      BIGINT NOT NULL REFERENCES products(id),
    price           NUMERIC(10,2) NOT NULL,
    valid_from      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    valid_until     TIMESTAMPTZ,  -- NULL = vigente
    changed_by      BIGINT REFERENCES users(id),
    change_reason   TEXT
);

-- "¿Cuánto costaba el producto X el 15 de marzo?"
SELECT price
FROM product_prices
WHERE product_id = 42
  AND valid_from <= '2024-03-15'::TIMESTAMPTZ
  AND (valid_until IS NULL OR valid_until > '2024-03-15'::TIMESTAMPTZ);
```

---

## 13. El modelo de datos como decisión estratégica

### 13.1 Las decisiones de modelo que duran décadas

A diferencia del código, que puede refactorizarse con relativa facilidad, las decisiones de modelo de datos tienen una inercia enorme. Los datos se acumulan durante años. Cambiar la estructura de una tabla con 10 años de datos de negocio crítico es una operación de riesgo alto que nadie quiere hacer.

Algunas decisiones de diseño tomadas en los primeros días de un sistema siguen siendo visibles (y limitantes) diez años después:

- **El identificador principal:** ¿UUID v4, BIGSERIAL, UUID v7, o clave de negocio natural? Una vez que hay datos en producción y el ID es la referencia en decenas de tablas, cambiar el tipo es casi imposible.
- **La zona horaria de los timestamps:** Un sistema que empezó almacenando timestamps sin zona horaria y luego se expandió a múltiples países puede tener años de datos cuyo orden temporal real es irrecuperable.
- **La granularidad del modelo:** ¿Los ítems de una orden son una tabla separada o un JSONB en la orden? La decisión inicial determina qué queries son posibles años después.
- **El esquema multi-tenant vs. base de datos por tenant:** Una vez elegido, es extremadamente costoso cambiar.

### 13.2 La conversación que no ocurre lo suficiente

El modelo de datos raramente recibe la atención estratégica que merece en las organizaciones. Las decisiones arquitectónicas sobre microservicios, lenguajes de programación, frameworks, y cloud providers reciben debates extensos. La decisión sobre el modelo de datos suele tomarse rápidamente, bajo presión, por el desarrollador que inicia el proyecto.

El costo de esa falta de atención se paga después: en migraciones costosas, en bugs de integridad de datos, en reportes que no pueden construirse porque los datos no se capturaron correctamente, en sistemas que no pueden escalar porque el esquema no fue diseñado para ello.

La conversación que el equipo debería tener al inicio de cualquier proyecto importante incluye:

- **¿Qué preguntas de negocio necesitamos poder responder con estos datos dentro de 3 años?**
- **¿Qué datos necesitamos preservar por razones legales o de auditoría?**
- **¿Cuánto pueden crecer estos datos? ¿El modelo soporta ese crecimiento?**
- **¿Qué sistemas externos necesitarán acceder a estos datos? ¿En qué formato?**

### 13.3 El diseñador de datos como colaborador estratégico

El profesional con conocimiento profundo de diseño de datos no es solo un técnico que crea tablas. Es alguien que puede:

- **Traducir requerimientos de negocio a estructuras de datos correctas y duraderas.**
- **Identificar riesgos en modelos ajenos antes de que lleguen a producción.**
- **Planificar y ejecutar migraciones de esquema en sistemas en producción sin downtime.**
- **Diseñar para la escala futura sin sobrediseñar para el presente.**
- **Comunicar las implicaciones de las decisiones de modelo a audiencias no técnicas.**

Estas son habilidades que separan a un desarrollador que conoce SQL de un diseñador de datos con experiencia real.

---

## 14. Síntesis del curso

### 14.1 Los 16 módulos y su hilo conductor

Este curso recorrió el diseño de bases de datos desde sus fundamentos matemáticos hasta su aplicación en sistemas en producción. El hilo conductor fue siempre el mismo: **entender el porqué detrás de cada técnica**, para poder aplicarla con criterio en lugar de por convención.

```
EJE 1 — TEORÍA (Módulos 1-2)
  El modelo relacional como matemática → Las tablas son relaciones, 
  las operaciones tienen propiedades formales, el álgebra relacional 
  es la base de SQL.
  
  Las formas normales como consecuencia → No son reglas a memorizar;
  son el resultado inevitable de eliminar dependencias redundantes.

EJE 2 — DISEÑO (Módulos 3-6)
  Modelado conceptual → Capturar el dominio antes de comprometerse
  con una implementación.
  
  Integridad → Las invariantes del negocio deben estar en la base de datos,
  no en el código de la aplicación.
  
  Patrones → Los problemas de modelado se repiten; conocer las soluciones
  conocidas evita reinventar ruedas defectuosas.
  
  Diseño para el tiempo → Los datos cambian; el historial es información,
  no ruido.

EJE 3 — MOTOR (Módulos 7-9)
  Internals → El optimizador toma decisiones basadas en estadísticas;
  entender esas decisiones permite ayudarlo en lugar de obstaculizarlo.
  
  Transacciones → ACID no es magia; MVCC, aislamiento y niveles de
  concurrencia tienen consecuencias concretas en el comportamiento
  del sistema.
  
  Modelado físico → El mismo modelo lógico puede tener rendimientos
  radicalmente distintos dependiendo del modelo físico.

EJE 4 — ANALÍTICO (Módulos 10-11)
  OLTP vs OLAP → Son filosofías fundamentalmente incompatibles que
  requieren modelos distintos.
  
  Kimball → El modelo dimensional es el lenguaje del data warehouse;
  hechos, dimensiones y granularidad son sus conceptos centrales.

EJE 5 — INTEGRACIÓN (Módulos 12-16)
  Modelos alternativos → El relacional no es siempre la herramienta
  correcta; conocer las alternativas es saber cuándo el relacional
  empieza a ser el problema.
  
  Sistemas distribuidos → Cuando los datos viven en múltiples nodos,
  el modelo relacional enfrenta límites que CAP hace explícitos.
  
  Patrones de Fowler → La frontera entre el modelo de objetos y el
  modelo relacional es el origen de toda la complejidad de los ORMs.
  
  ORMs → Son herramientas poderosas cuando se entienden; son generadores
  de bugs cuando no.
  
  El oficio → Todo lo anterior aplicado al trabajo real, con sus
  plazos, sistemas heredados, y restricciones.
```

### 14.2 Las ideas que perduran

De todos los conceptos cubiertos, hay algunas ideas que trascienden la tecnología específica y perduran independientemente de qué motor, ORM, o arquitectura se use:

**1. La integridad pertenece a la base de datos, no al código de la aplicación.**
El código puede tener bugs. La base de datos con constraints correctas garantiza que ciertos estados inválidos son imposibles por construcción.

**2. El NULL tiene una semántica precisa que debe ser definida, no asumida.**
"Este campo puede ser NULL" es una declaración de diseño, no una forma de evitar pensar en los casos borde.

**3. El tiempo es la dimensión más ignorada y la más costosa de agregar después.**
Si hay preguntas históricas posibles sobre los datos, el modelo debe capturar el tiempo desde el inicio.

**4. El modelo de datos es más difícil de cambiar que el código.**
Las decisiones de esquema tienen inercia por los datos que acumulan. Invertir tiempo en diseñar bien al inicio cuesta menos que migrar después.

**5. El modelo correcto no existe independientemente de los patrones de acceso.**
Un modelo puede ser lógicamente correcto pero físicamente inadecuado para los queries del sistema. La capa física no es un detalle de implementación.

**6. La distribución no es una extensión del modelo centralizado.**
El modelo distribuido requiere repensar las suposiciones fundamentales sobre consistencia, transacciones y joins.

**7. Los ORMs implementan los patrones de Fowler; entender los patrones permite usar los ORMs.**
El N+1, el LazyInitializationException, y las transacciones implícitas no son bugs del ORM; son consecuencias de no entender los patrones que el ORM implementa.

**8. El oficio es la aplicación de la teoría dentro de restricciones reales.**
La habilidad más valiosa no es conocer todos los patrones; es saber cuál aplicar en qué contexto, y cómo hacerlo con los recursos y el tiempo disponibles.

---

## 15. Ejercicios de comprensión

**Ejercicio 1 — La conversación con el cliente.** Un cliente te describe el siguiente requerimiento:

> *"Necesitamos una plataforma para gestionar proyectos de consultoría. Los consultores trabajan en proyectos para clientes. Los proyectos tienen tareas, y los consultores registran las horas que trabajan en cada tarea. Al final del mes facturamos al cliente por las horas trabajadas."*

a) Formula exactamente 10 preguntas que harías al cliente antes de empezar a diseñar el modelo. Para cada pregunta, explica qué aspecto del modelo cambia según la respuesta.

b) Asumiendo las respuestas más complejas posibles (las que generan más entidades y relaciones), diseña el modelo conceptual completo en texto (entidades, atributos clave, relaciones con cardinalidades).

c) Traduce el modelo conceptual a un esquema relacional completo con DDL PostgreSQL, incluyendo todas las constraints que se deducen del modelo.

d) Identifica al menos tres reglas de negocio que emergen del dominio y que deberían estar capturadas como constraints en la base de datos (no en el código de la aplicación).

---

**Ejercicio 2 — Auditoría de un esquema heredado.** Te entregan el siguiente esquema de un sistema de gestión de órdenes con 5 años de historia:

```sql
CREATE TABLE clientes (
    id INT,
    nombre VARCHAR(200),
    tipo VARCHAR(50),
    email VARCHAR(100),
    telefono1 VARCHAR(20),
    telefono2 VARCHAR(20),
    telefono3 VARCHAR(20),
    dir_calle VARCHAR(200),
    dir_ciudad VARCHAR(100),
    dir_pais VARCHAR(50),
    empresa_nombre VARCHAR(200),
    empresa_ruc VARCHAR(20),
    activo INT,
    fecha_creacion TIMESTAMP,
    usr_creador VARCHAR(50)
);

CREATE TABLE ordenes (
    id INT,
    cliente_id INT,
    cliente_nombre VARCHAR(200),
    cliente_email VARCHAR(100),
    fecha TIMESTAMP,
    estado VARCHAR(50),
    total FLOAT,
    descuento FLOAT,
    total_final FLOAT,
    notas TEXT,
    pagado INT,
    fecha_pago TIMESTAMP,
    metodo_pago VARCHAR(100)
);

CREATE TABLE orden_items (
    id INT,
    orden_id INT,
    producto_nombre VARCHAR(200),
    producto_sku VARCHAR(50),
    cantidad INT,
    precio_unitario FLOAT,
    total_linea FLOAT
);
```

a) Lista todos los problemas de diseño que encuentras en este esquema. Organízalos por categoría: (1) Problemas de integridad, (2) Problemas de normalización, (3) Problemas de tipos de datos, (4) Problemas de diseño temporal, (5) Otros.

b) Diseña el esquema corregido completo con DDL PostgreSQL. Para cada cambio significativo, explica brevemente por qué lo haces.

c) Para uno de los problemas más graves que identificaste, diseña el plan de migración usando el patrón Expand/Contract. Incluye las tres fases con el SQL de cada una.

d) La tabla `orden_items` no tiene referencia a una tabla `productos`. ¿Es esto siempre un error? ¿Hay casos donde esto sea una decisión de diseño válida? Explica.

---

**Ejercicio 3 — Migración sin downtime.** Un sistema de e-commerce en producción tiene una tabla `products` con 80 millones de filas. El equipo quiere hacer los siguientes cambios:

- Cambio A: Renombrar la columna `price` (NUMERIC) a `base_price`.
- Cambio B: Agregar una columna `currency` (TEXT, NOT NULL, default 'USD') que actualmente no existe.
- Cambio C: Dividir la columna `full_name` (TEXT, e.g., 'Widget Pro 500GB') en `brand` y `model` separados.
- Cambio D: Cambiar la columna `category` (TEXT, e.g., 'electronics') por una FK a una nueva tabla `categories`.

Para cada cambio:

a) Describe el plan Expand/Contract completo con las tres fases.

b) Identifica las operaciones que son peligrosas (pueden causar downtime o bloquear escrituras) y cómo evitarlas.

c) Para el Cambio D, el backfill requiere crear las categorías en la nueva tabla y actualizar las FKs. ¿Cómo harías este backfill sin downtime? ¿Qué pasa con los valores de `category` que no tienen correspondencia exacta (variaciones de capitalización, errores tipográficos)?

d) ¿En qué orden ejecutarías los cuatro cambios y por qué?

---

**Ejercicio 4 — Seguridad a nivel de datos.** Un sistema de salud necesita implementar las siguientes restricciones de acceso:

- Los médicos solo pueden ver los registros de sus propios pacientes.
- El personal administrativo puede ver el nombre y datos de contacto de todos los pacientes pero no sus diagnósticos ni medicamentos.
- El personal de facturación puede ver los datos de facturación de todos los pacientes pero no sus diagnósticos.
- Los auditores externos pueden ver todos los datos pero no pueden modificar nada.
- Todos los accesos (lectura y escritura) deben quedar registrados en un log de auditoría.

a) Diseña el esquema de tablas principal para este sistema (pacientes, médicos, registros médicos, facturas) con la seguridad en mente.

b) Implementa las políticas de Row-Level Security en PostgreSQL para las restricciones de los médicos.

c) Diseña la estructura de la vista enmascarada para el personal administrativo.

d) Implementa el trigger de auditoría para la tabla de registros médicos. ¿Qué información debe capturarse en cada acceso?

e) ¿Por qué no es suficiente implementar estas restricciones solo en el código de la aplicación? ¿Qué escenarios quedan sin protección si no hay RLS?

---

**Ejercicio 5 — Síntesis: el sistema completo.** Diseña el modelo de datos completo para el siguiente sistema:

> *Una plataforma de aprendizaje en línea donde instructores publican cursos con múltiples módulos y lecciones. Los estudiantes se suscriben a cursos, progresan por las lecciones, y reciben certificados al completar el curso. Los instructores reciben ingresos basados en las suscripciones. La plataforma opera en múltiples monedas y zonas horarias.*

El diseño debe cubrir:

a) **Modelo conceptual:** Diagrama EER textual con todas las entidades, relaciones, y cardinalidades.

b) **Modelo lógico:** DDL completo con todas las constraints, incluyendo gestión temporal para los precios de los cursos (que pueden cambiar), el progreso de los estudiantes (con timestamps precisos), y el historial de pagos a instructores.

c) **Modelo físico:** Los índices necesarios para soportar eficientemente las siguientes queries:
   - "Dame todos los cursos activos con su número de estudiantes actuales y rating promedio."
   - "Dame el progreso del estudiante X en el curso Y: qué lecciones completó y cuándo."
   - "Dame el reporte mensual de ingresos del instructor Z."

d) **Seguridad:** ¿Qué datos son sensibles en este sistema? Diseña las políticas de acceso para al menos tres roles distintos.

e) **Evolución futura:** El equipo de producto está considerando añadir en el futuro: (1) live sessions con instructores, (2) foros de discusión por curso, (3) sistema de refunds. Para cada uno, identifica si el modelo actual soportaría el cambio sin migraciones mayores, y qué habría que agregar.

---

*Este módulo cierra el curso. El modelo de datos es, en última instancia, la representación formal del entendimiento que un equipo tiene de su dominio. Diseñarlo bien es, simultáneamente, un acto técnico y un acto de conocimiento. Los mejores diseños de datos no son los más elaborados; son los que capturan con precisión y simplicidad lo que el negocio necesita recordar.*

*El oficio es aprender a ver ambas cosas al mismo tiempo.*
