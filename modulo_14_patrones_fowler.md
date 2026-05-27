# Módulo 14 — Patrones de Fowler y la capa de acceso a datos

> *"La base de datos no sabe nada del código que la usa. El código de dominio no debería saber nada de la base de datos. Entre estos dos mundos existe una frontera, y en esa frontera vive toda la complejidad que los ORMs intentan esconder y que los patrones de Fowler intentan hacer explícita. Ignorar esa frontera no hace que desaparezca: la empuja hacia el interior del código de dominio, donde corrompe la lógica de negocio con detalles de persistencia hasta que los dos son indistinguibles y ninguno es fácil de cambiar."*

---

## Tabla de contenidos

1. [El problema de la frontera objeto-relacional](#1-el-problema-de-la-frontera-objeto-relacional)
2. [El origen: "Patterns of Enterprise Application Architecture"](#2-el-origen-patterns-of-enterprise-application-architecture)
3. [Table Data Gateway: acceso a tabla como objeto](#3-table-data-gateway-acceso-a-tabla-como-objeto)
4. [Row Data Gateway: acceso a fila como objeto](#4-row-data-gateway-acceso-a-fila-como-objeto)
5. [Active Record: el objeto que sabe persistirse](#5-active-record-el-objeto-que-sabe-persistirse)
6. [Data Mapper: la capa que separa los mundos](#6-data-mapper-la-capa-que-separa-los-mundos)
7. [Repository: la colección que habla con la base de datos](#7-repository-la-colección-que-habla-con-la-base-de-datos)
8. [Unit of Work: rastrear cambios para persistirlos juntos](#8-unit-of-work-rastrear-cambios-para-persistirlos-juntos)
9. [Identity Map: una sola instancia por registro](#9-identity-map-una-sola-instancia-por-registro)
10. [Lazy Load: cargar datos solo cuando se necesitan](#10-lazy-load-cargar-datos-solo-cuando-se-necesitan)
11. [Cómo los patrones se combinan en la práctica](#11-cómo-los-patrones-se-combinan-en-la-práctica)
12. [Cuándo usar cada patrón](#12-cuándo-usar-cada-patrón)
13. [Los patrones en el contexto de sistemas distribuidos](#13-los-patrones-en-el-contexto-de-sistemas-distribuidos)
14. [Resumen y conexión con el resto del curso](#14-resumen-y-conexión-con-el-resto-del-curso)
15. [Ejercicios de comprensión](#15-ejercicios-de-comprensión)

---

## 1. El problema de la frontera objeto-relacional

### 1.1 Dos modelos fundamentalmente distintos

Para entender por qué los patrones de este módulo existen, hay que empezar con la incompatibilidad fundamental entre los dos mundos que deben comunicarse: el mundo del **código orientado a objetos** y el mundo del **modelo relacional**.

Esta incompatibilidad es tan profunda y tan reconocida que tiene nombre propio: la **impedancia objeto-relacional** (object-relational impedance mismatch). El término viene de la electrónica, donde la impedancia describe la resistencia de un circuito a la corriente alterna; cuando dos componentes tienen distinta impedancia, hay pérdida de señal en la frontera.

Las diferencias son concretas:

**Identidad:**
- En el modelo relacional, la identidad de una fila viene de su clave primaria: dos filas son la misma si y solo si tienen el mismo valor de PK.
- En el modelo de objetos, la identidad puede ser por referencia (dos variables que apuntan al mismo objeto en memoria) o por valor (dos objetos que son "iguales" según su método `equals()`). Estos dos conceptos de identidad no coinciden necesariamente.

**Relaciones:**
- En el modelo relacional, las relaciones son simétricas y se navegan mediante JOINs. La tabla `orders` tiene una FK hacia `customers`, pero desde `customers` no hay nada que apunte hacia `orders`; el JOIN se construye al momento de la query.
- En el modelo de objetos, las relaciones son navegables de forma directa: un objeto `Customer` puede tener un campo `List<Order> orders` que permite navegar hacia los pedidos directamente, sin SQL. La relación puede tener referencias en ambas direcciones.

**Herencia:**
- El modelo de objetos tiene herencia de primera clase: una clase `Employee` puede extender una clase `Person`, y el código puede tratar un `Employee` como un `Person` gracias al polimorfismo.
- El modelo relacional no tiene herencia. Hay varias estrategias para mapear jerarquías de herencia a tablas (Table Per Hierarchy, Table Per Type, Table Per Concrete Class), ninguna perfecta.

**Grafos de objetos:**
- Los objetos pueden apuntar a otros objetos, que apuntan a otros, formando un grafo potencialmente profundo. `customer.orders[0].items[2].product.category.name` es una expresión perfectamente válida en código.
- En el modelo relacional, ese grafo requeriría múltiples JOINs. Y cargar el grafo completo puede ser innecesario si solo se necesita el nombre de la categoría de ese producto específico.

**Tipos de datos:**
- Los lenguajes de programación tienen un sistema de tipos rico: colecciones, generics, enums, tipos nullable, tipos de valor vs tipos de referencia, tipos personalizados con comportamiento.
- El modelo relacional tiene tipos primitivos (entero, texto, fecha, booleano) y tipos más avanzados (arrays, JSON) en algunos motores, pero no hay herencia de tipos ni comportamiento asociado a los tipos.

### 1.2 La consecuencia práctica: el código de acceso a datos

Si se deja que esta impedancia se resuelva de forma ad-hoc, el código de la aplicación termina mezclando dos preocupaciones que deberían estar separadas:

```python
# Sin ningún patrón: el caos que emerge naturalmente
class OrderService:
    def complete_order(self, order_id: int, user_id: int):
        conn = get_db_connection()
        cursor = conn.cursor()
        
        # 1. Verificar que el usuario existe (lógica de negocio + SQL mezclados)
        cursor.execute("SELECT id, credit_limit FROM users WHERE id = %s", (user_id,))
        user_row = cursor.fetchone()
        if not user_row:
            raise ValueError("Usuario no encontrado")
        
        # 2. Obtener la orden (más SQL mezclado con lógica)
        cursor.execute("""
            SELECT o.id, o.status, o.total, oi.product_id, oi.quantity, p.price
            FROM orders o
            JOIN order_items oi ON oi.order_id = o.id
            JOIN products p ON p.id = oi.product_id
            WHERE o.id = %s AND o.user_id = %s
        """, (order_id, user_id))
        order_rows = cursor.fetchall()
        if not order_rows:
            raise ValueError("Orden no encontrada")
        
        # 3. Lógica de negocio: ¿puede completarse?
        total = sum(row[5] * row[4] for row in order_rows)  # ¿qué es row[5]? ¿row[4]?
        if total > user_row[1]:  # ¿qué es user_row[1]?
            raise ValueError("Crédito insuficiente")
        
        # 4. Actualizar estado (más SQL)
        cursor.execute(
            "UPDATE orders SET status = 'completed' WHERE id = %s",
            (order_id,)
        )
        conn.commit()
        
        # ¿Dónde se cierra la conexión si hay una excepción? ¿El rollback?
        # ¿Si mañana se agrega un campo, cuántos row[N] hay que actualizar?
```

Este código tiene múltiples problemas graves:

- **SQL y lógica de negocio entrelazados:** Para entender la regla de negocio ("¿puede completarse la orden?"), hay que leer también el SQL.
- **Fragilidad ante cambios del esquema:** Si se renombra una columna, hay que encontrar todos los `row[N]` que la referencian.
- **No testeable sin base de datos:** Para probar la lógica de negocio, se necesita una base de datos real.
- **Duplicación:** Si otro servicio necesita cargar un usuario o una orden, escribe el mismo SQL de nuevo.
- **Manejo de conexiones, transacciones y errores** disperso por todo el código.

Los patrones de Fowler son la respuesta sistemática a este problema.

---

## 2. El origen: "Patterns of Enterprise Application Architecture"

### 2.1 El libro y su contexto

En 2002, Martin Fowler publicó **"Patterns of Enterprise Application Architecture"** (PoEAA), que documentó y nombró los patrones que los mejores desarrolladores de aplicaciones empresariales ya estaban usando implícitamente, pero sin un vocabulario común.

Fowler no inventó estos patrones; los **documentó**: les dio nombre, describió el problema que resuelven, las consecuencias de usarlos, y cuándo son apropiados. Esta contribución es tan valiosa como la invención, porque permite que los equipos hablen sobre arquitectura con precisión.

El libro organiza los patrones en varias categorías. Los que cubren la capa de acceso a datos son:

- **Data Source Architectural Patterns:** Table Data Gateway, Row Data Gateway, Active Record, Data Mapper.
- **Object-Relational Behavioral Patterns:** Unit of Work, Identity Map, Lazy Load.
- **Object-Relational Structural Patterns:** Identity Field, Foreign Key Mapping, Association Table Mapping, Dependent Mapping, Embedded Value, Serialized LOB, Single Table Inheritance, Class Table Inheritance, Concrete Table Inheritance, Inheritance Mappers.
- **Object-Relational Metadata Mapping Patterns:** Metadata Mapping, Query Object, Repository.

Este módulo cubre los más fundamentales y los más frecuentemente implementados en la práctica moderna.

### 2.2 La idea unificadora: separar qué de cómo

Antes de entrar en los patrones individuales, es importante entender la idea que los unifica: la **separación del modelo de dominio del mecanismo de persistencia**.

```
┌────────────────────────────────────────────────────────────────────┐
│                        CAPAS DE LA APLICACIÓN                      │
├────────────────────────────────────────────────────────────────────┤
│  PRESENTACIÓN / API                                                 │
│  HTTP handlers, validación de input, serialización de output       │
├────────────────────────────────────────────────────────────────────┤
│  DOMINIO / LÓGICA DE NEGOCIO                                        │
│  Entidades, Value Objects, Domain Services, reglas de negocio      │
│  No sabe nada de SQL, ni de tablas, ni de cómo se persiste         │
├────────────────────────────────────────────────────────────────────┤
│  CAPA DE ACCESO A DATOS  ←── Los patrones de Fowler viven aquí     │
│  Repository, Data Mapper, Unit of Work, Identity Map               │
│  Traduce entre el modelo de dominio y el modelo relacional         │
├────────────────────────────────────────────────────────────────────┤
│  BASE DE DATOS                                                      │
│  Tablas, índices, constraints, SQL, transactions                   │
└────────────────────────────────────────────────────────────────────┘
```

La capa de acceso a datos es el **traductor** entre los dos mundos. Su responsabilidad es que el código de dominio pueda expresarse en términos de su propio modelo (usuarios, pedidos, productos) sin contaminar ese modelo con SQL, nombres de tablas, o detalles del motor de base de datos.

---

## 3. Table Data Gateway: acceso a tabla como objeto

### 3.1 La idea

El **Table Data Gateway** es el patrón más simple de los cuatro en Data Source Architectural Patterns. La idea es directa: por cada tabla de la base de datos, crear un objeto (una clase) que sea la **única puerta de acceso** a esa tabla. Toda la SQL para esa tabla vive dentro del Table Data Gateway de esa tabla.

```
Base de datos:           Código:
┌──────────────┐        ┌──────────────────────────────┐
│ Tabla users  │◀──────▶│ UsersGateway                  │
├──────────────┤        │  + find_by_id(id)             │
│ id           │        │  + find_by_email(email)       │
│ name         │        │  + find_all_active()          │
│ email        │        │  + insert(name, email, ...)   │
│ created_at   │        │  + update(id, name, ...)      │
└──────────────┘        │  + delete(id)                 │
                        └──────────────────────────────┘
```

El Table Data Gateway devuelve datos en algún formato genérico: filas (como diccionarios, records, o DataSets). No devuelve objetos de dominio ricos; devuelve datos en bruto que el código que llama puede usar.

```python
class UsersGateway:
    """
    Table Data Gateway para la tabla 'users'.
    Toda la SQL relacionada con users vive aquí.
    """
    
    def __init__(self, connection):
        self._conn = connection
    
    def find_by_id(self, user_id: int) -> dict | None:
        cursor = self._conn.cursor(dictionary=True)
        cursor.execute(
            "SELECT id, name, email, credit_limit, created_at "
            "FROM users WHERE id = %s",
            (user_id,)
        )
        return cursor.fetchone()  # Devuelve un dict, no un objeto User
    
    def find_by_email(self, email: str) -> dict | None:
        cursor = self._conn.cursor(dictionary=True)
        cursor.execute(
            "SELECT id, name, email, credit_limit, created_at "
            "FROM users WHERE email = %s",
            (email,)
        )
        return cursor.fetchone()
    
    def find_all_active(self) -> list[dict]:
        cursor = self._conn.cursor(dictionary=True)
        cursor.execute(
            "SELECT id, name, email, credit_limit "
            "FROM users WHERE active = TRUE ORDER BY name"
        )
        return cursor.fetchall()
    
    def insert(self, name: str, email: str, credit_limit: float) -> int:
        cursor = self._conn.cursor()
        cursor.execute(
            "INSERT INTO users (name, email, credit_limit) VALUES (%s, %s, %s)",
            (name, email, credit_limit)
        )
        self._conn.commit()
        return cursor.lastrowid
    
    def update_credit_limit(self, user_id: int, new_limit: float) -> None:
        cursor = self._conn.cursor()
        cursor.execute(
            "UPDATE users SET credit_limit = %s WHERE id = %s",
            (new_limit, user_id)
        )
        self._conn.commit()
    
    def delete(self, user_id: int) -> None:
        cursor = self._conn.cursor()
        cursor.execute("DELETE FROM users WHERE id = %s", (user_id,))
        self._conn.commit()
```

### 3.2 Cómo se usa

El código de la aplicación interactúa con el Table Data Gateway directamente, trabajando con diccionarios (datos en bruto):

```python
class OrderService:
    def __init__(self, users_gateway: UsersGateway, orders_gateway: OrdersGateway):
        self._users = users_gateway
        self._orders = orders_gateway
    
    def complete_order(self, order_id: int, user_id: int) -> None:
        user = self._users.find_by_id(user_id)  # dict
        if not user:
            raise ValueError("Usuario no encontrado")
        
        order = self._orders.find_by_id(order_id)  # dict
        if not order or order['user_id'] != user_id:
            raise ValueError("Orden no encontrada")
        
        # La lógica de negocio usa los dicts directamente
        if order['total'] > user['credit_limit']:
            raise ValueError("Crédito insuficiente")
        
        self._orders.update_status(order_id, 'completed')
```

### 3.3 Ventajas y desventajas

**Ventajas:**
- **Simplicidad extrema:** Es fácil de entender, escribir, y explicar a nuevos miembros del equipo.
- **SQL centralizado:** Toda la SQL para una tabla está en un lugar. Si cambia el esquema, hay un solo lugar a modificar.
- **Bajo overhead:** Sin abstracciones complejas, sin mapeos. Lo que ves es lo que ocurre.

**Desventajas:**
- **El código que llama trabaja con datos en bruto (dicts/records)**, no con objetos con comportamiento. La lógica de negocio tiene que saber qué campos contiene el dict.
- **No hay modelo de dominio:** Si "completar un pedido" implica varias reglas de negocio, esas reglas viven en el service o están dispersas. No hay un objeto `Order` que las encapsule.
- **No escala bien para lógica de dominio compleja:** Para aplicaciones CRUD simples es perfecto. Para dominios ricos, las limitaciones se vuelven obstáculos.

**Cuándo usar Table Data Gateway:** Aplicaciones CRUD simples (paneles de administración, dashboards, reportes), scripts de migración, servicios pequeños con lógica de negocio mínima. Es la elección de mínima complejidad.

---

## 4. Row Data Gateway: acceso a fila como objeto

### 4.1 La idea

El **Row Data Gateway** es un paso adelante del Table Data Gateway. En lugar de que el Gateway devuelva datos en bruto (dicts), cada fila de la base de datos se envuelve en un objeto que tiene campos con tipos correctos y métodos para acceder a ellos. Pero ese objeto no tiene lógica de negocio: es solo un contenedor de datos con acceso a la base de datos.

```python
class UserRow:
    """
    Row Data Gateway para una fila de la tabla 'users'.
    Envuelve una fila con propiedades tipadas y métodos de persistencia.
    No tiene lógica de negocio.
    """
    
    def __init__(self, connection, user_id: int, name: str, 
                 email: str, credit_limit: float):
        self._conn = connection
        self.id = user_id
        self.name = name
        self.email = email
        self.credit_limit = credit_limit
    
    @classmethod
    def find(cls, connection, user_id: int) -> 'UserRow | None':
        """Factory method: carga desde base de datos."""
        cursor = connection.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit FROM users WHERE id = %s",
            (user_id,)
        )
        row = cursor.fetchone()
        if not row:
            return None
        return cls(connection, row[0], row[1], row[2], row[3])
    
    @classmethod
    def find_by_email(cls, connection, email: str) -> 'UserRow | None':
        cursor = connection.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit FROM users WHERE email = %s",
            (email,)
        )
        row = cursor.fetchone()
        if not row:
            return None
        return cls(connection, row[0], row[1], row[2], row[3])
    
    def save(self) -> None:
        """Persiste el estado actual en la base de datos."""
        cursor = self._conn.cursor()
        cursor.execute(
            "UPDATE users SET name = %s, email = %s, credit_limit = %s WHERE id = %s",
            (self.name, self.email, self.credit_limit, self.id)
        )
        self._conn.commit()
    
    def delete(self) -> None:
        cursor = self._conn.cursor()
        cursor.execute("DELETE FROM users WHERE id = %s", (self.id,))
        self._conn.commit()

# Uso:
user_row = UserRow.find(conn, user_id=42)
if user_row:
    user_row.credit_limit = 5000.0
    user_row.save()
```

### 4.2 Diferencia con Table Data Gateway

La diferencia clave: en el Table Data Gateway, el objeto representa la **tabla** y devuelve datos en bruto. En el Row Data Gateway, el objeto representa una **fila específica** con sus campos tipados y métodos para persistirse.

```
Table Data Gateway:                Row Data Gateway:
UsersGateway                       UserRow (instancia de una fila específica)
  - Métodos de clase (find, etc.)    - Propiedades: id, name, email
  - Devuelve dicts                   - Métodos de instancia: save(), delete()
  - Representa la tabla entera       - Factory methods: find(), find_by_email()
```

### 4.3 Ventajas y desventajas

**Ventajas sobre Table Data Gateway:**
- Los datos tienen tipos correctos (no `row['credit_limit']` como string, sino `user.credit_limit` como float).
- Más legible: `user.credit_limit` es más claro que `user_data['credit_limit']`.

**Desventajas:**
- Sigue sin tener lógica de negocio: `UserRow` solo guarda y carga datos. La lógica de dominio sigue estando fuera.
- La distinción entre Row Data Gateway y Active Record es sutil. En la práctica, los desarrolladores frecuentemente añaden lógica de negocio directamente en el Row Data Gateway, convirtiendo el patrón en Active Record sin planearlo.

**Cuándo usar Row Data Gateway:** Es un patrón de transición. Se usa cuando se quiere tipado pero no se quiere la complejidad del Data Mapper. En la práctica moderna, es poco común porque se usa o Active Record (si se quiere algo simple) o Data Mapper (si se quiere separación de preocupaciones).

---

## 5. Active Record: el objeto que sabe persistirse

### 5.1 La idea central

El **Active Record** es el patrón más popular en frameworks como Ruby on Rails, Laravel, Django (parcialmente), y muchos otros. La idea es combinar en un solo objeto tanto los **datos del dominio** como la **lógica de persistencia**.

Un objeto Active Record es al mismo tiempo:
- Un objeto de dominio con propiedades y métodos de negocio.
- Un objeto que sabe cómo guardarse, cargarse, actualizarse y eliminarse de la base de datos.

```
Active Record:
┌────────────────────────────────────────────────┐
│  User (Active Record)                          │
├────────────────────────────────────────────────┤
│  Datos:                                        │
│    id: int                                     │
│    name: str                                   │
│    email: str                                  │
│    credit_limit: float                         │
├────────────────────────────────────────────────┤
│  Lógica de negocio:                            │
│    can_place_order(amount) → bool              │
│    upgrade_to_premium() → None                 │
│    send_welcome_email() → None                 │
├────────────────────────────────────────────────┤
│  Persistencia:                                 │
│    find(id) → User                [clase]      │
│    find_by_email(email) → User    [clase]      │
│    find_all_active() → List[User] [clase]      │
│    save() → None                  [instancia]  │
│    delete() → None                [instancia]  │
└────────────────────────────────────────────────┘
```

Implementación en Python:

```python
class User:
    """
    Active Record para la entidad User.
    Combina datos de dominio, lógica de negocio y persistencia.
    """
    
    # ── Persistencia (métodos de clase) ──────────────────────────────
    
    @classmethod
    def find(cls, user_id: int) -> 'User | None':
        db = Database.get_connection()
        cursor = db.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE id = %s",
            (user_id,)
        )
        row = cursor.fetchone()
        if not row:
            return None
        user = cls.__new__(cls)
        user.id = row[0]
        user.name = row[1]
        user.email = row[2]
        user.credit_limit = row[3]
        user.active = row[4]
        user._is_new = False
        return user
    
    @classmethod
    def find_all_active(cls) -> list['User']:
        db = Database.get_connection()
        cursor = db.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE active = TRUE"
        )
        results = []
        for row in cursor.fetchall():
            user = cls.__new__(cls)
            user.id, user.name, user.email = row[0], row[1], row[2]
            user.credit_limit, user.active = row[3], row[4]
            user._is_new = False
            results.append(user)
        return results
    
    def __init__(self, name: str, email: str, credit_limit: float = 0.0):
        """Constructor para crear nuevos usuarios (no para cargar desde DB)."""
        self.id = None
        self.name = name
        self.email = email
        self.credit_limit = credit_limit
        self.active = True
        self._is_new = True
    
    def save(self) -> None:
        db = Database.get_connection()
        cursor = db.cursor()
        if self._is_new:
            cursor.execute(
                "INSERT INTO users (name, email, credit_limit, active) "
                "VALUES (%s, %s, %s, %s)",
                (self.name, self.email, self.credit_limit, self.active)
            )
            self.id = cursor.lastrowid
            self._is_new = False
        else:
            cursor.execute(
                "UPDATE users SET name=%s, email=%s, credit_limit=%s, active=%s "
                "WHERE id = %s",
                (self.name, self.email, self.credit_limit, self.active, self.id)
            )
        db.commit()
    
    def delete(self) -> None:
        db = Database.get_connection()
        cursor = db.cursor()
        cursor.execute("DELETE FROM users WHERE id = %s", (self.id,))
        db.commit()
        self.id = None
        self._is_new = True
    
    # ── Lógica de negocio ─────────────────────────────────────────────
    
    def can_place_order(self, amount: float) -> bool:
        """Regla de negocio: el usuario puede realizar el pedido
        solo si tiene crédito suficiente."""
        return self.active and self.credit_limit >= amount
    
    def upgrade_to_premium(self) -> None:
        """Regla de negocio: actualizar el límite de crédito a nivel premium."""
        if not self.active:
            raise ValueError("No se puede actualizar un usuario inactivo.")
        self.credit_limit = max(self.credit_limit, 10000.0)
        self.save()
    
    def deactivate(self) -> None:
        """Regla de negocio: desactivar el usuario."""
        self.active = False
        self.save()
```

### 5.2 Cómo se usa

La elegancia del Active Record está en lo fluido que resulta el código que lo usa:

```python
# Crear un usuario nuevo:
user = User(name="María García", email="maria@ejemplo.com", credit_limit=1500.0)
user.save()

# Cargar y modificar:
user = User.find(42)
if user and user.can_place_order(200.0):
    # procesar el pedido...
    user.credit_limit -= 200.0
    user.save()

# Actualizar a premium:
user = User.find(42)
user.upgrade_to_premium()  # guarda solo si es necesario

# El código del servicio es limpio:
class OrderService:
    def process_order(self, user_id: int, amount: float) -> None:
        user = User.find(user_id)
        if not user:
            raise ValueError("Usuario no encontrado")
        if not user.can_place_order(amount):
            raise ValueError("Crédito insuficiente")
        
        order = Order(user_id=user.id, total=amount)
        order.save()
        
        user.credit_limit -= amount
        user.save()
```

### 5.3 El problema del Active Record: acoplamiento

La conveniencia del Active Record tiene un costo: el objeto de dominio **conoce la base de datos**. Esto crea acoplamiento en múltiples niveles:

**1. No testeable sin base de datos:**
```python
def test_can_place_order():
    # Para probar can_place_order, necesito un User...
    # que necesita una conexión a base de datos para cargarse.
    # No puedo instanciar un User en memoria sin que haya un INSERT en la DB.
    user = User.find(42)  # ← necesita DB
    assert user.can_place_order(100.0) == True
    # Los tests unitarios se convierten en tests de integración.
```

**2. La lógica de dominio y la persistencia están entrelazadas:**
Si mañana necesitas cambiar de PostgreSQL a MongoDB, o necesitas que tu `User` pueda venir de un servicio REST externo en lugar de una base de datos, el `User` mismo tiene que cambiar, porque lleva la SQL dentro.

**3. El modelo de dominio conoce el esquema:**
La estructura de la clase `User` refleja directamente las columnas de la tabla `users`. Si el esquema cambia, la clase cambia. El modelo de dominio está "anclado" al esquema relacional.

**4. Comportamiento implícito peligroso:**
`user.upgrade_to_premium()` tiene un side-effect oculto: llama a `self.save()` internamente. El código que llama no sabe que esta llamada produce SQL. En el contexto de una transacción más grande, puede generar múltiples commits parciales.

### 5.4 Cuándo el Active Record es la elección correcta

A pesar de sus limitaciones, el Active Record es la elección correcta cuando:

- **El dominio es simple y la aplicación es CRUD-heavy:** Aplicaciones web donde la mayoría de las operaciones son crear, leer, actualizar y eliminar registros, con poca lógica de negocio compleja.
- **La velocidad de desarrollo es prioritaria:** Rails, Laravel, Django con su Active Record integrado permiten construir aplicaciones funcionales muy rápidamente.
- **El equipo es pequeño:** Los problemas de testabilidad y acoplamiento se vuelven serios cuando muchos desarrolladores trabajan en el mismo código. En equipos pequeños, la simplicidad puede pesar más.
- **El esquema y el dominio son isomorfos:** Cuando la estructura de los objetos de dominio refleja naturalmente la estructura del esquema (un usuario tiene exactamente los campos de la tabla `users`), el Active Record es natural.

---

## 6. Data Mapper: la capa que separa los mundos

### 6.1 La idea central

El **Data Mapper** es el patrón que resuelve el acoplamiento del Active Record. La idea fundamental es:

> **Los objetos de dominio no saben nada de la base de datos. Un mapper separado se encarga de mover datos entre los objetos y la base de datos.**

```
┌──────────────────────┐        ┌───────────────────┐        ┌─────────────────┐
│  DOMINIO             │        │   DATA MAPPER     │        │  BASE DE DATOS  │
│                      │        │                   │        │                 │
│  class User:         │◀──────▶│  class UserMapper:│◀──────▶│  tabla users    │
│    id: int           │        │    find(id)        │        │    id           │
│    name: str         │        │    save(user)      │        │    name         │
│    email: str        │        │    delete(user)    │        │    email        │
│    credit_limit:float│        │    find_all_active │        │    credit_limit │
│                      │        │                   │        │                 │
│    # SOLO lógica de  │        │  # SOLO SQL y     │        │                 │
│    # negocio         │        │  # mapeo          │        │                 │
└──────────────────────┘        └───────────────────┘        └─────────────────┘
```

La diferencia con Active Record es radical: el objeto de dominio `User` no tiene ninguna referencia a la base de datos ni sabe cómo persistirse. Es un objeto puro de dominio.

```python
# El objeto de dominio: puro, sin SQL, sin imports de base de datos
class User:
    """
    Objeto de dominio puro. No sabe nada de la base de datos.
    Solo encapsula datos y lógica de negocio.
    Perfectamente testeable sin ninguna base de datos.
    """
    
    def __init__(self, user_id: int | None, name: str, 
                 email: str, credit_limit: float, active: bool = True):
        self.id = user_id
        self.name = name
        self.email = email
        self.credit_limit = credit_limit
        self.active = active
    
    # Solo lógica de negocio: sin SQL, sin save(), sin find()
    
    def can_place_order(self, amount: float) -> bool:
        return self.active and self.credit_limit >= amount
    
    def upgrade_to_premium(self) -> None:
        if not self.active:
            raise ValueError("No se puede actualizar un usuario inactivo.")
        self.credit_limit = max(self.credit_limit, 10000.0)
        # NOTA: NO hay self.save() aquí. El dominio no sabe persistirse.
    
    def deactivate(self) -> None:
        self.active = False
        # NOTA: NO hay self.save() aquí.
    
    def __eq__(self, other) -> bool:
        if not isinstance(other, User):
            return False
        return self.id == other.id
    
    def __repr__(self) -> str:
        return f"User(id={self.id}, name='{self.name}', email='{self.email}')"


# El Data Mapper: solo SQL y mapeo, sin lógica de negocio
class UserMapper:
    """
    Data Mapper para User.
    Traduce entre objetos User (dominio) y filas de la tabla users (DB).
    No tiene lógica de negocio: solo SQL y traducción.
    """
    
    def __init__(self, connection):
        self._conn = connection
    
    def find(self, user_id: int) -> User | None:
        """Carga un User desde la base de datos por su ID."""
        cursor = self._conn.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE id = %s",
            (user_id,)
        )
        row = cursor.fetchone()
        if not row:
            return None
        return self._row_to_user(row)
    
    def find_by_email(self, email: str) -> User | None:
        cursor = self._conn.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE email = %s",
            (email,)
        )
        row = cursor.fetchone()
        return self._row_to_user(row) if row else None
    
    def find_all_active(self) -> list[User]:
        cursor = self._conn.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE active = TRUE ORDER BY name"
        )
        return [self._row_to_user(row) for row in cursor.fetchall()]
    
    def save(self, user: User) -> None:
        """Persiste un User. Detecta automáticamente si es INSERT o UPDATE."""
        cursor = self._conn.cursor()
        if user.id is None:
            # INSERT
            cursor.execute(
                "INSERT INTO users (name, email, credit_limit, active) "
                "VALUES (%s, %s, %s, %s)",
                (user.name, user.email, user.credit_limit, user.active)
            )
            user.id = cursor.lastrowid  # Asigna el ID generado
        else:
            # UPDATE
            cursor.execute(
                "UPDATE users SET name=%s, email=%s, credit_limit=%s, active=%s "
                "WHERE id = %s",
                (user.name, user.email, user.credit_limit, user.active, user.id)
            )
    
    def delete(self, user: User) -> None:
        cursor = self._conn.cursor()
        cursor.execute("DELETE FROM users WHERE id = %s", (user.id,))
        user.id = None
    
    def _row_to_user(self, row) -> User:
        """Traduce una fila de la DB a un objeto User."""
        return User(
            user_id=row[0],
            name=row[1],
            email=row[2],
            credit_limit=row[3],
            active=bool(row[4])
        )
```

### 6.2 Testabilidad: el beneficio más importante

Con el Data Mapper, la lógica de dominio es completamente testeable sin base de datos:

```python
# Test unitario puro: sin base de datos, sin SQL, sin fixtures
def test_can_place_order_with_sufficient_credit():
    user = User(user_id=1, name="Test", email="test@test.com", credit_limit=1000.0)
    assert user.can_place_order(500.0) == True

def test_cannot_place_order_with_insufficient_credit():
    user = User(user_id=1, name="Test", email="test@test.com", credit_limit=100.0)
    assert user.can_place_order(500.0) == False

def test_upgrade_to_premium_sets_minimum_limit():
    user = User(user_id=1, name="Test", email="test@test.com", credit_limit=500.0)
    user.upgrade_to_premium()
    assert user.credit_limit == 10000.0

def test_cannot_upgrade_inactive_user():
    user = User(user_id=1, name="Test", email="test@test.com", 
                credit_limit=500.0, active=False)
    with pytest.raises(ValueError):
        user.upgrade_to_premium()
```

Estos tests se ejecutan en microsegundos porque no hay acceso a base de datos. Son deterministas: siempre dan el mismo resultado. Son fáciles de escribir y mantener.

### 6.3 Cómo se usa el Data Mapper en el servicio

```python
class OrderService:
    def __init__(self, user_mapper: UserMapper, order_mapper: OrderMapper, 
                 connection):
        self._user_mapper = user_mapper
        self._order_mapper = order_mapper
        self._conn = connection
    
    def process_order(self, user_id: int, amount: float) -> int:
        # El servicio trabaja con objetos de dominio, no con SQL
        user = self._user_mapper.find(user_id)
        if not user:
            raise ValueError("Usuario no encontrado")
        
        if not user.can_place_order(amount):
            raise ValueError("Crédito insuficiente")
        
        # Modificar el dominio (sin saber nada de SQL)
        order = Order(user_id=user.id, total=amount, status='pending')
        user.credit_limit -= amount
        
        # Persistir los cambios (el mapper sabe el SQL)
        self._conn.begin()
        try:
            self._order_mapper.save(order)
            self._user_mapper.save(user)
            self._conn.commit()
        except Exception:
            self._conn.rollback()
            raise
        
        return order.id
```

### 6.4 La desventaja del Data Mapper: complejidad

El Data Mapper tiene una desventaja real: requiere escribir y mantener más código. Para cada entidad del dominio, se necesita tanto la clase de dominio como el mapper. En aplicaciones con muchas entidades, esto puede ser una cantidad significativa de código de infraestructura.

Esta es la razón por la que los ORMs existen: automatizan la generación del Data Mapper (y otros patrones) a partir de metadatos (anotaciones, convenciones, configuración). Hibernate en Java, SQLAlchemy en Python, y Entity Framework en .NET son implementaciones de Data Mapper con meta-mapeo.

---

## 7. Repository: la colección que habla con la base de datos

### 7.1 La idea central

El **Repository** es quizás el patrón más influyente de los catalogados por Fowler, especialmente en el contexto del Diseño Dirigido por el Dominio (Domain-Driven Design, DDD) de Eric Evans. La idea es:

> **El Repository es una abstracción que simula ser una colección en memoria de todos los objetos de un tipo.** El código de dominio le pide al Repository que le dé objetos o que guarde objetos, sin saber si esos objetos vienen de una base de datos, un archivo, una caché, un servicio REST, o están en memoria.

```python
# Desde la perspectiva del código que usa el Repository,
# es como si los usuarios estuvieran todos en memoria:

# "Dame el usuario con id 42"
user = user_repository.get(42)

# "Dame todos los usuarios premium"
premium_users = user_repository.find_all_premium()

# "Guarda este usuario"
user_repository.save(user)

# "Elimina este usuario"
user_repository.remove(user)

# El código NO sabe si esto va a SQL, Redis, un archivo CSV, o está en memoria.
# Solo trabaja con la abstracción de "colección de usuarios".
```

### 7.2 Cómo se diferencia del Data Mapper

La diferencia entre Repository y Data Mapper es de abstracción e intención:

- **Data Mapper:** Sabe que hay una base de datos. Sus métodos tienen nombres que reflejan operaciones de persistencia (`find`, `save`, `delete`). Es una capa de traducción.
- **Repository:** Finge que no hay base de datos. Sus métodos tienen nombres que reflejan operaciones sobre colecciones (`get`, `find_all_by_criteria`, `add`, `remove`). Es una abstracción de dominio.

En la práctica, un Repository típicamente usa un Data Mapper (o un ORM) en su implementación.

```python
from abc import ABC, abstractmethod

# Interfaz del Repository (contrato del dominio)
class UserRepository(ABC):
    """
    Interfaz del Repository de usuarios.
    Define el contrato desde la perspectiva del dominio.
    No menciona SQL, tablas, ni base de datos.
    """
    
    @abstractmethod
    def get(self, user_id: int) -> User | None:
        """Retorna el usuario con el id dado, o None si no existe."""
        ...
    
    @abstractmethod
    def get_by_email(self, email: str) -> User | None:
        """Retorna el usuario con el email dado, o None si no existe."""
        ...
    
    @abstractmethod
    def find_all_active(self) -> list[User]:
        """Retorna todos los usuarios activos."""
        ...
    
    @abstractmethod
    def find_premium(self) -> list[User]:
        """Retorna los usuarios con crédito >= 10,000."""
        ...
    
    @abstractmethod
    def save(self, user: User) -> None:
        """Persiste el usuario (INSERT si es nuevo, UPDATE si existe)."""
        ...
    
    @abstractmethod
    def remove(self, user: User) -> None:
        """Elimina el usuario de la colección."""
        ...


# Implementación concreta: va a PostgreSQL
class PostgresUserRepository(UserRepository):
    """
    Implementación concreta del UserRepository usando PostgreSQL.
    Esta clase sí sabe sobre la base de datos, pero es un detalle
    de infraestructura oculto detrás de la interfaz abstracta.
    """
    
    def __init__(self, connection):
        self._conn = connection
    
    def get(self, user_id: int) -> User | None:
        cursor = self._conn.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE id = %s",
            (user_id,)
        )
        row = cursor.fetchone()
        return self._to_user(row) if row else None
    
    def get_by_email(self, email: str) -> User | None:
        cursor = self._conn.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE email = %s",
            (email,)
        )
        row = cursor.fetchone()
        return self._to_user(row) if row else None
    
    def find_all_active(self) -> list[User]:
        cursor = self._conn.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE active = TRUE ORDER BY name"
        )
        return [self._to_user(row) for row in cursor.fetchall()]
    
    def find_premium(self) -> list[User]:
        cursor = self._conn.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE credit_limit >= 10000 AND active = TRUE"
        )
        return [self._to_user(row) for row in cursor.fetchall()]
    
    def save(self, user: User) -> None:
        cursor = self._conn.cursor()
        if user.id is None:
            cursor.execute(
                "INSERT INTO users (name, email, credit_limit, active) "
                "VALUES (%s, %s, %s, %s)",
                (user.name, user.email, user.credit_limit, user.active)
            )
            user.id = cursor.lastrowid
        else:
            cursor.execute(
                "UPDATE users SET name=%s, email=%s, credit_limit=%s, active=%s "
                "WHERE id = %s",
                (user.name, user.email, user.credit_limit, user.active, user.id)
            )
    
    def remove(self, user: User) -> None:
        cursor = self._conn.cursor()
        cursor.execute("DELETE FROM users WHERE id = %s", (user.id,))
        user.id = None
    
    def _to_user(self, row) -> User:
        return User(row[0], row[1], row[2], float(row[3]), bool(row[4]))


# Implementación alternativa: va a memoria (para tests)
class InMemoryUserRepository(UserRepository):
    """
    Implementación en memoria del UserRepository.
    Usada en tests unitarios: sin base de datos, extremadamente rápida.
    """
    
    def __init__(self):
        self._users: dict[int, User] = {}
        self._next_id: int = 1
    
    def get(self, user_id: int) -> User | None:
        return self._users.get(user_id)
    
    def get_by_email(self, email: str) -> User | None:
        return next(
            (u for u in self._users.values() if u.email == email), 
            None
        )
    
    def find_all_active(self) -> list[User]:
        return [u for u in self._users.values() if u.active]
    
    def find_premium(self) -> list[User]:
        return [u for u in self._users.values() 
                if u.active and u.credit_limit >= 10000]
    
    def save(self, user: User) -> None:
        if user.id is None:
            user.id = self._next_id
            self._next_id += 1
        self._users[user.id] = user
    
    def remove(self, user: User) -> None:
        self._users.pop(user.id, None)
        user.id = None
```

### 7.3 El beneficio del polimorfismo

El poder del Repository como interfaz abstracta es que el código de negocio puede trabajar con cualquier implementación sin cambios:

```python
class OrderService:
    """
    Este servicio trabaja con cualquier implementación de UserRepository.
    No sabe si los usuarios vienen de PostgreSQL, MySQL, o están en memoria.
    """
    
    def __init__(self, user_repo: UserRepository, order_repo: OrderRepository):
        self._users = user_repo
        self._orders = order_repo
    
    def process_order(self, user_id: int, amount: float) -> int:
        user = self._users.get(user_id)
        if not user:
            raise ValueError("Usuario no encontrado")
        if not user.can_place_order(amount):
            raise ValueError("Crédito insuficiente")
        
        order = Order(user_id=user.id, total=amount, status='pending')
        user.credit_limit -= amount
        
        self._orders.save(order)
        self._users.save(user)
        
        return order.id

# En producción:
service = OrderService(
    PostgresUserRepository(pg_connection),
    PostgresOrderRepository(pg_connection)
)

# En tests: sin base de datos
user_repo = InMemoryUserRepository()
user_repo.save(User(None, "Test User", "test@test.com", 1000.0))
service = OrderService(user_repo, InMemoryOrderRepository())
result = service.process_order(1, 200.0)
assert result is not None
```

### 7.4 El Specification Pattern: criterios de búsqueda como objetos

Un problema que aparece con los Repositories es cómo manejar queries complejas. Con un conjunto fijo de métodos (`find_all_active`, `find_premium`, `find_by_email`), el Repository puede crecer indefinidamente a medida que el dominio necesita más formas de filtrar.

El **Specification Pattern** complementa al Repository encapsulando los criterios de búsqueda en objetos:

```python
# Especificación: un objeto que representa un criterio de filtrado
class Specification(ABC):
    @abstractmethod
    def is_satisfied_by(self, candidate: User) -> bool: ...
    
    def and_(self, other: 'Specification') -> 'AndSpecification':
        return AndSpecification(self, other)
    
    def or_(self, other: 'Specification') -> 'OrSpecification':
        return OrSpecification(self, other)

class ActiveUserSpec(Specification):
    def is_satisfied_by(self, user: User) -> bool:
        return user.active

class PremiumUserSpec(Specification):
    def is_satisfied_by(self, user: User) -> bool:
        return user.credit_limit >= 10000.0

class EmailDomainSpec(Specification):
    def __init__(self, domain: str):
        self._domain = domain
    
    def is_satisfied_by(self, user: User) -> bool:
        return user.email.endswith(f"@{self._domain}")

class AndSpecification(Specification):
    def __init__(self, left: Specification, right: Specification):
        self._left, self._right = left, right
    
    def is_satisfied_by(self, user: User) -> bool:
        return self._left.is_satisfied_by(user) and self._right.is_satisfied_by(user)

# Uso: composición de criterios
active_premium = ActiveUserSpec().and_(PremiumUserSpec())
corporate_users = EmailDomainSpec("empresa.com").and_(ActiveUserSpec())

# El Repository acepta especificaciones:
users = user_repository.find_by_spec(active_premium)
```

---

## 8. Unit of Work: rastrear cambios para persistirlos juntos

### 8.1 El problema que resuelve

En el ejemplo del Data Mapper y el Repository, cuando el servicio modifica tanto un `User` como un `Order`, llama explícitamente a `save()` para cada uno. Esto tiene varios problemas:

**1. Multiple commits parciales:**
```python
# Si esto falla después del primer save pero antes del segundo:
self._orders.save(order)   # ← commit #1 (orden guardada)
self._users.save(user)     # ← si falla aquí, la orden está guardada pero
                           #   el crédito del usuario no se actualizó.
                           #   Estado inconsistente.
```

**2. El servicio tiene que saber el orden correcto:**
En un dominio complejo con muchas entidades interrelacionadas, determinar en qué orden deben guardarse para no violar constraints de FK puede ser complicado.

**3. Operaciones innecesarias:**
Si el mismo objeto se modifica múltiples veces en la misma unidad de trabajo, sin un mecanismo de rastreo puede terminar siendo guardado múltiples veces (múltiples UPDATEs del mismo registro).

### 8.2 La solución: rastrear el estado de los objetos

El **Unit of Work** es un objeto que rastrea todos los cambios realizados a los objetos de dominio durante una operación de negocio, y al final de esa operación, persiste todos esos cambios en una sola transacción de base de datos.

```
┌─────────────────────────────────────────────────────────────────┐
│                         UNIT OF WORK                           │
│                                                                 │
│  Registros internos:                                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  new_objects:     [Order(id=None)]  ← para INSERT       │   │
│  │  dirty_objects:   [User(id=42)]     ← para UPDATE       │   │
│  │  removed_objects: []               ← para DELETE        │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  Métodos:                                                       │
│    register_new(obj)    ← marcar para INSERT                   │
│    register_dirty(obj)  ← marcar para UPDATE                   │
│    register_removed(obj)← marcar para DELETE                   │
│    commit()             ← ejecutar todos los cambios en una    │
│                           sola transacción                     │
└─────────────────────────────────────────────────────────────────┘
```

```python
class UnitOfWork:
    """
    Unit of Work: rastrea todos los cambios y los persiste 
    en una sola transacción atómica.
    """
    
    def __init__(self, connection):
        self._conn = connection
        self._new: list = []      # Objetos a insertar
        self._dirty: list = []    # Objetos a actualizar
        self._removed: list = []  # Objetos a eliminar
        
        # Mappers registrados para cada tipo de objeto
        self._mappers: dict = {}
    
    def register_mapper(self, entity_class, mapper):
        self._mappers[entity_class] = mapper
    
    def register_new(self, obj) -> None:
        """Marca un objeto como nuevo (pendiente de INSERT)."""
        assert obj not in self._dirty, "Un objeto nuevo no puede estar dirty"
        assert obj not in self._removed, "Un objeto removido no puede ser nuevo"
        if obj not in self._new:
            self._new.append(obj)
    
    def register_dirty(self, obj) -> None:
        """Marca un objeto como modificado (pendiente de UPDATE)."""
        assert obj not in self._removed, "Un objeto removido no puede estar dirty"
        if obj not in self._dirty and obj not in self._new:
            self._dirty.append(obj)
    
    def register_removed(self, obj) -> None:
        """Marca un objeto como eliminado (pendiente de DELETE)."""
        if obj in self._new:
            self._new.remove(obj)
            return  # Era nuevo: simplemente lo descartamos
        if obj in self._dirty:
            self._dirty.remove(obj)
        if obj not in self._removed:
            self._removed.append(obj)
    
    def commit(self) -> None:
        """Ejecuta todos los cambios pendientes en una sola transacción."""
        try:
            self._conn.begin()
            
            # INSERT todos los objetos nuevos
            for obj in self._new:
                mapper = self._mappers[type(obj)]
                mapper.insert(obj)
            
            # UPDATE todos los objetos modificados
            for obj in self._dirty:
                mapper = self._mappers[type(obj)]
                mapper.update(obj)
            
            # DELETE todos los objetos removidos
            for obj in self._removed:
                mapper = self._mappers[type(obj)]
                mapper.delete(obj)
            
            self._conn.commit()
            self._clear()
            
        except Exception:
            self._conn.rollback()
            self._clear()
            raise
    
    def rollback(self) -> None:
        self._conn.rollback()
        self._clear()
    
    def _clear(self) -> None:
        self._new.clear()
        self._dirty.clear()
        self._removed.clear()
    
    # Context manager para uso idiomático
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.commit()
        else:
            self.rollback()
        return False  # No suprime la excepción
```

### 8.3 Cómo se usa el Unit of Work

```python
class OrderService:
    def __init__(self, user_repo: UserRepository, order_repo: OrderRepository,
                 uow_factory):
        self._users = user_repo
        self._orders = order_repo
        self._uow_factory = uow_factory
    
    def process_order(self, user_id: int, amount: float) -> int:
        user = self._users.get(user_id)
        if not user:
            raise ValueError("Usuario no encontrado")
        if not user.can_place_order(amount):
            raise ValueError("Crédito insuficiente")
        
        order = Order(user_id=user.id, total=amount, status='pending')
        user.credit_limit -= amount
        
        # Todos los cambios se aplican en UNA sola transacción
        with self._uow_factory() as uow:
            uow.register_new(order)          # → INSERT
            uow.register_dirty(user)         # → UPDATE
            # uow.commit() se llama automáticamente al salir del with
        
        return order.id
    
    def cancel_order(self, order_id: int, user_id: int) -> None:
        order = self._orders.get(order_id)
        user = self._users.get(user_id)
        
        if not order or order.user_id != user_id:
            raise ValueError("Orden no encontrada")
        if order.status != 'pending':
            raise ValueError("Solo se pueden cancelar órdenes pendientes")
        
        order.status = 'cancelled'
        user.credit_limit += order.total   # Devolver el crédito
        
        with self._uow_factory() as uow:
            uow.register_dirty(order)     # → UPDATE orders
            uow.register_dirty(user)      # → UPDATE users
            # Ambos UPDATEs en la misma transacción
```

### 8.4 Variante automática: change tracking

Los ORMs modernos implementan el Unit of Work con **change tracking automático**: el mapper (o el ORM) registra una "foto" del estado de cada objeto cuando se carga, y al hacer `commit()`, compara el estado actual con la foto para detectar automáticamente qué objetos cambiaron.

```python
class TrackingUnitOfWork(UnitOfWork):
    """
    Variante con change tracking automático.
    Los objetos no necesitan llamar register_dirty() manualmente.
    """
    
    def __init__(self, connection):
        super().__init__(connection)
        self._snapshots: dict[int, dict] = {}  # id → estado original
    
    def track(self, obj) -> None:
        """Toma una 'foto' del estado del objeto al cargarlo."""
        snapshot = {k: v for k, v in vars(obj).items() 
                    if not k.startswith('_')}
        self._snapshots[id(obj)] = snapshot
    
    def commit(self) -> None:
        """Detecta automáticamente los objetos que cambiaron."""
        for obj_id, snapshot in self._snapshots.items():
            obj = ...  # recuperar el objeto por id (en memoria)
            current = {k: v for k, v in vars(obj).items() 
                      if not k.startswith('_')}
            if current != snapshot:  # ¿cambió algo?
                if obj not in self._dirty and obj not in self._new:
                    self._dirty.append(obj)
        
        super().commit()
```

Hibernate (Java), Entity Framework (.NET) y SQLAlchemy (Python) implementan change tracking de esta forma.

---

## 9. Identity Map: una sola instancia por registro

### 9.1 El problema: múltiples instancias del mismo registro

Sin un mecanismo de control, es posible que el mismo registro de base de datos produzca múltiples instancias distintas en memoria:

```python
# Sin Identity Map:
user_a = user_repo.get(42)  # Carga user id=42 desde DB → instancia A
user_b = user_repo.get(42)  # Carga user id=42 desde DB → instancia B (diferente objeto)

user_a is user_b  # False: son objetos distintos en memoria
user_a == user_b  # Depende de cómo esté implementado __eq__

# Problema:
user_a.credit_limit = 5000.0
# user_b.credit_limit sigue siendo el valor original
# Hay dos "versiones" del mismo usuario en memoria
# ¿Cuál es el "verdadero"?

user_repo.save(user_a)  # Guarda credit_limit = 5000.0
user_repo.save(user_b)  # Sobreescribe con el valor original !
```

Este problema es especialmente grave cuando el mismo objeto es accedido por múltiples partes del sistema dentro de la misma unidad de trabajo (mismo request HTTP, misma transacción).

### 9.2 La solución: un mapa de objetos ya cargados

El **Identity Map** es un diccionario en memoria que mapea identidades (claves primarias) a instancias de objetos. Antes de cargar un objeto desde la base de datos, se verifica si ya existe en el mapa.

```python
class IdentityMap:
    """
    Identity Map: garantiza que una clave primaria mapee a 
    una única instancia en memoria por unidad de trabajo.
    """
    
    def __init__(self):
        # { tipo_clase → { id → instancia } }
        self._registry: dict[type, dict[any, object]] = {}
    
    def get(self, entity_class: type, entity_id) -> object | None:
        """Retorna la instancia si ya fue cargada, None si no."""
        return self._registry.get(entity_class, {}).get(entity_id)
    
    def put(self, entity_id, entity: object) -> None:
        """Registra una instancia en el mapa."""
        entity_class = type(entity)
        if entity_class not in self._registry:
            self._registry[entity_class] = {}
        self._registry[entity_class][entity_id] = entity
    
    def contains(self, entity_class: type, entity_id) -> bool:
        return entity_id in self._registry.get(entity_class, {})
    
    def remove(self, entity_class: type, entity_id) -> None:
        if entity_class in self._registry:
            self._registry[entity_class].pop(entity_id, None)
    
    def clear(self) -> None:
        self._registry.clear()


# El Repository usa el Identity Map:
class PostgresUserRepository(UserRepository):
    
    def __init__(self, connection, identity_map: IdentityMap):
        self._conn = connection
        self._identity_map = identity_map
    
    def get(self, user_id: int) -> User | None:
        # Primero, verificar si ya está en el mapa
        cached = self._identity_map.get(User, user_id)
        if cached is not None:
            return cached  # Devuelve la misma instancia: sin consulta a DB
        
        # Si no está, cargar desde DB
        cursor = self._conn.cursor()
        cursor.execute(
            "SELECT id, name, email, credit_limit, active "
            "FROM users WHERE id = %s",
            (user_id,)
        )
        row = cursor.fetchone()
        if not row:
            return None
        
        user = self._to_user(row)
        self._identity_map.put(user.id, user)  # Registrar en el mapa
        return user
    
    def save(self, user: User) -> None:
        # ... SQL de save ...
        # Asegurarse de que el mapa tenga la instancia actualizada
        if user.id:
            self._identity_map.put(user.id, user)
    
    # ... resto de métodos ...
```

### 9.3 Identity Map y Unit of Work son complementarios

El Identity Map y el Unit of Work trabajan juntos: el Identity Map garantiza que hay una sola instancia por registro (evitando conflictos de escritura), y el Unit of Work rastrea todos los cambios a esas instancias para confirmarlos juntos.

En la práctica, ambos patrones suelen vivir en el mismo objeto (la "sesión" o el contexto de persistencia), que también actúa como el ámbito de una unidad de trabajo:

```python
class PersistenceContext:
    """
    Combina Unit of Work e Identity Map en un solo objeto.
    Es equivalente a la 'Session' de SQLAlchemy o el
    'EntityManager' de JPA/Hibernate.
    """
    
    def __init__(self, connection):
        self._conn = connection
        self._identity_map = IdentityMap()
        self._new: list = []
        self._dirty: list = []
        self._removed: list = []
    
    def load_user(self, user_id: int) -> User | None:
        # Primero: verificar Identity Map
        user = self._identity_map.get(User, user_id)
        if user:
            return user  # Sin acceso a DB
        
        # Cargar desde DB
        cursor = self._conn.cursor()
        cursor.execute("SELECT ... FROM users WHERE id = %s", (user_id,))
        row = cursor.fetchone()
        if not row:
            return None
        
        user = User(row[0], row[1], row[2], float(row[3]), bool(row[4]))
        self._identity_map.put(user_id, user)  # Registrar
        return user
    
    def add(self, entity) -> None:
        """Registra un nuevo objeto para INSERT."""
        self._new.append(entity)
    
    def mark_modified(self, entity) -> None:
        """Marca un objeto como modificado."""
        if entity not in self._dirty and entity not in self._new:
            self._dirty.append(entity)
    
    def commit(self) -> None:
        """Confirma todos los cambios en una sola transacción."""
        # ... igual que UnitOfWork.commit() ...
```

---

## 10. Lazy Load: cargar datos solo cuando se necesitan

### 10.1 El problema: los grafos de objetos y el costo de cargarlos

Un objeto `User` en un sistema de e-commerce puede tener relacionados: `orders` (pedidos), y cada `Order` tiene `order_items`, y cada `OrderItem` tiene un `Product`, y cada `Product` tiene una `Category`. Es un grafo.

```
User
└── orders: List[Order]
    └── items: List[OrderItem]
        └── product: Product
            └── category: Category
```

Si se carga todo el grafo eagerly (con anticipación) cuando se carga un `User`, se ejecutarán múltiples JOINs costosos aunque el código que pidió el usuario solo necesite su nombre y email. No tiene sentido.

```python
# Carga eager: trae TODO el grafo aunque no se necesite
user = user_repo.get_with_all_relations(user_id=42)
# ↑ ejecuta: 
#   SELECT users...
#   JOIN orders ON ...
#   JOIN order_items ON ...
#   JOIN products ON ...
#   JOIN categories ON ...
# Todo esto para mostrar solo user.name
print(user.name)
```

El **Lazy Load** resuelve esto: los datos relacionados no se cargan hasta que se accede a ellos por primera vez.

### 10.2 Las cuatro implementaciones de Lazy Load

Fowler documentó cuatro formas de implementar el Lazy Load, con distintos trade-offs:

**Implementación 1: Lazy Initialization (la más simple)**

El campo relacionado es `None` inicialmente. La primera vez que se accede a él, se carga desde la base de datos.

```python
class User:
    def __init__(self, user_id: int, name: str, email: str, 
                 credit_limit: float, active: bool):
        self.id = user_id
        self.name = name
        self.email = email
        self.credit_limit = credit_limit
        self.active = active
        self._orders = None  # None = no cargado aún; [] = cargado pero vacío
    
    @property
    def orders(self) -> list['Order']:
        """Lazy load: carga los pedidos la primera vez que se accede."""
        if self._orders is None:
            # El repository está inyectado o se obtiene de algún registry
            self._orders = order_repo.find_by_user_id(self.id)
        return self._orders
```

**Problema de Lazy Initialization:** El objeto de dominio necesita una referencia al repository o a la base de datos, rompiendo el principio de separación. El objeto de dominio "puro" ya no es tan puro.

**Implementación 2: Virtual Proxy**

En lugar de devolver la lista de pedidos directamente, se devuelve un **proxy** que se comporta exactamente como una lista, pero carga los datos de la base de datos la primera vez que alguien la usa.

```python
class LazyLoadingList:
    """
    Virtual Proxy para una colección lazy.
    Se comporta como una lista normal, pero carga los datos 
    desde la base de datos la primera vez que se accede.
    """
    
    def __init__(self, loader_fn):
        """
        loader_fn: función sin argumentos que retorna la lista real
                   cuando se llama.
        """
        self._loader = loader_fn
        self._data: list | None = None
    
    def _ensure_loaded(self) -> None:
        if self._data is None:
            self._data = self._loader()
    
    def __iter__(self):
        self._ensure_loaded()
        return iter(self._data)
    
    def __len__(self):
        self._ensure_loaded()
        return len(self._data)
    
    def __getitem__(self, index):
        self._ensure_loaded()
        return self._data[index]
    
    def __repr__(self):
        if self._data is None:
            return "LazyLoadingList(not loaded)"
        return f"LazyLoadingList({self._data!r})"

# En el mapper:
class UserMapper:
    def find(self, user_id: int) -> User | None:
        cursor = self._conn.cursor()
        cursor.execute("SELECT ... FROM users WHERE id = %s", (user_id,))
        row = cursor.fetchone()
        if not row:
            return None
        
        user = User(row[0], row[1], row[2], float(row[3]), bool(row[4]))
        
        # Asigna un proxy que cargará los pedidos CUANDO se acceda a ellos
        user.orders = LazyLoadingList(
            loader_fn=lambda: self._order_mapper.find_by_user_id(user.id)
        )
        
        return user

# Uso: idéntico, el proxy es transparente
user = user_mapper.find(42)
print(user.name)         # Sin acceso a DB (solo campos directos)
print(len(user.orders))  # AQUÍ se dispara la query a la DB (primer acceso)
for order in user.orders:  # Ya cargado, sin nueva query
    print(order.total)
```

**Implementación 3: Value Holder**

Similar al Virtual Proxy, pero en lugar de un proxy que imita a la colección, se usa un objeto envoltorio explícito con un método `get()`:

```python
class ValueHolder:
    def __init__(self, loader_fn):
        self._loader = loader_fn
        self._value = None
        self._loaded = False
    
    def get(self):
        if not self._loaded:
            self._value = self._loader()
            self._loaded = True
        return self._value

# Uso:
user.orders_holder = ValueHolder(lambda: order_mapper.find_by_user_id(user.id))
orders = user.orders_holder.get()  # Carga en el primer .get()
```

La diferencia con Virtual Proxy es que el Value Holder no imita a la colección: requiere llamar `.get()` explícitamente. Es más explícito pero menos transparente.

**Implementación 4: Ghost**

Un **Ghost** es un objeto que sabe su ID pero que no tiene ninguno de sus datos cargados. La primera vez que se accede a cualquier campo (además del ID), se carga todo el objeto desde la base de datos.

```python
class GhostUser:
    """
    Ghost: objeto que solo tiene su ID cargado.
    Al acceder a cualquier otro campo, se carga el resto desde DB.
    """
    
    STATE_GHOST = 'ghost'     # Solo tiene ID
    STATE_LOADED = 'loaded'   # Todos los datos están en memoria
    
    def __init__(self, user_id: int):
        object.__setattr__(self, '_id', user_id)
        object.__setattr__(self, '_state', self.STATE_GHOST)
        object.__setattr__(self, '_name', None)
        object.__setattr__(self, '_email', None)
        object.__setattr__(self, '_credit_limit', None)
    
    def __getattribute__(self, name):
        if name.startswith('_') or name == 'id':
            return object.__getattribute__(self, name)
        
        # Cargar si es ghost
        state = object.__getattribute__(self, '_state')
        if state == GhostUser.STATE_GHOST:
            # Cargar desde DB
            user_id = object.__getattribute__(self, '_id')
            self._load_from_db(user_id)
        
        return object.__getattribute__(self, f'_{name}')
    
    def _load_from_db(self, user_id: int) -> None:
        # Aquí iría el acceso a DB
        row = db.execute("SELECT ... FROM users WHERE id = %s", (user_id,))
        object.__setattr__(self, '_name', row[1])
        object.__setattr__(self, '_email', row[2])
        object.__setattr__(self, '_credit_limit', float(row[3]))
        object.__setattr__(self, '_state', GhostUser.STATE_LOADED)
    
    @property
    def id(self): return self._id
    @property
    def name(self): return self._name       # Dispara carga si es ghost
    @property
    def email(self): return self._email     # Dispara carga si es ghost
```

### 10.3 El problema más importante del Lazy Load: el N+1

El Lazy Load es conveniente pero puede destruir el rendimiento cuando se usa en bucles. Este es el famoso **problema N+1**:

```python
# Con lazy loading en una colección:
users = user_repo.find_all_active()  # 1 query: SELECT * FROM users WHERE active=TRUE
                                     # Retorna 1000 usuarios

for user in users:
    print(f"{user.name}: {len(user.orders)} órdenes")
    # user.orders dispara una query POR CADA USUARIO
    # → SELECT * FROM orders WHERE user_id = 1
    # → SELECT * FROM orders WHERE user_id = 2
    # → SELECT * FROM orders WHERE user_id = 3
    # ... 1000 queries más
    
# Total: 1 query inicial + 1000 queries lazy = 1001 queries
# De ahí viene el nombre: N+1
```

El N+1 es insidioso porque no es visible en el código: `user.orders` parece una simple propiedad. Solo se revela cuando se monitoriza la base de datos (o se usa un profiler de SQL).

**Solución:** Eager loading explícito cuando se sabe que se necesitarán las relaciones:

```python
# Con eager loading (JOIN en la query inicial):
users_with_orders = user_repo.find_all_active_with_orders()
# → SELECT u.*, o.* FROM users u LEFT JOIN orders o ON o.user_id = u.id
#   WHERE u.active = TRUE
# 1 query total, sin importar cuántos usuarios haya

for user in users_with_orders:
    print(f"{user.name}: {len(user.orders)} órdenes")
    # user.orders ya está cargado: sin queries adicionales
```

El módulo 15 profundiza en el N+1 en el contexto de los ORMs.

---

## 11. Cómo los patrones se combinan en la práctica

### 11.1 La arquitectura típica con todos los patrones

En una aplicación bien diseñada, los patrones de Fowler no se usan de forma aislada. Se combinan en capas:

```
┌─────────────────────────────────────────────────────────────────────────┐
│  APPLICATION SERVICE (Servicio de aplicación)                           │
│  Orquesta el flujo de negocio. No sabe de SQL.                         │
│                                                                         │
│  process_order(user_id, amount):                                        │
│    user = user_repo.get(user_id)        ← Repository                  │
│    order = Order(...)                   ← Domain Object                │
│    user.credit_limit -= amount          ← Domain Logic                 │
│    uow.add(order)                       ← Unit of Work                 │
│    uow.mark_modified(user)              ← Unit of Work                 │
│    uow.commit()                         ← Unit of Work                 │
└─────────────────────────────────────────────────────────────────────────┘
           │                │
           ▼                ▼
┌──────────────────┐  ┌─────────────────────────────────────────────────┐
│  REPOSITORY      │  │  UNIT OF WORK + IDENTITY MAP                    │
│  UserRepository  │  │                                                  │
│  OrderRepository │  │  Rastrea cambios.                               │
│                  │  │  Garantiza una instancia por ID.                │
│  Abstrae el      │  │  Confirma en una transacción.                   │
│  acceso a datos. │  │                                                  │
└──────────────────┘  └─────────────────────────────────────────────────┘
           │                │
           ▼                ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  DATA MAPPER                                                            │
│  UserMapper, OrderMapper                                                │
│  Traduce entre objetos de dominio y filas de la DB.                    │
│  Contiene toda la SQL.                                                  │
└─────────────────────────────────────────────────────────────────────────┘
           │
           ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  BASE DE DATOS (PostgreSQL, MySQL, SQLite, etc.)                        │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.2 Los ORMs son implementaciones de estos patrones

Los ORMs modernos son, esencialmente, implementaciones genéricas y automáticas de los patrones de Fowler. La equivalencia es directa:

| Patrón de Fowler | Implementación en SQLAlchemy (Python) | Implementación en Hibernate (Java) |
|---|---|---|
| Data Mapper | `Session` + modelos declarativos | `EntityManager` + entidades JPA |
| Repository | Clase `Session` o repositorios custom | `Repository` interface de Spring Data |
| Unit of Work | `Session` / `Session.flush()` | `EntityManager` / `flush()` |
| Identity Map | `Session` interna (first-level cache) | First-level cache de Hibernate |
| Lazy Load | `lazy='select'` (default) | `FetchType.LAZY` (default) |
| Eager Load | `joinedload()`, `selectinload()` | `FetchType.EAGER`, `@EntityGraph` |

Entender los patrones de Fowler permite usar los ORMs con comprensión real en lugar de por convención ciega. Cuando un ORM genera queries inesperadas, cuando hay N+1 en producción, cuando una sesión lanza un error de "detached entity", el diagnóstico es inmediato si se entienden los patrones subyacentes.

---

## 12. Cuándo usar cada patrón

### 12.1 La guía de selección

La elección del patrón correcto depende de la complejidad del dominio y los requerimientos de testabilidad:

```
¿Cuán compleja es la lógica de dominio?

  BAJA (mayormente CRUD, poco comportamiento en los objetos)
  │
  ├── ¿Necesitas una capa de abstracción limpia?
  │     No → Table Data Gateway: el más simple
  │     Sí → Active Record: conveniente para CRUD con algo de lógica
  │
  MEDIA/ALTA (reglas de negocio complejas, muchos objetos interrelacionados)
  │
  ├── ¿Necesitas testear la lógica de negocio sin base de datos?
  │     No → Active Record puede funcionar, pero considera las limitaciones
  │     Sí → Data Mapper + Repository + Unit of Work + Identity Map
  │
  ¿El dominio cambia independientemente del esquema?
  (modelos de dominio ricos, DDD, múltiples fuentes de datos)
  │
  └── Data Mapper + Repository + Unit of Work (+ un ORM si el volumen lo justifica)
```

### 12.2 Tabla comparativa de los patrones

| Patrón | Complejidad de implementación | Testabilidad | Acoplamiento con DB | Adecuado para |
|---|---|---|---|---|
| **Table Data Gateway** | Muy baja | Baja (datos en bruto) | Alto | Scripts, CRUD simple, reportes |
| **Row Data Gateway** | Baja | Media | Alto | Transición, datos tipados simples |
| **Active Record** | Baja-Media | Media (requiere DB para tests) | Alto | Apps CRUD, frameworks web simples |
| **Data Mapper** | Alta | Alta (dominio testeable) | Bajo | Dominios ricos, DDD, microservicios |
| **Repository** | Media | Muy Alta (intercambiable) | Muy Bajo | DDD, apps con múltiples fuentes |
| **Unit of Work** | Alta | Alta | Bajo | Transacciones complejas, consistencia |
| **Identity Map** | Media | Alta | Bajo | Siempre junto a UoW y Repository |
| **Lazy Load** | Depende de la impl. | Media (N+1 hidden) | Bajo-Medio | Grafos de objetos grandes |

### 12.3 El costo real de la complejidad

Es importante no sobreingeniería. Para una aplicación web pequeña con 10 tablas y lógica de negocio mínima, implementar todos los patrones de Fowler manualmente es trabajo de semanas que no aporta valor real.

**Regla práctica:**
- Si usas un framework web moderno (Rails, Laravel, Django), su ORM ya implementa muchos de estos patrones. Úsalo.
- Si el dominio es complejo y tiene lógica de negocio rica, separa explícitamente el dominio de la persistencia con Data Mapper + Repository.
- Si el dominio es simple y los datos fluyen directamente al front-end, Table Data Gateway o Active Record son elecciones completamente válidas.
- Los patrones Unit of Work e Identity Map son más relevantes cuando manejas múltiples entidades en la misma transacción y quieres garantías de atomicidad y consistencia de instancias.

---

## 13. Los patrones en el contexto de sistemas distribuidos

### 13.1 Por qué los patrones asumen una sola base de datos

Todos los patrones de Fowler fueron diseñados asumiendo implícitamente que hay **una sola base de datos**. El Unit of Work asume que puede confirmar todos los cambios en una sola transacción ACID. El Identity Map asume que "el objeto con ID 42" tiene una única fuente de verdad. El Lazy Load asume que la relación que se está cargando está en la misma base de datos que el objeto padre.

Cuando los datos están distribuidos en múltiples servicios o múltiples bases de datos (como en los microservicios del módulo 13), estas suposiciones se rompen.

### 13.2 Las adaptaciones necesarias

**Unit of Work en un sistema distribuido:**
En lugar de una transacción local, el Unit of Work necesita coordinar con el patrón Saga (módulo 13) para garantizar consistencia eventual entre múltiples servicios. El "commit" ya no es atómico sino una secuencia de pasos con compensaciones.

**Identity Map en un sistema distribuido:**
El Identity Map vive en el ámbito de una sola request/sesión. En un sistema distribuido, si el usuario X es pedido al Servicio A y también al Servicio B, cada servicio tiene su propio Identity Map. No hay un mapa global.

**Repository en microservicios:**
La interfaz `UserRepository` puede tener una implementación que llame a un microservicio de usuarios a través de HTTP en lugar de ir a PostgreSQL. La abstracción del Repository es especialmente valiosa aquí: el código de negocio no sabe si los usuarios están en una base de datos local o en un servicio remoto.

```python
class HttpUserRepository(UserRepository):
    """
    Implementación del Repository que llama al microservicio de usuarios.
    El código de negocio no sabe que hay una red de por medio.
    """
    
    def __init__(self, base_url: str, http_client):
        self._base_url = base_url
        self._client = http_client
    
    def get(self, user_id: int) -> User | None:
        response = self._client.get(f"{self._base_url}/users/{user_id}")
        if response.status_code == 404:
            return None
        data = response.json()
        return User(data['id'], data['name'], data['email'], 
                    data['credit_limit'], data['active'])
    
    # ... resto de métodos ...
```

**Lazy Load en sistemas distribuidos:**
El Lazy Load es potencialmente peligroso en sistemas distribuidos porque el costo de "cargar" una relación puede ser una llamada de red a otro servicio (latencia alta, posibilidad de fallo). Los sistemas distribuidos modernos prefieren explícitamente el eager loading en los casos donde se sabe que se necesitarán los datos relacionados, o el diseño de aggregates que encapsulen todos los datos relacionados que se necesitan juntos.

---

## 14. Resumen y conexión con el resto del curso

### Los conceptos clave y cuándo aplicarlos

| Concepto | Cuándo usarlo |
|---|---|
| **Table Data Gateway** | CRUD simple, scripts, dashboards, lógica mínima |
| **Row Data Gateway** | Datos tipados sin lógica de negocio compleja |
| **Active Record** | Apps web CRUD, frameworks con AR integrado, equipos pequeños |
| **Data Mapper** | Dominios ricos, testabilidad alta, separación de preocupaciones |
| **Repository** | DDD, múltiples fuentes de datos, tests sin base de datos |
| **Unit of Work** | Transacciones que afectan múltiples entidades |
| **Identity Map** | Siempre junto a Unit of Work para evitar instancias duplicadas |
| **Lazy Load** | Grafos de objetos donde no siempre se necesitan las relaciones |
| **Eager Load** | Cuando se sabe de antemano que se necesitarán las relaciones |

### La meta-lección del módulo

Los patrones de Fowler no son recetas que se aplican mecánicamente. Son soluciones a problemas concretos, y aplicarlos sin tener el problema es sobreingeniería.

La pregunta correcta no es "¿qué patrón debo usar?" sino:

1. **¿Cuán compleja es la lógica de negocio?** — Si es simple, Active Record o Table Data Gateway son perfectamente válidos. Si es compleja, Data Mapper + Repository + Unit of Work pagan su costo.

2. **¿Necesito testear la lógica de negocio sin base de datos?** — Si sí, Data Mapper y Repository son necesarios. Si no, Active Record es suficiente.

3. **¿El esquema y el dominio van a divergir?** — Si el modelo de objetos puede necesitar representaciones distintas del esquema (por herencia, por agregados de dominio complejos, por múltiples fuentes de datos), Data Mapper es necesario.

4. **¿El equipo usa un ORM?** — Si sí, el ORM ya implementa muchos de estos patrones. Entender los patrones permite usarlo con comprensión, no con fe ciega.

La clave es entender qué garantía aporta cada patrón y qué complejidad introduce, y elegir el nivel mínimo de complejidad que resuelve el problema real.

### Conexión con módulos futuros

- **Módulo 15 (ORMs e impedancia objeto-relacional):** Los ORMs son implementaciones automáticas de los patrones de este módulo. Entender Data Mapper, Unit of Work, Identity Map y Lazy Load es el prerequisito para entender cómo funciona un ORM, por qué genera el SQL que genera, y por qué falla de la forma en que falla. El problema N+1 que el módulo 15 trata en profundidad es directamente consecuencia del Lazy Load mal aplicado.

- **Módulo 16 (El oficio):** Las decisiones de qué capa de acceso a datos usar son parte de las decisiones arquitectónicas que se toman al inicio de un proyecto. Evaluar un modelo ajeno incluye evaluar si la capa de acceso a datos está bien diseñada: ¿hay SQL mezclado con lógica de dominio? ¿Hay N+1 ocultos? ¿Las transacciones abarcan exactamente lo que deben? Estas preguntas se responden con el vocabulario de los patrones de Fowler.

---

## 15. Ejercicios de comprensión

**Ejercicio 1.** Dado el siguiente código de un servicio de gestión de biblioteca:

```python
class BookService:
    def checkout_book(self, book_id: int, member_id: int):
        db = Database.get_connection()
        cursor = db.cursor()
        
        cursor.execute("SELECT id, available, title FROM books WHERE id = %s", (book_id,))
        book = cursor.fetchone()
        if not book or not book[1]:
            raise ValueError("Libro no disponible")
        
        cursor.execute("SELECT id, active, loans_count FROM members WHERE id = %s", (member_id,))
        member = cursor.fetchone()
        if not member or not member[1]:
            raise ValueError("Miembro inactivo")
        
        if member[2] >= 5:
            raise ValueError("Límite de préstamos alcanzado")
        
        cursor.execute("UPDATE books SET available = FALSE WHERE id = %s", (book_id,))
        cursor.execute(
            "INSERT INTO loans (book_id, member_id, checkout_date) VALUES (%s, %s, NOW())",
            (book_id, member_id)
        )
        cursor.execute(
            "UPDATE members SET loans_count = loans_count + 1 WHERE id = %s", 
            (member_id,)
        )
        db.commit()
```

a) Identifica todos los problemas de diseño de este código con respecto a la separación de preocupaciones y los patrones de Fowler.

b) Refactoriza el código usando el patrón **Active Record**. Implementa las clases `Book` y `Member` como Active Records con la lógica de negocio apropiada.

c) Refactoriza el código usando **Data Mapper + Repository + Unit of Work**. Define las interfaces `BookRepository` y `MemberRepository`, y las clases de dominio puras `Book` y `Member`.

d) Escribe 3 tests unitarios para la lógica de negocio de la clase `Member` en la versión del inciso (c) que no requieran base de datos.

---

**Ejercicio 2.** Un sistema de gestión de proyectos tiene las siguientes entidades:
- `Project` (proyecto) con campos: id, name, status, budget.
- `Task` (tarea) con campos: id, project_id, title, status, assigned_to_user_id.
- `User` (usuario) con campos: id, name, email, role.
- `Comment` (comentario) con campos: id, task_id, user_id, text, created_at.

Las queries más comunes son:
1. Cargar un proyecto con todas sus tareas y el nombre del usuario asignado a cada tarea.
2. Cargar una tarea con sus comentarios, incluyendo el nombre del usuario de cada comentario.
3. Listar todos los proyectos activos con su número de tareas pendientes.

a) Para la query #1, ¿usarías Lazy Load o Eager Load? ¿Por qué? Escribe el SQL que generaría un Eager Load bien diseñado.

b) Para la query #2, identifica el riesgo de N+1 si se usara Lazy Load para los usuarios de los comentarios. ¿Cuántas queries se generarían para una tarea con 20 comentarios? Propón la solución.

c) Diseña la interfaz `ProjectRepository` con los métodos necesarios para soportar las tres queries. Para la query #3, ¿cómo implementarías el conteo de tareas pendientes de forma eficiente?

d) Diseña el Identity Map para este sistema. ¿Qué tipo de entidades deben estar en el Identity Map? ¿Deben estar los `Comment`?

---

**Ejercicio 3.** Un servicio de pagos necesita procesar una transferencia entre dos cuentas. La operación debe:
1. Verificar que la cuenta origen tiene fondos suficientes.
2. Crear un registro de transferencia con estado "pending".
3. Debitar la cuenta origen.
4. Acreditar la cuenta destino.
5. Actualizar el estado de la transferencia a "completed".

Si cualquier paso falla, todos los cambios deben revertirse.

a) Implementa el Unit of Work para este escenario. Incluye el manejo de transacciones y el rollback automático.

b) ¿Qué garantías proporciona el Unit of Work aquí que no tendría una secuencia de `save()` individuales?

c) Identifica un escenario donde el Identity Map sea crítico para la correctitud de este proceso. (Pista: considera qué pasa si la cuenta origen y la cuenta destino son accedidas desde distintos lugares del código.)

d) Si el sistema de pagos decide separar las cuentas en un microservicio distinto del servicio de transferencias, ¿cómo cambiaría el diseño? ¿Puede el Unit of Work local seguir garantizando atomicidad?

---

**Ejercicio 4.** Dado el siguiente esquema de una red social simplificada:

```sql
CREATE TABLE users (id BIGINT PK, name TEXT, bio TEXT);
CREATE TABLE posts (id BIGINT PK, user_id BIGINT FK, content TEXT, created_at TIMESTAMPTZ);
CREATE TABLE followers (follower_id BIGINT FK, followed_id BIGINT FK, PRIMARY KEY(follower_id, followed_id));
CREATE TABLE likes (post_id BIGINT FK, user_id BIGINT FK, PRIMARY KEY(post_id, user_id));
```

a) Diseña el modelo de objetos de dominio (clases `User`, `Post`, `Like`) sin ninguna referencia a SQL ni a la base de datos.

b) En la vista de perfil de un usuario, se muestra: nombre, bio, número de seguidores, los últimos 10 posts con el número de likes de cada uno. Diseña el método del Repository que cargue todos estos datos eficientemente. ¿Cuántas queries SQL ejecuta?

c) En el feed de un usuario, se muestran los posts de todos los usuarios que sigue, ordenados por fecha, con paginación de 20 en 20. ¿Cómo implementarías el método del Repository para esta query? Escribe el SQL.

d) Los `likes` son un dato muy leído y poco escrito. ¿Valdría la pena implementar un caché en el Repository de `Like`? Diseña una implementación `CachedLikeRepository` que use Redis como caché y PostgreSQL como fuente de verdad.

---

**Ejercicio 5.** Considera los cuatro patrones principales: Table Data Gateway, Row Data Gateway, Active Record, y Data Mapper. Para cada uno de los siguientes sistemas, indica cuál usarías y justifica:

a) Un panel de administración interno que permite a los administradores de una empresa ver, editar y eliminar registros de usuarios, órdenes y productos. Tiene 15 tablas. La lógica de negocio es mínima (principalmente validaciones básicas).

b) Un sistema bancario que gestiona cuentas, transferencias, préstamos, y pagos. Tiene reglas de negocio complejas, regulaciones de cumplimiento, y múltiples invariantes que deben mantenerse (el saldo nunca puede ser negativo, las transferencias deben cumplir límites diarios, etc.).

c) Un servicio de importación de datos que lee un CSV con millones de filas y las inserta en la base de datos después de validaciones básicas.

d) Un sistema de reservas de vuelos con múltiples reglas de negocio: políticas de cancelación, cargos por cambio, asignación de asientos, gestión de listas de espera. Necesita ser testeable porque las reglas cambian frecuentemente.

---

*Próximo módulo: ORMs, impedancia objeto-relacional y SQL a mano — donde exploraremos cómo los ORMs implementan los patrones de Fowler, cuándo generan SQL eficiente y cuándo lo destruyen, el problema N+1 en profundidad, y cómo decidir cuándo usar un ORM y cuándo escribir SQL directamente.*
