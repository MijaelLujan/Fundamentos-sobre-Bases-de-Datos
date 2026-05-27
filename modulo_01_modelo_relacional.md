# Módulo 1 — El modelo relacional como matemática

> *"La mayoría empieza con SQL. Nosotros empezamos con el modelo relacional como matemática."*

---

## Tabla de contenidos

1. [Por qué importa empezar desde la matemática](#1-por-qué-importa-empezar-desde-la-matemática)
2. [Qué es una relación en sentido formal](#2-qué-es-una-relación-en-sentido-formal)
3. [Tuplas y dominios](#3-tuplas-y-dominios)
4. [El modelo lógico vs la implementación física](#4-el-modelo-lógico-vs-la-implementación-física)
5. [Las 12 reglas de Codd](#5-las-12-reglas-de-codd)
6. [El problema del valor nulo](#6-el-problema-del-valor-nulo)
7. [Álgebra relacional completa](#7-álgebra-relacional-completa)
8. [Composición de operadores y equivalencias](#8-composición-de-operadores-y-equivalencias)
9. [Resumen y conexión con el resto del curso](#9-resumen-y-conexión-con-el-resto-del-curso)

---

## 1. Por qué importa empezar desde la matemática

Cuando alguien aprende bases de datos de forma autodidacta o en un bootcamp, generalmente el primer contacto es con SQL: `SELECT`, `FROM`, `WHERE`, `JOIN`. Eso no está mal, pero crea una ilusión peligrosa: la ilusión de que SQL *es* el modelo relacional, de que las tablas *son* relaciones, de que las filas *son* tuplas en el sentido matemático.

Esa confusión tiene consecuencias prácticas concretas:

- Se diseñan esquemas pensando en "cómo quedará la tabla" en lugar de "qué relación matemática estoy definiendo".
- Se escriben queries que funcionan pero no son correctas según el modelo (y que pueden producir resultados incorrectos en casos borde).
- Se acepta NULL como algo natural cuando en realidad es una concesión pragmática con consecuencias teóricas severas.
- Se confunden las garantías del modelo con las garantías de la implementación.

El objetivo de este módulo es construir la base teórica correcta. No porque vayamos a escribir álgebra relacional en producción, sino porque entender el modelo formal nos permite **razonar con precisión** sobre diseño, corrección e integridad.

---

## 2. Qué es una relación en sentido formal

### 2.1 La definición matemática

En matemáticas, una **relación** es un subconjunto del producto cartesiano de dos o más conjuntos.

Sea un ejemplo concreto. Supongamos que tenemos dos conjuntos:

```
A = {1, 2, 3}
B = {a, b, c}
```

El producto cartesiano de A y B es el conjunto de todos los pares ordenados posibles:

```
A × B = {(1,a), (1,b), (1,c), (2,a), (2,b), (2,c), (3,a), (3,b), (3,c)}
```

Una **relación** sobre A y B es cualquier subconjunto de ese producto cartesiano. Por ejemplo:

```
R = {(1,a), (2,b), (3,c)}
```

Eso es una relación matemática perfectamente válida.

Codd tomó esta idea y la extendió: en lugar de solo dos conjuntos, permite **n conjuntos** (de ahí "modelo relacional"), y le da una estructura nombrada a cada posición.

### 2.2 La relación de Codd

En el modelo relacional, una relación se define así:

- Tiene un **nombre**.
- Tiene un conjunto de **atributos**, cada uno también con un nombre.
- Cada atributo está asociado a un **dominio** (el conjunto de valores posibles).
- El **esquema** de la relación es el conjunto de sus atributos con sus dominios.
- La **extensión** (o instancia) de la relación es un conjunto de tuplas que satisfacen ese esquema.

Formalmente, si una relación `R` tiene atributos `A₁, A₂, ..., Aₙ` con dominios `D₁, D₂, ..., Dₙ`, entonces la extensión de `R` es un subconjunto de:

```
D₁ × D₂ × ... × Dₙ
```

### 2.3 Propiedades que se derivan de esta definición

Aquí es donde empieza a ser interesante. Porque si una relación es un **conjunto**, entonces:

**No puede haber tuplas duplicadas.** Los conjuntos no tienen elementos repetidos. Esto implica que en una relación "verdadera" no puede haber dos filas idénticas. En SQL esto es violable (las tablas son bolsas, no conjuntos — más sobre esto después).

**El orden de las tuplas no importa.** Un conjunto no tiene orden. La relación `{(1,a), (2,b)}` es exactamente igual a `{(2,b), (1,a)}`. Esto implica que preguntar "dame la primera fila" no tiene sentido en el modelo relacional. En SQL, el orden es indefinido a menos que uses `ORDER BY`.

**El orden de los atributos no importa.** Los atributos se identifican por nombre, no por posición. La relación con atributos `{nombre, edad}` es la misma que con atributos `{edad, nombre}`. Esto es importante porque implica que `SELECT *` en SQL es potencialmente peligroso si dependes del orden de columnas.

### 2.4 La diferencia entre tabla y relación

Una **tabla SQL** no es necesariamente una relación:

| Propiedad | Relación (matemática) | Tabla SQL |
|---|---|---|
| Tuplas duplicadas | Imposible | Permitidas (sin PRIMARY KEY) |
| Orden de filas | No definido | No definido (pero consultas pueden asumir orden) |
| Valores NULL | No existen | Permitidos |
| Atributos identificados por | Nombre | Nombre (y posición en `SELECT *`) |

Cuando una tabla SQL tiene clave primaria, se aproxima a una relación en el sentido formal. Pero incluso así, NULL introduce diferencias irreconciliables.

---

## 3. Tuplas y dominios

### 3.1 Qué es una tupla

Una **tupla** es un elemento de una relación. Si la relación tiene esquema `(nombre: STRING, edad: INTEGER, activo: BOOLEAN)`, una tupla válida sería:

```
(nombre: "María", edad: 34, activo: true)
```

Nótese que en el modelo formal, los componentes de la tupla están identificados por nombre de atributo, no por posición. Esto es técnicamente diferente a una n-upla matemática ordenada `(a, b, c)`.

### 3.2 Qué es un dominio

Un **dominio** es el conjunto de todos los valores atómicos posibles para un atributo. La palabra clave es **atómico**: primera forma normal (que veremos en profundidad en el módulo 2) requiere que los dominios sean atómicos, es decir, indivisibles.

Ejemplos de dominios:
- `ENTERO_POSITIVO`: todos los enteros mayores que cero.
- `NOMBRE_PERSONA`: cadenas de hasta 100 caracteres.
- `PORCENTAJE`: números decimales entre 0.0 y 100.0.
- `MONEDA_COP`: números con exactamente dos decimales, representando pesos colombianos.

Un dominio no es solo un tipo de dato. Es una **restricción semántica**. Por ejemplo, `edad_empleado` y `código_departamento` podrían ambos ser enteros positivos, pero mezclarlos en una operación de comparación no tendría sentido semántico aunque fuera sintácticamente válido.

### 3.3 Dominio vs tipo de dato

Esta distinción es sutil pero importante.

Un **tipo de dato** es una categoría técnica del lenguaje: `INT`, `VARCHAR(100)`, `BOOLEAN`, `DATE`. Es una restricción del sistema sobre qué valores puede almacenar.

Un **dominio** es una restricción semántica del modelo: no solo qué tipo tiene el valor, sino qué valores específicos son válidos y qué significado tienen. Dos atributos pueden compartir tipo de dato pero pertenecer a dominios completamente distintos.

En SQL, los dominios se implementan de forma incompleta. La sentencia `CREATE DOMAIN` existe en el estándar y en PostgreSQL, pero la mayoría de los motores no la implementan. Se suelen simular con `CHECK` constraints, tipos enumerados, o simplemente con documentación.

```sql
-- Esto existe en PostgreSQL (aunque rara vez se usa)
CREATE DOMAIN porcentaje AS NUMERIC(5,2)
  CHECK (VALUE >= 0 AND VALUE <= 100);

-- Luego se usa como tipo:
CREATE TABLE descuentos (
    producto_id INT,
    descuento porcentaje  -- es un dominio, no solo un tipo
);
```

La importancia de pensar en dominios (no solo en tipos) es que **las operaciones del álgebra relacional solo tienen sentido entre atributos del mismo dominio**. Comparar `edad` con `código_postal` puede ser sintácticamente válido en SQL pero es semánticamente absurdo, y el modelo formal lo prohíbe.

---

## 4. El modelo lógico vs la implementación física

Esta es una de las distinciones más importantes de toda la arquitectura de bases de datos, y una que se confunde constantemente.

### 4.1 Los tres niveles de la arquitectura ANSI/SPARC

La arquitectura ANSI/SPARC (1975) define tres niveles de abstracción:

```
┌─────────────────────────────────────┐
│         NIVEL EXTERNO               │  ← Vistas, lo que ve cada aplicación
├─────────────────────────────────────┤
│         NIVEL CONCEPTUAL            │  ← El modelo lógico: relaciones, atributos, restricciones
├─────────────────────────────────────┤
│         NIVEL INTERNO               │  ← La implementación física: páginas, índices, archivos
└─────────────────────────────────────┘
```

**Nivel externo (vistas):** Cada aplicación o usuario ve solo la parte de los datos que le concierne, posiblemente con una estructura diferente a la del modelo conceptual. Una vista de "empleados activos con su departamento" puede existir para una aplicación de reportes sin que esa aplicación sepa nada de cómo se almacenan los datos.

**Nivel conceptual (modelo lógico):** Es el modelo relacional como matemática. Aquí viven las relaciones, los atributos, los dominios, las claves, las restricciones de integridad referencial. Es independiente de cualquier DBMS específico. Aquí es donde deberíamos pasar la mayor parte del tiempo en diseño.

**Nivel interno (modelo físico):** Es cómo el DBMS almacena los datos en disco. Tipos de archivos, estructuras de índices, distribución de páginas, compresión. Este nivel es completamente dependiente del motor.

### 4.2 La independencia de datos

El objetivo de esta separación es lograr **independencia de datos**:

- **Independencia lógica:** Puedo cambiar el modelo conceptual sin afectar las aplicaciones que usan el nivel externo.
- **Independencia física:** Puedo cambiar cómo se almacenan los datos físicamente sin cambiar el modelo conceptual.

Por ejemplo: si agrego un índice a una tabla (cambio en el nivel interno), las queries del nivel externo siguen siendo correctas. Si elimino y recreo una vista (cambio en el nivel externo), el modelo conceptual no cambia.

### 4.3 Por qué la confusión entre niveles es costosa

Cuando un desarrollador diseña pensando en "cómo quedará en la base de datos" en lugar de "cuál es el modelo correcto del dominio", mezcla el nivel conceptual con el físico. Esto genera:

- **Diseños guiados por rendimiento prematuro:** "Voy a desnormalizar porque los JOINs son lentos" antes de haber medido nada.
- **Acoplamiento al motor:** El esquema supone características específicas de MySQL, PostgreSQL o SQL Server, haciendo difícil migrar.
- **Pérdida de integridad:** Al pensar en implementación, se omiten restricciones que el modelo correcto requeriría.

La regla práctica es: **primero diseña el modelo lógico correcto, sin pensar en rendimiento**. Después, en el modelo físico, agrega índices, particiones, y ajustes específicos del motor. Si el modelo lógico es correcto, el modelo físico tiene una base sólida sobre la cual optimizar.

---

## 5. Las 12 reglas de Codd

En 1985, Edgar F. Codd publicó una lista de 12 reglas (más una regla cero) para determinar si un sistema de gestión de bases de datos puede ser considerado "verdaderamente relacional". La publicó en respuesta a que muchos productos de la época se anunciaban como "relacionales" sin serlo.

Es instructivo revisar estas reglas porque revelan qué tan lejos están los DBMS modernos del ideal teórico.

### Regla 0 — La regla fundamental
> *Todo sistema que se afirme relacional debe ser capaz de gestionar sus bases de datos completamente a través de sus capacidades relacionales.*

Es la meta-regla: si un sistema requiere medios no relacionales para operaciones básicas, no cumple el modelo.

### Regla 1 — Información representada como tablas
> *Toda información en la base de datos se representa en el nivel lógico exclusivamente como valores en tablas.*

Esto implica que no debería haber "canales ocultos" para transmitir información: punteros, ordenamientos implícitos, estructuras de árbol. Solo valores en columnas de tablas.

**¿La cumplen los DBMS modernos?** Parcialmente. PostgreSQL, por ejemplo, tiene tipos de datos como arrays y JSON que son valores compuestos, no atómicos. Técnicamente violan la noción de dominio atómico.

### Regla 2 — Acceso garantizado
> *Todo dato individual en la base de datos debe ser accesible especificando el nombre de la tabla, el valor de la clave primaria, y el nombre de la columna.*

Esto garantiza que el modelo tiene un mecanismo de direccionamiento completo y sin ambigüedades.

**¿La cumplen los DBMS modernos?** Generalmente sí, siempre que haya una clave primaria definida. El problema es que SQL permite tablas sin clave primaria, lo que viola esta regla.

### Regla 3 — Tratamiento sistemático del valor nulo
> *Los valores nulos deben ser soportados de forma uniforme para representar información faltante o inaplicable, independientemente del tipo de dato, en un manejo distinto a los valores por defecto.*

Esta es la regla más polémica. Codd la incluyó porque reconocía la necesidad práctica de representar "dato desconocido", pero su inclusión en el modelo es fuente de problemas teóricos profundos (los veremos en detalle en la sección 6).

### Regla 4 — Catálogo dinámico en línea
> *La descripción de la base de datos (metadatos) debe representarse al mismo nivel que los datos ordinarios, de modo que los usuarios autorizados puedan consultarla con el mismo lenguaje relacional.*

En la práctica: el catálogo del sistema (información sobre tablas, columnas, índices, etc.) debe ser consultable con SQL.

**¿La cumplen los DBMS modernos?** Sí. En PostgreSQL existe `information_schema` y `pg_catalog`. En SQL Server, `sys.*`. En MySQL, `information_schema`. Esta es una de las reglas mejor implementadas.

### Regla 5 — Sublenguaje de datos completo
> *El sistema debe soportar al menos un lenguaje relacional que tenga sintaxis lineal, pueda usarse interactivamente y en programas de aplicación, soporte operaciones de definición de datos, manipulación de datos, restricciones de integridad, autorización y gestión de transacciones.*

SQL es el lenguaje diseñado para esto. Que el estándar SQL exista y los motores lo implementen (con variaciones) satisface esta regla en general.

### Regla 6 — Actualización de vistas
> *Todas las vistas que sean teóricamente actualizables deben ser actualizables por el sistema.*

Una vista es "teóricamente actualizable" si, dada una modificación en la vista, existe exactamente una modificación equivalente en las tablas base. Por ejemplo, una vista que es una selección y proyección de una sola tabla, sin aggregations ni joins, es actualizable.

**¿La cumplen los DBMS modernos?** Parcialmente. La mayoría permite actualizar vistas simples pero falla en casos más complejos. PostgreSQL tiene `INSTEAD OF` triggers como solución parcial.

### Regla 7 — Inserción, actualización y borrado de alto nivel
> *El sistema debe soportar inserción, actualización y borrado de conjuntos de filas (no solo fila a fila).*

SQL satisface esto con `INSERT ... SELECT`, `UPDATE` con predicados, `DELETE` con predicados. No deberías necesitar iterar fila a fila para modificaciones masivas.

### Regla 8 — Independencia física de datos
> *Los programas de aplicación y las actividades del terminal no deben ser afectados cuando se realizan cambios en los métodos de acceso físico o en las estructuras de almacenamiento.*

Si agrego un índice, reorganizo el almacenamiento, o cambio el archivo de datos, las queries deben seguir funcionando correctamente.

**¿La cumplen los DBMS modernos?** Generalmente sí, con matices. Un cambio en la estructura de almacenamiento puede cambiar el rendimiento de una query, y si el plan de ejecución cambia, el comportamiento puede ser diferente (aunque lógicamente correcto).

### Regla 9 — Independencia lógica de datos
> *Los programas de aplicación no deben ser afectados cuando se realizan cambios en las tablas base que preservan la información.*

Si agrego una columna a una tabla, o separo una tabla en dos con una vista que las une, las aplicaciones que usaban la tabla original deben seguir funcionando.

**¿La cumplen los DBMS modernos?** Parcialmente. SQL `SELECT *` viola esto inmediatamente: si agrego una columna, `SELECT *` devuelve más columnas y puede romper código que asumía un número fijo de columnas.

### Regla 10 — Independencia de integridad
> *Las restricciones de integridad deben ser especificadas en el sublenguaje de datos relacional y almacenadas en el catálogo, no en los programas de aplicación.*

Las restricciones (claves primarias, claves foráneas, CHECK constraints) deben vivir en la base de datos, no en el código de la aplicación.

**¿La cumplen los DBMS modernos?** Técnicamente sí, pero en la práctica muchos desarrolladores ponen las restricciones en el ORM o en el código de aplicación, violando el espíritu de esta regla. Esto es un error de práctica, no de capacidad del motor.

### Regla 11 — Independencia de distribución
> *El sistema no debe requerir que los usuarios o programas cambien su comportamiento si los datos son redistribuidos a través de la red o el sistema se convierte en distribuido.*

Las queries deben funcionar igual si los datos están en un servidor o distribuidos en muchos.

**¿La cumplen los DBMS modernos?** Esta es la regla más difícil de cumplir. Los sistemas distribuidos modernos (Cassandra, DynamoDB, etc.) sacrifican explícitamente consistencia por disponibilidad, violando esta regla. Los DBMS tradicionales la cumplen en sistemas centralizados.

### Regla 12 — Regla de no subversión
> *Si el sistema tiene un lenguaje de bajo nivel (un registro a la vez), ese lenguaje de bajo nivel no debe poder subvertir las restricciones de integridad expresadas en el lenguaje relacional de alto nivel.*

No debería ser posible saltarse las restricciones de integridad accediendo a los datos "por debajo" del motor relacional.

**¿La cumplen los DBMS modernos?** En general sí dentro del motor. Pero existen mecanismos como acceso directo a archivos, herramientas de recuperación, o ciertas extensiones que pueden bypasear restricciones. En uso normal, se cumple.

### Resumen del cumplimiento

La conclusión honesta es que **ningún DBMS moderno cumple todas las 12 reglas**. SQL mismo, el lenguaje oficial del modelo relacional, viola varias de ellas (permite duplicados, tiene NULLs con semántica problemática, permite tablas sin clave primaria). Pero conocer las reglas nos permite entender qué estamos sacrificando cuando usamos ciertos features, y tomar decisiones informadas.

---

## 6. El problema del valor nulo

### 6.1 Por qué existe NULL

NULL fue introducido por Codd como solución al problema práctico de información ausente. En el mundo real, a veces simplemente no sabemos el valor de un atributo:

- Un empleado sin teléfono de contacto registrado.
- Una orden sin fecha de entrega aún (no ha sido entregada).
- Un producto sin precio establecido todavía.

La alternativa a NULL sería inventar un valor centinela: usar `0` para "sin precio", `"N/A"` para "sin nombre", `1900-01-01` para "sin fecha". Pero esos valores centinela son peores aún: son datos falsos que se mezclan con datos reales y requieren lógica especial en cada consulta.

Entonces Codd incluyó NULL como "valor" especial que significa "desconocido o inaplicable". Pero al hacerlo, rompió algo fundamental.

### 6.2 La lógica trivaluada

En lógica clásica (bivaluada), cualquier proposición es verdadera o falsa. No hay término medio. Esto es lo que permite razonar formalmente sobre datos.

Cuando introduces NULL, los predicados ya no devuelven solo `TRUE` o `FALSE`. Devuelven un tercer valor: `UNKNOWN`.

Considera: ¿Es `NULL = 5` verdadero o falso?
- No podemos decir que es verdadero (no sabemos si el valor es 5).
- No podemos decir que es falso (no sabemos si el valor no es 5).
- La respuesta es `UNKNOWN`.

Esto crea una **lógica trivaluada** con las siguientes tablas de verdad:

**AND:**
| | TRUE | FALSE | UNKNOWN | 
|---|---|---|---|
| **TRUE** | TRUE | FALSE | UNKNOWN |
| **FALSE** | FALSE | FALSE | FALSE |
| **UNKNOWN** | UNKNOWN | FALSE | UNKNOWN |

**OR:**
| | TRUE | FALSE | UNKNOWN |
|---|---|---|---|
| **TRUE** | TRUE | TRUE | TRUE |
| **FALSE** | TRUE | FALSE | UNKNOWN |
| **UNKNOWN** | TRUE | UNKNOWN | UNKNOWN |

**NOT:**
| | Resultado |
|---|---|
| TRUE | FALSE |
| FALSE | TRUE |
| UNKNOWN | UNKNOWN |

### 6.3 Las consecuencias prácticas y los bugs que genera

Esta lógica trivaluada produce resultados que parecen bugs pero son comportamiento correcto según el estándar SQL.

**Ejemplo 1: La ley de tercero excluido falla**

En lógica clásica, `P OR NOT P` siempre es verdadero. Pero con NULL:

```sql
-- Suponiendo que x es NULL:
SELECT CASE
    WHEN x = 5 THEN 'igual a 5'
    WHEN x != 5 THEN 'distinto de 5'
    ELSE '¿qué pasó aquí?'
END;
-- Resultado: '¿qué pasó aquí?'
-- Porque tanto (NULL = 5) como (NULL != 5) son UNKNOWN
```

**Ejemplo 2: NULL no es igual a NULL**

```sql
SELECT NULL = NULL;     -- UNKNOWN, no TRUE
SELECT NULL IS NULL;    -- TRUE (operador especial necesario)
SELECT NULL != NULL;    -- UNKNOWN, no FALSE
```

Por esto, la comparación normal no funciona para detectar NULLs. Se necesita `IS NULL` / `IS NOT NULL`.

**Ejemplo 3: Aggregations excluyen NULL silenciosamente**

```sql
CREATE TABLE ventas (monto DECIMAL);
INSERT INTO ventas VALUES (100), (200), (NULL), (300);

SELECT AVG(monto) FROM ventas;
-- Resultado: 200 (promedio de 100, 200, 300 = tres valores)
-- NO es 150 (que sería 600/4 si NULL fuera 0)
-- El NULL fue silenciosamente ignorado
```

Esto puede producir estadísticas incorrectas si el NULL debería representar "0" o algún otro valor.

**Ejemplo 4: COUNT(*) vs COUNT(columna)**

```sql
SELECT COUNT(*) FROM ventas;         -- 4 (cuenta todas las filas)
SELECT COUNT(monto) FROM ventas;     -- 3 (excluye NULLs)
```

**Ejemplo 5: El UNIQUE constraint y NULL**

En SQL estándar, múltiples NULLs en una columna UNIQUE son permitidos, porque `NULL != NULL` es UNKNOWN, no FALSE. Esto significa que UNIQUE no garantiza unicidad si hay NULLs.

```sql
CREATE TABLE prueba (codigo INT UNIQUE);
INSERT INTO prueba VALUES (NULL);
INSERT INTO prueba VALUES (NULL);  -- ¿Error? Depende del motor.
-- PostgreSQL: permite múltiples NULLs en columna UNIQUE
-- SQL Server: solo permite un NULL en columna UNIQUE
```

### 6.4 El argumento de C.J. Date contra NULL

C.J. Date, uno de los teóricos relacionales más influyentes, argumenta que **NULL debería eliminarse del modelo relacional**. Su argumento:

1. NULL introduce lógica trivaluada, lo que invalida muchas de las propiedades algebraicas que hacen al modelo relacional poderoso y predecible.
2. NULL significa cosas distintas en distintos contextos: "valor desconocido", "valor inaplicable", "valor que existe pero no fue capturado". Estos son conceptos semánticamente diferentes que NULL trata como si fueran lo mismo.
3. Hay formas de manejar información ausente sin NULL: usando tablas separadas para representar la ausencia explícitamente.

**La solución propuesta por Date: tablas separadas**

En lugar de:
```sql
-- Con NULL:
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100),
    telefono VARCHAR(20)  -- NULL si no tiene teléfono
);
```

Usar:
```sql
-- Sin NULL:
CREATE TABLE empleados (
    id INT PRIMARY KEY,
    nombre VARCHAR(100)
    -- sin columna teléfono aquí
);

CREATE TABLE telefonos_empleados (
    empleado_id INT REFERENCES empleados(id),
    telefono VARCHAR(20) NOT NULL,
    PRIMARY KEY (empleado_id)
);
```

Si un empleado no tiene teléfono, simplemente no tiene fila en `telefonos_empleados`. La ausencia está representada estructuralmente, no con un valor especial.

**¿Es esto práctico?** Depende. Para atributos verdaderamente opcionales que se consultan frecuentemente, la tabla separada introduce joins que complican las queries. En la práctica, usar `NOT NULL` donde es posible y ser extremadamente consciente de los NULLs cuando son necesarios es el compromiso más razonable.

### 6.5 Reglas prácticas para vivir con NULL

Dado que NULL existe en todos los DBMS que usamos, estas son las reglas para manejarlo correctamente:

1. **Usa `NOT NULL` por defecto.** Solo permite NULL cuando hay una razón semántica real, no por comodidad.
2. **Nunca compares con `= NULL`. Siempre usa `IS NULL` / `IS NOT NULL`.**
3. **Usa `COALESCE` para dar valores por defecto en expresiones.** `COALESCE(precio, 0)` devuelve el precio o 0 si es NULL.
4. **Sé explícito en aggregations.** Si un AVG o SUM puede ignorar NULLs, asegúrate de que eso es lo que quieres.
5. **Documenta el significado semántico de cada NULL.** ¿Significa "desconocido", "no aplica", o "aún no disponible"? Son conceptos distintos.

---

## 7. Álgebra relacional completa

El álgebra relacional es el lenguaje matemático formal para operar sobre relaciones. Es la base teórica de SQL: todas las queries SQL (en teoría) pueden expresarse como composiciones de operaciones del álgebra relacional.

A diferencia de SQL, el álgebra relacional:
- Trabaja con relaciones (sin duplicados, sin NULL en el modelo puro).
- Tiene propiedades algebraicas demostrables.
- Permite razonar sobre equivalencia de expresiones (si dos expresiones son equivalentes, el optimizador puede elegir la más eficiente).

Hay un **conjunto mínimo** de operaciones que forman un sistema completo (del que se pueden derivar todas las demás) y operaciones **derivadas** que son convenientes pero redundantes teóricamente.

**Operaciones primitivas (conjunto mínimo):**
1. Selección (σ)
2. Proyección (π)
3. Producto cartesiano (×)
4. Unión (∪)
5. Diferencia (−)
6. Renombramiento (ρ)

**Operaciones derivadas (definibles a partir de las primitivas):**
7. Intersección (∩)
8. Join natural (⋈)
9. División (÷)
10. Join con condición (θ-join)

Veremos cada una con detalle.

---

### 7.1 Selección (σ — sigma)

**Qué hace:** Filtra tuplas de una relación que satisfacen una condición. Opera sobre **filas**.

**Notación:** `σ_condición(R)`

**Equivalente SQL:** `WHERE`

**Ejemplo:**

Sea la relación `Empleados`:

| id | nombre | departamento | salario |
|---|---|---|---|
| 1 | Ana | Ingeniería | 8000 |
| 2 | Bruno | Marketing | 5000 |
| 3 | Carla | Ingeniería | 9000 |
| 4 | Diego | RRHH | 4500 |

La operación `σ_(departamento = 'Ingeniería')(Empleados)` produce:

| id | nombre | departamento | salario |
|---|---|---|---|
| 1 | Ana | Ingeniería | 8000 |
| 3 | Carla | Ingeniería | 9000 |

**Propiedades importantes:**

- El resultado es una relación con el mismo esquema que la relación original.
- Las condiciones pueden combinarse con AND (∧), OR (∨), NOT (¬).
- `σ_(c1 AND c2)(R) = σ_c1(σ_c2(R))` — Las selecciones se pueden "cascadear". El optimizador usa esto.
- `σ_(c1 AND c2)(R) = σ_(c2 AND c1)(R)` — La selección es conmutativa.

---

### 7.2 Proyección (π — pi)

**Qué hace:** Selecciona ciertos atributos (columnas) de una relación, eliminando los demás. Opera sobre **columnas**.

**Notación:** `π_(lista_de_atributos)(R)`

**Equivalente SQL:** `SELECT columnas` (pero con eliminación automática de duplicados)

**Ejemplo:**

`π_(nombre, departamento)(Empleados)` produce:

| nombre | departamento |
|---|---|
| Ana | Ingeniería |
| Bruno | Marketing |
| Carla | Ingeniería |
| Diego | RRHH |

**Punto crítico sobre duplicados:**

Si proyectamos sobre un conjunto de atributos que no incluye la clave, puede haber duplicados en el resultado. En álgebra relacional, la proyección **elimina duplicados automáticamente** (porque el resultado debe ser un conjunto).

Sea `Proyectos`:

| empleado_id | proyecto | horas |
|---|---|---|
| 1 | Alpha | 10 |
| 1 | Beta | 20 |
| 2 | Alpha | 15 |

`π_(proyecto)(Proyectos)` produce:

| proyecto |
|---|
| Alpha |
| Beta |

*(Solo dos filas, aunque había dos filas con "Alpha" — los duplicados se eliminan)*

En SQL, `SELECT proyecto FROM Proyectos` devuelve **tres filas** (con duplicado). Para emular la proyección relacional, necesitas `SELECT DISTINCT proyecto FROM Proyectos`. Esta es una diferencia real entre SQL y el álgebra relacional.

---

### 7.3 Producto cartesiano (× — cruz)

**Qué hace:** Combina cada tupla de una relación con cada tupla de otra, produciendo todas las combinaciones posibles. Es la operación más "primitiva" de combinación entre relaciones.

**Notación:** `R × S`

**Equivalente SQL:** `FROM R, S` (sin condición de join) o `CROSS JOIN`

**Ejemplo:**

Sea `Empleados_Mini`:

| id | nombre |
|---|---|
| 1 | Ana |
| 2 | Bruno |

Y `Proyectos_Mini`:

| codigo | nombre_proyecto |
|---|---|
| P1 | Alpha |
| P2 | Beta |

`Empleados_Mini × Proyectos_Mini` produce:

| Empleados_Mini.id | Empleados_Mini.nombre | Proyectos_Mini.codigo | Proyectos_Mini.nombre_proyecto |
|---|---|---|---|
| 1 | Ana | P1 | Alpha |
| 1 | Ana | P2 | Beta |
| 2 | Bruno | P1 | Alpha |
| 2 | Bruno | P2 | Beta |

Si R tiene m tuplas y S tiene n tuplas, R × S tiene m×n tuplas.

**¿Para qué sirve si genera combinaciones absurdas?** El producto cartesiano solo es útil en combinación con la selección. La combinación `σ_condición(R × S)` es la base del **θ-join** (join con condición), que veremos más adelante.

---

### 7.4 Unión (∪)

**Qué hace:** Produce el conjunto de todas las tuplas que están en R, en S, o en ambas. Requiere que R y S sean **compatibles en unión**: mismo número de atributos, con dominios compatibles en cada posición.

**Notación:** `R ∪ S`

**Equivalente SQL:** `UNION` (con eliminación de duplicados) o `UNION ALL` (sin eliminación)

**Ejemplo:**

`Empleados_Activos`:

| id | nombre |
|---|---|
| 1 | Ana |
| 2 | Bruno |

`Empleados_Contratados_Este_Mes`:

| id | nombre |
|---|---|
| 2 | Bruno |
| 3 | Carla |

`Empleados_Activos ∪ Empleados_Contratados_Este_Mes` produce:

| id | nombre |
|---|---|
| 1 | Ana |
| 2 | Bruno |
| 3 | Carla |

*(Bruno aparece solo una vez — es un conjunto)*

**Propiedades:**
- `R ∪ S = S ∪ R` (conmutativa)
- `(R ∪ S) ∪ T = R ∪ (S ∪ T)` (asociativa)
- `R ∪ R = R` (idempotente)

---

### 7.5 Diferencia (−)

**Qué hace:** Produce el conjunto de tuplas que están en R pero **no** en S. También requiere compatibilidad en unión.

**Notación:** `R − S`

**Equivalente SQL:** `EXCEPT`

**Ejemplo:**

`Todos_Los_Empleados − Empleados_Con_Proyecto` produce los empleados que no tienen ningún proyecto asignado.

Usando las relaciones anteriores:

`Empleados_Activos − Empleados_Contratados_Este_Mes` produce:

| id | nombre |
|---|---|
| 1 | Ana |

*(Solo Ana, porque Bruno está en ambas, y Carla solo está en la segunda)*

**Propiedades:**
- `R − S ≠ S − R` (NO es conmutativa)
- `R − R = ∅` (diferencia con sí misma es vacío)
- `R − ∅ = R`

---

### 7.6 Renombramiento (ρ — rho)

**Qué hace:** Renombra una relación y/o sus atributos. Es una operación "administrativa" necesaria para poder hacer operaciones como el producto cartesiano de una relación consigo misma.

**Notación:** `ρ_(nuevo_nombre)(R)` o `ρ_(nuevo_nombre(attr1→nuevo1, attr2→nuevo2))(R)`

**Equivalente SQL:** `AS` en aliases de tabla y columna.

**Ejemplo — por qué es necesario:**

Quiero encontrar pares de empleados que trabajan en el mismo departamento. Necesito hacer el producto cartesiano de `Empleados` con sí misma. Pero si tengo `Empleados × Empleados`, hay ambigüedad: ¿a qué `id` me refiero?

Con renombramiento:

```
ρ_E1(Empleados) × ρ_E2(Empleados)
```

Ahora tengo `E1.id`, `E1.nombre`, `E2.id`, `E2.nombre`, que son atributos distintos y puedo escribir la condición correctamente.

---

### 7.7 Intersección (∩) — derivada

**Qué hace:** Produce las tuplas que están en ambas relaciones. Es derivada porque `R ∩ S = R − (R − S)`.

**Equivalente SQL:** `INTERSECT`

**Ejemplo:**

`Empleados_En_Proyecto_Alpha ∩ Empleados_En_Proyecto_Beta` = empleados que trabajan en ambos proyectos.

---

### 7.8 Join natural (⋈)

**Qué hace:** Es la operación más usada en la práctica. Combina dos relaciones uniendo las tuplas que tienen el mismo valor en los **atributos de igual nombre**, y proyecta para que esos atributos no aparezcan duplicados.

**Notación:** `R ⋈ S`

**Definición formal:**
```
R ⋈ S = π_(atributos sin duplicar)(σ_(R.A = S.A para cada A en común)(R × S))
```

**Equivalente SQL:** `NATURAL JOIN` (aunque raramente recomendado en práctica por ser implícito)

**Ejemplo:**

`Empleados`:

| emp_id | nombre | dep_id |
|---|---|---|
| 1 | Ana | D1 |
| 2 | Bruno | D2 |

`Departamentos`:

| dep_id | nombre_dep | presupuesto |
|---|---|---|
| D1 | Ingeniería | 100000 |
| D2 | Marketing | 50000 |

`Empleados ⋈ Departamentos` produce:

| emp_id | nombre | dep_id | nombre_dep | presupuesto |
|---|---|---|---|---|
| 1 | Ana | D1 | Ingeniería | 100000 |
| 2 | Bruno | D2 | Marketing | 50000 |

El atributo `dep_id` aparece una sola vez (no duplicado) y solo se combinan las tuplas donde `Empleados.dep_id = Departamentos.dep_id`.

**¿Por qué no usar `NATURAL JOIN` en SQL?**

Porque en SQL, `NATURAL JOIN` es **implícito**: une automáticamente por todos los atributos de igual nombre. Si agregas una columna `nombre` a `Departamentos` (quizás porque la migraste de otro sistema), el join cambia silenciosamente de comportamiento. El `NATURAL JOIN` del álgebra relacional es correcto como concepto; el de SQL es peligroso por su implicitez.

En práctica se usa `JOIN ... ON condición` o `JOIN ... USING (columna)` para ser explícito.

---

### 7.9 Theta-join (θ-join)

**Qué hace:** Combina dos relaciones con una condición de join arbitraria. Es el join general.

**Notación:** `R ⋈_θ S = σ_θ(R × S)`

**Ejemplo:**

`Empleados ⋈_(salario > presupuesto/10) Departamentos` — empleados cuyo salario supera el 10% del presupuesto de algún departamento.

**El equi-join** es el caso especial del θ-join donde la condición es de igualdad entre atributos. Es el tipo más común en la práctica.

---

### 7.10 División (÷)

**Qué hace:** Es la operación más difícil de entender intuitivamente. Dadas dos relaciones R y S, `R ÷ S` produce las tuplas de R que están relacionadas con **todas** las tuplas de S.

**Cuándo se usa:** Cuando la pregunta tiene la forma "dame todos los X que tienen relación con **todos** los Y". Por ejemplo: "dame los estudiantes que han tomado **todos** los cursos requeridos" o "dame los proveedores que suministran **todos** los productos que necesitamos".

**Definición formal:**

Si R tiene atributos (A, B) y S tiene atributos (B), entonces:

```
R ÷ S = π_A(R) − π_A((π_A(R) × S) − R)
```

Es decir: toma todas las A, resta las A que, combinadas con alguna B de S, no están en R.

**Ejemplo:**

`Matriculas` (estudiante, curso):

| estudiante | curso |
|---|---|
| Ana | Mat |
| Ana | Fis |
| Ana | Qui |
| Bruno | Mat |
| Bruno | Fis |
| Carla | Mat |
| Carla | Fis |
| Carla | Qui |

`Cursos_Requeridos` (curso):

| curso |
|---|
| Mat |
| Fis |
| Qui |

`Matriculas ÷ Cursos_Requeridos` produce los estudiantes que tomaron **todos** los cursos requeridos:

| estudiante |
|---|
| Ana |
| Carla |

*(Bruno no aparece porque no tomó Qui)*

**Equivalente SQL:** No hay operador SQL directo. Se implementa típicamente con `NOT EXISTS` doble:

```sql
-- Estudiantes que tomaron TODOS los cursos requeridos
SELECT DISTINCT m.estudiante
FROM Matriculas m
WHERE NOT EXISTS (
    -- No existe ningún curso requerido...
    SELECT * FROM Cursos_Requeridos cr
    WHERE NOT EXISTS (
        -- ...que este estudiante no haya tomado
        SELECT * FROM Matriculas m2
        WHERE m2.estudiante = m.estudiante
          AND m2.curso = cr.curso
    )
);
```

La división relacional es el fundamento matemático de este patrón de query "universal".

---

## 8. Composición de operadores y equivalencias

Una de las propiedades más poderosas del álgebra relacional es que las operaciones se pueden **componer** libremente (el resultado de una operación es siempre una relación, y puede usarse como entrada de otra), y existen **equivalencias algebraicas** que el optimizador de queries puede explotar.

### 8.1 Equivalencias que usa el query optimizer

El query planner de un DBMS conoce estas equivalencias y las aplica para transformar la expresión en algo más eficiente:

**Cascada de selecciones:**
```
σ_(c1 AND c2)(R) = σ_c1(σ_c2(R))
```
*Permite "empujar" selecciones hacia abajo en el árbol de operaciones.*

**Conmutatividad de selección con proyección:**
```
σ_c(π_A(R)) = π_A(σ_c(R))  (si los atributos de c están en A)
```
*Primero filtra, luego proyecta — reduces datos más temprano.*

**Conmutatividad de join:**
```
R ⋈ S = S ⋈ R
```
*El optimizador puede elegir qué tabla "va primero" para minimizar el tamaño del resultado intermedio.*

**Asociatividad de join:**
```
(R ⋈ S) ⋈ T = R ⋈ (S ⋈ T)
```
*El optimizador puede reordenar los joins para minimizar los costos intermedios.*

### 8.2 Un ejemplo de optimización

Considera esta query:

```sql
SELECT e.nombre, d.nombre_dep
FROM Empleados e
JOIN Departamentos d ON e.dep_id = d.dep_id
WHERE e.salario > 7000 AND d.presupuesto > 80000;
```

En álgebra relacional sin optimizar:
```
π_(nombre, nombre_dep)(σ_(salario>7000 AND presupuesto>80000)(Empleados ⋈ Departamentos))
```

Esto hace el join completo (todas las combinaciones) y luego filtra. Si hay 10,000 empleados y 500 departamentos, el join produce hasta 5,000,000 de tuplas antes de filtrar.

El optimizador puede reescribirlo como:
```
π_(nombre, nombre_dep)(σ_(salario>7000)(Empleados) ⋈ σ_(presupuesto>80000)(Departamentos))
```

Primero filtra empleados con salario alto (quizás 200), luego filtra departamentos con presupuesto alto (quizás 10), y luego hace el join (200 × 10 = 2,000 combinaciones máximo). Drásticamente más eficiente.

Este proceso se llama **predicado pushdown** y es una de las optimizaciones más importantes que realiza cualquier query planner moderno. El fundamento matemático que lo hace posible son las equivalencias del álgebra relacional.

---

## 9. Resumen y conexión con el resto del curso

Hemos cubierto el modelo relacional como sistema matemático. Los puntos clave:

**Lo que une todo este módulo:**

- Una **relación** no es una tabla. Es un conjunto de tuplas que satisfacen un esquema. Las propiedades de los conjuntos (sin duplicados, sin orden) se derivan de esto.
- Los **dominios** son restricciones semánticas más ricas que los tipos de dato. Pensar en dominios lleva a modelos más correctos.
- El **modelo lógico** (la estructura conceptual) debe separarse del **modelo físico** (la implementación). Diseñar en el nivel lógico primero es fundamental.
- Las **12 reglas de Codd** revelan qué tan lejos está SQL del ideal teórico. Ningún DBMS las cumple todas; conocerlas nos permite saber qué estamos sacrificando.
- **NULL** introduce lógica trivaluada que rompe propiedades fundamentales del modelo. Debe usarse con conciencia, no por default.
- El **álgebra relacional** es el lenguaje matemático de las operaciones sobre datos. SQL es su implementación práctica (imperfecta). Entender el álgebra permite entender la optimización de queries.

**Conexión con los módulos siguientes:**

- **Módulo 2 (Dependencias y normalización):** Las formas normales se derivan formalmente de dependencias funcionales. Sin entender qué es una relación en sentido formal, las formas normales parecen recetas arbitrarias.
- **Módulo 7 (Internals y query planner):** El optimizador opera exactamente con las equivalencias del álgebra relacional que vimos en la sección 8. Cuando leas un plan de ejecución, estarás viendo esas transformaciones aplicadas.
- **Módulo 15 (ORMs e impedancia objeto-relacional):** El "problema de impedancia" existe precisamente porque el modelo relacional tiene propiedades matemáticas que el modelo de objetos no tiene.

---

## Ejercicios de comprensión

Para consolidar este módulo, trabaja sobre los siguientes ejercicios:

**Ejercicio 1.** Dada una tabla `Pedidos(pedido_id, cliente_id, producto_id, fecha, cantidad, precio_unitario)`, escribe la operación de álgebra relacional (usando la notación del módulo) que devuelve el `cliente_id` y la `fecha` de los pedidos donde la `cantidad` es mayor a 10.

**Ejercicio 2.** ¿Por qué el siguiente query SQL puede dar un resultado diferente al esperado?
```sql
SELECT * FROM empleados WHERE departamento != 'RRHH';
```
*(Pista: ¿qué pasa si algunos empleados tienen `departamento = NULL`?)*

**Ejercicio 3.** Diseña el esquema relacional (sin NULLs) para representar empleados que pueden o no tener un vehículo de empresa asignado. ¿Cómo representas la ausencia del vehículo?

**Ejercicio 4.** Dado `Clientes(cliente_id, nombre)` y `Compras(cliente_id, producto_id)`, escribe en álgebra relacional la operación que devuelve los clientes que **nunca han comprado nada**. (Pista: usa la diferencia.)

**Ejercicio 5.** Explica con tus palabras por qué `SELECT *` en SQL puede violar la independencia lógica de datos (Regla 9 de Codd) si se agrega una columna nueva a la tabla.

---

*Próximo módulo: Dependencias funcionales y normalización real — donde las formas normales dejarán de ser reglas a memorizar y se convertirán en consecuencias inevitables del modelo formal que acabamos de construir.*
