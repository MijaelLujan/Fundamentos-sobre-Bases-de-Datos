# Módulo 5 — Patrones de modelado para dominios reales

> *"Los dominios más comunes tienen estructuras que se repiten. Reconocer el patrón antes de empezar a dibujar tablas es la diferencia entre un modelo que dura cinco años y uno que requiere una migración costosa al año siguiente."*

---

## Tabla de contenidos

1. [Por qué existen los patrones de modelado](#1-por-qué-existen-los-patrones-de-modelado)
2. [El patrón Party](#2-el-patrón-party)
3. [El patrón de roles](#3-el-patrón-de-roles)
4. [Jerarquías recursivas y bill of materials](#4-jerarquías-recursivas-y-bill-of-materials)
5. [El patrón de productos y servicios configurables](#5-el-patrón-de-productos-y-servicios-configurables)
6. [Modelos de precios con vigencia temporal](#6-modelos-de-precios-con-vigencia-temporal)
7. [Clasificaciones flexibles sin alterar el esquema](#7-clasificaciones-flexibles-sin-alterar-el-esquema)
8. [El anti-patrón Entity-Attribute-Value (EAV)](#8-el-anti-patrón-entity-attribute-value-eav)
9. [Resumen y conexión con el resto del curso](#9-resumen-y-conexión-con-el-resto-del-curso)

---

## 1. Por qué existen los patrones de modelado

### 1.1 El problema de reinventar la rueda en modelado

Existe una clase de problemas de diseño que se encuentran en casi todos los sistemas empresariales, independientemente del dominio específico:

- "Necesitamos modelar tanto personas físicas como empresas como clientes."
- "Un usuario puede ser cliente, proveedor, y empleado al mismo tiempo."
- "Tenemos un catálogo de productos donde cada categoría tiene atributos diferentes."
- "Los precios cambian con el tiempo y necesitamos saber cuánto costaba algo en una fecha pasada."
- "Nuestra estructura organizacional es una jerarquía de profundidad variable."

Estos problemas tienen soluciones conocidas, probadas, y con trade-offs bien entendidos. Los **patrones de modelado** son exactamente eso: soluciones reutilizables a problemas de diseño que aparecen repetidamente en contextos diferentes.

Conocer estos patrones tiene tres beneficios:

**Velocidad:** No necesitas derivar la solución desde cero. Reconoces el patrón y aplicas la solución conocida.

**Calidad:** Los patrones documentados han sido refinados por años de uso en producción. Sus limitaciones son conocidas.

**Comunicación:** "Vamos a usar el patrón Party para el módulo de terceros" es mucho más preciso que "vamos a hacer una tabla de personas que también sirve para empresas pero es un poco diferente".

### 1.2 Los patrones que cubriremos

Los patrones de este módulo vienen principalmente de dos fuentes:

- **"The Data Model Resource Book"** de Len Silverston (1997, actualizado en 2001 y 2009): la colección más comprehensiva de patrones de modelado de datos para dominios empresariales.
- **"Analysis Patterns"** de Martin Fowler (1996): patrones conceptuales de alto nivel para modelado.
- **La práctica acumulada de la industria:** patrones que emergen repetidamente en proyectos reales.

---

## 2. El patrón Party

### 2.1 El problema que resuelve

En casi todos los sistemas de negocio hay entidades que son "actores": personas, empresas, organizaciones, departamentos. Estas entidades comparten una propiedad fundamental: son **partes** (parties) en algún tipo de relación comercial o contractual.

El problema de modelado que aparece repetidamente es:

> "Los clientes pueden ser tanto personas físicas como empresas. Las personas tienen nombre, apellido, fecha de nacimiento, número de cédula. Las empresas tienen razón social, NIT, fecha de constitución. Ambas tienen dirección, teléfonos, correo. Ambas hacen pedidos."

El diseño ingenuo resuelve esto con dos tablas independientes:

```sql
-- Diseño ingenuo
CREATE TABLE clientes_persona (
    persona_id      INT PRIMARY KEY,
    primer_nombre   VARCHAR(100),
    apellido        VARCHAR(100),
    fecha_nac       DATE,
    cedula          VARCHAR(20),
    direccion       TEXT,
    telefono        VARCHAR(20),
    email           VARCHAR(100)
);

CREATE TABLE clientes_empresa (
    empresa_id      INT PRIMARY KEY,
    razon_social    VARCHAR(200),
    nit             VARCHAR(20),
    fecha_const     DATE,
    direccion       TEXT,
    telefono        VARCHAR(20),
    email           VARCHAR(100)
);

-- Ahora los pedidos tienen que referenciar una de las dos:
CREATE TABLE pedidos (
    pedido_id       INT PRIMARY KEY,
    -- ¿Cuál FK uso? ¿Las dos son nullable?
    cliente_persona_id INT REFERENCES clientes_persona,
    cliente_empresa_id INT REFERENCES clientes_empresa,
    -- Y una constraint que garantiza que exactamente una de las dos no es NULL...
    -- que resulta compleja de expresar
    fecha           DATE NOT NULL
);
```

Este diseño tiene problemas inmediatos: atributos comunes duplicados (`direccion`, `telefono`, `email`), la referencia en pedidos es incómoda, y si aparece un tercer tipo de cliente (p.ej., una entidad gubernamental), hay que agregar una tercera tabla y una tercera FK en `pedidos`.

### 2.2 La solución: el patrón Party

El patrón **Party** crea una entidad abstracta que representa a cualquier actor que puede participar en relaciones del negocio. Los tipos concretos (persona, organización) son especializaciones de esa entidad abstracta.

El nombre "Party" viene del término legal: una "parte" en un contrato puede ser cualquier tipo de entidad legal.

```
                   ┌─────────────────┐
                   │      PARTY      │  ← entidad abstracta
                   │   (party_id)    │
                   │   nombre        │
                   │   tipo          │
                   └────────┬────────┘
                            │ especialización
              ┌─────────────┼─────────────┐
              ▼             ▼             ▼
      ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
      │    PERSONA   │ │ ORGANIZACIÓN │ │  GOB_ENTITY  │
      │  fecha_nac   │ │  nit         │ │  cod_entidad │
      │  cedula      │ │  fecha_const │ │  ...         │
      └──────────────┘ └──────────────┘ └──────────────┘
```

### 2.3 Implementación completa del patrón Party

```sql
-- La entidad abstracta Party
CREATE TABLE parties (
    party_id        SERIAL          PRIMARY KEY,
    party_type      VARCHAR(20)     NOT NULL
        CHECK (party_type IN ('PERSON', 'ORGANIZATION', 'GOV_ENTITY')),
    -- Atributos comunes a todas las parties:
    nombre_display  VARCHAR(300)    NOT NULL,   -- nombre para mostrar (razón social o nombre completo)
    activo          BOOLEAN         NOT NULL DEFAULT TRUE,
    creado_en       TIMESTAMPTZ     NOT NULL DEFAULT NOW()
);

-- Especialización: persona física
CREATE TABLE personas (
    party_id        INT             PRIMARY KEY
        REFERENCES parties(party_id) ON DELETE CASCADE,
    primer_nombre   VARCHAR(100)    NOT NULL,
    segundo_nombre  VARCHAR(100),
    primer_apellido VARCHAR(100)    NOT NULL,
    segundo_apellido VARCHAR(100),
    fecha_nac       DATE            NOT NULL,
    num_documento   VARCHAR(20)     NOT NULL UNIQUE,
    tipo_documento  VARCHAR(10)     NOT NULL
        CHECK (tipo_documento IN ('CC', 'CE', 'PASAPORTE', 'NIT'))
);

-- Especialización: organización
CREATE TABLE organizaciones (
    party_id        INT             PRIMARY KEY
        REFERENCES parties(party_id) ON DELETE CASCADE,
    razon_social    VARCHAR(300)    NOT NULL,
    nit             VARCHAR(20)     NOT NULL UNIQUE,
    fecha_const     DATE,
    tipo_org        VARCHAR(50)
        CHECK (tipo_org IN ('SAS', 'SA', 'LTDA', 'ONG', 'COOPERATIVA', 'OTRO'))
);

-- Datos de contacto: pertenecen a la party, no al subtipo
CREATE TABLE contactos_party (
    contacto_id     SERIAL          PRIMARY KEY,
    party_id        INT             NOT NULL REFERENCES parties(party_id),
    tipo_contacto   VARCHAR(20)     NOT NULL
        CHECK (tipo_contacto IN ('EMAIL', 'TEL_MOVIL', 'TEL_FIJO', 'FAX', 'WEB')),
    valor           VARCHAR(200)    NOT NULL,
    es_principal    BOOLEAN         NOT NULL DEFAULT FALSE,
    vigente_desde   DATE            NOT NULL DEFAULT CURRENT_DATE,
    vigente_hasta   DATE,
    CONSTRAINT chk_vigencia CHECK (vigente_hasta IS NULL OR vigente_hasta >= vigente_desde)
);

-- Direcciones: también pertenecen a la party
CREATE TABLE direcciones_party (
    direccion_id    SERIAL          PRIMARY KEY,
    party_id        INT             NOT NULL REFERENCES parties(party_id),
    tipo_dir        VARCHAR(20)     NOT NULL
        CHECK (tipo_dir IN ('FISCAL', 'COMERCIAL', 'ENTREGA', 'CORRESPONDENCIA')),
    linea1          VARCHAR(200)    NOT NULL,
    linea2          VARCHAR(200),
    ciudad          VARCHAR(100)    NOT NULL,
    departamento    VARCHAR(100),
    pais            CHAR(2)         NOT NULL DEFAULT 'CO',
    codigo_postal   VARCHAR(10),
    vigente_desde   DATE            NOT NULL DEFAULT CURRENT_DATE,
    vigente_hasta   DATE,
    CONSTRAINT chk_vigencia_dir CHECK (vigente_hasta IS NULL OR vigente_hasta >= vigente_desde)
);

-- Ahora los pedidos referencian parties, sin importar si es persona u organización:
CREATE TABLE pedidos (
    pedido_id       SERIAL          PRIMARY KEY,
    cliente_id      INT             NOT NULL REFERENCES parties(party_id),
    fecha           DATE            NOT NULL DEFAULT CURRENT_DATE,
    total           DECIMAL(12,2)   NOT NULL CHECK (total >= 0)
);
```

### 2.4 Relaciones entre parties

Una extensión natural del patrón Party es modelar relaciones entre parties. Las organizaciones tienen personas como empleados, socios, representantes. Las personas pueden tener relaciones con otras personas (familia, referidos). Estas relaciones entre parties son también un patrón recurrente:

```sql
-- Tipos de relación entre parties
CREATE TABLE tipos_relacion_party (
    tipo_id         SERIAL          PRIMARY KEY,
    nombre          VARCHAR(100)    NOT NULL UNIQUE,
    descripcion     TEXT,
    -- ¿Qué tipos de party pueden tener esta relación?
    parte_a_tipo    VARCHAR(20)     CHECK (parte_a_tipo IN ('PERSON', 'ORGANIZATION', 'ANY')),
    parte_b_tipo    VARCHAR(20)     CHECK (parte_b_tipo IN ('PERSON', 'ORGANIZATION', 'ANY'))
);

-- Instancias de relaciones entre parties
CREATE TABLE relaciones_party (
    relacion_id     SERIAL          PRIMARY KEY,
    tipo_id         INT             NOT NULL REFERENCES tipos_relacion_party,
    party_a_id      INT             NOT NULL REFERENCES parties(party_id),
    party_b_id      INT             NOT NULL REFERENCES parties(party_id),
    desde           DATE            NOT NULL,
    hasta           DATE,
    notas           TEXT,
    CONSTRAINT chk_parties_distintas CHECK (party_a_id != party_b_id),
    CONSTRAINT chk_vigencia_rel CHECK (hasta IS NULL OR hasta >= desde)
);

-- Datos de ejemplo de tipos de relación:
-- ('EMPLEADO_DE', 'PERSON', 'ORGANIZATION')
-- ('ACCIONISTA_DE', 'PERSON', 'ORGANIZATION')
-- ('FILIAL_DE', 'ORGANIZATION', 'ORGANIZATION')
-- ('REPRESENTANTE_LEGAL_DE', 'PERSON', 'ORGANIZATION')
-- ('REFERIDO_POR', 'ANY', 'ANY')
```

### 2.5 Cuándo usar Party y cuándo no

**Usa Party cuando:**
- El sistema maneja múltiples tipos de actores (personas y organizaciones) que participan en los mismos procesos.
- Los tipos de actores comparten muchos atributos y relaciones.
- Es probable que aparezcan nuevos tipos de actores en el futuro.
- Necesitas registrar relaciones entre los actores mismos.

**No uses Party cuando:**
- Solo hay un tipo de actor (solo personas, o solo empresas). El patrón agrega complejidad innecesaria.
- Los tipos de actores son tan distintos que no comparten nada. No hay beneficio en unificarlos.
- El equipo es pequeño y la complejidad adicional supera el beneficio de flexibilidad.

### 2.6 La consulta típica con Party

Una preocupación válida con Party es la complejidad de las queries. El patrón requiere joins adicionales. Esto se puede mitigar con vistas:

```sql
-- Vista que une la información de persona completa
CREATE VIEW vista_personas_completa AS
SELECT
    p.party_id,
    p.activo,
    per.primer_nombre,
    per.primer_apellido,
    per.segundo_apellido,
    per.primer_nombre || ' ' || per.primer_apellido AS nombre_completo,
    per.fecha_nac,
    per.num_documento,
    c.valor AS email_principal,
    d.linea1 AS direccion_principal,
    d.ciudad
FROM parties p
JOIN personas per ON p.party_id = per.party_id
LEFT JOIN contactos_party c ON p.party_id = c.party_id
    AND c.tipo_contacto = 'EMAIL' AND c.es_principal = TRUE
LEFT JOIN direcciones_party d ON p.party_id = d.party_id
    AND d.tipo_dir = 'FISCAL' AND d.vigente_hasta IS NULL;
```

---

## 3. El patrón de roles

### 3.1 El problema: una entidad en múltiples contextos

El patrón de roles resuelve un problema relacionado pero distinto al Party: el mismo "objeto" puede tener comportamientos y atributos distintos según el **contexto** en que participa.

Ejemplos:
- Una persona puede ser cliente Y empleado Y proveedor de la misma empresa.
- Una empresa puede ser cliente para algunos productos y proveedor para otros.
- Una cuenta bancaria puede ser cuenta de origen Y cuenta de destino en distintas transacciones.
- Un producto puede ser componente de otro producto Y producto final vendible.

El diseño ingenuo para "una persona que puede ser cliente o empleado" es crear columnas de flags:

```sql
-- Diseño ingenuo con flags
CREATE TABLE personas (
    persona_id  INT PRIMARY KEY,
    nombre      VARCHAR(100),
    es_cliente  BOOLEAN DEFAULT FALSE,
    es_empleado BOOLEAN DEFAULT FALSE,
    es_proveedor BOOLEAN DEFAULT FALSE,
    -- atributos específicos de cliente:
    limite_credito      DECIMAL,
    segmento_cliente    VARCHAR(50),
    -- atributos específicos de empleado:
    cargo               VARCHAR(100),
    salario             DECIMAL,
    fecha_contratacion  DATE,
    -- atributos específicos de proveedor:
    condiciones_pago    VARCHAR(100),
    categoria_proveedor VARCHAR(50)
);
```

Los problemas son los de siempre: NULLs en todas las columnas que no aplican al rol actual, imposibilidad de poner `NOT NULL` en atributos obligatorios de cada rol, y la tabla crece con cada nuevo rol.

### 3.2 La solución: tablas de rol

El patrón de roles separa la **identidad** de la entidad (quién es) de su **comportamiento en un contexto** (qué rol juega):

```sql
-- La entidad central (puede usar Party si aplica, o una tabla propia)
CREATE TABLE personas (
    persona_id      SERIAL          PRIMARY KEY,
    nombre_completo VARCHAR(200)    NOT NULL,
    num_documento   VARCHAR(20)     NOT NULL UNIQUE,
    fecha_nac       DATE            NOT NULL
);

-- Rol: cliente
CREATE TABLE clientes (
    cliente_id      SERIAL          PRIMARY KEY,
    persona_id      INT             NOT NULL REFERENCES personas(persona_id),
    -- Una persona puede tener múltiples "cuentas de cliente" si el negocio lo permite,
    -- o UNIQUE(persona_id) si solo puede ser cliente una vez.
    codigo_cliente  VARCHAR(20)     NOT NULL UNIQUE,
    limite_credito  DECIMAL(12,2)   NOT NULL DEFAULT 0,
    segmento        VARCHAR(50)     NOT NULL DEFAULT 'ESTANDAR'
        CHECK (segmento IN ('ESTANDAR', 'PREMIUM', 'CORPORATIVO')),
    activo_desde    DATE            NOT NULL DEFAULT CURRENT_DATE
);

-- Rol: empleado
CREATE TABLE empleados (
    empleado_id     SERIAL          PRIMARY KEY,
    persona_id      INT             NOT NULL REFERENCES personas(persona_id),
    -- UNIQUE si una persona solo puede ser empleada una vez:
    UNIQUE (persona_id),
    codigo_empleado VARCHAR(20)     NOT NULL UNIQUE,
    cargo           VARCHAR(100)    NOT NULL,
    salario         DECIMAL(12,2)   NOT NULL CHECK (salario > 0),
    departamento_id INT             NOT NULL REFERENCES departamentos(departamento_id),
    fecha_ingreso   DATE            NOT NULL,
    fecha_egreso    DATE,
    CONSTRAINT chk_fechas_emp CHECK (fecha_egreso IS NULL OR fecha_egreso >= fecha_ingreso)
);

-- Rol: proveedor
CREATE TABLE proveedores (
    proveedor_id    SERIAL          PRIMARY KEY,
    persona_id      INT             NOT NULL REFERENCES personas(persona_id),
    UNIQUE (persona_id),
    codigo_prov     VARCHAR(20)     NOT NULL UNIQUE,
    condiciones_pago VARCHAR(100)   NOT NULL,
    categoria       VARCHAR(50)     NOT NULL
);
```

### 3.3 La dimensión temporal de los roles

Los roles frecuentemente tienen vigencia: alguien puede ser empleado por un período y luego dejar de serlo. Modelar esa temporalidad en el rol mismo es parte del patrón:

```sql
-- Versión con historial completo de roles
CREATE TABLE historial_empleado (
    historial_id    SERIAL          PRIMARY KEY,
    persona_id      INT             NOT NULL REFERENCES personas(persona_id),
    cargo           VARCHAR(100)    NOT NULL,
    departamento_id INT             NOT NULL REFERENCES departamentos(departamento_id),
    salario         DECIMAL(12,2)   NOT NULL,
    desde           DATE            NOT NULL,
    hasta           DATE,
    motivo_cambio   VARCHAR(200),
    CONSTRAINT chk_fechas CHECK (hasta IS NULL OR hasta >= desde)
);

-- El "empleado actual" es el registro sin fecha_hasta
-- El historial completo está en la misma tabla
```

Esto conecta directamente con el Módulo 6 (Diseño para el tiempo), que profundizará en esta dirección.

### 3.4 Restricciones de integridad en roles

El patrón de roles puede requerir restricciones que garanticen la consistencia entre la entidad y sus roles:

```sql
-- Garantizar que si una persona es empleada, también sea mayor de 18 años:
CREATE OR REPLACE FUNCTION verificar_edad_minima_empleado()
RETURNS TRIGGER AS $$
DECLARE
    edad_persona INT;
BEGIN
    SELECT EXTRACT(YEAR FROM AGE(CURRENT_DATE, fecha_nac))
    INTO edad_persona
    FROM personas WHERE persona_id = NEW.persona_id;

    IF edad_persona < 18 THEN
        RAISE EXCEPTION 'Una persona menor de 18 años no puede ser empleada';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_edad_empleado
    BEFORE INSERT ON empleados
    FOR EACH ROW EXECUTE FUNCTION verificar_edad_minima_empleado();
```

### 3.5 Rol vs subtipo: cuándo usar cada uno

La pregunta de si modelar como jerarquía EER (subtipos) o como patrón de roles no siempre tiene una respuesta obvia. El criterio:

| Criterio | Usar subtipo (jerarquía) | Usar rol |
|---|---|---|
| ¿Puede una entidad cambiar de tipo? | No (un Perro no se convierte en Gato) | Sí (un cliente puede dejar de serlo) |
| ¿Puede pertenecer a múltiples tipos simultáneamente? | Solo si solapado | Sí, por diseño |
| ¿El tipo define qué es el objeto o qué hace? | Define qué es | Define qué hace en un contexto |
| ¿Los atributos son de la identidad o del comportamiento? | De la identidad | Del comportamiento contextual |

Ejemplo:
- `Persona` → `Médico` → `Cirujano`: jerarquía de subtipos (un cirujano *es* un tipo de médico, no *actúa como* médico temporalmente).
- `Persona` → `[cliente, empleado, proveedor]`: patrón de roles (una persona *actúa como* cliente en ciertos contextos, puede dejar de serlo).

---

## 4. Jerarquías recursivas y bill of materials

### 4.1 El problema de las jerarquías de profundidad variable

Muchos dominios tienen estructuras jerárquicas donde la profundidad no es fija:

- Estructura organizacional: empresa → división → área → equipo → empleado
- Árbol de categorías de productos: Electrónica → Computación → Laptops → Gaming
- Árbol de comentarios anidados (como en Reddit)
- Estructura geográfica: país → departamento → municipio → vereda
- **Bill of Materials (BOM):** un producto compuesto por subcomponentes que a su vez son compuestos por otros.

La característica común es que la **profundidad de la jerarquía no es conocida en tiempo de diseño** y puede variar entre ramas.

### 4.2 Modelo adjacency list (lista de adyacencia)

La solución más simple: cada nodo apunta a su padre directo.

```sql
CREATE TABLE categorias (
    categoria_id    SERIAL          PRIMARY KEY,
    nombre          VARCHAR(100)    NOT NULL,
    padre_id        INT             REFERENCES categorias(categoria_id),
    -- NULL para nodos raíz
    descripcion     TEXT,
    activo          BOOLEAN         NOT NULL DEFAULT TRUE,
    CONSTRAINT chk_no_auto_ref CHECK (categoria_id != padre_id)
);

-- Datos de ejemplo:
INSERT INTO categorias (categoria_id, nombre, padre_id) VALUES
    (1, 'Electrónica', NULL),        -- raíz
    (2, 'Computación', 1),
    (3, 'Laptops', 2),
    (4, 'Gaming', 3),
    (5, 'Smartphones', 1),
    (6, 'Android', 5);
```

**Ventajas:**
- Muy simple de entender e implementar.
- Inserciones, actualizaciones y borrados son simples: solo cambias `padre_id`.
- Mover un subárbol completo es una sola operación.

**Desventajas:**
- Consultar toda la jerarquía (o un subárbol) requiere queries recursivos.
- Hasta SQL:1999 (CTEs recursivos), esto era muy difícil de hacer en SQL estándar.
- En DBMS sin soporte de CTEs recursivos, requería múltiples queries o procedimientos almacenados.

### 4.3 Consultando la lista de adyacencia con CTEs recursivos

```sql
-- Obtener toda la jerarquía de una categoría (de raíz hacia abajo)
WITH RECURSIVE jerarquia AS (
    -- Caso base: el nodo inicial
    SELECT
        categoria_id,
        nombre,
        padre_id,
        0 AS nivel,
        nombre::TEXT AS ruta
    FROM categorias
    WHERE categoria_id = 1  -- empezamos desde "Electrónica"
      AND activo = TRUE

    UNION ALL

    -- Caso recursivo: hijos de los nodos actuales
    SELECT
        c.categoria_id,
        c.nombre,
        c.padre_id,
        j.nivel + 1,
        j.ruta || ' > ' || c.nombre
    FROM categorias c
    JOIN jerarquia j ON c.padre_id = j.categoria_id
    WHERE c.activo = TRUE
)
SELECT * FROM jerarquia ORDER BY ruta;

-- Resultado:
-- categoria_id | nombre       | nivel | ruta
-- 1            | Electrónica  | 0     | Electrónica
-- 2            | Computación  | 1     | Electrónica > Computación
-- 3            | Laptops      | 2     | Electrónica > Computación > Laptops
-- 4            | Gaming       | 3     | Electrónica > Computación > Laptops > Gaming
-- 5            | Smartphones  | 1     | Electrónica > Smartphones
-- 6            | Android      | 2     | Electrónica > Smartphones > Android
```

```sql
-- Obtener los antecesores (camino hacia la raíz) de un nodo específico
WITH RECURSIVE antecesores AS (
    SELECT categoria_id, nombre, padre_id, 0 AS nivel
    FROM categorias
    WHERE categoria_id = 4  -- "Gaming"

    UNION ALL

    SELECT c.categoria_id, c.nombre, c.padre_id, a.nivel + 1
    FROM categorias c
    JOIN antecesores a ON c.categoria_id = a.padre_id
)
SELECT * FROM antecesores ORDER BY nivel DESC;
-- Devuelve: Electrónica > Computación > Laptops > Gaming (el breadcrumb)
```

### 4.4 Modelos alternativos para jerarquías

La lista de adyacencia es la más simple pero no la más eficiente para ciertos patrones de consulta. Existen tres alternativas:

---

#### Modelo de conjunto anidado (Nested Set Model)

Cada nodo almacena un rango `(lft, rgt)` que representa su posición en un recorrido preorden del árbol. Los descendientes de un nodo tienen `lft` y `rgt` dentro del rango del ancestro.

```sql
CREATE TABLE categorias_ns (
    categoria_id    INT     PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL,
    lft             INT     NOT NULL,
    rgt             INT     NOT NULL,
    CONSTRAINT chk_lft_rgt CHECK (lft < rgt)
);

-- Para el árbol anterior:
-- Electrónica:     lft=1, rgt=12
-- Computación:     lft=2, rgt=7
-- Laptops:         lft=3, rgt=6
-- Gaming:          lft=4, rgt=5
-- Smartphones:     lft=8, rgt=11
-- Android:         lft=9, rgt=10

-- Obtener todos los descendientes de Electrónica (categoría_id=1, lft=1, rgt=12):
SELECT * FROM categorias_ns WHERE lft > 1 AND rgt < 12;
-- Sin recursión. Muy eficiente con índice en (lft, rgt).

-- Obtener todos los ancestros de Gaming (lft=4, rgt=5):
SELECT * FROM categorias_ns WHERE lft < 4 AND rgt > 5;
-- También sin recursión.
```

**Ventajas:** Leer subárboles y ancestros es muy eficiente (no requiere recursión).
**Desventajas:** Insertar un nodo requiere actualizar todos los nodos a la derecha (una operación O(n)). Muy costoso para árboles con muchas escrituras.

---

#### Modelo de ruta de materiales (Materialized Path)

Cada nodo almacena la ruta completa desde la raíz hasta él mismo como una cadena.

```sql
CREATE TABLE categorias_mp (
    categoria_id    INT     PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL,
    ruta            VARCHAR(500) NOT NULL,  -- p.ej. '/1/2/3/'
    profundidad     INT     NOT NULL
);

-- Datos:
-- Electrónica:  ruta='/1/',       profundidad=0
-- Computación:  ruta='/1/2/',     profundidad=1
-- Laptops:      ruta='/1/2/3/',   profundidad=2
-- Gaming:       ruta='/1/2/3/4/', profundidad=3

-- Todos los descendientes de Computación (ruta='/1/2/'):
SELECT * FROM categorias_mp WHERE ruta LIKE '/1/2/%';
-- Eficiente si hay índice en ruta (índice GIN o texto en PostgreSQL).

-- Mover un subárbol es un UPDATE con REPLACE en la ruta:
UPDATE categorias_mp
SET ruta = REPLACE(ruta, '/1/2/', '/1/5/')
WHERE ruta LIKE '/1/2/%';
```

**Ventajas:** Fácil de entender, las queries de subárbol son simples, mover subárboles es un UPDATE.
**Desventajas:** La ruta puede volverse muy larga. Los LIKE con % al inicio no usan índices B-tree (requiere índices especiales).

---

#### Tabla de cierre (Closure Table)

Almacena **todos** los pares (ancestro, descendiente) de la jerarquía, incluyendo cada nodo consigo mismo.

```sql
CREATE TABLE categorias (
    categoria_id    INT     PRIMARY KEY,
    nombre          VARCHAR(100) NOT NULL
);

CREATE TABLE cierre_categorias (
    ancestro_id     INT     NOT NULL REFERENCES categorias,
    descendiente_id INT     NOT NULL REFERENCES categorias,
    profundidad     INT     NOT NULL DEFAULT 0,  -- 0 = el nodo consigo mismo
    PRIMARY KEY (ancestro_id, descendiente_id)
);

-- Para el árbol: Electrónica(1) > Computación(2) > Laptops(3) > Gaming(4)
-- La tabla de cierre contiene:
-- (1,1,0), (1,2,1), (1,3,2), (1,4,3)  ← ancestros de cada nodo con Electrónica
-- (2,2,0), (2,3,1), (2,4,2)            ← ancestros de cada nodo con Computación
-- (3,3,0), (3,4,1)                     ← ancestros de cada nodo con Laptops
-- (4,4,0)                              ← Gaming consigo mismo

-- Todos los descendientes de Computación (ID=2):
SELECT c.* FROM categorias c
JOIN cierre_categorias cc ON c.categoria_id = cc.descendiente_id
WHERE cc.ancestro_id = 2 AND cc.profundidad > 0;

-- Todos los ancestros de Gaming (ID=4):
SELECT c.* FROM categorias c
JOIN cierre_categorias cc ON c.categoria_id = cc.ancestro_id
WHERE cc.descendiente_id = 4 AND cc.profundidad > 0;
```

**Ventajas:** Las queries de árbol son muy eficientes. Soporta bien tanto lectura de subárboles como de ancestros. Flexible.
**Desventajas:** La tabla de cierre puede ser grande (O(n²) en el peor caso). Insertar un nodo requiere insertar filas en la tabla de cierre para todos sus ancestros.

### 4.5 Comparación de modelos

| Modelo | Leer subárbol | Leer ancestros | Insertar nodo | Mover nodo | Espacio |
|---|---|---|---|---|---|
| Lista de adyacencia | CTE recursivo | CTE recursivo | Simple | Simple (update padre_id) | O(n) |
| Nested Set | Muy rápido | Muy rápido | Muy costoso (O(n)) | Muy costoso | O(n) |
| Materialized Path | Rápido (LIKE) | Rápido (LIKE) | Simple | UPDATE en ruta | O(n) |
| Closure Table | Rápido (JOIN) | Rápido (JOIN) | Moderado | Moderado | O(n²) |

**Recomendación práctica:**
- **Muchas lecturas, pocas escrituras, profundidad variable:** Closure Table o Nested Set.
- **Muchas escrituras (inserciones frecuentes):** Lista de adyacencia con CTEs recursivos.
- **Balance moderado:** Lista de adyacencia + Materialized Path para acelerar queries frecuentes.
- **DBMS con soporte nativo de árboles** (p.ej. `ltree` en PostgreSQL): usar la extensión.

### 4.6 Bill of Materials (BOM)

El Bill of Materials es una jerarquía recursiva específica del dominio de manufactura: un producto está compuesto por subcomponentes, que a su vez pueden ser compuestos por otros subcomponentes.

Lo que lo hace diferente de una jerarquía simple es que la relación entre padre e hijo **tiene atributos propios**: la cantidad del subcomponente que se necesita, la unidad de medida, y si es opcional o requerido.

```sql
CREATE TABLE componentes (
    componente_id   SERIAL          PRIMARY KEY,
    codigo          VARCHAR(50)     NOT NULL UNIQUE,
    descripcion     VARCHAR(200)    NOT NULL,
    tipo            VARCHAR(20)     NOT NULL
        CHECK (tipo IN ('MATERIA_PRIMA', 'SUBCONJUNTO', 'PRODUCTO_FINAL')),
    unidad_medida   VARCHAR(20)     NOT NULL,
    costo_unitario  DECIMAL(12,4)   NOT NULL CHECK (costo_unitario >= 0)
);

-- La estructura BOM: qué componentes forman cada componente
CREATE TABLE bom (
    bom_id          SERIAL          PRIMARY KEY,
    padre_id        INT             NOT NULL REFERENCES componentes(componente_id),
    hijo_id         INT             NOT NULL REFERENCES componentes(componente_id),
    cantidad        DECIMAL(12,4)   NOT NULL CHECK (cantidad > 0),
    unidad_medida   VARCHAR(20)     NOT NULL,
    es_opcional     BOOLEAN         NOT NULL DEFAULT FALSE,
    version_bom     INT             NOT NULL DEFAULT 1,
    vigente_desde   DATE            NOT NULL DEFAULT CURRENT_DATE,
    vigente_hasta   DATE,
    CONSTRAINT chk_no_auto_composicion CHECK (padre_id != hijo_id),
    CONSTRAINT chk_vigencia CHECK (vigente_hasta IS NULL OR vigente_hasta > vigente_desde),
    UNIQUE (padre_id, hijo_id, version_bom)
);

-- Calcular el costo total de un producto (explosión del BOM):
WITH RECURSIVE explosion_bom AS (
    -- Nivel 0: el producto raíz
    SELECT
        b.padre_id,
        b.hijo_id,
        b.cantidad,
        c.costo_unitario,
        b.cantidad * c.costo_unitario AS costo_nivel,
        1 AS nivel
    FROM bom b
    JOIN componentes c ON b.hijo_id = c.componente_id
    WHERE b.padre_id = 100  -- ID del producto final
      AND b.vigente_hasta IS NULL

    UNION ALL

    -- Niveles subsiguientes: subcomponentes de subcomponentes
    SELECT
        b.padre_id,
        b.hijo_id,
        e.cantidad * b.cantidad AS cantidad_acumulada,
        c.costo_unitario,
        e.cantidad * b.cantidad * c.costo_unitario AS costo_nivel,
        e.nivel + 1
    FROM bom b
    JOIN explosion_bom e ON b.padre_id = e.hijo_id
    JOIN componentes c ON b.hijo_id = c.componente_id
    WHERE b.vigente_hasta IS NULL
)
SELECT
    SUM(costo_nivel) AS costo_total_producto
FROM explosion_bom;
```

---

## 5. El patrón de productos y servicios configurables

### 5.1 El problema del catálogo heterogéneo

Muchos sistemas necesitan un catálogo de productos donde cada tipo de producto tiene atributos completamente diferentes:

- Una **laptop** tiene: procesador, RAM, almacenamiento, tamaño de pantalla, peso.
- Una **camiseta** tiene: talla, color, material, género.
- Un **vuelo** tiene: origen, destino, aerolínea, duración, clase.
- Un **servicio de consultoría** tiene: horas, perfil del consultor, modalidad.

No hay un conjunto común de columnas que funcione para todos los tipos. El diseño con una sola tabla y columnas para todos los atributos posibles genera una tabla con cientos de columnas casi todas NULL — el anti-patrón que describiremos en la sección 8.

### 5.2 Solución 1: herencia de tablas (TPT aplicado al catálogo)

```sql
-- Tabla base: atributos comunes a todos los productos
CREATE TABLE productos (
    producto_id     SERIAL          PRIMARY KEY,
    sku             VARCHAR(50)     NOT NULL UNIQUE,
    nombre          VARCHAR(200)    NOT NULL,
    precio_base     DECIMAL(12,2)   NOT NULL CHECK (precio_base >= 0),
    categoria_id    INT             NOT NULL REFERENCES categorias,
    activo          BOOLEAN         NOT NULL DEFAULT TRUE
);

-- Subtipo: laptops
CREATE TABLE productos_laptop (
    producto_id     INT             PRIMARY KEY REFERENCES productos ON DELETE CASCADE,
    procesador      VARCHAR(100)    NOT NULL,
    ram_gb          INT             NOT NULL CHECK (ram_gb > 0),
    almacenamiento_gb INT           NOT NULL,
    tipo_almacenamiento VARCHAR(10) NOT NULL CHECK (tipo_almacenamiento IN ('SSD', 'HDD', 'NVMe')),
    pantalla_pulgadas DECIMAL(4,1)  NOT NULL,
    peso_kg         DECIMAL(4,2)
);

-- Subtipo: ropa
CREATE TABLE productos_ropa (
    producto_id     INT             PRIMARY KEY REFERENCES productos ON DELETE CASCADE,
    talla           VARCHAR(10)     NOT NULL,
    color           VARCHAR(50)     NOT NULL,
    material        VARCHAR(100)    NOT NULL,
    genero          CHAR(1)         NOT NULL CHECK (genero IN ('M', 'F', 'U'))  -- U=unisex
);
```

**Cuándo usar este enfoque:** Cuando los tipos de producto son estables y conocidos en tiempo de diseño. Si la empresa vende solo laptops, ropa y libros, y eso no va a cambiar, TPT es correcto.

**El límite:** Si aparecen constantemente nuevos tipos de producto, hay que agregar una nueva tabla por cada tipo, lo que puede volverse inmanejable.

### 5.3 Solución 2: tabla de atributos genérica con tipos controlados

Esta solución se posiciona entre la rigidez de TPT y la flexibilidad peligrosa de EAV. Define los atributos posibles de forma centralizada pero permite asociarlos dinámicamente a los productos.

```sql
-- Definición de atributos disponibles por tipo de producto
CREATE TABLE tipos_producto (
    tipo_id         SERIAL          PRIMARY KEY,
    nombre          VARCHAR(100)    NOT NULL UNIQUE
);

CREATE TABLE definicion_atributos (
    atributo_id     SERIAL          PRIMARY KEY,
    tipo_id         INT             NOT NULL REFERENCES tipos_producto,
    nombre_atributo VARCHAR(100)    NOT NULL,
    tipo_dato       VARCHAR(20)     NOT NULL
        CHECK (tipo_dato IN ('TEXT', 'INTEGER', 'DECIMAL', 'BOOLEAN', 'DATE')),
    es_obligatorio  BOOLEAN         NOT NULL DEFAULT FALSE,
    valor_default   TEXT,
    unidad          VARCHAR(20),
    UNIQUE (tipo_id, nombre_atributo)
);

-- Productos con su tipo
CREATE TABLE productos (
    producto_id     SERIAL          PRIMARY KEY,
    tipo_id         INT             NOT NULL REFERENCES tipos_producto,
    sku             VARCHAR(50)     NOT NULL UNIQUE,
    nombre          VARCHAR(200)    NOT NULL,
    precio_base     DECIMAL(12,2)   NOT NULL
);

-- Valores de atributos por producto (controlado: solo atributos definidos)
CREATE TABLE valores_atributo (
    producto_id     INT             NOT NULL REFERENCES productos ON DELETE CASCADE,
    atributo_id     INT             NOT NULL REFERENCES definicion_atributos,
    valor_texto     TEXT,
    valor_entero    INT,
    valor_decimal   DECIMAL(12,4),
    valor_booleano  BOOLEAN,
    valor_fecha     DATE,
    PRIMARY KEY (producto_id, atributo_id),
    -- FK que garantiza que el atributo pertenece al tipo del producto:
    CONSTRAINT chk_atributo_compatible
        -- Esta constraint requiere un trigger en la práctica
        -- porque involucra múltiples tablas
    CHECK (1 = 1)  -- placeholder; ver trigger abajo
);
```

El trigger que garantiza la compatibilidad:

```sql
CREATE OR REPLACE FUNCTION verificar_compatibilidad_atributo()
RETURNS TRIGGER AS $$
DECLARE
    tipo_producto INT;
    tipo_atributo INT;
BEGIN
    -- Obtener el tipo del producto
    SELECT tipo_id INTO tipo_producto
    FROM productos WHERE producto_id = NEW.producto_id;

    -- Obtener el tipo al que pertenece el atributo
    SELECT tipo_id INTO tipo_atributo
    FROM definicion_atributos WHERE atributo_id = NEW.atributo_id;

    IF tipo_producto != tipo_atributo THEN
        RAISE EXCEPTION 'El atributo % no corresponde al tipo de producto %',
            NEW.atributo_id, tipo_producto;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_compatibilidad_atributo
    BEFORE INSERT OR UPDATE ON valores_atributo
    FOR EACH ROW EXECUTE FUNCTION verificar_compatibilidad_atributo();
```

### 5.4 Solución 3: JSONB para atributos variables (PostgreSQL)

En PostgreSQL, el tipo `JSONB` (JSON binario, indexable) ofrece una alternativa pragmática para atributos heterogéneos cuando los patrones de acceso son predecibles:

```sql
CREATE TABLE productos (
    producto_id     SERIAL          PRIMARY KEY,
    sku             VARCHAR(50)     NOT NULL UNIQUE,
    nombre          VARCHAR(200)    NOT NULL,
    precio_base     DECIMAL(12,2)   NOT NULL,
    tipo            VARCHAR(50)     NOT NULL,
    atributos       JSONB           NOT NULL DEFAULT '{}'::jsonb
);

-- Insertar una laptop:
INSERT INTO productos (sku, nombre, precio_base, tipo, atributos) VALUES
(
    'LAPTOP-001',
    'ThinkPad X1 Carbon',
    3500000,
    'LAPTOP',
    '{"procesador": "Intel i7-1365U", "ram_gb": 16, "ssd_gb": 512,
      "pantalla_pulgadas": 14.0, "peso_kg": 1.12}'::jsonb
);

-- Consultar por atributo específico:
SELECT sku, nombre, atributos->>'procesador' AS procesador
FROM productos
WHERE tipo = 'LAPTOP'
  AND (atributos->>'ram_gb')::int >= 16;

-- Índice GIN para acelerar búsquedas en JSONB:
CREATE INDEX idx_productos_atributos ON productos USING GIN (atributos);

-- Índice en un campo específico del JSON (aún más rápido para campos frecuentes):
CREATE INDEX idx_productos_ram ON productos
    USING BTREE ((atributos->>'ram_gb'));
```

**Cuándo JSONB es apropiado:**
- Los atributos son muy heterogéneos y difíciles de predecir.
- Siempre se accede al producto completo (no se filtra por atributos individuales frecuentemente).
- El equipo está dispuesto a sacrificar restricciones de tipo en atributos individuales.
- Se usa PostgreSQL (JSONB es específico de PostgreSQL; otros motores tienen JSON pero sin el mismo soporte de índices).

**Cuándo JSONB NO es apropiado:**
- Necesitas restricciones de integridad en atributos individuales.
- Haces filtros o joins frecuentes por atributos específicos.
- El equipo prefiere que el esquema documente los atributos posibles.

---

## 6. Modelos de precios con vigencia temporal

### 6.1 El problema de los precios que cambian

Los precios de casi cualquier producto o servicio cambian con el tiempo. La pregunta que el modelo debe poder responder no es solo "¿cuánto cuesta este producto ahora?", sino también:

- "¿Cuánto costaba en la fecha en que se emitió esta factura?"
- "¿Qué precio tenía la semana pasada?"
- "¿Cuándo entró en vigor el precio actual?"

Si el modelo solo guarda el precio actual (sobreescribiendo el anterior), las facturas históricas no se pueden reconstruir correctamente.

### 6.2 El anti-patrón: precio en la tabla de productos

```sql
-- Anti-patrón: precio sobreescrito
CREATE TABLE productos (
    producto_id INT PRIMARY KEY,
    nombre      VARCHAR(200),
    precio      DECIMAL(12,2) NOT NULL  -- ← se sobreescribe cuando cambia
);

-- Problema: si el precio del producto P01 cambió 5 veces este año,
-- no hay forma de saber cuánto costaba en enero.
-- Las facturas de enero quedan incorrectamente sin precio histórico.
```

### 6.3 Solución: tabla de historial de precios

```sql
CREATE TABLE productos (
    producto_id     SERIAL          PRIMARY KEY,
    sku             VARCHAR(50)     NOT NULL UNIQUE,
    nombre          VARCHAR(200)    NOT NULL
    -- SIN columna precio aquí
);

-- Historial de precios: cada cambio de precio es una nueva fila
CREATE TABLE precios (
    precio_id       SERIAL          PRIMARY KEY,
    producto_id     INT             NOT NULL REFERENCES productos,
    precio          DECIMAL(12,2)   NOT NULL CHECK (precio >= 0),
    moneda          CHAR(3)         NOT NULL DEFAULT 'COP',
    tipo_precio     VARCHAR(30)     NOT NULL DEFAULT 'LISTA'
        CHECK (tipo_precio IN ('LISTA', 'MAYORISTA', 'VIP', 'PROMOCION')),
    vigente_desde   DATE            NOT NULL,
    vigente_hasta   DATE,
    creado_por      VARCHAR(100)    NOT NULL DEFAULT CURRENT_USER,
    CONSTRAINT chk_vigencia CHECK (vigente_hasta IS NULL OR vigente_hasta > vigente_desde)
);

-- Garantizar que no haya solapamiento de precios del mismo tipo para el mismo producto:
-- (Esta restricción requiere un trigger en la mayoría de DBMS)
CREATE OR REPLACE FUNCTION verificar_solapamiento_precios()
RETURNS TRIGGER AS $$
BEGIN
    IF EXISTS (
        SELECT 1 FROM precios
        WHERE producto_id = NEW.producto_id
          AND tipo_precio = NEW.tipo_precio
          AND moneda = NEW.moneda
          AND precio_id != COALESCE(NEW.precio_id, -1)
          AND (
              -- El nuevo rango se superpone con alguno existente
              NEW.vigente_desde < COALESCE(vigente_hasta, 'infinity'::date)
              AND COALESCE(NEW.vigente_hasta, 'infinity'::date) > vigente_desde
          )
    ) THEN
        RAISE EXCEPTION 'Solapamiento de precios para el producto % tipo %',
            NEW.producto_id, NEW.tipo_precio;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tg_solapamiento_precios
    BEFORE INSERT OR UPDATE ON precios
    FOR EACH ROW EXECUTE FUNCTION verificar_solapamiento_precios();
```

### 6.4 Consultando el precio vigente

```sql
-- Precio actual de un producto:
SELECT p.sku, p.nombre, pr.precio, pr.moneda
FROM productos p
JOIN precios pr ON p.producto_id = pr.producto_id
WHERE p.producto_id = 42
  AND pr.tipo_precio = 'LISTA'
  AND pr.vigente_desde <= CURRENT_DATE
  AND (pr.vigente_hasta IS NULL OR pr.vigente_hasta > CURRENT_DATE);

-- Precio en una fecha específica (para reconstrucción histórica):
SELECT pr.precio
FROM precios pr
WHERE pr.producto_id = 42
  AND pr.tipo_precio = 'LISTA'
  AND pr.vigente_desde <= '2023-06-15'
  AND (pr.vigente_hasta IS NULL OR pr.vigente_hasta > '2023-06-15');
```

### 6.5 Precios por segmento y canal

En sistemas comerciales reales, el precio no es único: varía por tipo de cliente, canal de venta, volumen, y combinaciones de estos factores:

```sql
-- Estructura de precios por dimensiones múltiples
CREATE TABLE lista_precios (
    lista_id        SERIAL          PRIMARY KEY,
    nombre          VARCHAR(100)    NOT NULL,
    segmento        VARCHAR(50),    -- NULL = aplica a todos
    canal           VARCHAR(50),    -- NULL = aplica a todos
    prioridad       INT             NOT NULL DEFAULT 0
    -- Mayor prioridad = se aplica primero si hay múltiples listas vigentes
);

CREATE TABLE precios_lista (
    precio_id       SERIAL          PRIMARY KEY,
    lista_id        INT             NOT NULL REFERENCES lista_precios,
    producto_id     INT             NOT NULL REFERENCES productos,
    precio          DECIMAL(12,2)   NOT NULL CHECK (precio >= 0),
    descuento_pct   DECIMAL(5,2)    DEFAULT 0
        CHECK (descuento_pct BETWEEN 0 AND 100),
    -- Vigencia temporal:
    vigente_desde   DATE            NOT NULL,
    vigente_hasta   DATE,
    -- Vigencia por volumen:
    cantidad_min    INT             DEFAULT 1,
    cantidad_max    INT,
    CONSTRAINT chk_vigencia CHECK (vigente_hasta IS NULL OR vigente_hasta > vigente_desde),
    CONSTRAINT chk_cantidades CHECK (cantidad_max IS NULL OR cantidad_max >= cantidad_min)
);
```

---

## 7. Clasificaciones flexibles sin alterar el esquema

### 7.1 El problema de las clasificaciones que evolucionan

Las clasificaciones y etiquetas son una necesidad constante en sistemas de negocio:

- Productos con múltiples etiquetas.
- Clientes con segmentos y sub-segmentos.
- Documentos con categorías y sub-categorías.
- Casos de soporte con tipos, prioridades, y áreas.

El enfoque rígido de crear una columna por cada dimensión de clasificación falla cuando:
- El número de dimensiones crece con el tiempo.
- Los valores posibles de cada dimensión cambian.
- Una entidad puede tener múltiples valores de la misma dimensión (p.ej., múltiples etiquetas).

### 7.2 El patrón de taxonomías

La solución es separar la **definición** de las clasificaciones (qué dimensiones existen y qué valores tienen) de la **asignación** de esas clasificaciones a las entidades:

```sql
-- Definición de dimensiones de clasificación
CREATE TABLE dimensiones_clasificacion (
    dimension_id    SERIAL          PRIMARY KEY,
    entidad_tipo    VARCHAR(50)     NOT NULL,  -- 'PRODUCTO', 'CLIENTE', etc.
    nombre          VARCHAR(100)    NOT NULL,
    descripcion     TEXT,
    permite_multiple BOOLEAN        NOT NULL DEFAULT FALSE,
    -- ¿Puede una entidad tener múltiples valores de esta dimensión?
    es_requerida    BOOLEAN         NOT NULL DEFAULT FALSE,
    UNIQUE (entidad_tipo, nombre)
);

-- Valores posibles de cada dimensión
CREATE TABLE valores_clasificacion (
    valor_id        SERIAL          PRIMARY KEY,
    dimension_id    INT             NOT NULL REFERENCES dimensiones_clasificacion,
    codigo          VARCHAR(50)     NOT NULL,
    nombre          VARCHAR(100)    NOT NULL,
    descripcion     TEXT,
    padre_valor_id  INT             REFERENCES valores_clasificacion,  -- para jerarquías
    activo          BOOLEAN         NOT NULL DEFAULT TRUE,
    orden           INT             DEFAULT 0,
    UNIQUE (dimension_id, codigo)
);

-- Asignación de clasificaciones a entidades (usando polimorfismo de tabla)
CREATE TABLE clasificaciones_producto (
    producto_id     INT             NOT NULL REFERENCES productos,
    valor_id        INT             NOT NULL REFERENCES valores_clasificacion,
    asignado_en     TIMESTAMPTZ     NOT NULL DEFAULT NOW(),
    PRIMARY KEY (producto_id, valor_id)
);
```

### 7.3 Datos de ejemplo y consultas

```sql
-- Configuración: dimensiones para productos
INSERT INTO dimensiones_clasificacion (entidad_tipo, nombre, permite_multiple, es_requerida)
VALUES
    ('PRODUCTO', 'Categoría', FALSE, TRUE),
    ('PRODUCTO', 'Etiquetas', TRUE, FALSE),
    ('PRODUCTO', 'Estado', FALSE, TRUE),
    ('PRODUCTO', 'Mercado objetivo', TRUE, FALSE);

-- Valores para "Estado":
INSERT INTO valores_clasificacion (dimension_id, codigo, nombre)
VALUES
    (3, 'BORRADOR', 'Borrador'),
    (3, 'ACTIVO', 'Activo en venta'),
    (3, 'DESCONTINUADO', 'Descontinuado'),
    (3, 'AGOTADO', 'Agotado temporalmente');

-- Consulta: todos los productos activos con sus clasificaciones
SELECT
    p.nombre,
    MAX(vc.nombre) FILTER (WHERE dc.nombre = 'Categoría') AS categoria,
    STRING_AGG(vc.nombre, ', ') FILTER (WHERE dc.nombre = 'Etiquetas') AS etiquetas,
    MAX(vc.nombre) FILTER (WHERE dc.nombre = 'Estado') AS estado
FROM productos p
JOIN clasificaciones_producto cp ON p.producto_id = cp.producto_id
JOIN valores_clasificacion vc ON cp.valor_id = vc.valor_id
JOIN dimensiones_clasificacion dc ON vc.dimension_id = dc.dimension_id
WHERE EXISTS (
    SELECT 1 FROM clasificaciones_producto cp2
    JOIN valores_clasificacion vc2 ON cp2.valor_id = vc2.valor_id
    JOIN dimensiones_clasificacion dc2 ON vc2.dimension_id = dc2.dimension_id
    WHERE cp2.producto_id = p.producto_id
      AND dc2.nombre = 'Estado'
      AND vc2.codigo = 'ACTIVO'
)
GROUP BY p.producto_id, p.nombre;
```

---

## 8. El anti-patrón Entity-Attribute-Value (EAV)

### 8.1 Qué es EAV y por qué es tentador

El patrón **Entity-Attribute-Value (EAV)** almacena los atributos de una entidad como filas en lugar de columnas. En lugar de:

```sql
-- Modelo relacional normal
CREATE TABLE productos (
    id      INT PRIMARY KEY,
    nombre  VARCHAR(200),
    precio  DECIMAL,
    color   VARCHAR(50),
    talla   VARCHAR(10)
);
INSERT INTO productos VALUES (1, 'Camiseta', 29.99, 'Rojo', 'M');
```

EAV almacena:

```sql
-- El anti-patrón EAV
CREATE TABLE entidades (id INT PRIMARY KEY, tipo VARCHAR(50));
CREATE TABLE atributos (id INT PRIMARY KEY, nombre VARCHAR(100), tipo_dato VARCHAR(20));
CREATE TABLE valores_eav (
    entidad_id  INT REFERENCES entidades,
    atributo_id INT REFERENCES atributos,
    valor       TEXT,  -- todos los valores como texto
    PRIMARY KEY (entidad_id, atributo_id)
);

-- Datos:
INSERT INTO entidades VALUES (1, 'PRODUCTO');
INSERT INTO atributos VALUES (1, 'nombre', 'TEXT'), (2, 'precio', 'DECIMAL'), ...;
INSERT INTO valores_eav VALUES (1, 1, 'Camiseta'), (1, 2, '29.99'), ...;
```

La tentación de EAV es real: parece que resuelve el problema de atributos heterogéneos para siempre. Puedes agregar cualquier atributo a cualquier entidad sin modificar el esquema.

### 8.2 Por qué EAV es un error de diseño

EAV sacrifica **todo** lo que hace valioso al modelo relacional:

**1. No hay tipos de dato:**
Todo se almacena como `TEXT`. No puedes poner `CHECK (precio > 0)` porque el precio es un texto. Las comparaciones numéricas y de fecha requieren casteos que son lentos y propensos a errores.

```sql
-- Para consultar productos con precio > 100 en EAV:
SELECT e.id
FROM entidades e
JOIN valores_eav v ON e.id = v.entidad_id
JOIN atributos a ON v.atributo_id = a.id
WHERE a.nombre = 'precio'
  AND v.valor::DECIMAL > 100;  -- cast explícito, puede fallar si hay datos sucios
```

**2. No hay restricciones de integridad:**
- No puedes hacer `NOT NULL` en un atributo específico.
- No puedes garantizar que el atributo "precio" exista para todos los productos.
- No puedes poner UNIQUE en "email" de una entidad.

**3. Las queries son pesadillas:**
Recuperar un "registro completo" de una entidad requiere un `PIVOT` manual o múltiples JOINs:

```sql
-- Obtener nombre, precio y color de un producto en EAV (el famoso "EAV pivot"):
SELECT
    MAX(CASE WHEN a.nombre = 'nombre' THEN v.valor END) AS nombre,
    MAX(CASE WHEN a.nombre = 'precio' THEN v.valor::DECIMAL END) AS precio,
    MAX(CASE WHEN a.nombre = 'color' THEN v.valor END) AS color
FROM entidades e
JOIN valores_eav v ON e.id = v.entidad_id
JOIN atributos a ON v.atributo_id = a.id
WHERE e.id = 1
GROUP BY e.id;
```

vs en un modelo normal:
```sql
SELECT nombre, precio, color FROM productos WHERE id = 1;
```

**4. El rendimiento es terrible a escala:**
Una entidad con 20 atributos en EAV genera 20 filas. Si tienes 1 millón de productos, son 20 millones de filas en la tabla EAV. Las queries que recuperan múltiples atributos de múltiples productos hacen cruces de datos enormes.

**5. El esquema no documenta el modelo:**
Mirando el esquema EAV, es imposible saber qué atributos tiene un producto. La documentación del modelo está en los datos, no en la estructura.

### 8.3 Cuándo EAV es la única solución pragmática

A pesar de sus problemas, hay escenarios donde EAV (o una variante de él) es la solución más razonable disponible:

**Formularios dinámicos configurables por el usuario final:**
Un sistema de formularios donde los usuarios pueden agregar sus propios campos personalizados. No puedes alterar el esquema cada vez que un usuario agrega un campo.

**Metadatos arbitrarios:**
Almacenar metadatos de configuración o propiedades extendidas que son completamente libres en estructura y que no se usarán en queries de filtrado.

**Integración con sistemas legacy:**
Algunos sistemas legacy exportan datos en formato EAV. Puede ser necesario recibirlos así.

**Datos de configuración dispersos:**
Tablas del tipo `parametros_sistema(clave, valor)` para configuración de la aplicación son técnicamente EAV pero su uso es limitado y controlado.

### 8.4 Lo que se sacrifica al usar EAV

Si decides usar EAV en alguna de estas situaciones, debes ser completamente consciente del costo:

```
Se pierde:
├── Tipos de dato → todo es texto, validaciones manuales
├── NOT NULL por atributo → ausencia de atributos no detectada automáticamente
├── UNIQUE por atributo → duplicados posibles si no se controla manualmente
├── Índices simples → los índices EAV son costosos y limitados
├── Legibilidad del esquema → el modelo no se puede leer del esquema
├── Herramientas de ORM → la mayoría no entienden EAV nativamente
└── Rendimiento en queries analíticas → los PIVOTs son costosos
```

### 8.5 Alternativas modernas a EAV

En lugar de EAV, considera:

| Necesidad | Alternativa a EAV |
|---|---|
| Atributos heterogéneos por tipo | Herencia de tablas (TPT) |
| Atributos realmente dinámicos | JSONB (PostgreSQL) |
| Formularios configurables | Tabla de definición de campos + tabla de respuestas |
| Metadatos extensibles | JSONB o tabla auxiliar con hstore |
| Configuración del sistema | Tabla `configuracion(clave VARCHAR PK, valor TEXT, tipo VARCHAR)` con validación por tipo |

---

## 9. Resumen y conexión con el resto del curso

### Los patrones vistos y cuándo aplicarlos

| Patrón | Problema que resuelve | Señal de que lo necesitas |
|---|---|---|
| **Party** | Múltiples tipos de actores (personas/empresas) en los mismos procesos | "Nuestros clientes pueden ser personas o empresas" |
| **Roles** | Una entidad con comportamientos diferentes según el contexto | "Un usuario puede ser cliente y empleado al mismo tiempo" |
| **Jerarquía recursiva** | Estructuras de profundidad variable | "Las categorías tienen subcategorías de subcategorías" |
| **Bill of Materials** | Productos compuestos de subproductos | "Necesitamos explotar los componentes de un producto" |
| **Productos configurables** | Catálogo con tipos de atributos heterogéneos | "Cada tipo de producto tiene atributos completamente distintos" |
| **Precios con vigencia** | Valores que cambian con el tiempo y necesitan historial | "¿Cuánto costaba este producto cuando se emitió esta factura?" |
| **Clasificaciones flexibles** | Múltiples dimensiones de categorización que evolucionan | "Queremos poder agregar nuevas etiquetas sin tocar el esquema" |
| **EAV** | Atributos completamente arbitrarios definidos por el usuario | Solo como último recurso, con plena consciencia del costo |

### La meta-lección del módulo

Los patrones de modelado no son recetas. Son soluciones conocidas a problemas conocidos, cada una con trade-offs específicos. El valor real de conocerlos no es aplicarlos mecánicamente, sino:

1. **Reconocer el problema** antes de empezar a dibujar tablas.
2. **Saber el abanico de soluciones** disponibles y sus consecuencias.
3. **Elegir conscientemente** según las restricciones del dominio específico.
4. **Comunicar la decisión** al equipo usando un vocabulario compartido.

Un modelo que usa el patrón Party cuando el dominio lo requiere es robusto ante nuevos tipos de actores. Un modelo que usa una jerarquía recursiva con closure table cuando las lecturas dominan es eficiente. Un modelo que usa EAV porque "era flexible" es técnicamente deuda que alguien pagará con intereses.

### Conexión con módulos futuros

- **Módulo 6 (Diseño para el tiempo):** Los precios con vigencia temporal son una introducción al tema completo del módulo siguiente: cómo modelar datos que cambian con el tiempo manteniendo el historial completo. Tiempo válido vs tiempo de transacción, tablas bitemporales, y Slowly Changing Dimensions son la extensión formal de lo que vimos en la sección de precios.

- **Módulo 7 (Internals y query planner):** Los CTEs recursivos para jerarquías, las queries de EAV con PIVOTs, y las queries de clasificación con múltiples JOINs tienen comportamientos de rendimiento muy distintos. Entender el query planner es esencial para saber si el modelo elegido es eficiente en producción.

- **Módulo 12 (Modelos de datos alternativos):** Las bases de datos de grafos son la alternativa natural para jerarquías profundas y redes complejas de relaciones. Las bases de datos de documentos son la alternativa natural para productos con atributos heterogéneos. Entender cuándo el modelo relacional es la solución correcta y cuándo un modelo alternativo resuelve el problema de forma más natural es el tema de ese módulo.

- **Módulo 15 (ORMs):** Los patrones como Party, roles, y jerarquías tienen implicaciones específicas para cómo los ORMs los representan. El patrón N+1 aparece con frecuencia al traversar jerarquías con ORMs que usan lazy loading.

---

## Ejercicios de comprensión

**Ejercicio 1.** Una empresa de logística necesita modelar sus "actores": clientes (que pueden ser personas naturales o empresas), transportistas (siempre son empresas), y conductores (siempre son personas). Los clientes y transportistas firman contratos con la empresa. Los conductores trabajan para los transportistas.

a) ¿Aplica el patrón Party? ¿Para qué tipos de actores?
b) ¿Aplica el patrón de roles? ¿Para qué casos?
c) Diseña el esquema relacional con las restricciones de integridad correctas.

**Ejercicio 2.** Dado el siguiente árbol de categorías de una tienda online:

```
Hogar
├── Cocina
│   ├── Electrodomésticos pequeños
│   └── Utensilios
├── Baño
└── Dormitorio
    └── Ropa de cama
        ├── Sábanas
        └── Cobijas
```

a) Implementa el modelo de lista de adyacencia.
b) Escribe la query CTE recursiva que devuelve el "breadcrumb" completo de "Sábanas" (p.ej. "Hogar > Dormitorio > Ropa de cama > Sábanas").
c) ¿Qué modelo alternativo (Nested Set, Materialized Path, Closure Table) elegirías si el árbol tiene 50,000 nodos y se leen subárboles completos 10,000 veces por día pero se modifican solo 5 veces? Justifica.

**Ejercicio 3.** Tienes un sistema de e-commerce con tres tipos de productos: libros (con ISBN, autor, editorial, número de páginas), electrónicos (con voltaje, garantía_años, peso_kg), y alimentos (con peso_g, calorías_por_100g, ingredientes, fecha_vencimiento).

a) Diseña el esquema usando TPT (herencia de tablas).
b) Diseña el esquema alternativo usando JSONB para los atributos específicos.
c) ¿En qué condiciones preferirías cada enfoque?

