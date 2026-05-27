# Módulo 3 — Modelado conceptual con profundidad

> *"Un error en el modelo conceptual es invisible una vez que se traduce a tablas. Lo que parece una tabla más o una columna de más es, en realidad, una decisión de diseño que determina qué preguntas podrá y no podrá responder el sistema durante años."*

---

## Tabla de contenidos

1. [Por qué el modelado conceptual merece su propio módulo](#1-por-qué-el-modelado-conceptual-merece-su-propio-módulo)
2. [El modelo Entidad-Relación Extendido (EER)](#2-el-modelo-entidad-relación-extendido-eer)
3. [Tipos de entidad, instancias y conjuntos](#3-tipos-de-entidad-instancias-y-conjuntos)
4. [Atributos: taxonomía completa](#4-atributos-taxonomía-completa)
5. [Entidades débiles y su semántica](#5-entidades-débiles-y-su-semántica)
6. [Tipos de relación: cardinalidad y participación](#6-tipos-de-relación-cardinalidad-y-participación)
7. [Relaciones ternarias y n-arias: cuándo son irreducibles](#7-relaciones-ternarias-y-n-arias-cuándo-son-irreducibles)
8. [Agregación](#8-agregación)
9. [Jerarquías de generalización y especialización](#9-jerarquías-de-generalización-y-especialización)
10. [De EER a esquema relacional: el proceso sistemático](#10-de-eer-a-esquema-relacional-el-proceso-sistemático)
11. [Errores de modelado que se vuelven invisibles](#11-errores-de-modelado-que-se-vuelven-invisibles)
12. [Un caso completo de modelado](#12-un-caso-completo-de-modelado)
13. [Resumen y conexión con el resto del curso](#13-resumen-y-conexión-con-el-resto-del-curso)

---

## 1. Por qué el modelado conceptual merece su propio módulo

### 1.1 El lugar del modelo conceptual en el proceso de diseño

El diseño de una base de datos ocurre en tres fases:

```
FASE 1: Modelado conceptual
    ↓  (independiente del DBMS, del lenguaje, de la tecnología)
    Diagrama EER — captura el "qué" del dominio

FASE 2: Modelado lógico
    ↓  (independiente del DBMS pero dentro del modelo relacional)
    Esquema relacional — tablas, columnas, claves, restricciones

FASE 3: Modelado físico
    ↓  (específico del DBMS)
    Índices, particiones, tipos de almacenamiento
```

El modelo conceptual es el más importante y el más descuidado. Es el único que se produce en conversación directa con las personas que conocen el dominio: el cliente, el analista de negocio, el experto en el área. Es donde se captura el significado de los datos, no su estructura técnica.

### 1.2 Por qué los errores conceptuales son los más costosos

Un error en el modelo físico (un índice mal configurado) se corrige en minutos sin tocar los datos. Un error en el modelo lógico (una tabla mal normalizada) requiere una migración pero no cambia el significado. Un error en el modelo conceptual significa que el modelo no refleja el dominio del problema: hay entidades que no están, relaciones que se modelaron incorrectamente, restricciones semánticas que se ignoraron.

Cuando ese error llega a producción, corregirlo requiere:
- Rediseñar tablas existentes.
- Migrar datos ya almacenados.
- Cambiar queries y código de aplicación.
- Potencialmente perder información que nunca se capturó correctamente.

El costo crece exponencialmente con el tiempo que pasa entre el error y su corrección.

### 1.3 El ER básico vs el EER

El modelo Entidad-Relación (ER) original, propuesto por Peter Chen en 1976, fue un gran avance: proporcionar una notación visual para capturar la estructura de los datos antes de hablar de implementación. Pero el ER básico es limitado: no puede expresar jerarquías, especializaciones, ni ciertas restricciones semánticas comunes.

El modelo **Entidad-Relación Extendido (EER)** agrega al ER original:
- Generalización y especialización (jerarquías de herencia).
- Categorías y uniones de tipos.
- Semántica más rica en participación y cardinalidad.

Este módulo trabaja con el EER completo porque es lo que se necesita para modelar dominios reales.

---

## 2. El modelo Entidad-Relación Extendido (EER)

### 2.1 Los componentes del EER

El EER tiene los siguientes elementos:

| Elemento | Símbolo clásico (Chen) | Descripción |
|---|---|---|
| Tipo de entidad | Rectángulo | Una clase de objetos del dominio |
| Tipo de entidad débil | Rectángulo doble | Entidad cuya existencia depende de otra |
| Atributo | Elipse | Propiedad de una entidad o relación |
| Atributo clave | Elipse con texto subrayado | Identifica unívocamente a la entidad |
| Atributo multivaluado | Elipse doble | Puede tener múltiples valores |
| Atributo derivado | Elipse punteada | Se calcula a partir de otros atributos |
| Atributo compuesto | Elipse con sub-elipses | Tiene estructura interna |
| Tipo de relación | Rombo | Asociación entre entidades |
| Relación de identificación | Rombo doble | Relaciona una entidad débil con su propietaria |
| Generalización/Especialización | Triángulo | Jerarquía de herencia entre tipos |

### 2.2 Notaciones alternativas

Existen varias notaciones para diagramas ER/EER. Las más comunes son:

**Notación de Chen (la original):** Usa rectángulos para entidades, rombos para relaciones, elipses para atributos. Es la más expresiva semánticamente pero ocupa mucho espacio.

**Notación Crow's Foot (pata de cuervo):** Usa líneas con símbolos en los extremos para representar cardinalidades. Es la más usada en herramientas como ERD en dbdiagram.io, Lucidchart, o MySQL Workbench.

**Notación UML (diagrama de clases):** Usa rectángulos con compartimentos. Muchos equipos la prefieren porque los desarrolladores ya la conocen.

En este módulo usaremos descripciones textuales y tablas en lugar de diagramas gráficos, pero los conceptos aplican a cualquier notación.

---

## 3. Tipos de entidad, instancias y conjuntos

### 3.1 La distinción fundamental: tipo vs instancia

Un **tipo de entidad** (o clase de entidad) es la descripción abstracta de un conjunto de objetos del mundo real que comparten los mismos atributos y participan en los mismos tipos de relación. Define la estructura.

Una **instancia de entidad** es un objeto concreto del mundo real que pertenece a ese tipo. Es un valor específico.

Ejemplos:
```
Tipo de entidad:   EMPLEADO
Instancias:        (id=1, nombre="Ana García", cargo="Desarrolladora")
                   (id=2, nombre="Bruno Ríos", cargo="DBA")
                   (id=3, nombre="Carla Vega", cargo="PM")

Tipo de entidad:   PRODUCTO
Instancias:        (id=P01, descripción="Tornillo M6", precio=0.05)
                   (id=P02, descripción="Tuerca M6", precio=0.03)
```

La distinción importa porque el modelo conceptual describe **tipos**, no instancias. El diagrama EER no muestra datos; muestra la estructura que los datos deben tener.

### 3.2 Qué merece ser una entidad

No todo sustantivo en la descripción del dominio merece ser una entidad. La heurística es:

**Merece ser entidad si:**
- Tiene existencia independiente en el dominio.
- Tiene múltiples atributos propios (más allá de solo el nombre).
- Participa en múltiples relaciones con otras entidades.
- El negocio necesita rastrear su historia o estado de forma independiente.

**No merece ser entidad (probablemente es atributo) si:**
- Solo tiene un valor descriptivo sin estructura propia.
- Su existencia depende completamente de otra entidad.
- El negocio nunca necesita consultarlo de forma independiente.

**Ejemplo:** En un sistema de recursos humanos:
- `EMPLEADO` → entidad (tiene id, nombre, cargo, salario, fecha de contratación...)
- `DEPARTAMENTO` → entidad (tiene código, nombre, presupuesto, gerente...)
- `CIUDAD` → ¿entidad o atributo?

`CIUDAD` merece ser entidad si el sistema necesita rastrear información sobre las ciudades (población, país, código postal, zona horaria), si múltiples entidades tienen relación con ciudades (empleados viven en ciudades, sucursales están en ciudades), o si las ciudades tienen su propia vida en el sistema. Si solo es un campo descriptivo que nadie consulta de forma independiente, puede ser un atributo simple.

### 3.3 La trampa de los "objetos disfrazados de atributos"

Un error común en modelado es tomar algo que debería ser una entidad y colapsarlo en un atributo. Esto es especialmente peligroso cuando:

- El "atributo" puede tener múltiples valores (debería ser entidad en relación 1:N).
- El "atributo" tiene propiedades propias (debería ser entidad con sus propios atributos).
- El "atributo" aparece en múltiples lugares del modelo con la misma semántica.

**Ejemplo:**

```
-- Diseño incorrecto: categoría como atributo
PRODUCTO(producto_id, nombre, precio, categoria)

-- Problema: si "categoria" tiene descripción, color de etiqueta, impuesto aplicable...
-- esos atributos no tienen dónde vivir.

-- Diseño correcto: categoría como entidad
PRODUCTO(producto_id, nombre, precio, categoria_id)
CATEGORIA(categoria_id, nombre, descripción, tasa_impuesto, color_etiqueta)
```

---

## 4. Atributos: taxonomía completa

### 4.1 Atributos simples vs compuestos

Un **atributo simple** (o atómico) tiene un único valor indivisible desde la perspectiva del dominio.

Ejemplos: `precio`, `fecha_nacimiento`, `activo`.

Un **atributo compuesto** tiene una estructura interna con componentes que tienen significado propio.

Ejemplo clásico: `dirección` se compone de `calle`, `número`, `ciudad`, `código_postal`, `país`.

```
dirección
├── calle
├── número
├── apartamento  (puede ser nulo)
├── ciudad
├── código_postal
└── país
```

**¿Cuándo mantener el atributo compuesto vs descomponerlo?**

Si el sistema necesita consultar, filtrar u ordenar por algún componente individual, debe descomponerse. Si la dirección siempre se trata como un bloque opaco (solo se muestra, nunca se filtra por ciudad), puede mantenerse como texto.

En la práctica: cuando hay duda, descomponer. Es más fácil concatenar componentes en una query que extraer componentes de un texto libre en producción.

### 4.2 Atributos de valor único vs multivaluados

Un **atributo de valor único** tiene exactamente un valor por instancia de entidad.

Un **atributo multivaluado** puede tener cero, uno o varios valores por instancia.

Ejemplos:
- `fecha_nacimiento` → valor único (una persona tiene una sola fecha de nacimiento)
- `números_de_teléfono` → multivaluado (una persona puede tener 0, 1 o varios teléfonos)
- `idiomas_que_habla` → multivaluado
- `títulos_académicos` → multivaluado

**Los atributos multivaluados en el esquema relacional:**

Como vimos en el módulo anterior, los atributos multivaluados violan 1FN si se almacenan directamente. La traducción correcta es siempre una tabla separada:

```
-- Atributo multivaluado en el modelo EER:
EMPLEADO con atributo multivaluado {teléfonos}

-- Se traduce a:
empleados(empleado_id, nombre, ...)
telefonos_empleado(empleado_id, tipo_telefono, numero)
--  clave: {empleado_id, numero} o {empleado_id, tipo_telefono}
```

### 4.3 Atributos almacenados vs derivados

Un **atributo almacenado** existe en el modelo y se persiste en la base de datos.

Un **atributo derivado** se puede calcular a partir de otros atributos o relaciones. No se almacena (normalmente); se calcula cuando se necesita.

Ejemplos:
- `edad` se deriva de `fecha_nacimiento` y la fecha actual.
- `total_pedido` se deriva de `sum(cantidad × precio_unitario)` de las líneas del pedido.
- `antigüedad_empleado` se deriva de `fecha_contratación` y la fecha actual.
- `número_de_empleados` de un departamento se deriva contando las instancias relacionadas.

**¿Cuándo almacenar un atributo derivado?**

Normalmente no se almacena porque genera redundancia: si cambia la fuente, el derivado queda desactualizado. Pero hay excepciones justificadas:

- Cuando el cálculo es muy costoso y el valor no cambia frecuentemente.
- Cuando necesitas el valor histórico (la edad que tenía alguien en un momento específico).
- Cuando el derivado requiere datos que luego se borran (el total de una factura después de eliminar las líneas).

### 4.4 Atributos clave

Un **atributo clave** (o identificador) es el atributo o conjunto de atributos que identifica unívocamente cada instancia de un tipo de entidad. En la notación de Chen, se subraya.

En el modelo EER, la clave es siempre del tipo de entidad, nunca de una instancia. Hay que garantizar que el atributo elegido como clave **nunca tenga duplicados en ninguna instancia posible del tipo**, no solo en los datos actuales.

Trampas comunes al elegir claves:
- Usar `nombre` como clave: pueden existir dos personas con el mismo nombre.
- Usar combinaciones que parecen únicas pero no lo son: `{nombre, fecha_nacimiento}` puede tener duplicados (gemelos con el mismo nombre).
- Usar datos externos sin verificar su unicidad garantizada: un número de teléfono puede reasignarse.

En el modelo EER se pueden indicar múltiples claves candidatas subrayándolas a todas. La elección de cuál será la clave primaria en el esquema relacional se hace en la fase de modelado lógico.

---

## 5. Entidades débiles y su semántica

### 5.1 Qué es una entidad débil

Una **entidad débil** es un tipo de entidad que **no tiene suficientes atributos para formar una clave propia**. Su identificación depende de una entidad "propietaria" (también llamada entidad fuerte o entidad de identificación).

La entidad débil tiene un **atributo discriminador** (o clave parcial): un atributo que la distingue entre las instancias que dependen de la misma entidad propietaria, pero no la identifica globalmente.

La clave completa de una instancia de entidad débil es: **clave de la entidad propietaria + atributo discriminador**.

### 5.2 Un ejemplo concreto

Considera el dominio de pedidos de una empresa:

```
PEDIDO(pedido_id, fecha, estado)
LÍNEA_DE_PEDIDO(número_línea, cantidad, precio_unitario)
```

Una `LÍNEA_DE_PEDIDO` no existe sin un `PEDIDO`. Su identificación no tiene sentido de forma independiente: "la línea número 3" no es suficiente sin saber "la línea número 3 del pedido P001".

La clave completa de una línea es `{pedido_id, número_línea}`:
- `pedido_id` viene de la entidad propietaria.
- `número_línea` es el discriminador (número 1, 2, 3... dentro del pedido).

### 5.3 Dependencia existencial vs dependencia de identificación

Esta distinción es sutil y frecuentemente confundida.

**Dependencia existencial:** Una entidad no puede existir sin otra. Si borras el pedido, desaparecen sus líneas. Esta es la restricción de "cascade delete".

**Dependencia de identificación:** Una entidad no puede ser identificada sin otra. La clave incluye la clave de la propietaria.

Toda entidad débil tiene dependencia de identificación (que implica dependencia existencial). Pero una entidad puede tener dependencia existencial sin ser débil:

```
-- EMPLEADO depende existencialmente de DEPARTAMENTO
-- (no puede existir un empleado sin departamento, por política del negocio)
-- Pero EMPLEADO tiene su propia clave (empleado_id).
-- Por lo tanto, EMPLEADO no es una entidad débil.
```

La entidad débil es aquella que, además de la dependencia existencial, necesita "pedir prestada" parte de su identidad a la propietaria.

### 5.4 Ejemplos adicionales de entidades débiles

**Sistema bancario:**
```
CUENTA_BANCARIA (fuerte) → TRANSACCIÓN (débil)
Una transacción se identifica como "transacción número 45 de la cuenta 00123456".
La transacción número 45 de otra cuenta es una entidad completamente diferente.
```

**Sistema de recursos humanos:**
```
EMPLEADO (fuerte) → FAMILIAR_DEPENDIENTE (débil)
"El dependiente 'hijo mayor' del empleado E001" se identifica por (E001, "hijo mayor").
```

**Sistema de manufactura:**
```
ORDEN_DE_TRABAJO (fuerte) → OPERACIÓN (débil)
"Operación 3 de la orden OT-2024-001" es la clave completa.
```

### 5.5 Entidades débiles en el esquema relacional

La traducción es directa: la tabla de la entidad débil incluye la clave foránea a la entidad propietaria, y esa clave foránea forma parte de la clave primaria compuesta.

```sql
CREATE TABLE pedidos (
    pedido_id    INT         PRIMARY KEY,
    fecha        DATE        NOT NULL,
    cliente_id   INT         NOT NULL REFERENCES clientes(cliente_id)
);

CREATE TABLE lineas_pedido (
    pedido_id       INT  NOT NULL REFERENCES pedidos(pedido_id) ON DELETE CASCADE,
    numero_linea    INT  NOT NULL,
    producto_id     INT  NOT NULL REFERENCES productos(producto_id),
    cantidad        INT  NOT NULL CHECK (cantidad > 0),
    precio_unitario DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (pedido_id, numero_linea)  -- clave compuesta: propietaria + discriminador
);
```

---

## 6. Tipos de relación: cardinalidad y participación

### 6.1 Qué es un tipo de relación

Un **tipo de relación** es una asociación entre dos o más tipos de entidad que tiene significado en el dominio. Igual que con las entidades, hay que distinguir entre el **tipo** de relación (la asociación abstracta, p.ej. "EMPLEADO trabaja_en DEPARTAMENTO") y las **instancias** de relación (los hechos concretos: "Ana trabaja en Ingeniería").

Una relación puede tener **atributos propios**: información que pertenece a la asociación, no a ninguna de las entidades participantes.

Ejemplo: La relación `TRABAJA_EN` entre `EMPLEADO` y `PROYECTO` tiene atributos propios:
- `fecha_inicio_asignación`
- `horas_semanales`
- `rol_en_proyecto`

Ninguno de estos atributos pertenece al empleado (varían por proyecto) ni al proyecto (varían por empleado). Pertenecen a la asociación específica.

### 6.2 Cardinalidad: cuántas instancias participan

La **cardinalidad** de un tipo de relación especifica cuántas instancias de un tipo de entidad pueden estar asociadas con cuántas instancias del otro tipo.

Las tres cardinalidades básicas son:

**Uno a Uno (1:1)**

Cada instancia de A se asocia con exactamente una instancia de B, y viceversa.

Ejemplo: `EMPLEADO` — `dirige` — `DEPARTAMENTO`
- Un empleado puede dirigir a lo sumo un departamento.
- Un departamento tiene a lo sumo un director.

Este es el tipo de relación más raro en la práctica. Cuando aparece, frecuentemente indica que las dos entidades deberían ser una sola (si siempre van juntas) o que la relación es más compleja de lo que parece.

**Uno a Muchos (1:N)**

Una instancia de A se puede asociar con muchas instancias de B. Cada instancia de B se asocia con exactamente una de A.

Ejemplo: `DEPARTAMENTO` — `tiene` — `EMPLEADO`
- Un departamento tiene muchos empleados.
- Un empleado pertenece a exactamente un departamento.

Es la cardinalidad más común. Es el patrón central del modelo relacional: una clave foránea en la tabla del lado "muchos" apuntando a la tabla del lado "uno".

**Muchos a Muchos (M:N)**

Una instancia de A puede asociarse con muchas de B, y una instancia de B puede asociarse con muchas de A.

Ejemplo: `EMPLEADO` — `asignado_a` — `PROYECTO`
- Un empleado puede trabajar en muchos proyectos.
- Un proyecto puede tener muchos empleados asignados.

Se implementa mediante una **tabla de unión** (también llamada tabla de junction o tabla de relación) en el esquema relacional.

### 6.3 Cardinalidades mínimas y máximas (notación (min, max))

La notación de cardinalidades básicas (1:1, 1:N, M:N) es insuficiente porque no captura las restricciones mínimas. La notación `(min, max)` es más precisa:

```
(0, 1)  → opcional, a lo sumo uno
(1, 1)  → exactamente uno (obligatorio)
(0, N)  → opcional, puede ser muchos
(1, N)  → al menos uno, puede ser muchos
(m, n)  → entre m y n instancias (m ≤ n)
```

Ejemplo: `EMPLEADO` — `asignado_a (0,N)` — `(1,N)` — `PROYECTO`

Se lee: un empleado puede estar asignado a cero o más proyectos; un proyecto debe tener asignado al menos un empleado y puede tener muchos.

Esto expresa dos restricciones que la notación básica no capturaba:
- Un empleado puede existir en el sistema sin estar asignado a ningún proyecto (mínimo 0).
- Un proyecto no puede existir sin al menos un empleado asignado (mínimo 1).

### 6.4 Participación total vs parcial

**Participación total** (o restricción de existencia): **todas** las instancias de un tipo de entidad deben participar en el tipo de relación. Se representa con doble línea en la notación de Chen, o con `|` (pipa vertical) en Crow's Foot.

**Participación parcial**: Algunas instancias pueden no participar en la relación. Es el caso por defecto.

Ejemplos:

```
EMPLEADO — trabaja_en → DEPARTAMENTO
  Participación de EMPLEADO: total (todo empleado debe trabajar en un departamento)
  Participación de DEPARTAMENTO: parcial (puede haber departamentos sin empleados aún)

EMPLEADO — dirige → DEPARTAMENTO
  Participación de EMPLEADO: parcial (no todos los empleados dirigen un departamento)
  Participación de DEPARTAMENTO: parcial (puede haber departamentos sin director temporal)
```

**La participación total tiene una consecuencia directa en el esquema relacional:** implica una restricción `NOT NULL` en la clave foránea (si la participación es total en el lado "muchos") o una clave foránea obligatoria con otras restricciones.

### 6.5 Por qué una elección incorrecta de cardinalidad destruye el modelo

La cardinalidad no es una descripción de los datos actuales: es una **restricción del dominio del negocio**. Elegirla incorrectamente introduce dos tipos de problemas:

**Restricción demasiado laxa (modelar M:N cuando es 1:N):**

Si el negocio dice "un empleado pertenece a exactamente un departamento" pero modelamos la relación como M:N (muchos departamentos por empleado), el sistema permitirá datos que el negocio no permite. Las queries serán más complejas de lo necesario, y las restricciones deberán mantenerse en la aplicación en lugar de en la base de datos.

**Restricción demasiado estricta (modelar 1:N cuando es M:N):**

Si modelamos como 1:N (un empleado, un proyecto) pero en realidad la relación es M:N, en algún momento el sistema no podrá representar el estado real del negocio. Llegar a ese punto en producción requiere una migración costosa.

La pregunta correcta al determinar cardinalidad es siempre sobre el **dominio, no sobre los datos actuales:**
> "¿Podría alguna vez existir una instancia de A asociada a más de una instancia de B?"
> "¿El negocio requiere que siempre haya al menos una asociación?"

---

## 7. Relaciones ternarias y n-arias: cuándo son irreducibles

### 7.1 El problema de las relaciones ternarias

Una **relación ternaria** involucra tres tipos de entidad simultáneamente. La pregunta que siempre hay que hacerse es: ¿puede esta relación ternaria descomponerse en relaciones binarias sin perder información?

La respuesta es: **a veces sí, a veces no**. Y la diferencia es crucial.

### 7.2 Cuándo una ternaria ES reducible a binarias

```
Relación ternaria: MÉDICO — atiende — PACIENTE en HOSPITAL
```

Si la semántica es:
- Un médico trabaja en un hospital (independientemente de los pacientes).
- Un paciente está asignado a un hospital (independientemente del médico).
- Un médico atiende a un paciente (independientemente del hospital).

Y el hecho de que "un médico atiende a un paciente" implica automáticamente que "lo atiende en el hospital donde ambos están", entonces la ternaria no agrega información sobre las tres binarias:

```
MÉDICO — trabaja_en — HOSPITAL       (binaria)
PACIENTE — asignado_a — HOSPITAL     (binaria)
MÉDICO — atiende — PACIENTE          (binaria)
```

El hospital de atención se puede derivar de las relaciones binarias. La ternaria es redundante.

### 7.3 Cuándo una ternaria NO ES reducible: el caso clásico

```
Relación ternaria: PROVEEDOR — suministra — PRODUCTO para PROYECTO
```

Semántica:
- El mismo proveedor puede suministrar el mismo producto a distintos proyectos.
- El mismo proveedor puede suministrar distintos productos al mismo proyecto.
- El mismo producto puede ser suministrado por distintos proveedores al mismo proyecto.

¿Qué pasa si la descomponemos en binarias?

```
PROVEEDOR — suministra_producto — PRODUCTO
PROVEEDOR — suministra_a — PROYECTO
PRODUCTO — usado_en — PROYECTO
```

Si un proveedor P1 suministra el producto PR1 (relación binaria 1), y P1 suministra algo al proyecto PY1 (relación binaria 2), y PR1 se usa en PY1 (relación binaria 3), ¿podemos concluir que P1 suministra PR1 a PY1?

**No necesariamente.** P1 podría suministrar PR2 a PY1, y PR1 podría ser suministrado por P2 a PY1. Las tres binarias juntas no implican la ternaria original.

Esta es la "trampa de las relaciones ternarias": el join de las tres binarias puede producir **instancias espurias** que no corresponden a hechos reales del dominio.

### 7.4 La prueba formal de irreducibilidad

Para probar que una relación ternaria `R(A, B, C)` es irreducible a binarias:

Construye un contraejemplo: un conjunto de instancias donde las tres binarias son verdaderas para ciertas combinaciones, pero la ternaria no. Si ese contraejemplo es posible en el dominio, la ternaria es irreducible.

Para `SUMINISTRA(proveedor, producto, proyecto)`:

| proveedor | producto | proyecto |
|---|---|---|
| P1 | PR1 | PY1 |
| P2 | PR2 | PY1 |

Binarias derivadas:
- `PROVEEDOR-PRODUCTO`: (P1, PR1), (P2, PR2)
- `PROVEEDOR-PROYECTO`: (P1, PY1), (P2, PY1)
- `PRODUCTO-PROYECTO`: (PR1, PY1), (PR2, PY1)

Join de las tres binarias:
```
(P1, PR1, PY1)  ← instancia real ✓
(P2, PR2, PY1)  ← instancia real ✓
(P1, PR2, PY1)  ← ESPURIA: P1 no suministra PR2 al proyecto PY1
(P2, PR1, PY1)  ← ESPURIA: P2 no suministra PR1 al proyecto PY1
```

El join produce instancias espurias. La ternaria es irreducible.

### 7.5 Implementación de relaciones ternarias en el esquema relacional

Una relación ternaria M:N:P se implementa con una tabla de unión que tiene las claves foráneas de las tres entidades participantes, más los atributos de la relación.

```sql
CREATE TABLE suministros (
    proveedor_id  INT  NOT NULL REFERENCES proveedores(proveedor_id),
    producto_id   INT  NOT NULL REFERENCES productos(producto_id),
    proyecto_id   INT  NOT NULL REFERENCES proyectos(proyecto_id),
    cantidad      INT  NOT NULL,
    fecha_entrega DATE,
    PRIMARY KEY (proveedor_id, producto_id, proyecto_id)
);
```

La clave primaria compuesta garantiza que no haya dos registros del mismo proveedor suministrando el mismo producto al mismo proyecto.

### 7.6 La regla práctica

Cuando encuentres una posible relación ternaria:
1. Pregúntate: "¿El hecho de que A-B, B-C, y A-C sean ciertos implica necesariamente que A-B-C es cierto?"
2. Si sí → descomponer en binarias.
3. Si no (puedes construir un contraejemplo) → mantener la ternaria.

La mayoría de relaciones "ternarias" que aparecen en diseño inicial son en realidad relaciones binarias que tienen una entidad "escondida" que actúa como tercer participante. Identifica esa entidad y modélala explícitamente.

---

## 8. Agregación

### 8.1 El problema que motiva la agregación

Hay situaciones donde necesitamos relacionar una entidad con una **relación** (no con otra entidad). El modelo ER básico no permite esto: las relaciones asocian entidades entre sí, no entidades con relaciones.

La **agregación** es el mecanismo del EER para tratar un tipo de relación (junto con sus entidades participantes) como si fuera una entidad de nivel superior, permitiendo que otras relaciones participen con ese conjunto.

### 8.2 El ejemplo clásico

Consideremos el dominio donde:
- Un `EMPLEADO` trabaja en un `PROYECTO`.
- La relación `TRABAJA_EN` tiene atributos: `horas_semanales`, `rol`.
- Un `GERENTE` supervisa la participación de un `EMPLEADO` en un `PROYECTO` específico.

La relación `SUPERVISA` no es entre `GERENTE` y `EMPLEADO`, ni entre `GERENTE` y `PROYECTO`. Es entre `GERENTE` y la **instancia de participación específica**: el hecho de que cierto empleado trabaje en cierto proyecto.

Sin agregación, esto es imposible de modelar correctamente. Con agregación, tratamos la relación `TRABAJA_EN` (junto con `EMPLEADO` y `PROYECTO`) como una entidad de nivel superior y establecemos `SUPERVISA` entre `GERENTE` y esa agregación.

### 8.3 Implementación en el esquema relacional

La agregación se traduce así:

```sql
-- Las entidades
CREATE TABLE empleados (empleado_id INT PRIMARY KEY, nombre VARCHAR(100));
CREATE TABLE proyectos (proyecto_id INT PRIMARY KEY, nombre VARCHAR(100));
CREATE TABLE gerentes (gerente_id INT PRIMARY KEY, nombre VARCHAR(100));

-- La relación base (agregación)
CREATE TABLE trabaja_en (
    empleado_id     INT NOT NULL REFERENCES empleados,
    proyecto_id     INT NOT NULL REFERENCES proyectos,
    horas_semanales INT NOT NULL,
    rol             VARCHAR(50),
    PRIMARY KEY (empleado_id, proyecto_id)
);

-- La relación que apunta a la agregación
CREATE TABLE supervisa (
    gerente_id   INT NOT NULL REFERENCES gerentes,
    empleado_id  INT NOT NULL,
    proyecto_id  INT NOT NULL,
    fecha_inicio DATE NOT NULL,
    FOREIGN KEY (empleado_id, proyecto_id) REFERENCES trabaja_en(empleado_id, proyecto_id),
    PRIMARY KEY (gerente_id, empleado_id, proyecto_id)
);
```

La clave foránea compuesta `(empleado_id, proyecto_id) → trabaja_en` es la forma de implementar la "relación con una relación" en SQL.

### 8.4 Cuándo necesitas agregación

La señal de que necesitas agregación es cuando un atributo de una relación en realidad "apunta" a otra entidad, no es un valor descriptivo. Si la relación `TRABAJA_EN` tiene un atributo `gerente_supervisor`, ese atributo no es un valor descriptivo: es una referencia a la entidad `GERENTE`. Eso es señal de que en realidad hay una relación entre `GERENTE` y la agregación de `TRABAJA_EN`.

---

## 9. Jerarquías de generalización y especialización

### 9.1 Los dos sentidos del mismo concepto

La **generalización** y la **especialización** son el mismo mecanismo visto desde direcciones opuestas:

- **Especialización (top-down):** Parto de un tipo de entidad genérico y lo desgloso en subtipos más específicos con atributos o comportamientos adicionales. Ejemplo: `EMPLEADO` se especializa en `EMPLEADO_TIEMPO_COMPLETO` y `EMPLEADO_CONTRATISTA`.

- **Generalización (bottom-up):** Identifico que varios tipos de entidad distintos comparten atributos comunes y creo un supertipo que los generaliza. Ejemplo: `CLIENTE_PERSONA` y `CLIENTE_EMPRESA` comparten `nombre`, `dirección`, `límite_crédito`, por lo que los generalizo en `CLIENTE`.

En ambos casos el resultado es una jerarquía con un supertipo arriba y subtipos abajo. Los subtipos **heredan todos los atributos del supertipo** y agregan los suyos propios.

### 9.2 Restricciones en las jerarquías

Las jerarquías tienen dos restricciones independientes que hay que especificar explícitamente:

**Restricción de membresía: total vs parcial**

- **Total:** Toda instancia del supertipo debe pertenecer a al menos uno de los subtipos. No puede haber un `EMPLEADO` que no sea ni `TIEMPO_COMPLETO` ni `CONTRATISTA`. Se indica con doble línea entre el supertipo y el triángulo.

- **Parcial:** Una instancia del supertipo puede no pertenecer a ningún subtipo. Puede haber una `PERSONA` que no es ni `CLIENTE` ni `EMPLEADO` en el sistema. Es el caso por defecto.

**Restricción de solapamiento: disjunto vs solapado**

- **Disjunto (exclusivo):** Una instancia del supertipo puede pertenecer a **como máximo un subtipo**. Un empleado es tiempo completo O contratista, nunca ambos. Se indica con la letra `d` en el triángulo.

- **Solapado (overlapping):** Una instancia del supertipo puede pertenecer a **múltiples subtipos simultáneamente**. Una persona puede ser al mismo tiempo `CLIENTE` y `EMPLEADO` de la empresa. Se indica con la letra `o` en el triángulo.

Esto produce cuatro combinaciones posibles:

| Membresía \ Solapamiento | Disjunto | Solapado |
|---|---|---|
| **Total** | Toda instancia pertenece a exactamente un subtipo | Toda instancia pertenece a al menos un subtipo |
| **Parcial** | Cada instancia pertenece a como máximo un subtipo | Cada instancia puede pertenecer a cero o más subtipos |

### 9.3 Estrategias de traducción a esquema relacional

Esta es la decisión de diseño más importante cuando hay jerarquías. Hay tres estrategias, cada una con trade-offs diferentes.

---

#### Estrategia 1: Una tabla por jerarquía (Table Per Hierarchy — TPH)

**Descripción:** Una sola tabla que incluye todos los atributos del supertipo y de todos los subtipos, más una columna discriminadora que indica a qué subtipo pertenece cada fila.

```sql
CREATE TABLE empleados (
    empleado_id     INT PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL,   -- atributo del supertipo
    fecha_contrato  DATE NOT NULL,           -- atributo del supertipo
    tipo_empleado   VARCHAR(20) NOT NULL     -- discriminador: 'TC' o 'CONTRATISTA'
        CHECK (tipo_empleado IN ('TC', 'CONTRATISTA')),
    
    -- Atributos solo de TIEMPO_COMPLETO:
    salario_mensual DECIMAL(10,2),   -- NULL si es contratista
    beneficios_plan VARCHAR(50),     -- NULL si es contratista
    
    -- Atributos solo de CONTRATISTA:
    tarifa_hora     DECIMAL(10,2),   -- NULL si es tiempo completo
    empresa_cliente VARCHAR(100),    -- NULL si es tiempo completo
    contrato_fin    DATE             -- NULL si es tiempo completo
);
```

**Ventajas:**
- Una sola tabla → queries simples, sin joins para recuperar toda la información de un empleado.
- Rendimiento en lectura: una sola operación de I/O.

**Desventajas:**
- **Muchos NULLs:** Los atributos específicos de cada subtipo son NULL para todos los demás. En una jerarquía grande con muchos subtipos, la mayoría de columnas son NULL para cualquier fila.
- **No se pueden imponer restricciones NOT NULL** en atributos específicos de subtipos (porque deben ser NULL para las instancias de los otros subtipos). La integridad se pierde.
- **Tablas muy anchas** con muchas columnas de significado heterogéneo.

**Cuándo usarla:**
- Jerarquías pequeñas (2-3 subtipos).
- Cuando los subtipos tienen pocos atributos propios.
- Cuando el rendimiento de lectura es crítico y los NULLs son manejables.

---

#### Estrategia 2: Una tabla por subtipo (Table Per Type — TPT)

**Descripción:** Una tabla para el supertipo con sus atributos comunes, y una tabla adicional por cada subtipo con sus atributos propios. Las tablas de subtipos tienen la misma clave primaria que el supertipo, y una clave foránea hacia él.

```sql
-- Supertipo
CREATE TABLE empleados (
    empleado_id    INT PRIMARY KEY,
    nombre         VARCHAR(100) NOT NULL,
    fecha_contrato DATE NOT NULL
);

-- Subtipo 1
CREATE TABLE empleados_tc (
    empleado_id     INT PRIMARY KEY REFERENCES empleados(empleado_id),
    salario_mensual DECIMAL(10,2) NOT NULL,
    beneficios_plan VARCHAR(50) NOT NULL
);

-- Subtipo 2
CREATE TABLE empleados_contratista (
    empleado_id     INT PRIMARY KEY REFERENCES empleados(empleado_id),
    tarifa_hora     DECIMAL(10,2) NOT NULL,
    empresa_cliente VARCHAR(100) NOT NULL,
    contrato_fin    DATE NOT NULL
);
```

**Ventajas:**
- **Sin NULLs:** Cada tabla tiene solo los atributos que le corresponden, todos `NOT NULL` cuando aplica.
- **Integridad completa:** Todas las restricciones pueden expresarse.
- **Normalizado:** No hay redundancia.

**Desventajas:**
- **Joins necesarios:** Para ver la información completa de un empleado, siempre hay que hacer join entre `empleados` y la tabla del subtipo correspondiente.
- **Joins variables:** La query cambia según el subtipo que se busca. Para buscar "todos los empleados con sus detalles" se necesita un `LEFT JOIN` a cada subtipo o `UNION`.
- **El discriminador no está en el modelo:** Para saber a qué subtipo pertenece un empleado, hay que verificar qué tabla subtipo tiene una fila para ese `empleado_id`. Esto puede requerir múltiples queries o joins.

**Cuándo usarla:**
- Cuando la integridad de datos es prioritaria.
- Cuando los subtipos tienen muchos atributos propios.
- Cuando las queries típicas trabajan sobre un subtipo específico, no sobre la jerarquía completa.

---

#### Estrategia 3: Una tabla por clase concreta (Table Per Concrete Class — TPC)

**Descripción:** Una tabla separada por cada **subtipo concreto**, que incluye tanto los atributos del supertipo como los propios. No hay tabla para el supertipo.

```sql
-- Solo tablas para los subtipos, con todos los atributos
CREATE TABLE empleados_tc (
    empleado_id     INT PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL,   -- heredado del supertipo
    fecha_contrato  DATE NOT NULL,           -- heredado del supertipo
    salario_mensual DECIMAL(10,2) NOT NULL,  -- propio
    beneficios_plan VARCHAR(50) NOT NULL     -- propio
);

CREATE TABLE empleados_contratista (
    empleado_id     INT PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL,   -- heredado (duplicado)
    fecha_contrato  DATE NOT NULL,           -- heredado (duplicado)
    tarifa_hora     DECIMAL(10,2) NOT NULL,  -- propio
    empresa_cliente VARCHAR(100) NOT NULL,   -- propio
    contrato_fin    DATE NOT NULL            -- propio
);
```

**Ventajas:**
- Cada subtipo tiene su propia tabla completa → queries sobre un subtipo específico son simples y rápidas.
- Sin NULLs.

**Desventajas:**
- **Redundancia severa:** Los atributos del supertipo se duplican en cada tabla de subtipo. Si cambia un atributo del supertipo, hay que actualizar múltiples tablas.
- **Imposible imponer unicidad global:** No puedes poner una restricción de que `empleado_id` sea único entre TODOS los empleados (TC y Contratistas), porque están en tablas distintas sin relación entre sí.
- **Queries sobre la jerarquía completa requieren UNION:**
  ```sql
  SELECT empleado_id, nombre FROM empleados_tc
  UNION
  SELECT empleado_id, nombre FROM empleados_contratista;
  ```
  Que no puede usar índices de forma eficiente.

**Cuándo usarla:**
- Raramente es la mejor opción.
- Puede tener sentido cuando los subtipos son completamente distintos y **nunca** se necesita consultarlos juntos como supertipo.

### 9.4 La jerarquía correcta depende del dominio

La elección entre TPH, TPT, y TPC no tiene una respuesta universalmente correcta. El criterio debe ser:

1. **¿Con qué frecuencia se consulta la jerarquía completa vs subtipos individuales?** → Muchos queries sobre toda la jerarquía favorecen TPH. Queries sobre subtipos específicos favorecen TPT o TPC.

2. **¿Qué tan importantes son las restricciones de integridad?** → Si necesitas NOT NULL en atributos de subtipos, TPT es la única opción correcta.

3. **¿Cuántos subtipos hay y cuántos atributos propios tienen?** → Jerarquía con muchos subtipos y muchos atributos propios → los NULLs de TPH se vuelven inaceptables.

4. **¿Los subtipos son disjuntos o solapados?** → Las jerarquías solapadas son muy difíciles de implementar con TPH (una columna discriminadora no funciona). TPT maneja solapamiento naturalmente (una instancia puede tener filas en múltiples tablas de subtipos).

---

## 10. De EER a esquema relacional: el proceso sistemático

### 10.1 El mapa de traducción

La traducción de un diagrama EER a un esquema relacional tiene reglas sistemáticas para cada elemento. No es intuitiva: es un proceso formal.

### Regla 1 — Tipos de entidad fuertes

**Cada tipo de entidad fuerte → una tabla.**

Los atributos del tipo de entidad se convierten en columnas. El atributo clave se convierte en clave primaria. Los atributos compuestos se descomponen en sus componentes atómicos. Los atributos derivados generalmente no se almacenan.

```
EER: CLIENTE(cliente_id*, nombre, dirección{calle, ciudad, país}, email)
     (* indica atributo clave)

SQL:
CREATE TABLE clientes (
    cliente_id  INT         PRIMARY KEY,
    nombre      VARCHAR(100) NOT NULL,
    calle       VARCHAR(200),
    ciudad      VARCHAR(100),
    pais        VARCHAR(100),
    email       VARCHAR(100)
);
```

### Regla 2 — Atributos multivaluados

**Cada atributo multivaluado → una nueva tabla.**

La nueva tabla tiene: la clave foránea al tipo de entidad original, el valor del atributo multivaluado, y la combinación de ambos forma la clave primaria (o puede haber una clave propia si hay más semántica).

```
EER: EMPLEADO con atributos multivaluados {teléfonos} y {títulos_académicos}

SQL:
CREATE TABLE telefonos_empleado (
    empleado_id INT  NOT NULL REFERENCES empleados(empleado_id),
    tipo        VARCHAR(20), -- 'celular', 'trabajo', 'casa'
    numero      VARCHAR(20)  NOT NULL,
    PRIMARY KEY (empleado_id, numero)
);

CREATE TABLE titulos_empleado (
    empleado_id   INT         NOT NULL REFERENCES empleados(empleado_id),
    titulo        VARCHAR(100) NOT NULL,
    institucion   VARCHAR(100),
    año_obtencion INT,
    PRIMARY KEY (empleado_id, titulo)
);
```

### Regla 3 — Tipos de entidad débiles

**Cada tipo de entidad débil → una tabla.**

La tabla incluye: la clave foránea al tipo de entidad propietaria (con ON DELETE CASCADE), el atributo discriminador, y cualquier otro atributo de la entidad débil. La clave primaria es la combinación de la clave foránea y el discriminador.

```
EER: LÍNEA_PEDIDO (débil) identificada por PEDIDO, discriminador: número_línea

SQL:
CREATE TABLE lineas_pedido (
    pedido_id    INT NOT NULL REFERENCES pedidos(pedido_id) ON DELETE CASCADE,
    numero_linea INT NOT NULL,
    producto_id  INT NOT NULL REFERENCES productos(producto_id),
    cantidad     INT NOT NULL,
    PRIMARY KEY (pedido_id, numero_linea)
);
```

### Regla 4 — Relaciones 1:1

**Para relaciones 1:1 hay tres opciones**, dependiendo de la participación:

**Opción A (participación total de un lado):** Incluir la clave foránea en la tabla del lado con participación total.

```
EER: EMPLEADO (0,1) — dirige — (1,1) DEPARTAMENTO
    (Todo departamento tiene exactamente un director, un empleado dirige a lo sumo un depto)

SQL: Agregar la clave foránea en DEPARTAMENTO (lado con participación total):
ALTER TABLE departamentos ADD COLUMN director_id INT REFERENCES empleados(empleado_id);
-- director_id NOT NULL porque la participación es total en DEPARTAMENTO
```

**Opción B (participación parcial de ambos lados):** Incluir la clave foránea en cualquiera de los lados (elegir el que minimice NULLs), con NULL permitido.

**Opción C (ambas tablas unidas):** Si la participación es total en ambos lados (toda instancia de A tiene una instancia de B y viceversa), se puede considerar fusionar ambas tablas en una. Pero esto raramente se justifica porque hace el modelo menos flexible a cambios futuros.

### Regla 5 — Relaciones 1:N

**Para relaciones 1:N:** La clave primaria del lado "1" se convierte en clave foránea en la tabla del lado "N".

```
EER: DEPARTAMENTO (1,1) — tiene — (0,N) EMPLEADO
    (Un departamento tiene muchos empleados; un empleado pertenece a exactamente un depto)

SQL: Agregar clave foránea en EMPLEADO (el lado N):
ALTER TABLE empleados ADD COLUMN departamento_id INT NOT NULL
    REFERENCES departamentos(departamento_id);
-- NOT NULL porque la participación de EMPLEADO es total (mínimo 1)
```

Si la relación tiene atributos propios, estos también van en la tabla del lado "N".

### Regla 6 — Relaciones M:N

**Para relaciones M:N:** Crear una nueva tabla de relación (tabla de unión/junction) con las claves primarias de ambas entidades como claves foráneas. La clave primaria de la tabla de relación es la combinación de ambas claves foráneas (salvo que haya razones para una clave sustituta). Los atributos de la relación también van aquí.

```
EER: EMPLEADO (0,N) — asignado_a — (1,N) PROYECTO
    Atributos de la relación: horas_semanales, rol, fecha_inicio

SQL:
CREATE TABLE asignaciones (
    empleado_id     INT NOT NULL REFERENCES empleados(empleado_id),
    proyecto_id     INT NOT NULL REFERENCES proyectos(proyecto_id),
    horas_semanales INT NOT NULL,
    rol             VARCHAR(50),
    fecha_inicio    DATE NOT NULL,
    PRIMARY KEY (empleado_id, proyecto_id)
);
```

### Regla 7 — Relaciones n-arias

**Para relaciones n-arias:** Crear una tabla con las claves foráneas de todas las entidades participantes. La clave primaria es generalmente la combinación de todas las claves foráneas (con ajustes según la cardinalidad).

```
EER: PROVEEDOR — SUMINISTRA — PRODUCTO — PROYECTO (ternaria)

SQL:
CREATE TABLE suministros (
    proveedor_id INT NOT NULL REFERENCES proveedores,
    producto_id  INT NOT NULL REFERENCES productos,
    proyecto_id  INT NOT NULL REFERENCES proyectos,
    cantidad     INT NOT NULL,
    PRIMARY KEY (proveedor_id, producto_id, proyecto_id)
);
```

### Regla 8 — Jerarquías de especialización

Aplicar la estrategia adecuada (TPH, TPT, o TPC) según los criterios del dominio discutidos en la sección anterior.

---

## 11. Errores de modelado que se vuelven invisibles

### 11.1 El problema de los errores de semántica

Un error sintáctico (una tabla mal creada) falla inmediatamente. Un error semántico (una tabla que no refleja el dominio correctamente) puede pasar desapercibido durante meses o años, acumulando datos incorrectos que son muy difíciles de corregir retroactivamente.

### Error 1: Confundir una relación con una entidad

**Síntoma:** Una "entidad" que no tiene identidad propia más allá de las entidades que conecta, y sus únicos atributos son referencias a otras entidades.

```
-- Incorrecto: "Pertenencia" como entidad cuando es simplemente una relación
PERTENENCIA(pertenencia_id, empleado_id, departamento_id, fecha_ingreso)

-- Correcto: es una relación con un atributo propio
EMPLEADO — pertenece_a (con atributo fecha_ingreso) — DEPARTAMENTO
```

El `pertenencia_id` no aporta nada: no hay ningún proceso del negocio que necesite referenciar una "pertenencia" específica de forma independiente. Es una relación, no una entidad.

**Cuándo sí es correcto tener una entidad de "pertenencia":** Cuando hay procesos del negocio que necesitan referenciar el hecho de la pertenencia de forma independiente, como "historial de pertenencias" donde una pertenencia puede tener fecha_inicio, fecha_fin, motivo_cambio, y múltiples eventos asociados. En ese caso, la pertenencia sí merece ser entidad.

### Error 2: Cardinalidad incorrecta por pensar en los datos actuales

**Síntoma:** El diseño funciona perfectamente para los datos de hoy, pero cuando el negocio evoluciona (lo que siempre ocurre), requiere una migración costosa.

```
-- Error: modelar como 1:N porque "actualmente" cada empleado tiene un solo proyecto
EMPLEADOS tiene proyecto_id (FK a PROYECTOS)

-- Cuando el negocio decide que los empleados pueden participar en múltiples proyectos,
-- hay que migrar a una tabla de relación M:N. Si ya hay millones de registros, eso duele.

-- Correcto desde el inicio: preguntar "¿podría alguna vez un empleado
-- participar en más de un proyecto?" Si la respuesta es "quizás en el futuro",
-- modelar M:N desde el principio.
```

### Error 3: Atributos que son realmente entidades

**Síntoma:** Un "atributo" que empieza a acumular estructura propia con el tiempo, hasta que hay que migrar para convertirlo en entidad.

```
-- Incorrecto:
PRODUCTOS(producto_id, nombre, precio, categoria VARCHAR(50))

-- El día que necesites "color de etiqueta por categoría" o "tasa de impuesto por categoría"
-- o "descripción de la categoría", tienes que hacer una migración.

-- Correcto desde el inicio: si la categoría tiene significado propio en el dominio,
-- es una entidad.
CATEGORIAS(categoria_id, nombre, descripcion, tasa_impuesto, color_etiqueta)
PRODUCTOS(producto_id, nombre, precio, categoria_id FK → CATEGORIAS)
```

### Error 4: Perder la semántica de la participación total

**Síntoma:** Columnas que deberían ser `NOT NULL` se dejan con NULL porque "es más fácil", perdiendo la garantía de integridad.

```
-- Error: si todo pedido DEBE tener un cliente (participación total de PEDIDO en la relación)
-- pero la columna se deja nullable
CREATE TABLE pedidos (
    pedido_id  INT PRIMARY KEY,
    cliente_id INT,   -- ERROR: debería ser NOT NULL
    fecha      DATE NOT NULL
);

-- Correcto:
CREATE TABLE pedidos (
    pedido_id  INT PRIMARY KEY,
    cliente_id INT NOT NULL REFERENCES clientes(cliente_id),
    fecha      DATE NOT NULL
);
```

### Error 5: Modelar jerarquías con columnas booleanas múltiples

**Síntoma:** En lugar de una jerarquía de especialización, se usan múltiples columnas booleanas que intentan capturar el tipo de entidad.

```
-- Error: "flags" de tipo en lugar de jerarquía
CREATE TABLE personas (
    persona_id  INT PRIMARY KEY,
    nombre      VARCHAR(100),
    es_cliente  BOOLEAN,
    es_empleado BOOLEAN,
    es_proveedor BOOLEAN,
    -- atributos de cliente:
    limite_credito DECIMAL,
    -- atributos de empleado:
    cargo VARCHAR(50),
    salario DECIMAL,
    -- atributos de proveedor:
    condiciones_pago VARCHAR(100)
    -- La tabla crece indefinidamente con cada nuevo "rol"
);
```

El problema: NULLs por todas partes, no hay restricciones de integridad, y agregar un nuevo "rol" requiere alterar la tabla.

```sql
-- Correcto: jerarquía con TPT
CREATE TABLE personas (persona_id INT PRIMARY KEY, nombre VARCHAR(100));

CREATE TABLE clientes (
    persona_id     INT PRIMARY KEY REFERENCES personas,
    limite_credito DECIMAL NOT NULL
);

CREATE TABLE empleados (
    persona_id INT PRIMARY KEY REFERENCES personas,
    cargo      VARCHAR(50) NOT NULL,
    salario    DECIMAL NOT NULL
);

CREATE TABLE proveedores (
    persona_id        INT PRIMARY KEY REFERENCES personas,
    condiciones_pago  VARCHAR(100) NOT NULL
);
```

### Error 6: Ignorar la diferencia entre relaciones binarias y ternarias

Como vimos en la sección 7, descomponer una ternaria irreducible en binarias produce instancias espurias. El error contrario (no descomponer una ternaria reducible) genera redundancia innecesaria.

El test de si generar instancias espurias es posible es el que determina qué hacer.

### Error 7: Claves que dependen de reglas externas no garantizadas

**Síntoma:** Usar como clave primaria un identificador externo cuya unicidad depende de un sistema o institución externa.

```
-- Riesgoso:
CREATE TABLE empresas (rut VARCHAR(20) PRIMARY KEY, nombre VARCHAR(200));

-- Problemas: El formato del RUT puede cambiar. Puede haber errores de entrada.
-- En algunos países, los RUTs se reasignan en circunstancias específicas.

-- Más robusto: clave sustituta interna + unicidad del RUT
CREATE TABLE empresas (
    empresa_id INT PRIMARY KEY,   -- clave sustituta, nunca cambia
    rut        VARCHAR(20) UNIQUE NOT NULL,  -- el RUT sigue siendo único, pero no es la PK
    nombre     VARCHAR(200) NOT NULL
);
```

---

## 12. Un caso completo de modelado

### 12.1 El dominio: sistema de gestión de un hospital

Modelaremos un subconjunto de un sistema hospitalario para ilustrar todos los conceptos del módulo integrados.

**Descripción del dominio:**

1. El hospital tiene varias **áreas** (urgencias, cirugía, pediatría, etc.), cada una con nombre, piso y jefe de área.
2. Los **médicos** tienen número de colegiatura, nombre, especialidad y pueden trabajar en múltiples áreas.
3. Los **pacientes** tienen número de historia clínica, nombre, fecha de nacimiento y datos de contacto.
4. Las **consultas** representan la atención de un médico a un paciente en una fecha específica, con diagnóstico y tratamiento indicado.
5. Cada consulta genera una o varias **prescripciones** de medicamentos, con dosis y duración.
6. Los **medicamentos** tienen nombre comercial, principio activo y fabricante.
7. Un médico puede ser **jefe de área** de a lo sumo un área.

### 12.2 Identificación de entidades

| Entidad | Tipo | Justificación |
|---|---|---|
| ÁREA | Fuerte | Tiene atributos propios, vida independiente |
| MÉDICO | Fuerte | Tiene atributos propios, vida independiente |
| PACIENTE | Fuerte | Tiene atributos propios, vida independiente |
| CONSULTA | Fuerte* | Podría ser débil respecto al paciente, pero tiene vida propia (puede ser consultada, facturada) |
| PRESCRIPCIÓN | Débil | No existe sin CONSULTA, se identifica por consulta + número |
| MEDICAMENTO | Fuerte | Tiene vida independiente, se referencia desde prescripciones |

*La decisión de si CONSULTA es entidad fuerte o débil depende del dominio: si el sistema necesita hablar de "la consulta C001" de forma independiente al paciente (p.ej. para facturación), es fuerte. Si solo tiene sentido como "la segunda consulta del paciente P005", es débil.

### 12.3 Identificación de relaciones y cardinalidades

```
MÉDICO — trabaja_en — ÁREA
  (0,N) : (0,N)  →  M:N
  Un médico puede trabajar en múltiples áreas; un área puede tener múltiples médicos.
  Atributo de la relación: fecha_inicio_en_área

MÉDICO — jefe_de — ÁREA
  (0,1) : (0,1)  →  1:1 parcial-parcial
  Un médico puede ser jefe de a lo sumo un área.
  Un área puede tener a lo sumo un jefe (puede estar vacante).
  ↳ Implementación: FK jefe_medico_id en AREA (nullable)

MÉDICO — realiza — CONSULTA
  (1,N) : (1,1)  →  1:N
  Un médico realiza muchas consultas; cada consulta la realiza exactamente un médico.

PACIENTE — recibe — CONSULTA
  (1,N) : (1,1)  →  1:N
  Un paciente puede tener muchas consultas; cada consulta es de un paciente.

CONSULTA — genera — PRESCRIPCIÓN
  (0,N) : (1,1)  →  1:N con PRESCRIPCIÓN como débil
  Una consulta puede generar cero o más prescripciones.
  Cada prescripción pertenece a exactamente una consulta.

PRESCRIPCIÓN — indica — MEDICAMENTO
  (1,N) : (1,1)  →  1:N
  Un medicamento puede aparecer en muchas prescripciones.
  Cada prescripción indica exactamente un medicamento.
```

### 12.4 El esquema relacional resultante

```sql
-- Entidades fuertes base

CREATE TABLE areas (
    area_id      INT          PRIMARY KEY,
    nombre       VARCHAR(100) NOT NULL UNIQUE,
    piso         INT          NOT NULL,
    jefe_id      INT          REFERENCES medicos(medico_id)  -- FK nullable (1:1 parcial)
    -- jefe_id se agrega después de crear médicos (restricción de orden)
);

CREATE TABLE medicos (
    medico_id       INT          PRIMARY KEY,
    num_colegiatura VARCHAR(20)  NOT NULL UNIQUE,  -- clave candidata alternativa
    nombre          VARCHAR(100) NOT NULL,
    especialidad    VARCHAR(100) NOT NULL
);

-- Agregar FK de jefe_id ahora que médicos existe
ALTER TABLE areas ADD FOREIGN KEY (jefe_id) REFERENCES medicos(medico_id);

CREATE TABLE pacientes (
    paciente_id     INT          PRIMARY KEY,
    num_historia    VARCHAR(20)  NOT NULL UNIQUE,
    nombre          VARCHAR(100) NOT NULL,
    fecha_nac       DATE         NOT NULL,
    telefono        VARCHAR(20),
    email           VARCHAR(100)
);

CREATE TABLE medicamentos (
    medicamento_id  INT          PRIMARY KEY,
    nombre_comercial VARCHAR(100) NOT NULL,
    principio_activo VARCHAR(100) NOT NULL,
    fabricante       VARCHAR(100) NOT NULL
);

-- Relación M:N: médico trabaja en área

CREATE TABLE medico_area (
    medico_id        INT  NOT NULL REFERENCES medicos(medico_id),
    area_id          INT  NOT NULL REFERENCES areas(area_id),
    fecha_inicio     DATE NOT NULL,
    PRIMARY KEY (medico_id, area_id)
);

-- Entidad fuerte: consulta (1:N con médico y paciente)

CREATE TABLE consultas (
    consulta_id   INT          PRIMARY KEY,
    medico_id     INT          NOT NULL REFERENCES medicos(medico_id),
    paciente_id   INT          NOT NULL REFERENCES pacientes(paciente_id),
    fecha_hora    TIMESTAMP    NOT NULL,
    diagnostico   TEXT,
    tratamiento   TEXT,
    area_id       INT          REFERENCES areas(area_id)  -- área donde se realizó
);

-- Entidad débil: prescripción (depende de consulta)

CREATE TABLE prescripciones (
    consulta_id    INT          NOT NULL REFERENCES consultas(consulta_id) ON DELETE CASCADE,
    num_prescr     INT          NOT NULL,    -- discriminador
    medicamento_id INT          NOT NULL REFERENCES medicamentos(medicamento_id),
    dosis          VARCHAR(100) NOT NULL,
    duracion_dias  INT          NOT NULL CHECK (duracion_dias > 0),
    indicaciones   TEXT,
    PRIMARY KEY (consulta_id, num_prescr)
);
```

### 12.5 Decisiones de diseño explícitas

Documentar estas decisiones es parte del trabajo de modelado:

1. **CONSULTA como entidad fuerte:** Se eligió porque el sistema necesita referenciar consultas desde facturación, historiales y reportes de forma independiente al paciente.

2. **jefe_id en AREAS:** La relación 1:1 "jefe_de" se implementa como FK nullable en AREAS, no en MÉDICOS, porque el área es la entidad que "tiene" un jefe (semánticamente). Un médico existe independientemente de ser o no jefe.

3. **PRESCRIPCIÓN como entidad débil:** Tiene sentido porque fuera del contexto de su consulta, una prescripción no tiene identidad. "Prescripción número 2" solo tiene significado como "prescripción número 2 de la consulta C045".

4. **num_historia en PACIENTES como UNIQUE NOT NULL:** Es una clave candidata alternativa. La clave primaria es `paciente_id` (sustituta) para estabilidad, pero `num_historia` mantiene la unicidad semántica del dominio.

---

## 13. Resumen y conexión con el resto del curso

### Lo que construimos en este módulo

El modelado conceptual con EER provee un lenguaje formal para capturar la estructura de un dominio **antes** de comprometerse con una implementación. Sus elementos clave:

- **Tipos de entidad** con su distinción entre fuertes y débiles, y la semántica precisa de cada uno.
- **Atributos** en toda su taxonomía: simples, compuestos, multivaluados, derivados. Cada uno con una estrategia de traducción diferente.
- **Tipos de relación** con cardinalidades `(min, max)` que capturan restricciones del dominio que la notación básica 1:N pierde.
- **Relaciones n-arias** y el test formal de irreducibilidad para saber cuándo una ternaria no puede descomponerse en binarias sin generar instancias espurias.
- **Agregación** para los casos donde una relación necesita participar en otra relación.
- **Jerarquías EER** con las cuatro combinaciones de restricciones (total/parcial × disjunto/solapado) y las tres estrategias de implementación con sus trade-offs precisos.
- **El proceso sistemático** de traducción de EER a esquema relacional con reglas explícitas para cada elemento.

### Conexión con módulos futuros

- **Módulo 4 (Integridad y restricciones):** Las restricciones que emergen del modelo EER — `NOT NULL` para participación total, `UNIQUE` para claves candidatas, `FOREIGN KEY` para relaciones, `CHECK` para restricciones de dominio — son exactamente el tema del siguiente módulo.

- **Módulo 5 (Patrones de modelado):** Los patrones como "Party" y "roles" son extensiones de las jerarquías EER aplicadas a dominios específicos y recurrentes.

- **Módulo 6 (Diseño para el tiempo):** Modelar tiempo válido y tiempo de transacción requiere extender el EER con conceptos de temporalidad que cambian la estructura de las entidades y relaciones.

- **Módulo 15 (ORMs):** La "impedancia objeto-relacional" es en parte la dificultad de mapear jerarquías EER (especialmente las solapadas) al modelo de objetos. Las tres estrategias TPH/TPT/TPC tienen nombres y consecuencias específicas en ORMs como Hibernate, ActiveRecord, o SQLAlchemy.

---

## Ejercicios de comprensión

**Ejercicio 1.** Para el siguiente enunciado, construye el diagrama EER textual (lista de entidades, atributos, relaciones con cardinalidades):

*"Una universidad tiene facultades, cada una con nombre y decano. Cada facultad ofrece programas académicos. Los programas tienen código, nombre y número de créditos requeridos. Los estudiantes se matriculan en un programa y pueden tomar cursos de distintos programas. Cada curso tiene un código, nombre y créditos, y es ofrecido por un departamento (que puede ser de otra facultad). Los profesores pertenecen a un departamento y pueden dictar múltiples cursos en un semestre. Cada dictado de un curso en un semestre tiene un aula y horario asignados."*

**Ejercicio 2.** La siguiente relación ternaria: `ACTOR — ACTÚA_EN — PELÍCULA — DIRECTOR`. Bajo la semántica "un actor actúa en una película bajo la dirección de un director específico (un director puede tener actores favoritos que usa en múltiples películas)": ¿Es esta ternaria reducible a tres binarias o no? Construye el contraejemplo o la prueba de reducibilidad.

**Ejercicio 3.** Dado el siguiente esquema (producido por un modelado descuidado):
```sql
CREATE TABLE contratos (
    contrato_id     INT PRIMARY KEY,
    empleado_id     INT,
    empleado_nombre VARCHAR(100),
    empleado_cargo  VARCHAR(50),
    cliente_id      INT,
    cliente_nombre  VARCHAR(100),
    cliente_ciudad  VARCHAR(100),
    valor_contrato  DECIMAL(12,2),
    fecha_inicio    DATE,
    fecha_fin       DATE,
    tipo_contrato   VARCHAR(20),
    descuento_tc    DECIMAL(5,2),  -- solo aplica si tipo = 'CORPORATIVO'
    referido_por    VARCHAR(100)   -- solo aplica si tipo = 'REFERIDO'
);
```
a) Identifica todos los problemas de modelado.
b) Propón el modelo EER correcto.
c) Traduce a esquema relacional normalizado.

**Ejercicio 4.** Para la jerarquía `VEHÍCULO → {AUTOMÓVIL, MOTOCICLETA, CAMIÓN}` con las siguientes características:
- Todo vehículo tiene: placa, año, color, propietario_id.
- AUTOMÓVIL agrega: número_puertas, tipo_combustible.
- MOTOCICLETA agrega: cilindrada, tipo (deportiva/urbana/todoterreno).
- CAMIÓN agrega: capacidad_carga_toneladas, número_ejes.

¿Qué estrategia de jerarquía (TPH, TPT, TPC) elegirías si el sistema principal consulta con frecuencia "todos los vehículos del propietario X"? ¿Y si las consultas son casi siempre del tipo "todos los camiones disponibles para carga"? Justifica ambas decisiones.

**Ejercicio 5.** Identifica si la siguiente frase describe una entidad débil o una entidad fuerte con dependencia existencial, y justifica:

> *"En el sistema de un banco, una 'cuenta de ahorros' no puede existir sin un 'titular'. Si el titular cierra todas sus cuentas y se da de baja del banco, las cuentas se cierran también."*

---

*Próximo módulo: Integridad y restricciones como primera clase — donde todo lo que el modelo EER prometió (participaciones totales, cardinalidades máximas, restricciones de dominio) debe implementarse de forma robusta y declarativa en el esquema relacional.*
