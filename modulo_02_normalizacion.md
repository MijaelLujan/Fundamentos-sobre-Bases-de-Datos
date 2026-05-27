# Módulo 2 — Dependencias funcionales y normalización real

> *"Las formas normales no son recetas, son consecuencias de entender dependencias."*

---

## Tabla de contenidos

1. [El problema que resuelve la normalización](#1-el-problema-que-resuelve-la-normalización)
2. [Dependencias funcionales](#2-dependencias-funcionales)
3. [Los axiomas de Armstrong](#3-los-axiomas-de-armstrong)
4. [Cierre de un conjunto de atributos](#4-cierre-de-un-conjunto-de-atributos)
5. [Cobertura mínima](#5-cobertura-mínima)
6. [Claves candidatas vs claves primarias](#6-claves-candidatas-vs-claves-primarias)
7. [Primera Forma Normal (1FN)](#7-primera-forma-normal-1fn)
8. [Segunda Forma Normal (2FN)](#8-segunda-forma-normal-2fn)
9. [Tercera Forma Normal (3FN)](#9-tercera-forma-normal-3fn)
10. [Forma Normal de Boyce-Codd (FNBC)](#10-forma-normal-de-boyce-codd-fnbc)
11. [Cuarta Forma Normal (4FN) — Dependencias multivaluadas](#11-cuarta-forma-normal-4fn--dependencias-multivaluadas)
12. [Quinta Forma Normal (5FN) — Dependencias de join](#12-quinta-forma-normal-5fn--dependencias-de-join)
13. [Forma Normal de Domain/Key (FNDK)](#13-forma-normal-de-domainkey-fndk)
14. [¿Siempre hay que normalizar al máximo?](#14-siempre-hay-que-normalizar-al-máximo)
15. [Resumen y conexión con el resto del curso](#15-resumen-y-conexión-con-el-resto-del-curso)

---

## 1. El problema que resuelve la normalización

Antes de ver cualquier definición formal, necesitamos entender visceralmente el problema. La normalización existe para resolver algo muy concreto: las **anomalías de actualización**.

### 1.1 Un ejemplo que duele

Imagina que alguien diseñó la siguiente tabla para registrar los pedidos de una empresa:

```
PedidosCompletos
┌────────────┬────────────┬──────────────────┬────────────────┬────────────────┬──────────────┬──────────┬────────────┐
│ pedido_id  │ cliente_id │ cliente_nombre   │ cliente_ciudad │ producto_id    │ producto_nom │ cantidad │ precio_u   │
├────────────┼────────────┼──────────────────┼────────────────┼────────────────┼──────────────┼──────────┼────────────┤
│ P001       │ C01        │ Ferretería López │ Bogotá         │ PROD-A         │ Tornillo M6  │ 500      │ 0.05       │
│ P001       │ C01        │ Ferretería López │ Bogotá         │ PROD-B         │ Tuerca M6    │ 500      │ 0.03       │
│ P002       │ C01        │ Ferretería López │ Bogotá         │ PROD-A         │ Tornillo M6  │ 200      │ 0.05       │
│ P003       │ C02        │ Constructora ABC │ Medellín       │ PROD-C         │ Clavo 2"     │ 1000     │ 0.02       │
└────────────┴────────────┴──────────────────┴────────────────┴────────────────┴──────────────┴──────────┴────────────┘
```

A primera vista parece razonable. Tiene todo lo que necesitas para reportar un pedido. Pero tiene problemas graves:

**Anomalía de actualización:** La Ferretería López se muda de Bogotá a Cali. ¿Cuántas filas hay que actualizar? Todas las que contengan `C01`. Si se actualiza solo algunas (por error, por concurrencia, por bug), la base de datos queda en un estado **inconsistente**: el mismo cliente tiene dos ciudades distintas.

**Anomalía de inserción:** Quiero registrar un nuevo cliente (C03, "Distribuidora Norte", Barranquilla) antes de que haga su primer pedido. No puedo: la tabla no tiene sentido sin un pedido y un producto. El cliente no existe en el sistema hasta que compra algo. Eso es absurdo desde el punto de vista del negocio.

**Anomalía de borrado:** Se cancela el pedido P003 y se borra esa fila. Con esa operación desaparece también toda la información sobre PROD-C (Clavo 2"). Si era el único pedido de ese producto, el producto deja de existir en el sistema. Borraste un pedido pero destruiste un producto.

Estos tres tipos de anomalías — de actualización, de inserción y de borrado — son los enemigos que la normalización combate. Y todos tienen la misma causa raíz: **datos de distintos conceptos mezclados en la misma tabla**.

La normalización es el proceso formal de separar esos conceptos de manera correcta y sin pérdida de información.

### 1.2 La solución intuitiva vs la solución formal

Ante el ejemplo anterior, cualquier desarrollador con experiencia diría: "separa clientes, productos y pedidos en tablas distintas". Y estaría en lo correcto. Pero la intuición tiene límites:

- ¿Cómo sabes que ya terminaste de separar? ¿Cuándo un esquema está "suficientemente separado"?
- ¿Qué pasa cuando la situación es más sutil que "obviamente hay dos entidades mezcladas"?
- ¿Cómo garantizas que la separación no pierde información (que puedes reconstruir los datos originales)?

Aquí es donde la teoría formal de dependencias funcionales y formas normales nos da respuestas precisas. Las formas normales no son reglas arbitrarias: son **consecuencias lógicas** de las propiedades matemáticas de las dependencias funcionales en un esquema dado.

---

## 2. Dependencias funcionales

### 2.1 La definición formal

Dada una relación `R` con atributos, una **dependencia funcional** (DF) se escribe:

```
X → Y
```

y se lee: **"X determina funcionalmente a Y"** o **"Y depende funcionalmente de X"**.

Significa: para cualquier dos tuplas de R, si tienen el mismo valor en los atributos `X`, entonces **necesariamente** tienen el mismo valor en los atributos `Y`.

Formalmente: para toda instancia válida de R, para todo par de tuplas t₁ y t₂:
```
si t₁[X] = t₂[X], entonces t₁[Y] = t₂[Y]
```

### 2.2 Qué significa "determina" en el mundo real

Una dependencia funcional no es una observación estadística sobre los datos actuales. Es una **restricción semántica del dominio del problema**.

Ejemplo incorrecto de razonar sobre DFs:
> "Veo que todos los empleados de la ciudad 'Bogotá' tienen código de área '601'. Por lo tanto `ciudad → código_área`."

Eso puede ser verdad hoy pero no es necesariamente una DF válida: podría haber una ciudad con dos códigos de área, o el código podría cambiar. Una DF debe ser verdad para **todas las instancias posibles**, no solo para los datos que tenemos ahora.

Ejemplo correcto:
> "Por definición del negocio, un número de RUT (identificador tributario) identifica exactamente a una empresa. Por lo tanto `rut → nombre_empresa`, `rut → dirección_fiscal`, etc."

Esta es una restricción del mundo real que podemos garantizar que se cumplirá siempre, porque el sistema tributario garantiza la unicidad del RUT.

### 2.3 Tipos de dependencias funcionales

**Dependencia trivial:**

`X → Y` es trivial si `Y ⊆ X`. Es decir, Y es un subconjunto de X.

Ejemplo: `{nombre, ciudad} → nombre` es trivial. Siempre es verdad porque si dos tuplas tienen el mismo `nombre` y `ciudad`, trivialmente tienen el mismo `nombre`.

Las dependencias triviales no aportan información útil sobre la estructura del esquema.

**Dependencia no trivial:**

`X → Y` es no trivial si `Y ⊄ X`. Al menos algún atributo de Y no está en X.

Ejemplo: `cliente_id → cliente_nombre` es no trivial. El nombre no está "incluido" en el id; es una información adicional que el id determina.

**Dependencia completamente no trivial:**

`X → Y` es completamente no trivial si `X ∩ Y = ∅`. Ningún atributo de Y aparece en X.

**Dependencia parcial:**

`X → Y` es una dependencia parcial si existe `Z ⊂ X` (subconjunto propio de X) tal que `Z → Y`. Esto significa que no necesitamos todos los atributos de X para determinar Y; un subconjunto ya lo hace.

Ejemplo: En la relación `(pedido_id, producto_id, producto_nombre, cantidad)`, la dependencia `{pedido_id, producto_id} → producto_nombre` es parcial porque `producto_id → producto_nombre` ya funciona sola.

**Dependencia transitiva:**

`X → Y` es transitiva (respecto a una clave K) si existe un conjunto de atributos Z tal que:
- `K → Z`
- `Z → Y`
- `Z` no determina a `K` (Z no es clave candidata)
- `Y` no está en `Z`

Ejemplo: Si `pedido_id → cliente_id` y `cliente_id → cliente_ciudad`, entonces `pedido_id → cliente_ciudad` es una dependencia transitiva.

### 2.4 Cómo descubrir dependencias funcionales en un dominio

Las DFs se descubren analizando el **significado de los datos**, no los datos en sí. Las preguntas útiles son:

- **"¿Este valor puede cambiar independientemente de aquel?"** Si la ciudad de un cliente puede cambiar sin que cambie el cliente, no hay DF `ciudad → cliente`. Pero si el código postal de una ciudad nunca puede cambiar (definición externa), entonces `código_postal → ciudad`.

- **"¿Dado este valor, está determinado unívocamente aquel?"** Dado el `número_de_factura`, ¿está determinada la `fecha_de_emisión`? Sí: una factura tiene una sola fecha. Dado el `producto_id`, ¿está determinado el `precio`? Depende del negocio: ¿el precio varía por cliente o es único por producto?

- **"¿Puede el mismo valor de X corresponder a dos valores distintos de Y?"** Si sí, no hay DF. Si no, hay DF.

### 2.5 Ejemplos de DFs en contextos reales

**Sistema de recursos humanos:**
```
empleado_id → nombre, fecha_nacimiento, número_seguro_social
número_seguro_social → empleado_id, nombre  (si el SS es único por persona)
departamento_id → nombre_departamento, presupuesto_anual, gerente_id
{empleado_id, proyecto_id} → horas_asignadas, fecha_inicio_asignación
```

**Sistema de ventas:**
```
factura_id → fecha, cliente_id, vendedor_id
cliente_id → nombre_cliente, dirección, límite_crédito
{factura_id, producto_id} → cantidad, precio_unitario_aplicado
producto_id → descripción, precio_lista, categoría_id
```

**Sistema académico:**
```
estudiante_id → nombre, fecha_nacimiento, carrera_id
carrera_id → nombre_carrera, facultad_id
{estudiante_id, materia_id, período} → nota_final
materia_id → nombre_materia, créditos, departamento_id
{profesor_id, materia_id, período} → aula, horario
```

---

## 3. Los axiomas de Armstrong

Los axiomas de Armstrong son reglas de inferencia que permiten **derivar todas las dependencias funcionales** que se cumplen en un esquema dado a partir de un conjunto inicial de DFs. Son **completos** (pueden derivar todas las DFs válidas) y **correctos** (solo derivan DFs que son válidas).

Hay tres axiomas base y tres reglas derivadas.

### 3.1 Axiomas base

**Axioma 1 — Reflexividad:**
```
Si Y ⊆ X, entonces X → Y
```
Cualquier conjunto de atributos determina a cualquiera de sus subconjuntos. Genera las dependencias triviales.

Ejemplo: `{nombre, ciudad} → ciudad` ✓ (porque `ciudad ⊆ {nombre, ciudad}`)

**Axioma 2 — Aumentatividad (Augmentation):**
```
Si X → Y, entonces XZ → YZ  (para cualquier Z)
```
Si agregar los mismos atributos a ambos lados no rompe la dependencia.

Ejemplo: Si `cliente_id → ciudad`, entonces `{cliente_id, fecha} → {ciudad, fecha}`.

Intuitivamente: si dos tuplas coinciden en `cliente_id` y `fecha`, entonces ya coinciden en `cliente_id`, por lo que coinciden en `ciudad`, y como también coinciden en `fecha`, coinciden en `{ciudad, fecha}`.

**Axioma 3 — Transitividad:**
```
Si X → Y  e  Y → Z, entonces X → Z
```
Si X determina Y, y Y determina Z, entonces X determina Z.

Ejemplo: Si `pedido_id → cliente_id` y `cliente_id → ciudad`, entonces `pedido_id → ciudad`.

### 3.2 Reglas derivadas

Las siguientes reglas se pueden **demostrar** a partir de los tres axiomas base, pero son útiles tenerlas explícitas:

**Regla 1 — Unión:**
```
Si X → Y  e  X → Z, entonces X → YZ
```
Si X determina Y, y también determina Z, entonces determina a ambos juntos.

*Demostración:*
1. `X → Y` (dado)
2. `X → Z` (dado)
3. `X → XZ` (aumentatividad de 2 con X: `X → Z` implica `XX → XZ`, y `XX = X`)
4. `XY → YZ` (aumentatividad de 1 con Z)
5. `X → YZ` (transitividad de 3 y 4)

**Regla 2 — Descomposición:**
```
Si X → YZ, entonces X → Y  e  X → Z
```
Una dependencia a un conjunto puede descomponerse.

*Demostración:*
1. `X → YZ` (dado)
2. `YZ → Y` (reflexividad, ya que `Y ⊆ YZ`)
3. `X → Y` (transitividad de 1 y 2)
(análogamente para Z)

**Regla 3 — Pseudotransitividad:**
```
Si X → Y  e  WY → Z, entonces WX → Z
```

*Demostración:*
1. `X → Y` (dado)
2. `WX → WY` (aumentatividad de 1 con W)
3. `WY → Z` (dado)
4. `WX → Z` (transitividad de 2 y 3)

### 3.3 Por qué importan los axiomas en la práctica

Los axiomas de Armstrong permiten al diseñador (y a los algoritmos de normalización) **razonar completamente** sobre las consecuencias de un conjunto de DFs. Dado un esquema con DFs explícitas, se puede calcular sistemáticamente:

- Qué otras DFs se cumplen implícitamente.
- Cuáles atributos puede determinar un conjunto dado (el "cierre").
- Si un conjunto es clave candidata.
- Si el esquema está en una determinada forma normal.

---

## 4. Cierre de un conjunto de atributos

### 4.1 Definición

Dado un conjunto de dependencias funcionales `F` y un conjunto de atributos `X`, el **cierre de X bajo F** (denotado `X⁺`) es el conjunto de **todos los atributos que X determina**, directa o transitivamente, usando las DFs en F.

Formalmente:
```
X⁺ = {A | X →_F A}
```

### 4.2 Algoritmo para calcular el cierre

```
ALGORITMO: Calcular X⁺ dado F
──────────────────────────────
resultado = X

REPETIR hasta que resultado no cambie:
    PARA CADA DF (W → V) en F:
        SI W ⊆ resultado:
            resultado = resultado ∪ V

DEVOLVER resultado
```

### 4.3 Ejemplo paso a paso

Sea el esquema `R(A, B, C, D, E, G)` con el conjunto de DFs:
```
F = {
  A → BC,
  BC → AD,
  D → E,
  CF → B
}
```

Calculemos `{A}⁺`:

**Iteración 1:**
- `A → BC`: A ⊆ {A} ✓ → resultado = {A, B, C}
- `BC → AD`: BC ⊆ {A, B, C} ✓ → resultado = {A, B, C, D}
- `D → E`: D ⊆ {A, B, C, D} ✓ → resultado = {A, B, C, D, E}
- `CF → B`: CF ⊄ {A, B, C, D, E} (F no está) ✗

**Iteración 2:** Revisamos nuevamente con resultado = {A, B, C, D, E}
- No se agregan atributos nuevos.

**Resultado:** `{A}⁺ = {A, B, C, D, E}`

Esto significa que `A` determina funcionalmente a `A, B, C, D, E`. Como `G` no está en el cierre, `A` no determina `G`.

### 4.4 Aplicaciones del cierre

**Verificar si una DF se puede inferir de F:**

`X → Y` se puede inferir de F si y solo si `Y ⊆ X⁺`.

¿Se puede inferir `A → D` de F? Calculamos `{A}⁺ = {A, B, C, D, E}`. Como `D ∈ {A}⁺`, sí: `A → D` se infiere de F.

**Encontrar claves candidatas:**

Un conjunto `K` es superclave si `K⁺` contiene **todos los atributos** de la relación. Es clave candidata si además no existe ningún subconjunto propio de `K` que también sea superclave.

En el ejemplo anterior, ¿es `{A}` una superclave de `R(A, B, C, D, E, G)`?
- `{A}⁺ = {A, B, C, D, E}` — falta `G`.
- No es superclave.

¿Es `{A, G}` una superclave?
- Calculamos `{A, G}⁺`:
  - Inicio: {A, G}
  - `A → BC`: → {A, B, C, G}
  - `BC → AD`: → {A, B, C, D, G}
  - `D → E`: → {A, B, C, D, E, G}
- `{A, G}⁺ = {A, B, C, D, E, G}` — contiene todos los atributos. ✓
- Es superclave. ¿Es clave candidata?
  - ¿Es `{A}` superclave? No (como calculamos).
  - ¿Es `{G}` superclave? `{G}⁺ = {G}` (ninguna DF tiene G en el lado izquierdo). No.
- Por lo tanto, `{A, G}` es clave candidata (no tiene subconjuntos propios que sean superclaves).

---

## 5. Cobertura mínima

### 5.1 El problema de la redundancia en las DFs

Un conjunto de DFs puede contener redundancias. Por ejemplo:

```
F = {
  A → B,
  B → C,
  A → C    ← redundante (se deriva de A→B y B→C por transitividad)
}
```

La DF `A → C` es redundante porque puede inferirse de las otras dos. Trabajar con el conjunto completo de DFs, incluidas las redundantes, complica los algoritmos de normalización innecesariamente.

### 5.2 Definición de cobertura mínima

Un conjunto `Fm` es una **cobertura mínima** (o base canónica) de `F` si:

1. **Equivalencia:** `Fm` y `F` son equivalentes (cada DF de F puede inferirse de Fm, y viceversa).
2. **Lado derecho singular:** Cada DF en Fm tiene exactamente un atributo en el lado derecho.
3. **No hay DFs redundantes:** Eliminar cualquier DF de Fm produce un conjunto no equivalente a F.
4. **No hay atributos extraneous en el lado izquierdo:** No se puede eliminar ningún atributo del lado izquierdo de ninguna DF sin cambiar el conjunto que es equivalente a F.

### 5.3 Algoritmo para calcular la cobertura mínima

```
ALGORITMO: Cobertura mínima de F
─────────────────────────────────
PASO 1 — Descomponer lado derecho:
    Reemplazar cada DF X → Y₁Y₂...Yₙ
    por n DFs: X → Y₁, X → Y₂, ..., X → Yₙ

PASO 2 — Eliminar atributos extraneous en lado izquierdo:
    PARA CADA DF (XA → Y) en F donde X ≠ ∅:
        SI Y ∈ X⁺ (calculado sin usar A):
            Reemplazar XA → Y por X → Y

PASO 3 — Eliminar DFs redundantes:
    PARA CADA DF (X → Y) en F:
        SI Y ∈ (F − {X→Y})  [cierre de X sin esa DF]:
            Eliminar X → Y de F

DEVOLVER F (la cobertura mínima)
```

### 5.4 Ejemplo paso a paso

Sea:
```
F = {
  A → BC,
  B → C,
  A → B,
  AB → C
}
```

**Paso 1 — Descomponer lado derecho:**
```
F = {A→B, A→C, B→C, A→B, AB→C}
```
Eliminando la duplicada `A→B`:
```
F = {A→B, A→C, B→C, AB→C}
```

**Paso 2 — Eliminar atributos extraneous en lado izquierdo:**

Para `AB → C`: ¿es `A` extraneo? ¿Podemos inferir `B → C` sin `A`? Calculamos `{B}⁺` usando el resto de F:
- `{B}⁺`: con `B→C` → `{B,C}`. Sí, C ∈ `{B}⁺`.
- Por lo tanto, A es extraneo en `AB → C`. Reducimos a `B → C`.

Ahora F = `{A→B, A→C, B→C, B→C}`. Eliminamos duplicada:
```
F = {A→B, A→C, B→C}
```

**Paso 3 — Eliminar DFs redundantes:**

¿Es `A→C` redundante? Calculamos `{A}⁺` usando `F − {A→C} = {A→B, B→C}`:
- `{A}⁺`: `A→B` → `{A,B}`; `B→C` → `{A,B,C}`. C ∈ `{A}⁺`. ✓
- `A→C` es redundante. La eliminamos.

```
Fm = {A→B, B→C}
```

Esta es la cobertura mínima. Verificación: `{A→B, B→C}` implica `A→C` (por transitividad) y `AB→C` (aumentatividad de `B→C`). Equivalencia confirmada.

---

## 6. Claves candidatas vs claves primarias

### 6.1 Superclaves

Una **superclave** es cualquier conjunto de atributos que determina funcionalmente a **todos los demás atributos** de la relación. Es decir, `K` es superclave si `K⁺` contiene todos los atributos.

Propiedad importante: si `K` es superclave, entonces cualquier superconjunto de `K` también es superclave. Si `{empleado_id}` es superclave, entonces `{empleado_id, nombre}` también lo es (aunque de forma trivialmente redundante).

### 6.2 Claves candidatas

Una **clave candidata** es una superclave **mínima**: no existe ningún subconjunto propio suyo que también sea superclave.

Toda relación tiene al menos una clave candidata (en el peor caso, el conjunto de todos sus atributos es superclave y algún subconjunto mínimo de él es clave candidata).

Una relación puede tener **múltiples claves candidatas**. Esto es completamente normal y ocurre cuando hay varios atributos o conjuntos de atributos que identifican unívocamente a una tupla.

**Ejemplo:** En la relación `Empleados(empleado_id, número_seguro_social, nombre, departamento)`:
- `{empleado_id}` es clave candidata (identificador interno único).
- `{número_seguro_social}` es clave candidata (identificador externo único).
- `{empleado_id, nombre}` es superclave pero NO clave candidata (tiene un subconjunto propio que ya es superclave).

**Ejemplo con clave compuesta:** En `Matriculas(estudiante_id, materia_id, período, nota)`:
- `{estudiante_id, materia_id, período}` es clave candidata (un estudiante puede repetir una materia en distinto período).
- `{estudiante_id, materia_id}` podría o no ser clave candidata dependiendo de si el negocio permite cursar la misma materia dos veces.

### 6.3 Por qué la distinción entre candidatas y primaria importa

Una **clave primaria** es simplemente la clave candidata que el diseñador eligió para identificar oficialmente las filas. Es una decisión de diseño, no matemática.

La distinción importa porque:

**Las formas normales están definidas en términos de claves candidatas, no de la clave primaria.** Un error común es evaluar si un esquema está en FNBC o 3FN usando solo la clave primaria e ignorando las demás claves candidatas. Esto puede llevar a esquemas que están en 3FN según la clave primaria pero no según todas las candidatas.

**Las claves candidatas no elegidas como primaria también deben tener restricción de unicidad.** En SQL, se deben declarar con `UNIQUE NOT NULL` para preservar su semántica.

```sql
CREATE TABLE empleados (
    empleado_id     INT         PRIMARY KEY,           -- Clave primaria elegida
    nss             CHAR(10)    UNIQUE NOT NULL,        -- Clave candidata alternativa
    nombre          VARCHAR(100) NOT NULL,
    departamento_id INT         REFERENCES departamentos(id)
);
```

**Atributos primos:** Un atributo que forma parte de **alguna** clave candidata se llama **atributo primo**. Los demás son **atributos no primos**. Esta distinción es crucial para definir 2FN y 3FN correctamente.

---

## 7. Primera Forma Normal (1FN)

### 7.1 Definición formal

Una relación está en **Primera Forma Normal (1FN)** si y solo si los dominios de todos sus atributos son **atómicos**: cada atributo contiene un único valor indivisible, no conjuntos, listas, ni estructuras anidadas.

Esta es en realidad la definición básica de ser una relación en el sentido de Codd. Si no está en 1FN, ni siquiera es una relación: es otra estructura de datos.

### 7.2 Violaciones comunes de 1FN

**Grupos repetitivos (repeating groups):**

El modelo original de bases de datos de red (pre-relacional) permitía grupos repetitivos: una "fila" podía tener múltiples instancias de un grupo de atributos.

```
-- Representación conceptual NO relacional (viola 1FN)
Empleado:
  id: 1
  nombre: Ana
  teléfonos: [555-1234, 555-5678, 555-9999]
  proyectos: [(P1, "Alpha", 40h), (P2, "Beta", 20h)]
```

**Atributos multivaluados:**

```sql
-- Viola 1FN: varios valores en una columna
CREATE TABLE empleados_mal (
    id   INT,
    nombre VARCHAR(100),
    telefonos VARCHAR(200)  -- "555-1234, 555-5678, 555-9999"
);
```

**Atributos compuestos:**

```sql
-- Viola 1FN: un atributo que tiene estructura interna
CREATE TABLE empleados_mal (
    id         INT,
    nombre_completo VARCHAR(200)  -- "López, Ana María" (apellido+nombre mezclados)
);
```

**Valores en columnas codificadas:**

```sql
-- Viola 1FN en espíritu: un valor codificado que representa múltiples hechos
INSERT INTO empleados VALUES (1, 'Ana', 'MGR-ENG-001');
-- donde 'MGR-ENG-001' significa: Manager, Ingeniería, nivel 001
```

### 7.3 Cómo convertir a 1FN

**Para atributos multivaluados:** Crear una tabla separada.

```sql
-- En lugar de:
empleados(id, nombre, telefonos_concatenados)

-- Usar:
empleados(id, nombre)
telefonos_empleado(empleado_id, telefono)  -- una fila por teléfono
```

**Para atributos compuestos:** Separar en sus componentes atómicos.

```sql
-- En lugar de:
empleados(id, nombre_completo)

-- Usar:
empleados(id, primer_nombre, segundo_nombre, primer_apellido, segundo_apellido)
```

La decisión de qué tan atómico ser depende del uso. Si nunca vas a buscar por `primer_nombre` solo, quizás `nombre_completo` es "suficientemente atómico" para tu dominio. Pero si necesitas ordenar por apellido, necesitas separarlos.

### 7.4 1FN en los DBMS modernos: el problema de JSON y Arrays

Los DBMS modernos han introducido tipos de datos que técnicamente violan 1FN:

```sql
-- PostgreSQL permite esto:
CREATE TABLE empleados (
    id          INT PRIMARY KEY,
    nombre      VARCHAR(100),
    telefonos   TEXT[],           -- array: viola 1FN
    metadata    JSONB             -- estructura anidada: viola 1FN
);
```

¿Esto es siempre incorrecto? No necesariamente. La respuesta depende del patrón de uso:

- Si el array o JSON es un **blob opaco** que siempre se lee y escribe completo, y nunca se necesita consultar por elementos individuales, puede ser pragmáticamente aceptable.
- Si necesitas hacer queries como "dame todos los empleados cuyo teléfono sea 555-1234" o "dame todos los usuarios que tengan `metadata.departamento = 'Ingeniería'`", entonces la violación de 1FN tiene un costo real: queries complejas, imposibilidad de usar índices simples, semántica de actualización confusa.

El criterio práctico: si necesitas acceder a componentes individuales de un atributo en queries, ese atributo necesita ser atómico (o estar en tabla separada).

---

## 8. Segunda Forma Normal (2FN)

### 8.1 Definición formal

Una relación está en **Segunda Forma Normal (2FN)** si:
1. Está en 1FN.
2. No tiene **dependencias parciales**: ningún atributo **no primo** depende de un subconjunto propio de alguna clave candidata.

*Recordatorio:* un atributo primo es el que forma parte de alguna clave candidata. Un atributo no primo es el que no forma parte de ninguna clave candidata.

Otra forma de leerlo: **cada atributo no primo debe depender funcionalmente de la clave completa**, no de parte de ella.

**Nota importante:** La 2FN solo es relevante cuando hay **claves candidatas compuestas** (de más de un atributo). Si todas las claves candidatas son de un solo atributo, la relación automáticamente está en 2FN (no puede haber dependencias parciales respecto a un solo atributo).

### 8.2 Detectando una violación de 2FN

Volvamos a la tabla del principio, simplificada:

```
DetallesPedido(pedido_id, producto_id, cantidad, precio_unitario, descripcion_producto)
```

Dependencias funcionales:
```
{pedido_id, producto_id} → cantidad            -- DF 1
{pedido_id, producto_id} → precio_unitario     -- DF 2
producto_id → descripcion_producto             -- DF 3 ← PROBLEMA
```

La clave candidata es `{pedido_id, producto_id}`.

Atributos primos: `pedido_id`, `producto_id`.
Atributos no primos: `cantidad`, `precio_unitario`, `descripcion_producto`.

La DF 3 muestra que `descripcion_producto` depende solo de `producto_id`, que es un **subconjunto propio** de la clave candidata. Esto es una dependencia parcial: **viola 2FN**.

Las anomalías que genera:
- Si el mismo producto aparece en 1000 líneas de pedido, su descripción se repite 1000 veces.
- Si cambia la descripción del producto, hay que actualizar 1000 filas.
- Si se borran todos los pedidos de un producto, la descripción desaparece.

### 8.3 Algoritmo para llevar a 2FN

```
PARA CADA dependencia parcial X → Y donde X ⊂ clave_candidata:
    1. Crear nueva relación con esquema (X, Y)
    2. X es la clave primaria de la nueva relación
    3. Eliminar Y de la relación original
    4. Mantener X en la relación original (como referencia)
```

Aplicado al ejemplo:

```
-- Relación original violando 2FN:
DetallesPedido(pedido_id, producto_id, cantidad, precio_unitario, descripcion_producto)

-- DF parcial: producto_id → descripcion_producto

-- RESULTADO después de normalizar:

DetallesPedido(pedido_id, producto_id, cantidad, precio_unitario)
-- Clave: {pedido_id, producto_id}
-- precio_unitario aquí tiene sentido: es el precio *en el momento del pedido*,
-- que puede diferir del precio actual del producto.

Productos(producto_id, descripcion_producto)
-- Clave: {producto_id}
```

### 8.4 Un ejemplo más complejo

```
Inscripciones(estudiante_id, materia_id, nombre_estudiante, créditos_materia, nota, fecha_inscripción)
```

Claves candidatas: `{estudiante_id, materia_id}` (asumiendo que un estudiante solo puede inscribirse una vez por materia).

DFs:
```
{estudiante_id, materia_id} → nota, fecha_inscripción     -- clave completa → OK
estudiante_id → nombre_estudiante                          -- parcial → VIOLA 2FN
materia_id → créditos_materia                              -- parcial → VIOLA 2FN
```

Normalización a 2FN:

```
Inscripciones(estudiante_id, materia_id, nota, fecha_inscripción)
-- Clave: {estudiante_id, materia_id}

Estudiantes(estudiante_id, nombre_estudiante)
-- Clave: {estudiante_id}

Materias(materia_id, créditos_materia)
-- Clave: {materia_id}
```

---

## 9. Tercera Forma Normal (3FN)

### 9.1 Definición formal

Una relación está en **Tercera Forma Normal (3FN)** si:
1. Está en 2FN.
2. No tiene **dependencias transitivas**: ningún atributo no primo depende transitivamente de ninguna clave candidata.

Una dependencia `K → Z` es transitiva si existe `X` tal que:
- `K → X` (K determina X)
- `X → Z` (X determina Z)
- `X` no determina a `K` (X no es clave candidata — de lo contrario no sería "transitiva" en el sentido problemático)
- `Z` no está en ninguna clave candidata

Dicho de forma más directa: **cada atributo no primo debe depender directamente de las claves candidatas, no "a través de" otro atributo no primo**.

### 9.2 Detectando una violación de 3FN

```
Empleados(empleado_id, nombre, departamento_id, nombre_departamento, ubicación_departamento)
```

Asumamos:
- Clave candidata: `{empleado_id}`
- DFs:
  ```
  empleado_id → nombre, departamento_id                    -- directo, OK
  empleado_id → nombre_departamento                        -- transitivo, PROBLEMA
  empleado_id → ubicación_departamento                     -- transitivo, PROBLEMA
  departamento_id → nombre_departamento, ubicación_departamento  -- la causa raíz
  ```

La dependencia transitiva está aquí:
```
empleado_id → departamento_id → nombre_departamento
empleado_id → departamento_id → ubicación_departamento
```

`departamento_id` no es clave candidata de `Empleados`, pero determina atributos. Eso es una transitividad que viola 3FN.

Las anomalías que genera:
- Si el departamento D1 cambia de nombre o ubicación, hay que actualizar todas las filas de empleados del departamento D1.
- No se puede registrar un nuevo departamento sin asignarle al menos un empleado.
- Si el único empleado de un departamento se va, se pierde la información del departamento.

### 9.3 Algoritmo para llevar a 3FN

```
PARA CADA dependencia transitiva X → Y donde X no es clave candidata:
    1. Crear nueva relación con esquema (X, Y)
    2. X es la clave primaria de la nueva relación
    3. Eliminar Y de la relación original
    4. Mantener X en la relación original (como clave foránea)
```

Aplicado al ejemplo:

```
-- Original:
Empleados(empleado_id, nombre, departamento_id, nombre_departamento, ubicación_departamento)

-- Normalizado a 3FN:
Empleados(empleado_id, nombre, departamento_id)
-- Clave: {empleado_id}
-- departamento_id es clave foránea → Departamentos

Departamentos(departamento_id, nombre_departamento, ubicación_departamento)
-- Clave: {departamento_id}
```

### 9.4 La definición alternativa de 3FN (Codd mejorada)

Existe una definición equivalente de 3FN que es más elegante y útil:

> Una relación R está en 3FN si para toda DF no trivial `X → A` que se cumple en R, al menos una de las siguientes condiciones se cumple:
> 1. X es superclave de R.
> 2. A es atributo primo (pertenece a alguna clave candidata).

Esta definición permite ver claramente por qué 3FN es más permisiva que FNBC (que veremos a continuación): en 3FN, una DF puede violar la condición 1 siempre que cumpla la condición 2 (que el atributo determinado sea primo).

### 9.5 Un ejemplo que requiere tanto 2FN como 3FN

```
Pedidos(pedido_id, cliente_id, nombre_cliente, vendedor_id, región_vendedor, cantidad, fecha)
```

Claves candidatas: `{pedido_id}`

DFs:
```
pedido_id → cliente_id, nombre_cliente, vendedor_id, región_vendedor, cantidad, fecha
cliente_id → nombre_cliente         -- transitiva respecto a pedido_id
vendedor_id → región_vendedor       -- transitiva respecto a pedido_id
```

Violaciones de 3FN (dependencias transitivas):
- `pedido_id → cliente_id → nombre_cliente`
- `pedido_id → vendedor_id → región_vendedor`

Normalización:

```
Pedidos(pedido_id, cliente_id, vendedor_id, cantidad, fecha)
Clientes(cliente_id, nombre_cliente)
Vendedores(vendedor_id, región_vendedor)
```

---

## 10. Forma Normal de Boyce-Codd (FNBC)

### 10.1 La motivación: 3FN no es suficientemente fuerte

La 3FN tiene una "escapatoria": permite DFs no triviales donde el lado determinante no es superclave, siempre que el lado determinado sea un atributo primo. En la mayoría de los casos esto no es problema, pero existe una clase de anomalías que 3FN no elimina.

### 10.2 Definición formal de FNBC

Una relación está en **Forma Normal de Boyce-Codd (FNBC)** si para toda DF no trivial `X → Y` que se cumple en R:

> **X es superclave de R.**

No hay excepciones. No importa si Y es primo o no primo. Si la DF existe, X debe ser superclave.

La diferencia respecto a 3FN es exactamente la condición 2 de la definición alternativa de 3FN: en FNBC esa condición no existe. Cualquier DF que viola la condición 1 viola FNBC.

### 10.3 ¿Cuándo difiere 3FN de FNBC?

3FN y FNBC son equivalentes en relaciones con **una sola clave candidata**. La diferencia aparece cuando hay **múltiples claves candidatas que se superponen** (comparten atributos).

### 10.4 El ejemplo clásico

Sea la relación:

```
InscripcionesProfesor(estudiante_id, materia, profesor_id)
```

Con las siguientes reglas del negocio:
1. Un estudiante puede cursar una materia con un solo profesor a la vez: `{estudiante_id, materia} → profesor_id`
2. Un profesor solo enseña una materia: `profesor_id → materia`
3. Un estudiante puede estar inscrito con varios profesores de distintas materias.

Claves candidatas:
- `{estudiante_id, materia}` (por regla 1)
- `{estudiante_id, profesor_id}` (por combinación de reglas 1 y 2: si sé el estudiante y el profesor, sé la materia)

Todos los atributos son primos (`estudiante_id` está en ambas claves, `materia` en la primera, `profesor_id` en la segunda).

Evaluemos 3FN:

La DF `profesor_id → materia`:
- ¿Es `profesor_id` superclave? No (no determina `estudiante_id`).
- ¿Es `materia` un atributo primo? Sí (está en la primera clave candidata).

Por lo tanto, la DF `profesor_id → materia` cumple la condición 2 de 3FN. **La relación está en 3FN.**

Evaluemos FNBC:

La DF `profesor_id → materia`:
- ¿Es `profesor_id` superclave? No.

**Viola FNBC** (a pesar de estar en 3FN).

**¿Qué anomalía genera?**

Datos de ejemplo:

| estudiante_id | materia | profesor_id |
|---|---|---|
| E1 | Álgebra | P1 |
| E2 | Álgebra | P1 |
| E3 | Álgebra | P2 |
| E1 | Cálculo | P3 |

Si el profesor P1 deja de enseñar Álgebra y pasa a enseñar Estadística, hay que actualizar dos filas. Si olvidamos una, la base de datos dice que P1 enseña tanto Álgebra como Estadística, que contradice la regla del negocio.

### 10.5 Descomposición a FNBC

El proceso es el mismo: por cada DF que viola FNBC, crear una nueva relación.

Para `InscripcionesProfesor`, la DF problemática es `profesor_id → materia`:

```
-- Descomponer:
ProfesorMateria(profesor_id, materia)
-- Clave: {profesor_id}
-- Aquí vivé la regla "un profesor enseña una materia"

Inscripciones(estudiante_id, profesor_id)
-- Clave: {estudiante_id, profesor_id}
-- La materia se obtiene por join con ProfesorMateria
```

### 10.6 El costo de FNBC: pérdida de dependencias

La descomposición a FNBC **no siempre preserva todas las dependencias funcionales**.

En el ejemplo anterior, la DF `{estudiante_id, materia} → profesor_id` ya no puede verificarse directamente en ninguna tabla individual. Para verificarla, necesitas hacer un join de `Inscripciones` con `ProfesorMateria`. Eso significa que la restricción ya no puede expresarse como una simple `UNIQUE` constraint; requeriría un trigger o una verificación en la aplicación.

Este es el trade-off fundamental:
- **3FN garantiza** que siempre existe una descomposición sin pérdida de información (lossless join) y que preserva todas las dependencias funcionales.
- **FNBC garantiza** una descomposición sin pérdida de información, pero **puede no preservar** todas las dependencias funcionales.

En la práctica, si 3FN elimina las anomalías concretas que enfrentas y preserva todas tus restricciones, puede ser preferible a FNBC.

---

## 11. Cuarta Forma Normal (4FN) — Dependencias multivaluadas

### 11.1 El problema que 3FN y FNBC no resuelven

Existe otro tipo de redundancia que no es capturada por las dependencias funcionales: las **dependencias multivaluadas**.

### 11.2 Dependencias multivaluadas

Considera la siguiente relación (en FNBC):

```
CurrículumDocente(docente_id, materia, idioma)
```

Con la semántica de que un docente puede enseñar varias materias y hablar varios idiomas, y las combinaciones son independientes (el hecho de que enseñe Álgebra no tiene relación con qué idiomas habla).

Datos de ejemplo:

| docente_id | materia | idioma |
|---|---|---|
| D1 | Álgebra | Español |
| D1 | Álgebra | Inglés |
| D1 | Cálculo | Español |
| D1 | Cálculo | Inglés |

La clave candidata es `{docente_id, materia, idioma}` (la combinación completa). No hay dependencias funcionales no triviales. La relación está en FNBC.

Sin embargo, hay redundancia: el hecho de que D1 habla Inglés está duplicado (una vez por cada materia). Si D1 aprende Francés, hay que agregar tantas filas como materias tenga.

La causa es una **dependencia multivaluada**: el valor de `docente_id` determina **independientemente** el conjunto de valores de `materia` y el conjunto de valores de `idioma`.

**Notación:** `docente_id ↠ materia | idioma`

Se lee: "docente_id multidetermina materia" (y simétricamente idioma).

**Definición formal:** `X ↠ Y` (X multidetermina Y) en una relación R con atributos X, Y, Z si para cualquier dos tuplas con el mismo valor de X, el conjunto de valores de Y para ese valor de X es independiente del conjunto de valores de Z.

### 11.3 Definición de 4FN

Una relación está en **Cuarta Forma Normal (4FN)** si para toda dependencia multivaluada no trivial `X ↠ Y`:

> **X es superclave de R.**

Una dependencia multivaluada `X ↠ Y` es trivial si `Y ⊆ X` o `X ∪ Y` contiene todos los atributos de R.

### 11.4 Descomposición a 4FN

El esquema `CurrículumDocente(docente_id, materia, idioma)` viola 4FN. La descomposición separa las dos dependencias multivaluadas independientes:

```
MateriasDocente(docente_id, materia)
-- Clave: {docente_id, materia}

IdiomasDocente(docente_id, idioma)
-- Clave: {docente_id, idioma}
```

Ahora:
- Agregar un idioma a D1 es una operación en `IdiomasDocente`, independiente de las materias.
- Agregar una materia a D1 es una operación en `MateriasDocente`, independiente de los idiomas.
- Para ver todas las combinaciones (como en la tabla original), se hace el join natural.

### 11.5 La relación entre DFs y dependencias multivaluadas

Una DF es un caso especial de dependencia multivaluada: si `X → Y`, entonces `X ↠ Y` (cada valor de X determina exactamente un valor de Y, que es trivialmente un conjunto de un elemento). Por eso, 4FN implica FNBC.

---

## 12. Quinta Forma Normal (5FN) — Dependencias de join

### 12.1 Un nivel más de sutileza

La 5FN aborda un caso extremadamente específico que raramente se encuentra en diseño práctico, pero que es importante entender para completar la teoría.

### 12.2 Dependencias de join

Una relación `R` tiene una **dependencia de join** respecto a las proyecciones R₁, R₂, ..., Rₙ si `R = R₁ ⋈ R₂ ⋈ ... ⋈ Rₙ` — es decir, R puede reconstruirse completamente a partir del join de sus proyecciones.

Esto siempre es verdad para la descomposición en dos tablas (propiedad de join sin pérdida). La 5FN trata el caso donde una relación tiene redundancias que solo se manifiestan cuando se descompone en **tres o más** partes.

### 12.3 El ejemplo clásico: tres entidades relacionadas

```
ProveedorProductoProyecto(proveedor_id, producto_id, proyecto_id)
```

Semántica: un proveedor suministra un producto para un proyecto. Las tres entidades están interrelacionadas de forma que no se puede simplificar a relaciones binarias sin perder información en algunos casos.

Reglas del negocio que crean una dependencia de join cíclica:
1. Si un proveedor suministra un producto, y ese producto se usa en un proyecto, y el proveedor suministra algo a ese proyecto → el proveedor suministra ESE producto a ESE proyecto.

Esta regla es la que hace que la descomposición en tres tablas binarias `(proveedor, producto)`, `(producto, proyecto)`, `(proveedor, proyecto)` permita reconstruir la relación original mediante join.

### 12.4 Definición de 5FN

Una relación está en **Quinta Forma Normal (5FN)** si toda dependencia de join que se cumple en ella es una consecuencia de sus claves candidatas.

La 5FN garantiza que no hay más redundancias posibles derivadas de dependencias de join.

En la práctica, la 5FN se alcanza raramente y en casos muy específicos. La mayoría de esquemas bien diseñados en FNBC ya están en 5FN.

---

## 13. Forma Normal de Domain/Key (FNDK)

### 13.1 El techo teórico

La **Forma Normal de Domain/Key (FNDK)**, propuesta por Ronald Fagin en 1981, es el techo teórico de la normalización. Su promesa es radical:

> **Un esquema está en FNDK si y solo si toda restricción de integridad que se cumple en el esquema es una consecuencia lógica de las restricciones de dominio y de clave.**

Si un esquema está en FNDK, **no pueden existir anomalías de actualización**, por construcción matemática. No hay nada más que hacer; las restricciones de dominio y clave garantizan toda la integridad.

### 13.2 Por qué es difícil de alcanzar

La FNDK es más general que todas las formas normales anteriores (implica 5FN, 4FN, FNBC, etc.), pero su definición es difícil de trabajar algorítmicamente porque:

1. Requiere identificar **todas** las restricciones de integridad del dominio, no solo las dependencias funcionales.
2. No existe un algoritmo general para transformar un esquema arbitrario en FNDK.
3. La noción de "consecuencia lógica de dominios y claves" es muy amplia y difícil de verificar mecánicamente.

En la práctica, la FNDK sirve como **ideal teórico** que nos dice que si logramos que todas las restricciones sean expresables como restricciones de dominio y clave, el modelo es completo e íntegro. Es más una guía de diseño que un algoritmo aplicable.

---

## 14. ¿Siempre hay que normalizar al máximo?

### 14.1 La desnormalización intencional

La normalización maximiza la integridad y minimiza la redundancia, pero tiene un costo: **más joins en las queries**, lo que puede afectar el rendimiento.

En **sistemas OLTP** (transaccionales), la normalización hasta 3FN o FNBC es generalmente la decisión correcta porque:
- Las escrituras son frecuentes y la redundancia generaría muchas actualizaciones.
- Las queries suelen trabajar sobre pocas filas a la vez.
- Los joins sobre claves indexadas son rápidos.

En **sistemas OLAP** (analíticos, data warehouses), se desnormaliza intencionalmente porque:
- Las queries escanean millones de filas.
- Los joins sobre conjuntos grandes son costosos.
- Las escrituras son por lotes, no concurrentes.
- La redundancia es aceptable si los datos son de solo lectura.

### 14.2 Cuándo desnormalizar es una decisión informada

La desnormalización es una **decisión de modelo físico**, no de modelo lógico. El proceso correcto es:

1. Diseña el modelo lógico correctamente normalizado.
2. Implementa el modelo físico basado en él.
3. **Mide** el rendimiento en condiciones reales.
4. Si hay un problema real de rendimiento causado por joins específicos, evalúa si desnormalizar esas partes específicas resuelve el problema.
5. Desnormaliza solo lo que sea necesario, documentando el motivo.

Lo que **no** debe ocurrir: desnormalizar por intuición antes de medir, o diseñar directamente en el modelo físico sin pasar por el lógico normalizado.

### 14.3 Trade-offs de cada forma normal

| Forma Normal | Lo que elimina | Costo |
|---|---|---|
| 1FN | Atributos multivaluados y compuestos | Más tablas, más joins |
| 2FN | Dependencias parciales | Más tablas, más joins |
| 3FN | Dependencias transitivas | Más tablas, más joins |
| FNBC | Todas las DF donde el determinante no es superclave | Posible pérdida de preservación de dependencias |
| 4FN | Dependencias multivaluadas independientes | Más tablas |
| 5FN | Dependencias de join | Más joins necesarios para reconstruir información |

### 14.4 La regla práctica

Para la mayoría de sistemas transaccionales:
- **3FN es el mínimo aceptable** para un diseño serio.
- **FNBC es preferible** cuando sea posible sin sacrificar restricciones importantes.
- **4FN** debe alcanzarse cuando haya dependencias multivaluadas identificadas.
- **5FN** rara vez es necesario evaluar explícitamente.

La FNDK sirve como horizonte: si puedes demostrar que todas tus restricciones son consecuencia de dominios y claves, sabes que el modelo es formalmente correcto.

---

## 15. Resumen y conexión con el resto del curso

### Lo que construimos en este módulo

Las formas normales no son una lista de reglas a memorizar. Son una jerarquía de propiedades matemáticas derivadas de entender las dependencias en los datos:

```
1FN
 └─ 2FN (elimina dependencias parciales)
     └─ 3FN (elimina dependencias transitivas)
         └─ FNBC (cada DF tiene determinante superclave)
             └─ 4FN (elimina dependencias multivaluadas)
                 └─ 5FN (elimina dependencias de join)
                     └─ FNDK (techo teórico: solo restricciones de dominio y clave)
```

Cada nivel elimina una clase específica de redundancia y la anomalía de actualización que genera.

**El hilo conductor:** todo empieza con las dependencias funcionales. Las dependencias son restricciones semánticas del dominio. Los axiomas de Armstrong nos dan las herramientas para razonar sobre ellas. El cierre y la cobertura mínima nos permiten calcular algorítmicamente. Las claves candidatas son el pivote sobre el que se definen todas las formas normales.

### Conexión con módulos futuros

- **Módulo 3 (Modelado conceptual):** Al pasar de un diagrama EER a un esquema relacional, el resultado debe estar en al menos 3FN. Entender dependencias funcionales permite detectar si la traducción introduce anomalías.

- **Módulo 4 (Integridad y restricciones):** Las restricciones que se pueden expresar como clave primaria, UNIQUE y FOREIGN KEY son exactamente las que corresponden a las claves candidatas y relaciones entre relaciones que la normalización identifica.

- **Módulo 7 (Internals y query planner):** Un esquema altamente normalizado genera muchos joins. Entender los costos de diferentes tipos de join (nested loop, hash join, merge join) permite tomar mejores decisiones sobre hasta dónde normalizar en cada caso.

- **Módulo 10 (OLTP vs OLAP):** La desnormalización intencional en sistemas OLAP (Módulo 11) tiene sentido precisamente porque entendemos qué estamos sacrificando: las garantías de las formas normales superiores, aceptando redundancia a cambio de rendimiento de lectura.

---

## Ejercicios de comprensión

**Ejercicio 1.** Dado el esquema `Vuelos(vuelo_id, aerolinea_id, nombre_aerolinea, origen, destino, hora_salida, precio_base, impuesto_origen)` con las DFs:
```
vuelo_id → aerolinea_id, origen, destino, hora_salida, precio_base
aerolinea_id → nombre_aerolinea
origen → impuesto_origen
```
a) ¿Cuáles son las claves candidatas?
b) ¿Está en 1FN? ¿En 2FN? ¿En 3FN?
c) Normaliza a 3FN mostrando todos los esquemas resultantes.

**Ejercicio 2.** Dado el conjunto de DFs `F = {AB→C, C→B, A→D, D→E}` sobre el esquema `R(A,B,C,D,E)`:
a) Calcula `{A,B}⁺`.
b) Calcula `{C}⁺`.
c) ¿Cuáles son las claves candidatas de R?

**Ejercicio 3.** El siguiente esquema está en 3FN pero no en FNBC. Identifica la DF que viola FNBC y propón la descomposición:

```
ReservasAula(aula_id, período, materia_id, profesor_id)
```
Con las reglas:
- Una materia en un período usa exactamente un aula: `{materia_id, período} → aula_id`
- Un profesor en un período usa exactamente un aula: `{profesor_id, período} → aula_id`
- Un aula en un período tiene exactamente una materia: `{aula_id, período} → materia_id`
- Un aula en un período tiene exactamente un profesor: `{aula_id, período} → profesor_id`

**Ejercicio 4.** Calcula la cobertura mínima del siguiente conjunto de DFs:
```
F = {A→BC, B→C, A→B, AB→C, AC→B}
```

**Ejercicio 5.** Un colega propone el siguiente esquema para un sistema de biblioteca:
```
Préstamos(libro_isbn, título_libro, autor_libro, socio_id, nombre_socio, 
          fecha_préstamo, fecha_devolución, biblioteca_id, dirección_biblioteca)
```
a) Identifica todas las dependencias funcionales que puedas inferir razonablemente.
b) Identifica todas las anomalías que tiene este diseño.
c) Normaliza a 3FN.

---

*Próximo módulo: Modelado conceptual con profundidad — donde aplicaremos toda la teoría de dependencias a construir modelos Entidad-Relación Extendidos que se traduzcan correctamente a esquemas relacionales desde el inicio.*