**Ejercicio 4.** Una tienda tiene la siguiente estructura de precios:
- Precio de lista (para todos los clientes).
- Precio mayorista (para clientes con más de 100 compras históricas).
- Precio de promoción (válido solo en ciertas fechas).
- Precio por volumen (descuento escalonado: 5% por 10+ unidades, 10% por 50+ unidades).

Diseña el modelo de precios completo que pueda representar todos estos tipos, incluyendo las queries para obtener el precio aplicable a una compra específica.

**Ejercicio 5.** Evalúa el siguiente diseño que usa EAV e identifica los problemas concretos. Luego propón un diseño alternativo:

```sql
CREATE TABLE fichas_tecnicas (
    ficha_id    INT PRIMARY KEY,
    producto_id INT NOT NULL,
    atributo    VARCHAR(100) NOT NULL,
    valor       TEXT
);
-- Datos:
-- (1, 100, 'voltaje', '110V')
-- (2, 100, 'garantia', '2')
-- (3, 100, 'peso', '1.5')
-- (4, 101, 'isbn', '9780000000000')
-- (5, 101, 'paginas', '350')
-- (6, 101, 'autor', 'García Márquez')
```

---

*Próximo módulo: Diseño para el tiempo — donde exploraremos cómo modelar la dimensión que más frecuentemente se ignora en el diseño inicial: el hecho de que los datos del mundo real cambian, y que muchas veces necesitamos saber no solo cómo son las cosas ahora, sino cómo eran en cualquier momento del pasado.*
