# Módulo 15 — ORMs, impedancia objeto-relacional y SQL a mano

> *"Un ORM bien usado es una herramienta de productividad legítima. Un ORM mal usado es un generador automático de bugs de rendimiento que son invisibles en desarrollo y catastróficos en producción. La diferencia entre los dos no está en el ORM; está en si el desarrollador entiende qué SQL genera el ORM y por qué. El ORM no dispensa al desarrollador de entender la base de datos: lo obliga a entenderla dos veces, porque ahora debe entender tanto la API del ORM como el SQL que produce."*

---

## Tabla de contenidos

1. [La impedancia objeto-relacional en profundidad](#1-la-impedancia-objeto-relacional-en-profundidad)
2. [Qué es un ORM y qué garantiza](#2-qué-es-un-orm-y-qué-garantiza)
3. [Herencia en objetos: el problema sin solución perfecta](#3-herencia-en-objetos-el-problema-sin-solución-perfecta)
4. [Table Per Hierarchy (TPH / Single Table Inheritance)](#4-table-per-hierarchy-tph--single-table-inheritance)
5. [Table Per Type (TPT / Class Table Inheritance)](#5-table-per-type-tpt--class-table-inheritance)
6. [Table Per Concrete Class (TPC / Concrete Table Inheritance)](#6-table-per-concrete-class-tpc--concrete-table-inheritance)
7. [Cuándo elegir cada estrategia de herencia](#7-cuándo-elegir-cada-estrategia-de-herencia)
8. [El problema N+1: la trampa más frecuente de los ORMs](#8-el-problema-n1-la-trampa-más-frecuente-de-los-orms)
9. [Estrategias de fetching: cómo controlar qué carga el ORM](#9-estrategias-de-fetching-cómo-controlar-qué-carga-el-orm)
10. [Lazy loading fuera de sesión: el error más frustrante](#10-lazy-loading-fuera-de-sesión-el-error-más-frustrante)
11. [Transacciones implícitas y el comportamiento oculto del ORM](#11-transacciones-implícitas-y-el-comportamiento-oculto-del-orm)
12. [Cuándo el ORM genera SQL eficiente y cuándo no](#12-cuándo-el-orm-genera-sql-eficiente-y-cuándo-no)
13. [Cuándo escribir SQL a mano es inevitable](#13-cuándo-escribir-sql-a-mano-es-inevitable)
14. [La decisión: ORM, query builder, o SQL puro](#14-la-decisión-orm-query-builder-o-sql-puro)
15. [Resumen y conexión con el resto del curso](#15-resumen-y-conexión-con-el-resto-del-curso)
16. [Ejercicios de comprensión](#16-ejercicios-de-comprensión)

---

## 1. La impedancia objeto-relacional en profundidad

### 1.1 Revisión y profundización

El módulo 14 introdujo la impedancia objeto-relacional como motivación para los patrones de Fowler. En este módulo la examinamos más a fondo, porque los problemas específicos de los ORMs son consecuencias directas de cada dimensión de esa impedancia. Entender el problema en detalle es el prerequisito para entender por qué los ORMs fallan de la forma en que fallan.

Las dimensiones de la impedancia son cinco, y cada una genera una clase de problema diferente:

### 1.2 Primera dimensión: identidad

**El problema:** En el mundo relacional, la identidad de una fila está definida inequívocamente por su clave primaria. Dos filas son "el mismo registro" si y solo si tienen el mismo valor de PK. En el mundo de objetos hay dos conceptos de identidad que coexisten:

- **Identidad de referencia** (`is` en Python, `==` para referencias en Java): dos variables son el mismo objeto si apuntan a la misma dirección en memoria.
- **Igualdad de valor** (`==` con `__eq__` en Python, `.equals()` en Java): dos objetos son "iguales" si tienen los mismos valores de campos relevantes.

```python
user_a = User(id=42, name="María")
user_b = User(id=42, name="María")

user_a is user_b       # False: distintas instancias en memoria
user_a == user_b       # Depende de cómo esté implementado __eq__
                       # Si __eq__ compara id, entonces True
                       # Si __eq__ es el default (identidad de referencia), False

# Problema concreto:
session.add(user_a)
session.add(user_b)  # ¿Es esto un error (duplicado) o son dos entidades distintas?
                     # El ORM debe decidir basándose en el id, no en la identidad de referencia
```

El ORM resuelve esto con el **Identity Map** (módulo 14): garantiza que para cualquier PK, siempre existe una única instancia en memoria dentro de la sesión. Pero esto crea su propio problema: el alcance del Identity Map es la sesión, y cuando la sesión termina, las instancias se vuelven "detached" (desconectadas).

### 1.3 Segunda dimensión: relaciones y su dirección

**El problema:** En el modelo relacional, las relaciones son simétricas y bidireccionales por naturaleza. La FK en `orders.customer_id` no tiene una "dirección": puedo navegar desde una orden hacia su cliente con un JOIN, o desde un cliente hacia sus órdenes con otro JOIN. La tabla de FK no impone dirección.

En el modelo de objetos, las referencias tienen dirección. Si `Order` tiene un campo `customer: Customer`, puedo navegar de `Order` hacia `Customer`. Pero para navegar de `Customer` hacia sus órdenes, necesito un campo `orders: List[Order]` en `Customer`. Son dos referencias independientes que el ORM tiene que mantener sincronizadas.

```python
# En SQLAlchemy:
class Customer(Base):
    __tablename__ = 'customers'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    # Relación bidireccional: Customer conoce sus Orders
    orders = relationship("Order", back_populates="customer")

class Order(Base):
    __tablename__ = 'orders'
    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, ForeignKey('customers.id'))
    # Relación bidireccional: Order conoce su Customer
    customer = relationship("Customer", back_populates="orders")
```

Esta bidireccionalidad es conveniente pero introduce un problema sutil: cuando se modifica la relación desde un lado, el ORM debe actualizar el otro lado también para mantener la coherencia en memoria. Si no lo hace correctamente, hay inconsistencias en el grafo de objetos en memoria aunque la base de datos esté correcta.

```python
order = Order(customer_id=42)
session.add(order)

# Si accedo a customer.orders ANTES de hacer flush/commit:
customer = session.get(Customer, 42)
print(customer.orders)  # ¿Incluye la nueva order?
                        # Depende de cómo el ORM gestiona la coherencia
                        # del grafo en memoria vs. lo que hay en la DB.
```

### 1.4 Tercera dimensión: navegación vs. joins

**El problema:** En el código de objetos, navegar por el grafo es natural e intuitivo:

```python
# Código de dominio natural en objetos:
total_revenue = sum(
    item.unit_price * item.quantity
    for customer in active_customers
    for order in customer.orders
    if order.status == 'completed'
    for item in order.items
)
```

En el modelo relacional, esta misma operación requiere un JOIN de cuatro tablas. La navegación en código produce múltiples queries implícitas (el N+1) a menos que el ORM sepa de antemano que se necesitarán las relaciones (eager loading). Esta es la tensión central que los ORMs resuelven mal por defecto.

### 1.5 Cuarta dimensión: granularidad

**El problema:** El modelo de objetos tiene una granularidad fina: un `Address` puede ser un objeto separado con sus propios métodos, aunque en la base de datos sea solo un conjunto de columnas en la tabla `customers`.

```python
# Modelo de objetos: Address como value object separado
class Address:
    def __init__(self, street: str, city: str, zip_code: str, country: str):
        self.street = street
        self.city = city
        self.zip_code = zip_code
        self.country = country
    
    def format_for_label(self) -> str:
        return f"{self.street}\n{self.city}, {self.zip_code}\n{self.country}"

class Customer:
    def __init__(self, id: int, name: str, address: Address):
        self.id = id
        self.name = name
        self.address = address  # Un objeto, no columnas separadas
```

```sql
-- Modelo relacional: no hay un objeto Address; son columnas de customers
CREATE TABLE customers (
    id          BIGINT PRIMARY KEY,
    name        TEXT NOT NULL,
    street      TEXT,
    city        TEXT,
    zip_code    TEXT,
    country     TEXT
);
```

El ORM tiene que mapear un objeto `Address` a columnas de `customers`. Esta técnica se llama **Embedded Value** (Valor embebido): un objeto de valor que se almacena como columnas de la tabla del objeto que lo contiene.

### 1.6 Quinta dimensión: herencia

**El problema:** La herencia es un pilar del modelo de objetos. La herencia no existe en el modelo relacional. Esta es la dimensión más compleja de la impedancia y ocupa las siguientes cuatro secciones de este módulo.

---

## 2. Qué es un ORM y qué garantiza

### 2.1 Definición precisa

Un **ORM (Object-Relational Mapper)** es un framework que implementa automáticamente los patrones de acceso a datos del módulo 14 —Data Mapper, Unit of Work, Identity Map, Lazy Load— usando **metadatos** (anotaciones, decoradores, archivos de configuración, o convenciones) para conocer la correspondencia entre clases y tablas.

En lugar de escribir un `UserMapper` a mano, el ORM genera ese mapper a partir de la declaración de la clase:

```python
# Con SQLAlchemy (ORM de Python):
# En lugar de escribir UserMapper a mano, se declara la clase con anotaciones:

from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column, relationship

class Base(DeclarativeBase):
    pass

class User(Base):
    __tablename__ = 'users'
    
    # Mapeo de columnas
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100), nullable=False)
    email: Mapped[str] = mapped_column(String(200), nullable=False, unique=True)
    credit_limit: Mapped[float] = mapped_column(Numeric(10, 2), default=0.0)
    active: Mapped[bool] = mapped_column(Boolean, default=True)
    
    # Relación lazy (por defecto): no carga orders hasta que se acceda
    orders: Mapped[list['Order']] = relationship(back_populates='customer')
    
    # Lógica de dominio: el User sigue siendo un objeto de dominio
    def can_place_order(self, amount: float) -> bool:
        return self.active and self.credit_limit >= amount
    
    def upgrade_to_premium(self) -> None:
        if not self.active:
            raise ValueError("No se puede actualizar un usuario inactivo.")
        self.credit_limit = max(self.credit_limit, 10000.0)
        # El ORM detectará el cambio en credit_limit automáticamente (change tracking)
```

El ORM genera el `UserMapper` correspondiente de forma automática: sabe que `id` mapea a `users.id`, que `name` mapea a `users.name`, etc.

### 2.2 Qué hace el ORM por nosotros

El ORM maneja automáticamente:

- **SQL de CRUD:** Genera los `INSERT`, `UPDATE`, `DELETE`, `SELECT` correspondientes.
- **Change tracking:** Detecta qué propiedades cambiaron y genera solo los `UPDATE` necesarios.
- **Unit of Work:** Agrupa todos los cambios de la sesión en una transacción.
- **Identity Map:** Garantiza que la misma PK siempre produce la misma instancia en memoria durante la sesión.
- **Gestión de relaciones:** Genera los JOINs o las queries adicionales para cargar objetos relacionados.
- **Mapeo de tipos:** Convierte entre tipos Python/Java y tipos SQL (e.g., `datetime` Python ↔ `TIMESTAMP` SQL).

### 2.3 Qué no hace el ORM

Lo que el ORM **no resuelve** automáticamente (y que es la fuente de casi todos los problemas de rendimiento):

- **No sabe cuándo cargar relaciones:** Solo sabe que debe cargarlas cuando se accede a ellas (lazy) o siempre (eager). La decisión de cuándo hacer un JOIN o cuándo hacer queries separadas es tuya.
- **No optimiza queries complejas:** Una query con múltiples JOINs, subconsultas, window functions, o agregaciones complejas es difícil de expresar eficientemente en la API del ORM. Tiende a generar SQL subóptimo para estas queries.
- **No gestiona la vida de las sesiones:** Es tu responsabilidad abrir, cerrar y delimitar las sesiones correctamente.
- **No previene el N+1:** El lazy loading por defecto es la raíz del N+1. El ORM no detecta ni advierte sobre este antipatrón.

---

## 3. Herencia en objetos: el problema sin solución perfecta

### 3.1 El problema fundamental

Consideremos una jerarquía de herencia típica en un sistema de pagos:

```python
# Jerarquía de objetos
class Payment:
    """Pago base: atributos comunes a todo tipo de pago."""
    def __init__(self, id, amount, currency, status, created_at):
        self.id = id
        self.amount = amount
        self.currency = currency
        self.status = status
        self.created_at = created_at
    
    def can_be_refunded(self) -> bool:
        return self.status == 'completed'

class CreditCardPayment(Payment):
    """Pago con tarjeta de crédito."""
    def __init__(self, *args, card_last4, card_brand, authorization_code):
        super().__init__(*args)
        self.card_last4 = card_last4
        self.card_brand = card_brand
        self.authorization_code = authorization_code
    
    def get_receipt_description(self) -> str:
        return f"{self.card_brand} terminada en {self.card_last4}"

class BankTransferPayment(Payment):
    """Pago por transferencia bancaria."""
    def __init__(self, *args, bank_name, account_last4, transfer_reference):
        super().__init__(*args)
        self.bank_name = bank_name
        self.account_last4 = account_last4
        self.transfer_reference = transfer_reference
    
    def get_receipt_description(self) -> str:
        return f"Transferencia desde {self.bank_name}"

class CryptoPayment(Payment):
    """Pago con criptomoneda."""
    def __init__(self, *args, wallet_address, transaction_hash, coin):
        super().__init__(*args)
        self.wallet_address = wallet_address
        self.transaction_hash = transaction_hash
        self.coin = coin
    
    def get_receipt_description(self) -> str:
        return f"Pago en {self.coin}"
```

El problema es: ¿cómo persistir esta jerarquía en una base de datos relacional que no tiene herencia? Hay exactamente tres estrategias, y ninguna es perfecta.

---

## 4. Table Per Hierarchy (TPH / Single Table Inheritance)

### 4.1 La idea

En **Table Per Hierarchy** (TPH), también llamada **Single Table Inheritance** (STI), toda la jerarquía de herencia se mapea a **una sola tabla**. La tabla contiene columnas para los atributos de todas las clases de la jerarquía. Una columna especial, el **discriminador**, indica el tipo concreto de cada fila.

```sql
-- Una sola tabla para toda la jerarquía
CREATE TABLE payments (
    -- Columnas comunes a todos los pagos
    id                  BIGSERIAL PRIMARY KEY,
    payment_type        TEXT NOT NULL,       -- DISCRIMINADOR: 'credit_card', 'bank_transfer', 'crypto'
    amount              NUMERIC(12,2) NOT NULL,
    currency            CHAR(3) NOT NULL,
    status              TEXT NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    
    -- Columnas específicas de CreditCardPayment (NULL para otros tipos)
    card_last4          CHAR(4),
    card_brand          TEXT,
    authorization_code  TEXT,
    
    -- Columnas específicas de BankTransferPayment (NULL para otros tipos)
    bank_name           TEXT,
    account_last4       CHAR(4),
    transfer_reference  TEXT,
    
    -- Columnas específicas de CryptoPayment (NULL para otros tipos)
    wallet_address      TEXT,
    transaction_hash    TEXT,
    coin                TEXT
);
```

### 4.2 El discriminador

La columna `payment_type` es el **discriminador**: le dice al ORM qué clase concreta instanciar cuando carga una fila.

```python
# SQLAlchemy — configuración TPH
class Payment(Base):
    __tablename__ = 'payments'
    
    id = Column(BigInteger, primary_key=True)
    payment_type = Column(String, nullable=False)  # discriminador
    amount = Column(Numeric(12, 2), nullable=False)
    currency = Column(String(3), nullable=False)
    status = Column(String, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    
    __mapper_args__ = {
        'polymorphic_on': payment_type,    # columna discriminadora
        'polymorphic_identity': 'payment', # valor del discriminador para esta clase
    }

class CreditCardPayment(Payment):
    card_last4 = Column(String(4))
    card_brand = Column(String)
    authorization_code = Column(String)
    
    __mapper_args__ = {
        'polymorphic_identity': 'credit_card',  # valor del discriminador
    }

class BankTransferPayment(Payment):
    bank_name = Column(String)
    account_last4 = Column(String(4))
    transfer_reference = Column(String)
    
    __mapper_args__ = {
        'polymorphic_identity': 'bank_transfer',
    }

class CryptoPayment(Payment):
    wallet_address = Column(String)
    transaction_hash = Column(String)
    coin = Column(String)
    
    __mapper_args__ = {
        'polymorphic_identity': 'crypto',
    }
```

### 4.3 Queries con TPH

La gran ventaja del TPH es la simplicidad de las queries:

```python
# Cargar un pago por ID: una sola query, sin JOINs
payment = session.get(Payment, 123)
# SQL generado: SELECT * FROM payments WHERE id = 123
# El ORM lee payment_type y devuelve la instancia del tipo correcto

# Cargar todos los pagos: una sola query
all_payments = session.query(Payment).all()
# SQL: SELECT * FROM payments

# Cargar solo tarjetas: una sola query con filtro en el discriminador
card_payments = session.query(CreditCardPayment).all()
# SQL: SELECT * FROM payments WHERE payment_type = 'credit_card'

# Polymorfismo en código: funciona transparentemente
for payment in all_payments:
    print(payment.get_receipt_description())  # Cada tipo ejecuta su propio método
```

### 4.4 Ventajas y desventajas del TPH

**Ventajas:**

- **Queries simples:** Todo el polimorfismo se resuelve en una sola tabla. No hay JOINs para queries que atraviesan la jerarquía.
- **Fácil de agregar subtipos:** Añadir una nueva subclase solo requiere agregar columnas a la tabla (que son NULL para los registros existentes) y una nueva clase con `polymorphic_identity`.
- **Rendimiento de lectura excelente:** Una sola página de datos puede contener filas de distintos tipos. El acceso por índice de PK es siempre en una sola tabla.

**Desventajas:**

- **Muchos NULLs:** Los atributos específicos de cada subtipo son `NULL` para todos los demás subtipos. En una tabla con 10 subtipos y 5 atributos específicos por subtipo, hay 45 columnas que son `NULL` para el 90% de las filas.
- **No se pueden declarar NOT NULL en columnas de subtipo:** `card_last4` debería ser `NOT NULL` para pagos con tarjeta, pero como la columna también existe (como NULL) para transferencias y cripto, no se puede declarar `NOT NULL` a nivel de tabla. La integridad debe enforcearse en la aplicación.
- **La tabla crece horizontalmente:** Cada nuevo subtipo con nuevos atributos añade más columnas. La tabla puede llegar a tener decenas de columnas con nombres crípticos como `attr_12` o `field_type_3`.
- **Contaminación semántica:** La tabla `payments` contiene columnas sobre cuentas bancarias, wallets de cripto y tarjetas, mezcladas. Es difícil entender el esquema sin conocer la jerarquía de clases.

---

## 5. Table Per Type (TPT / Class Table Inheritance)

### 5.1 La idea

En **Table Per Type** (TPT), también llamada **Class Table Inheritance**, existe una tabla por cada clase en la jerarquía. La tabla base contiene los atributos comunes. Cada tabla de subtipo contiene solo los atributos específicos de ese subtipo, y tiene una FK hacia la tabla base con el mismo ID.

```sql
-- Tabla base: solo atributos comunes
CREATE TABLE payments (
    id          BIGSERIAL PRIMARY KEY,
    payment_type TEXT NOT NULL,  -- discriminador (sigue siendo necesario)
    amount      NUMERIC(12,2) NOT NULL,
    currency    CHAR(3) NOT NULL,
    status      TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tabla de subtipo: solo atributos específicos de tarjeta
CREATE TABLE credit_card_payments (
    payment_id         BIGINT PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
    card_last4         CHAR(4) NOT NULL,     -- ahora SÍ puede ser NOT NULL
    card_brand         TEXT NOT NULL,
    authorization_code TEXT NOT NULL
);

-- Tabla de subtipo: solo atributos específicos de transferencia
CREATE TABLE bank_transfer_payments (
    payment_id         BIGINT PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
    bank_name          TEXT NOT NULL,
    account_last4      CHAR(4) NOT NULL,
    transfer_reference TEXT NOT NULL
);

-- Tabla de subtipo: solo atributos específicos de cripto
CREATE TABLE crypto_payments (
    payment_id        BIGINT PRIMARY KEY REFERENCES payments(id) ON DELETE CASCADE,
    wallet_address    TEXT NOT NULL,
    transaction_hash  TEXT NOT NULL,
    coin              TEXT NOT NULL
);
```

### 5.2 Configuración en el ORM

```python
# SQLAlchemy — configuración TPT
class Payment(Base):
    __tablename__ = 'payments'
    
    id = Column(BigInteger, primary_key=True)
    payment_type = Column(String, nullable=False)
    amount = Column(Numeric(12, 2), nullable=False)
    currency = Column(String(3), nullable=False)
    status = Column(String, nullable=False)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    
    __mapper_args__ = {
        'polymorphic_on': payment_type,
        'polymorphic_identity': 'payment',
    }

class CreditCardPayment(Payment):
    __tablename__ = 'credit_card_payments'
    
    payment_id = Column(BigInteger, ForeignKey('payments.id'), primary_key=True)
    card_last4 = Column(String(4), nullable=False)
    card_brand = Column(String, nullable=False)
    authorization_code = Column(String, nullable=False)
    
    __mapper_args__ = {
        'polymorphic_identity': 'credit_card',
    }
```

### 5.3 El SQL que genera: el problema del JOIN

Cargar un `CreditCardPayment` con TPT requiere un JOIN entre `payments` y `credit_card_payments`:

```sql
-- Cargar un pago por ID (TPT):
SELECT payments.id, payments.payment_type, payments.amount, 
       payments.currency, payments.status, payments.created_at,
       credit_card_payments.card_last4, credit_card_payments.card_brand,
       credit_card_payments.authorization_code
FROM payments
LEFT OUTER JOIN credit_card_payments ON credit_card_payments.payment_id = payments.id
WHERE payments.id = 123;
```

Cargar cualquier pago polimórfico (sin saber el tipo de antemano) requiere JOINs con TODAS las tablas de subtipo:

```sql
-- Cargar todos los pagos (polimórfico, TPT):
SELECT payments.*,
       credit_card_payments.card_last4, credit_card_payments.card_brand, 
       credit_card_payments.authorization_code,
       bank_transfer_payments.bank_name, bank_transfer_payments.account_last4,
       bank_transfer_payments.transfer_reference,
       crypto_payments.wallet_address, crypto_payments.transaction_hash,
       crypto_payments.coin
FROM payments
LEFT OUTER JOIN credit_card_payments ON credit_card_payments.payment_id = payments.id
LEFT OUTER JOIN bank_transfer_payments ON bank_transfer_payments.payment_id = payments.id
LEFT OUTER JOIN crypto_payments ON crypto_payments.payment_id = payments.id;
```

Con 3 subtipos esto es manejable. Con 10 subtipos, la query tiene 10 LEFT JOINs. El rendimiento puede degradarse significativamente, especialmente si cada subtipo tiene muchas columnas.

### 5.4 Ventajas y desventajas del TPT

**Ventajas:**

- **Sin NULLs estructurales:** Cada tabla solo tiene las columnas que le corresponden. `card_last4` puede ser `NOT NULL` en `credit_card_payments`.
- **Esquema comprensible:** Cada tabla tiene una semántica clara. Es posible leer el esquema y entender la jerarquía sin conocer el código.
- **Espacio en disco más eficiente:** Sin columnas NULL que ocupan espacio.
- **Integridad a nivel de base de datos:** Las constraints `NOT NULL` y `CHECK` se pueden aplicar correctamente en cada tabla de subtipo.

**Desventajas:**

- **JOINs inevitables:** Cualquier query polimórfica (que carga la clase base sin saber el subtipo) requiere JOIN con todas las tablas de subtipo. A medida que crece la jerarquía, el rendimiento de estas queries se degrada.
- **Escritura más costosa:** Crear un `CreditCardPayment` requiere un INSERT en `payments` y luego un INSERT en `credit_card_payments`. Dos operaciones.
- **Consultas específicas de subtipo son eficientes:** `SELECT * FROM credit_card_payments JOIN payments ON ...` solo involucra dos tablas. El problema es solo con queries polimórficas.

---

## 6. Table Per Concrete Class (TPC / Concrete Table Inheritance)

### 6.1 La idea

En **Table Per Concrete Class** (TPC), también llamada **Concrete Table Inheritance**, solo existen tablas para las **clases concretas** (no abstractas). No hay tabla para la clase base. Cada tabla contiene todos los atributos de su clase, incluyendo los heredados de la clase base.

```sql
-- NO hay tabla payments (la clase base)

-- Tabla para CreditCardPayment: tiene TODOS los campos (comunes + específicos)
CREATE TABLE credit_card_payments (
    id                 BIGSERIAL PRIMARY KEY,
    amount             NUMERIC(12,2) NOT NULL,  -- heredado de Payment
    currency           CHAR(3) NOT NULL,          -- heredado
    status             TEXT NOT NULL,             -- heredado
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),  -- heredado
    card_last4         CHAR(4) NOT NULL,          -- propio
    card_brand         TEXT NOT NULL,             -- propio
    authorization_code TEXT NOT NULL              -- propio
);

-- Tabla para BankTransferPayment: también tiene todos los campos
CREATE TABLE bank_transfer_payments (
    id                 BIGSERIAL PRIMARY KEY,
    amount             NUMERIC(12,2) NOT NULL,   -- heredado (duplicado)
    currency           CHAR(3) NOT NULL,           -- heredado (duplicado)
    status             TEXT NOT NULL,              -- heredado (duplicado)
    created_at         TIMESTAMPTZ NOT NULL DEFAULT now(),  -- heredado (duplicado)
    bank_name          TEXT NOT NULL,
    account_last4      CHAR(4) NOT NULL,
    transfer_reference TEXT NOT NULL
);

-- Tabla para CryptoPayment: ídem
CREATE TABLE crypto_payments (
    id                BIGSERIAL PRIMARY KEY,
    amount            NUMERIC(12,2) NOT NULL,    -- heredado (duplicado)
    currency          CHAR(3) NOT NULL,            -- heredado (duplicado)
    status            TEXT NOT NULL,               -- heredado (duplicado)
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),  -- heredado (duplicado)
    wallet_address    TEXT NOT NULL,
    transaction_hash  TEXT NOT NULL,
    coin              TEXT NOT NULL
);
```

### 6.2 El problema fundamental de TPC: queries polimórficas con UNION

La clase base `Payment` no tiene tabla. Entonces una query polimórfica ("dame todos los pagos completados") tiene que buscar en todas las tablas concretas y combinar los resultados con `UNION ALL`:

```sql
-- Cargar todos los pagos (polimórfico, TPC):
SELECT 'credit_card' AS payment_type, id, amount, currency, status, created_at,
       card_last4, card_brand, authorization_code,
       NULL AS bank_name, NULL AS account_last4, NULL AS transfer_reference,
       NULL AS wallet_address, NULL AS transaction_hash, NULL AS coin
FROM credit_card_payments

UNION ALL

SELECT 'bank_transfer' AS payment_type, id, amount, currency, status, created_at,
       NULL, NULL, NULL,
       bank_name, account_last4, transfer_reference,
       NULL, NULL, NULL
FROM bank_transfer_payments

UNION ALL

SELECT 'crypto' AS payment_type, id, amount, currency, status, created_at,
       NULL, NULL, NULL,
       NULL, NULL, NULL,
       wallet_address, transaction_hash, coin
FROM crypto_payments;
```

Este UNION puede ser **extremadamente costoso** porque el motor tiene que escanear todas las tablas concretas, construir el conjunto resultado, y luego aplicar cualquier filtro o orden. La mayoría de los optimizadores no pueden "push down" los predicados eficientemente a través de un UNION.

Además, hay un problema grave con las PKs: los IDs son independientes en cada tabla. Puede haber un `credit_card_payments.id = 5` y también un `crypto_payments.id = 5`. Cuando el ORM necesita cargar un `Payment` polimórfico por ID, no puede simplemente hacer `WHERE id = 5` porque eso puede matchear en múltiples tablas.

### 6.3 Ventajas y desventajas del TPC

**Ventajas:**

- **Queries de subtipo concreto son perfectas:** `SELECT * FROM credit_card_payments WHERE status = 'completed'` es una query de una sola tabla, sin JOINs, sin NULLs.
- **Máxima integridad a nivel de BD:** Cada tabla tiene exactamente las columnas que necesita, todas pueden ser NOT NULL.
- **Sin overhead de JOINs para lecturas de subtipo.**

**Desventajas:**

- **Queries polimórficas son terribles:** El UNION ALL es costoso y difícil de optimizar.
- **IDs no son globalmente únicos:** El mismo ID puede existir en dos tablas concretas distintas.
- **Mantenimiento difícil:** Cambiar un atributo de la clase base requiere alterar todas las tablas concretas.
- **Muchos ORMs tienen soporte limitado o problemático para TPC.**

---

## 7. Cuándo elegir cada estrategia de herencia

### 7.1 La guía de decisión

| Criterio | TPH | TPT | TPC |
|---|---|---|---|
| **Queries polimórficas frecuentes** | ✅ Excelente (una tabla) | ⚠️ Costoso (muchos JOINs) | ❌ Muy costoso (UNION) |
| **Queries de subtipo específico** | ⚠️ Requiere filtro discriminador | ✅ Bueno (JOIN de 2 tablas) | ✅ Excelente (1 tabla) |
| **Integridad NOT NULL en subtipos** | ❌ Imposible | ✅ Posible | ✅ Posible |
| **Subtipos muy distintos (muchos attrs específicos)** | ❌ Tabla llena de NULLs | ✅ Apropiado | ✅ Apropiado |
| **Subtipos muy similares (pocos attrs específicos)** | ✅ Eficiente | ⚠️ JOINs innecesarios | ⚠️ Duplicación innecesaria |
| **Agregar nuevos subtipos frecuentemente** | ✅ Solo agregar columnas | ✅ Agregar nueva tabla | ✅ Agregar nueva tabla |
| **Profundidad de jerarquía > 2 niveles** | ✅ No importa la profundidad | ❌ Un JOIN por nivel | ❌ UNION por hoja |

### 7.2 La regla práctica

**Usa TPH cuando:**
- Las subclases tienen pocos atributos propios (la tabla no explotará en columnas).
- Las queries polimórficas son más frecuentes que las queries de subtipo específico.
- La integridad NOT NULL de los atributos de subtipo no es crítica (o se maneja en la aplicación).
- La velocidad de desarrollo es prioritaria.

**Usa TPT cuando:**
- Las subclases tienen muchos atributos propios con restricciones de integridad importantes.
- Las queries son mayoritariamente de subtipo específico, no polimórficas.
- El esquema debe ser comprensible sin el código de dominio.
- La legibilidad y corrección del esquema es más importante que el rendimiento de queries polimórficas.

**Usa TPC cuando:**
- Las instancias de la clase base NUNCA se cargan polimórficamente; siempre se trabaja con un subtipo específico conocido de antemano.
- La clase base es verdaderamente abstracta y solo existe como artificio de organización del código.
- Esta situación es rara. TPC es el patrón menos recomendado en la práctica.

### 7.3 La alternativa no-ORM: delegación

Hay una cuarta estrategia que no es estrictamente herencia en tablas sino **delegación**: en lugar de mapear la jerarquía de herencia directamente a tablas, se diseña el esquema de forma independiente a la jerarquía de clases, y el código de mapeo traduce entre ambos.

```sql
-- Esquema relacional diseñado para el dominio, no para la herencia del código
CREATE TABLE payments (
    id           BIGSERIAL PRIMARY KEY,
    payment_type TEXT NOT NULL,
    amount       NUMERIC(12,2) NOT NULL,
    currency     CHAR(3) NOT NULL,
    status       TEXT NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    metadata     JSONB  -- atributos específicos del tipo, sin columnas fijas
);
```

Usando JSONB para los atributos específicos de cada tipo, se evita el dilema TPH/TPT/TPC: la tabla es simple, hay una sola tabla para queries polimórficas, y los atributos específicos están en JSONB con sus tipos y constraints gestionadas por la aplicación o por constraints CHECK sobre el JSONB.

Esta es una solución pragmática para jerarquías con muchos subtipos o subtipos que cambian frecuentemente.

---

## 8. El problema N+1: la trampa más frecuente de los ORMs

### 8.1 Por qué el N+1 es el bug de rendimiento más común con ORMs

El **problema N+1** es el problema de rendimiento más frecuente, más silencioso, y más subestimado en aplicaciones que usan ORMs. Su nombre describe exactamente lo que ocurre: se ejecutan **N+1 queries** donde debería ejecutarse **1**.

El N+1 ocurre casi siempre de la misma forma:

1. Se carga una colección de objetos (1 query).
2. Se itera sobre la colección y se accede a una relación de cada objeto (N queries, una por objeto).

```python
# CÓDIGO INOCENTE QUE GENERA N+1:

# 1 query: SELECT * FROM orders WHERE status = 'completed'
orders = session.query(Order).filter(Order.status == 'completed').all()

# Si hay 500 órdenes, este bucle genera 500 queries:
for order in orders:
    # LAZY LOAD: acceder a order.customer dispara:
    # SELECT * FROM customers WHERE id = <order.customer_id>
    print(f"Orden {order.id} del cliente {order.customer.name}")

# Total: 1 (orders) + 500 (customers) = 501 queries
# Para 10,000 órdenes: 10,001 queries
```

El código parece inocente porque `order.customer` parece una propiedad simple. El N+1 no es visible leyendo el código; solo se revela mirando los logs de SQL.

### 8.2 Por qué los ORMs generan N+1 silenciosamente

El lazy loading es el comportamiento por defecto en la mayoría de los ORMs (SQLAlchemy, Hibernate, ActiveRecord). Cuando se define una relación, el ORM asume que no siempre será necesaria y configura el lazy load.

```python
# SQLAlchemy: lazy loading es el default
class Order(Base):
    __tablename__ = 'orders'
    id = Column(Integer, primary_key=True)
    customer_id = Column(Integer, ForeignKey('customers.id'))
    
    # lazy='select' es el DEFAULT: carga cuando se accede por primera vez
    customer = relationship("Customer", lazy='select')
    items = relationship("OrderItem", lazy='select')
```

El ORM no puede saber en el momento de definir la relación si se accederá a ella en un bucle de 10,000 objetos o solo en casos puntuales. Esa decisión la toma el código de la aplicación al usar los objetos.

### 8.3 Cómo detectar el N+1

**Método 1: Logging de SQL**

El método más directo: habilitar el logging de todas las queries SQL y observar patrones repetitivos.

```python
# SQLAlchemy: habilitar logging
import logging
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)

# Ahora cada query se imprime en la consola.
# Si ves esto al ejecutar un bucle, hay N+1:
# SELECT * FROM customers WHERE id = 1
# SELECT * FROM customers WHERE id = 2
# SELECT * FROM customers WHERE id = 3
# ...
```

**Método 2: Conteo de queries en tests**

```python
# En pytest con SQLAlchemy, contar las queries que ejecuta una operación:
def test_orders_page_does_not_have_n_plus_1(session):
    # Crear datos de prueba
    customer = Customer(name="Test")
    session.add(customer)
    for i in range(10):
        session.add(Order(customer=customer, total=100))
    session.commit()
    
    # Contar queries durante la operación
    query_count = 0
    original_execute = session.bind.execute
    def counting_execute(*args, **kwargs):
        nonlocal query_count
        query_count += 1
        return original_execute(*args, **kwargs)
    
    orders = session.query(Order).all()
    names = [o.customer.name for o in orders]
    
    # Si hay N+1, query_count será 11 (1 + 10). 
    # Con eager loading, debería ser 1 o 2.
    assert query_count <= 2, f"N+1 detectado: {query_count} queries"
```

**Método 3: Herramientas de monitoreo**

En producción, herramientas como Django Debug Toolbar, Bullet (Ruby), Hibernate Statistics, o APMs como Datadog y New Relic pueden detectar y alertar sobre N+1 automáticamente.

### 8.4 El costo real en producción

El N+1 es devastador porque escala con el tamaño de la colección. Una página que muestra 20 órdenes con sus clientes genera 21 queries. Si el sistema tiene 1000 usuarios concurrentes viendo esa página, son 21,000 queries concurrentes. Si cada query tarda 2ms, son 42 segundos de tiempo de CPU de base de datos por segundo de pared.

```
Escenario real con N+1:
  Endpoint: GET /orders (página de 50 órdenes con detalles del cliente)
  Con N+1: 1 (orders) + 50 (customers) = 51 queries × 2ms = 102ms por request
  Sin N+1: 2 queries × 2ms = 4ms por request
  
  Con 500 requests/segundo al endpoint:
  Con N+1: 51 × 500 = 25,500 queries/segundo a la DB
  Sin N+1:  2 × 500 =  1,000 queries/segundo a la DB
  
  Diferencia: 24,500 queries/segundo que no deberían existir.
```

---

## 9. Estrategias de fetching: cómo controlar qué carga el ORM

### 9.1 Las tres estrategias de fetching en SQLAlchemy

**Lazy Loading (`lazy='select'` — default):**
Carga la relación solo cuando se accede a ella. Genera N+1 si se usa en bucles.

```python
orders = session.query(Order).all()       # 1 query
names = [o.customer.name for o in orders] # N queries (N+1)
```

**Eager Loading con JOIN (`lazy='joined'` / `joinedload`):**
Carga la relación en la misma query de la entidad principal usando LEFT OUTER JOIN.

```python
from sqlalchemy.orm import joinedload

orders = session.query(Order)\
    .options(joinedload(Order.customer))\
    .all()
# SQL: SELECT orders.*, customers.*
#      FROM orders
#      LEFT OUTER JOIN customers ON customers.id = orders.customer_id

names = [o.customer.name for o in orders]  # Sin queries adicionales
```

Bueno para relaciones many-to-one (una orden → un cliente). Problemático para one-to-many (un cliente → muchas órdenes) porque puede multiplicar filas:

```python
# joinedload en one-to-many multiplica filas:
orders = session.query(Order)\
    .options(joinedload(Order.items))\
    .all()
# Si una orden tiene 5 items, aparece 5 veces en el resultado del JOIN.
# SQLAlchemy desduplicará automáticamente, pero se transfieren más datos.
```

**Eager Loading con subquery (`lazy='subquery'` / `subqueryload`):**
Carga la relación en una segunda query usando una subconsulta con los IDs de los objetos ya cargados.

```python
from sqlalchemy.orm import subqueryload

orders = session.query(Order)\
    .options(subqueryload(Order.items))\
    .all()
# Query 1: SELECT * FROM orders
# Query 2: SELECT order_items.* FROM order_items
#           WHERE order_items.order_id IN (1, 2, 3, 4, 5, ...)
#           -- los IDs vienen de la primera query

# Solo 2 queries total, sin multiplicación de filas.
# Ideal para one-to-many.
```

**Eager Loading con selectin (`selectin`):**
Variante moderna de subqueryload que usa `IN (SELECT ...)`. Es más eficiente en algunos motores.

```python
from sqlalchemy.orm import selectinload

orders = session.query(Order)\
    .options(selectinload(Order.items))\
    .all()
# SQL: SELECT * FROM orders
# SQL: SELECT * FROM order_items WHERE order_items.order_id IN (SELECT ...)
```

### 9.2 La regla de oro para el fetching

No hay una estrategia universalmente correcta. La decisión depende del contexto de uso:

```
¿Siempre necesitas la relación para este query en particular?
  SÍ → Eager loading
  
  ¿Es many-to-one (FK en la tabla principal)?
    SÍ → joinedload (un JOIN eficiente)
    NO (one-to-many) → subqueryload o selectinload (evita multiplicación de filas)
  
  NO → Lazy loading (pero asegúrate de que no se usará en un bucle)
```

### 9.3 Carga anidada: múltiples niveles de relaciones

SQLAlchemy y otros ORMs permiten eager loading en múltiples niveles:

```python
# Cargar orders → items → product → category en una sola operación
orders = session.query(Order)\
    .options(
        selectinload(Order.items)\
            .joinedload(OrderItem.product)\
            .joinedload(Product.category)
    )\
    .filter(Order.status == 'completed')\
    .all()

# Genera 2 queries:
# 1. SELECT * FROM orders WHERE status = 'completed'
# 2. SELECT order_items.*, products.*, categories.*
#    FROM order_items
#    JOIN products ON products.id = order_items.product_id
#    JOIN categories ON categories.id = products.category_id
#    WHERE order_items.order_id IN (...)
```

Esto es fundamentalmente diferente al N+1: son exactamente 2 queries independientemente de cuántas órdenes e ítems haya.

---

## 10. Lazy loading fuera de sesión: el error más frustrante

### 10.1 El contexto del problema

Los ORMs modernos organizan la interacción con la base de datos en **sesiones** (SQLAlchemy `Session`, Hibernate `EntityManager`). La sesión:
- Mantiene el Identity Map (una instancia por PK).
- Rastrea los cambios para el Unit of Work.
- Mantiene una conexión abierta a la base de datos.
- Es el contexto necesario para el lazy loading.

Cuando la sesión se cierra, los objetos cargados durante esa sesión se vuelven **"detached"** (desconectados). En ese estado, intentar acceder a una relación lazy genera un error.

### 10.2 LazyInitializationException / DetachedInstanceError

```python
# SQLAlchemy — DetachedInstanceError
def get_order_service():
    with Session() as session:
        order = session.get(Order, 42)
        return order  # La sesión se cierra al salir del `with`

order = get_order_service()

# AQUÍ la sesión ya está cerrada.
# order.customer está configurado como lazy.
# Intentar acceder dispara DetachedInstanceError:
print(order.customer.name)
# sqlalchemy.orm.exc.DetachedInstanceError: Instance <Order at 0x...>
# is not bound to a Session; attribute refresh operation cannot proceed
```

```java
// Hibernate — LazyInitializationException (el equivalente en Java)
public Order getOrder(long id) {
    EntityManager em = emf.createEntityManager();
    Order order = em.find(Order.class, id);
    em.close();  // Sesión cerrada
    return order;
}

Order order = getOrder(42);
// Aquí la sesión está cerrada.
// order.getCustomer() es lazy → LazyInitializationException
String name = order.getCustomer().getName();
// org.hibernate.LazyInitializationException: 
// could not initialize proxy - no Session
```

### 10.3 Por qué ocurre tan frecuentemente

Este error es extremadamente común porque la arquitectura típica de una aplicación web separa las capas:

```
HTTP Request
     │
     ▼
Controller / Handler
     │
     ▼
Service Layer           ← la sesión generalmente se abre y cierra aquí
     │
     ▼
Repository              ← carga los objetos
     │
     ▼
(sesión se cierra)
     │
     ▼
Controller              ← intenta acceder a relaciones lazy de los objetos
     │
     ▼
Serializer / View       ← también puede intentar acceder a relaciones lazy
     │
     ▼
HTTP Response
```

La sesión se cierra en el Service Layer o Repository. Luego el Controller o el Serializer intenta navegar por el grafo de objetos y accede a relaciones lazy que ya no tienen sesión.

### 10.4 Soluciones

**Solución 1: Open Session in View (OSIV)**

Mantener la sesión abierta durante toda la duración de la request HTTP, desde que entra hasta que sale la respuesta.

```python
# Django ORM usa OSIV por defecto.
# En Spring Boot con Hibernate, OSIV es configurable:
# spring.jpa.open-in-view=true (default en muchas versiones)
```

**Problema de OSIV:** La sesión puede generar queries inesperadas en el serializador o la vista, que el desarrollador no ve ni controla. Los N+1 son más difíciles de detectar porque ocurren en capas que "no deberían acceder a la base de datos". OSIV es conveniente pero peligroso.

**Solución 2: Eager loading explícito antes de cerrar la sesión**

La solución más correcta: dentro del ámbito de la sesión, cargar todos los datos que se necesitarán después.

```python
def get_order_with_details(order_id: int) -> Order:
    with Session() as session:
        order = session.query(Order)\
            .options(
                joinedload(Order.customer),      # Eager load del cliente
                selectinload(Order.items)         # Eager load de los ítems
            )\
            .filter(Order.id == order_id)\
            .one_or_none()
        # Al salir del with, la sesión se cierra,
        # pero customer e items ya están cargados en memoria.
        return order

order = get_order_with_details(42)
print(order.customer.name)  # No hay lazy load: ya está en memoria
```

**Solución 3: DTOs (Data Transfer Objects)**

Dentro de la sesión, mapear los datos a objetos simples sin lazy loading (POJOs, dataclasses, dicts) antes de cerrar la sesión.

```python
from dataclasses import dataclass

@dataclass
class OrderDTO:
    id: int
    total: float
    status: str
    customer_name: str
    item_count: int

def get_order_dto(order_id: int) -> OrderDTO:
    with Session() as session:
        order = session.query(Order)\
            .options(joinedload(Order.customer), selectinload(Order.items))\
            .filter(Order.id == order_id)\
            .one()
        
        # Mapear a DTO dentro de la sesión
        return OrderDTO(
            id=order.id,
            total=float(order.total),
            status=order.status,
            customer_name=order.customer.name,  # acceso dentro de la sesión
            item_count=len(order.items)
        )
        # La sesión se cierra; el DTO no tiene lazy loading
```

Los DTOs son la solución más limpia: el código fuera de la capa de acceso a datos trabaja con objetos simples y explícitos, sin riesgo de lazy loading accidental.

---

## 11. Transacciones implícitas y el comportamiento oculto del ORM

### 11.1 El Unit of Work automático

La mayoría de los ORMs implementan un Unit of Work implícito: todos los cambios hechos a los objetos durante la sesión se acumulan y se aplican automáticamente cuando se hace `session.commit()`. Este comportamiento es conveniente pero tiene consecuencias que no siempre son obvias.

```python
with Session() as session:
    user = session.get(User, 42)
    user.name = "Nuevo Nombre"  # El ORM registra el cambio internamente
    
    # No hay session.save(user) ni session.update(user).
    # El ORM rastrea el cambio automáticamente (change tracking).
    
    session.commit()
    # SQL generado automáticamente:
    # UPDATE users SET name = 'Nuevo Nombre' WHERE id = 42
```

### 11.2 El autoflush: queries dentro de la sesión que fuerzan un flush

SQLAlchemy y Hibernate tienen un comportamiento de **autoflush**: antes de ejecutar una query, el ORM automáticamente hace `flush()` de todos los cambios pendientes para garantizar que la query ve el estado actual.

```python
with Session() as session:
    user = session.get(User, 42)
    user.name = "Nuevo Nombre"    # Cambio pendiente, no persistido
    
    # Esta query fuerza un flush ANTES de ejecutarse:
    all_users = session.query(User).all()
    # Antes del SELECT, SQLAlchemy ejecuta:
    # UPDATE users SET name = 'Nuevo Nombre' WHERE id = 42  ← autoflush
    # Luego: SELECT * FROM users
    
    # Si no se hace commit, el UPDATE se revierte al cerrar la sesión
    session.rollback()
```

El autoflush puede causar sorpresas: operaciones de lectura que inesperadamente generan escrituras en la base de datos (el UPDATE del flush). Si luego se hace rollback, esas escrituras desaparecen, pero si se hace commit sin intención, pueden persistir.

### 11.3 Transacciones implícitas: el compromiso silencioso

El ORM gestiona transacciones de formas que pueden ser invisibles:

**Caso 1: Commit accidental por cerrar la sesión**

```python
# En algunos ORMs y configuraciones, cerrar la sesión hace commit automático.
session = Session()
user = session.get(User, 42)
user.name = "Cambio no intencional"
session.close()  # ¿Hace commit? Depende de la configuración.
                 # En SQLAlchemy, close() NO hace commit; hace rollback implícito.
                 # En otros ORMs, el comportamiento puede ser distinto.
```

**Caso 2: Transacción larga por sesión larga**

```python
# Una sesión que dura mucho tiempo mantiene una transacción abierta.
# En PostgreSQL con MVCC, esto genera bloat y puede bloquear VACUUM.

session = Session()
# ... operaciones que tardan 10 minutos ...
# Durante esos 10 minutos, la transacción está abierta.
session.commit()
```

**Caso 3: Múltiples operaciones en la misma transacción involuntariamente**

```python
# El desarrollador cree que cada operación es independiente:
def process_payment(order_id):
    session = Session()
    order = session.get(Order, order_id)
    order.status = 'processing'
    session.commit()  # Primera "transacción"
    
    # Llamada a servicio externo (puede tardar segundos o fallar)
    payment_result = payment_gateway.charge(order.total)
    
    order.status = 'completed' if payment_result.success else 'failed'
    session.commit()  # Segunda "transacción"
    session.close()

# Problema: si payment_gateway.charge() falla (network error, timeout),
# la orden queda en estado 'processing' indefinidamente.
# No hay rollback automático del primer commit.
```

### 11.4 La sesión como unidad de trabajo consciente

La solución es hacer explícita la delimitación de la transacción y usar el contexto de la sesión como el Unit of Work:

```python
def process_payment(order_id: int) -> bool:
    with Session() as session:
        with session.begin():  # La transacción se gestiona explícitamente
            order = session.get(Order, order_id)
            if order.status != 'pending':
                return False
            
            order.status = 'processing'
            # flush() para enviar el UPDATE a la DB dentro de la transacción
            session.flush()
        
        # La transacción de 'processing' se confirmó.
        # La llamada al gateway ocurre FUERA de la transacción de DB.
        payment_result = payment_gateway.charge(order.total)
        
        with session.begin():
            order = session.get(Order, order_id)  # re-cargar el estado actual
            order.status = 'completed' if payment_result.success else 'failed'
        
        return payment_result.success
```

---

## 12. Cuándo el ORM genera SQL eficiente y cuándo no

### 12.1 Lo que el ORM hace bien

Los ORMs son efectivos para SQL que corresponde a operaciones directas sobre entidades:

**CRUD simple:**
```python
# Inserción: el ORM genera INSERT eficiente
user = User(name="Ana", email="ana@test.com")
session.add(user)
session.commit()
# SQL: INSERT INTO users (name, email) VALUES ('Ana', 'ana@test.com') RETURNING id

# Actualización con change tracking: solo actualiza los campos que cambiaron
user = session.get(User, 42)
user.name = "Ana García"
session.commit()
# SQL: UPDATE users SET name = 'Ana García' WHERE id = 42
# (no actualiza email, credit_limit, etc. porque no cambiaron)

# Eliminación:
session.delete(user)
session.commit()
# SQL: DELETE FROM users WHERE id = 42
```

**Carga de entidades con relaciones conocidas:**
```python
orders = session.query(Order)\
    .filter(Order.status == 'pending', Order.created_at >= '2024-01-01')\
    .options(joinedload(Order.customer), selectinload(Order.items))\
    .order_by(Order.created_at.desc())\
    .limit(20)\
    .all()
# SQL limpio y eficiente con los JOINs correctos
```

### 12.2 Lo que el ORM hace mal

**Agregaciones complejas:**

```python
# ORM: intento de calcular el revenue total por cliente
for customer in session.query(Customer).all():
    total = sum(order.total for order in customer.orders)  # N+1 masivo
    print(f"{customer.name}: {total}")

# SQL correcto (que el ORM raramente genera solo):
# SELECT customers.name, SUM(orders.total) AS revenue
# FROM customers
# JOIN orders ON orders.customer_id = customers.id
# GROUP BY customers.id, customers.name
# ORDER BY revenue DESC
```

**Window functions:**

```python
# Rango de ventas por región: imposible de expresar limpiamente en la API del ORM
# Con SQLAlchemy se puede usar func(), pero es verbose y difícil de leer:
from sqlalchemy import func

result = session.query(
    Order.region,
    func.sum(Order.total).label('total'),
    func.rank().over(order_by=func.sum(Order.total).desc()).label('rank')
).group_by(Order.region).all()

# El SQL que se necesita es mucho más claro así:
# SELECT region,
#        SUM(total) AS total,
#        RANK() OVER (ORDER BY SUM(total) DESC) AS rank
# FROM orders
# GROUP BY region
```

**Queries con CTEs:**

```python
# CTEs recursivas para jerarquías: el ORM no puede ayudar
# Hay que escribir SQL nativo directamente:
from sqlalchemy import text

result = session.execute(text("""
    WITH RECURSIVE categoria_tree AS (
        SELECT id, nombre, parent_id, 0 AS nivel
        FROM categorias WHERE id = :root_id
        
        UNION ALL
        
        SELECT c.id, c.nombre, c.parent_id, ct.nivel + 1
        FROM categorias c
        INNER JOIN categoria_tree ct ON c.parent_id = ct.id
    )
    SELECT * FROM categoria_tree ORDER BY nivel, nombre
"""), {'root_id': 5})
```

**Upserts y operaciones bulk:**

```python
# INSERT masivo: el ORM hace un INSERT por fila (lento para miles de filas)
for i in range(10000):
    session.add(Product(name=f"Producto {i}", price=9.99))
session.commit()  # 10,000 INSERTs individuales → muy lento

# La solución: bulk insert directo
session.bulk_insert_mappings(Product, [
    {'name': f'Producto {i}', 'price': 9.99}
    for i in range(10000)
])
# o con INSERT ... VALUES (...), (...), (...) en una sola query
```

---

## 13. Cuándo escribir SQL a mano es inevitable

### 13.1 Los casos donde el ORM simplemente no llega

**1. Queries analíticas complejas:**
Window functions, CTEs complejas, GROUPING SETS, ROLLUP, CUBE, queries con subconsultas correlacionadas. El ORM puede generar estas queries con code verboso y difícil de mantener, o simplemente no puede.

```python
# SQL directo para una query analítica compleja:
result = session.execute(text("""
    SELECT 
        DATE_TRUNC('month', o.created_at) AS mes,
        c.region,
        COUNT(DISTINCT o.customer_id)          AS clientes_unicos,
        SUM(o.total)                            AS revenue,
        AVG(o.total)                            AS ticket_promedio,
        SUM(SUM(o.total)) OVER (
            PARTITION BY c.region 
            ORDER BY DATE_TRUNC('month', o.created_at)
        )                                       AS revenue_acumulado_por_region
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    WHERE o.status = 'completed'
      AND o.created_at >= :desde
    GROUP BY 1, 2
    ORDER BY 1, 2
"""), {'desde': '2024-01-01'})
```

**2. Operaciones bulk eficientes:**

```python
# UPDATE masivo con condición: el ORM no puede hacer esto eficientemente
# sin cargar todos los objetos primero.

# Con SQL directo (actualiza millones de filas en una sola query):
session.execute(text("""
    UPDATE products
    SET price = price * 1.1  -- aumento del 10%
    WHERE category_id = :cat_id
      AND active = TRUE
"""), {'cat_id': 15})

# Con INSERT ... ON CONFLICT (UPSERT): el ORM no soporta esto de forma nativa
session.execute(text("""
    INSERT INTO inventory (product_id, warehouse_id, stock)
    VALUES (:product_id, :warehouse_id, :stock)
    ON CONFLICT (product_id, warehouse_id)
    DO UPDATE SET stock = EXCLUDED.stock, updated_at = NOW()
"""), {'product_id': 42, 'warehouse_id': 7, 'stock': 150})
```

**3. Queries que necesitan hints o características específicas del motor:**

```sql
-- PostgreSQL: usar un índice específico (hint)
SELECT /*+ IndexScan(orders orders_status_idx) */
       id, total, status
FROM orders
WHERE status = 'pending';

-- PostgreSQL: EXPLAIN ANALYZE dentro de la aplicación para diagnóstico
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT ...;
```

**4. DDL dinámico o migraciones:**

```python
# Crear o alterar tablas dinámicamente: siempre con SQL directo o herramientas de migración
session.execute(text("ALTER TABLE users ADD COLUMN phone TEXT"))
session.execute(text("CREATE INDEX CONCURRENTLY idx_users_email ON users(email)"))
```

### 13.2 El patrón: ORM para entidades, SQL para analytics

La arquitectura más pragmática separa claramente los dos usos:

```
┌────────────────────────────────────────────────────────┐
│  ORM (SQLAlchemy, Hibernate, etc.)                     │
│  Para: CRUD de entidades de dominio, transacciones     │
│  de negocio, carga de objetos con relaciones           │
├────────────────────────────────────────────────────────┤
│  SQL a mano (session.execute(text(...)))               │
│  Para: queries analíticas, bulk operations, queries    │
│  con features SQL avanzadas, upserts, CTEs complejas   │
├────────────────────────────────────────────────────────┤
│  Query Builder (SQLAlchemy Core, Knex, jOOQ)          │
│  Para: queries dinámicas donde el SQL se construye     │
│  programáticamente pero necesita tipado y seguridad    │
└────────────────────────────────────────────────────────┘
```

Usar el ORM para operaciones de dominio y SQL para analytics es lo correcto. No es una señal de que el ORM es inadecuado; es reconocer que son herramientas para problemas distintos.

---

## 14. La decisión: ORM, query builder, o SQL puro

### 14.1 Los tres niveles de abstracción

**SQL puro:** Strings de SQL pasados directamente a la base de datos.

```python
# Python con psycopg2 directamente:
cursor.execute("""
    SELECT u.id, u.name, COUNT(o.id) as order_count
    FROM users u
    LEFT JOIN orders o ON o.user_id = u.id
    WHERE u.active = TRUE
    GROUP BY u.id, u.name
    HAVING COUNT(o.id) > 5
""")
rows = cursor.fetchall()
```

**Query Builder (SQLAlchemy Core, Knex.js, jOOQ):** Construir SQL programáticamente usando una API tipada, sin mapear a objetos de dominio.

```python
# SQLAlchemy Core (sin ORM):
from sqlalchemy import select, func, text

stmt = select(
    users.c.id,
    users.c.name,
    func.count(orders.c.id).label('order_count')
).select_from(
    users.outerjoin(orders, orders.c.user_id == users.c.id)
).where(
    users.c.active == True
).group_by(
    users.c.id, users.c.name
).having(
    func.count(orders.c.id) > 5
)

result = conn.execute(stmt).fetchall()
```

**ORM completo:** Mapeo automático a objetos de dominio, change tracking, Identity Map, lazy loading.

```python
# SQLAlchemy ORM:
users_with_orders = session.query(User)\
    .outerjoin(Order)\
    .filter(User.active == True)\
    .group_by(User.id)\
    .having(func.count(Order.id) > 5)\
    .options(selectinload(User.orders))\
    .all()
```

### 14.2 Cuándo elegir cada nivel

**SQL puro es correcto cuando:**
- La query es única y compleja (no se reutiliza con parámetros variables).
- Necesita features avanzadas del motor (CTEs, window functions, hints).
- El rendimiento es crítico y el SQL debe ser exactamente el óptimo.
- El equipo está más cómodo con SQL que con la API del ORM.

**Query builder es correcto cuando:**
- La query se construye dinámicamente (distintos filtros, ordenamientos, paginación según el input).
- Se necesita SQL relativamente complejo pero sin mapeo a objetos.
- Se quiere type safety y refactoring-friendly sin el overhead completo del ORM.

```python
# Ejemplo de query builder para búsqueda con filtros dinámicos:
def search_products(category_id=None, min_price=None, max_price=None, 
                    in_stock=None, order_by='name'):
    stmt = select(products)
    
    if category_id:
        stmt = stmt.where(products.c.category_id == category_id)
    if min_price:
        stmt = stmt.where(products.c.price >= min_price)
    if max_price:
        stmt = stmt.where(products.c.price <= max_price)
    if in_stock:
        stmt = stmt.where(products.c.stock > 0)
    
    # Ordenamiento dinámico y validado
    order_columns = {'name': products.c.name, 'price': products.c.price}
    if order_by in order_columns:
        stmt = stmt.order_by(order_columns[order_by])
    
    return conn.execute(stmt).fetchall()
```

**ORM completo es correcto cuando:**
- Se trabaja con entidades de dominio con lógica de negocio encapsulada.
- Las operaciones son principalmente CRUD sobre objetos.
- Se necesita change tracking, Unit of Work y Identity Map automáticos.
- El equipo usa DDD y el modelo de objetos es central.
- El rendimiento de las queries generadas es aceptable para el caso de uso.

### 14.3 La trampa del ORM "todo o nada"

Un error frecuente es adoptar el ORM como una decisión de todo o nada: o todo el acceso a datos pasa por el ORM, o todo pasa por SQL puro. La realidad es que la combinación es no solo válida sino óptima:

```python
class OrderRepository:
    def __init__(self, session):
        self._session = session
    
    # Método ORM: para carga de entidades de dominio
    def get_with_details(self, order_id: int) -> Order | None:
        return self._session.query(Order)\
            .options(joinedload(Order.customer), selectinload(Order.items))\
            .filter(Order.id == order_id)\
            .one_or_none()
    
    # Método SQL: para query analítica
    def get_revenue_by_status(self, from_date: date) -> list[dict]:
        result = self._session.execute(text("""
            SELECT status, 
                   COUNT(*) as order_count,
                   SUM(total) as total_revenue,
                   AVG(total) as avg_order_value
            FROM orders
            WHERE created_at >= :from_date
            GROUP BY status
            ORDER BY total_revenue DESC
        """), {'from_date': from_date})
        return [dict(row) for row in result]
    
    # Método bulk: para operaciones masivas
    def bulk_update_status(self, order_ids: list[int], new_status: str) -> int:
        result = self._session.execute(text("""
            UPDATE orders 
            SET status = :status, updated_at = NOW()
            WHERE id = ANY(:ids) AND status != :status
        """), {'status': new_status, 'ids': order_ids})
        return result.rowcount
```

Este repositorio usa el ORM para lo que hace bien (carga de entidades con relaciones), SQL directo para analytics, y SQL directo para bulk operations. No es inconsistente: es usar la herramienta correcta para cada trabajo.

### 14.4 Seguridad: SQL injection con texto directo

Al usar SQL a mano, el riesgo de **SQL injection** es real si no se usan parámetros correctamente:

```python
# PELIGROSO: interpolación de strings → SQL injection
user_input = "'; DROP TABLE orders; --"
session.execute(text(f"SELECT * FROM orders WHERE status = '{user_input}'"))
# SQL ejecutado: SELECT * FROM orders WHERE status = ''; DROP TABLE orders; --'

# CORRECTO: parámetros binding
session.execute(
    text("SELECT * FROM orders WHERE status = :status"),
    {'status': user_input}   # El driver escapa correctamente
)
```

**Regla absoluta:** Nunca interpolar variables de usuario directamente en strings SQL. Siempre usar parámetros (`:nombre` en SQLAlchemy, `$1` en psycopg2, `?` en JDBC).

---

## 15. Resumen y conexión con el resto del curso

### Los conceptos clave y cuándo aplicarlos

| Concepto | Problema que resuelve | Cuándo aplicarlo |
|---|---|---|
| **TPH** | Herencia con queries polimórficas frecuentes | Jerarquías poco profundas, subclases similares |
| **TPT** | Herencia con integridad de BD y subclases distintas | Subclases con muchos atributos propios |
| **TPC** | Herencia donde siempre se trabaja con el tipo concreto | Raro; solo clases base verdaderamente abstractas |
| **Eager loading (joinedload)** | N+1 en many-to-one | JOIN cuando la relación siempre se necesita |
| **Eager loading (selectinload)** | N+1 en one-to-many | Subquery cuando puede haber multiplicación de filas |
| **Lazy loading** | Cargar relaciones opcionales | Solo cuando no se usa en bucles |
| **DTO** | LazyInitializationException fuera de sesión | Siempre que los datos salgan de la capa de persistencia |
| **SQL directo** | Queries complejas que el ORM genera mal | Analytics, bulk, CTEs, window functions |
| **Query builder** | Queries dinámicas con type safety | Filtros variables, ordenamiento dinámico |

### La meta-lección del módulo

El ORM no es una caja negra mágica que resuelve el acceso a datos. Es una implementación automática de los patrones de Fowler (módulo 14) que funciona muy bien para operaciones de dominio simples y puede fallar silenciosamente (N+1, DetachedInstanceError, transacciones implícitas) cuando se usa sin comprender lo que hace.

Las tres preguntas que un desarrollador debe hacerse al usar un ORM:

1. **¿Qué SQL está generando esta operación?** No lo asumas; activa el logging y compruébalo, especialmente para operaciones en bucles y queries en producción.

2. **¿Dónde está el límite de la sesión?** Las sesiones deben ser cortas y delimitadas. Saber cuándo una sesión abre y cierra es fundamental para evitar DetachedInstanceErrors, transacciones largas y comportamientos inesperados.

3. **¿Esta query debería estar en el ORM o en SQL directo?** Para CRUD y operaciones de dominio: ORM. Para analytics, bulk, y queries complejas: SQL directo. No hay conflicto en combinar ambos en la misma capa de acceso a datos.

### Conexión con el módulo siguiente

- **Módulo 16 (El oficio):** Diagnosticar un sistema existente incluye identificar N+1s silenciosos, lazy loads fuera de sesión, y queries generadas por el ORM que nadie revisó. Refactorizar un esquema en producción implica gestionar las migraciones del ORM (Alembic, Flyway, Liquibase) con las técnicas de expand/contract que el módulo 16 cubre. El oficio de trabajar con bases de datos en el mundo real es inseparable de trabajar con ORMs, porque casi todos los sistemas modernos los usan.

---

## 16. Ejercicios de comprensión

**Ejercicio 1.** Un sistema de e-commerce tiene la siguiente jerarquía de productos:

```
Product (base)
  ├── PhysicalProduct (con: weight_kg, dimensions, shipping_class)
  │     ├── BookProduct (con: isbn, author, publisher, pages)
  │     └── ElectronicsProduct (con: voltage, warranty_years, battery_life_hours)
  └── DigitalProduct (con: file_size_mb, download_url, license_type)
        ├── SoftwareProduct (con: platform, version, min_os_version)
        └── MediaProduct (con: duration_minutes, format, resolution)
```

Los atributos comunes de `Product` son: `id`, `name`, `description`, `price`, `active`, `category_id`.

a) Para la jerarquía completa (2 niveles de herencia, 5 clases concretas), diseña el esquema con **TPH**. Identifica cuántas columnas tiene la tabla y cuántas son nullable.

b) Diseña el mismo modelo con **TPT**. ¿Cuántas tablas se crean? Escribe el DDL completo de las tablas para `Product`, `PhysicalProduct` y `BookProduct`.

c) La query más frecuente del sistema es: "dame los primeros 20 productos activos de la categoría X ordenados por precio". ¿Qué estrategia genera SQL más eficiente para esta query? ¿Por qué?

d) La segunda query más frecuente es: "dame todos los detalles del producto Y (incluidos atributos específicos de su tipo)". En TPT, ¿cuántos JOINs requiere esta query para un `BookProduct`? Escribe el SQL.

e) Si se decide añadir un nuevo tipo `SubscriptionProduct` con atributos `billing_period`, `trial_days`, y `max_devices`: ¿qué cambios requiere cada estrategia (TPH vs TPT)?

---

**Ejercicio 2.** Dado el siguiente código con SQLAlchemy ORM:

```python
class Department(Base):
    __tablename__ = 'departments'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    employees = relationship("Employee", back_populates="department")

class Employee(Base):
    __tablename__ = 'employees'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    salary = Column(Numeric)
    department_id = Column(Integer, ForeignKey('departments.id'))
    department = relationship("Department", back_populates="employees")
    projects = relationship("ProjectAssignment", back_populates="employee")

class ProjectAssignment(Base):
    __tablename__ = 'project_assignments'
    employee_id = Column(Integer, ForeignKey('employees.id'), primary_key=True)
    project_id = Column(Integer, ForeignKey('projects.id'), primary_key=True)
    hours_per_week = Column(Integer)
    employee = relationship("Employee", back_populates="projects")
    project = relationship("Project")

class Project(Base):
    __tablename__ = 'projects'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    budget = Column(Numeric)
```

Para cada uno de los siguientes fragmentos de código, indica:
- ¿Cuántas queries SQL genera? (asumiendo lazy loading por defecto)
- ¿Hay N+1? ¿Dónde?
- ¿Cómo lo corregirías con el eager loading apropiado?

```python
# Fragmento A:
departments = session.query(Department).all()
for dept in departments:
    total_salary = sum(e.salary for e in dept.employees)
    print(f"{dept.name}: salario total = {total_salary}")

# Fragmento B:
employees = session.query(Employee).filter(Employee.salary > 50000).all()
for emp in employees:
    project_names = [pa.project.name for pa in emp.projects]
    print(f"{emp.name}: {', '.join(project_names)}")

# Fragmento C:
emp = session.query(Employee).filter(Employee.id == 42).one()
dept_name = emp.department.name  # ¿Cuántas queries?
```

---

**Ejercicio 3.** Un sistema web con Django ORM tiene la siguiente vista:

```python
# views.py
def order_list(request):
    orders = Order.objects.filter(
        status='pending',
        created_at__gte=timezone.now() - timedelta(days=7)
    )[:50]
    
    return render(request, 'orders/list.html', {'orders': orders})
```

```html
<!-- orders/list.html -->
{% for order in orders %}
<tr>
  <td>{{ order.id }}</td>
  <td>{{ order.customer.name }}</td>     {# relación ForeignKey #}
  <td>{{ order.customer.email }}</td>    {# misma relación #}
  <td>{{ order.total }}</td>
  <td>
    {% for item in order.items.all %}    {# relación ManyToMany/ForeignKey #}
      {{ item.product.name }} × {{ item.quantity }}
    {% endfor %}
  </td>
</tr>
{% endfor %}
```

a) Identifica todos los N+1 presentes en este código. ¿Cuántas queries se generan para 50 órdenes, cada una con 3 ítems?

b) Corrige el código en `views.py` usando `select_related` (para ForeignKey) y `prefetch_related` (para relaciones inversas y ManyToMany). Explica la diferencia entre ambos.

c) ¿Sería recomendable aquí usar OSIV (Open Session in View)? ¿Qué problemas pueden surgir?

d) Propón una alternativa usando un DTO o un queryset con `values()` que evite cargar objetos ORM completos cuando solo se necesitan algunos campos.

---

**Ejercicio 4.** Un repositorio tiene los siguientes métodos que mezclan ORM y SQL:

```python
class SalesRepository:
    
    def get_order(self, order_id: int) -> Order:
        return self.session.get(Order, order_id)
    
    def get_monthly_summary(self, year: int, month: int) -> dict:
        # Esta query usa window functions y CTEs
        result = self.session.execute(text("""
            WITH monthly_sales AS (
                SELECT 
                    DATE_TRUNC('day', created_at) AS sale_date,
                    SUM(total) AS daily_total,
                    COUNT(*) AS order_count
                FROM orders
                WHERE EXTRACT(YEAR FROM created_at) = :year
                  AND EXTRACT(MONTH FROM created_at) = :month
                  AND status = 'completed'
                GROUP BY 1
            )
            SELECT 
                sale_date,
                daily_total,
                order_count,
                SUM(daily_total) OVER (ORDER BY sale_date) AS cumulative_total
            FROM monthly_sales
            ORDER BY sale_date
        """), {'year': year, 'month': month})
        return [dict(row) for row in result]
    
    def bulk_expire_old_discounts(self, before_date: date) -> int:
        result = self.session.execute(text("""
            UPDATE discounts
            SET status = 'expired', expired_at = NOW()
            WHERE valid_until < :before_date AND status = 'active'
        """), {'before_date': before_date})
        self.session.commit()
        return result.rowcount
```

a) ¿Es correcto mezclar ORM y SQL directo en el mismo repositorio? Justifica.

b) El método `bulk_expire_old_discounts` hace `self.session.commit()` dentro del repositorio. ¿Qué problema introduce esto si el código que llama al repositorio ya está dentro de una transacción más grande? ¿Cómo lo corregirías?

c) Identifica un posible riesgo de SQL injection en cualquiera de los métodos. ¿Existe en este código? Justifica.

d) El método `get_monthly_summary` devuelve una lista de dicts. ¿Cuándo es preferible devolver dicts en lugar de objetos ORM? ¿Cuándo es un error?

---

**Ejercicio 5.** Considera el siguiente escenario de LazyInitializationException:

```python
# service.py
class UserService:
    def __init__(self, session_factory):
        self._session_factory = session_factory
    
    def get_user_profile(self, user_id: int) -> UserProfile:
        with self._session_factory() as session:
            user = session.get(User, user_id)
            if not user:
                raise UserNotFoundError(user_id)
            return UserProfile(user)   # ← ¿Problema aquí?

# api.py
class UserAPI:
    def get_profile(self, user_id: int):
        profile = self.user_service.get_user_profile(user_id)
        
        # Serializa el perfil a JSON
        return {
            'id': profile.user.id,
            'name': profile.user.name,
            'orders_count': len(profile.user.orders),  # ← ¿Problema aquí?
            'recent_order': profile.user.orders[0].total if profile.user.orders else None
        }
```

a) Identifica exactamente en qué línea(s) puede ocurrir el `DetachedInstanceError`.

b) Propón tres soluciones distintas a este problema, explicando el trade-off de cada una:
   - Solución usando eager loading dentro de la sesión.
   - Solución usando un DTO.
   - Solución usando OSIV.

c) ¿Cuál de las tres soluciones recomendarías para un sistema con alta concurrencia (1000 requests/segundo) y por qué?

d) Si `User.orders` tiene `lazy='dynamic'` (en SQLAlchemy, esto devuelve una query object en lugar de una lista), ¿cambia el comportamiento fuera de la sesión? ¿Cómo?

---

*Próximo módulo: El oficio — del requerimiento al modelo en producción. Todo lo que hemos aprendido se aplica al trabajo real: extraer el modelo de una conversación con el cliente, evaluar modelos ajenos, hacer refactoring de esquemas en sistemas vivos, y tomar las decisiones arquitectónicas que duran años.*
