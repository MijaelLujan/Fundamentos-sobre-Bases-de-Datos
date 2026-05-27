# Módulo 13 — Modelado para sistemas distribuidos

> *"Un sistema de base de datos en una sola máquina tiene un contrato simple: o la transacción se confirma y todos ven el resultado, o falla y es como si nunca hubiera ocurrido. En un sistema distribuido ese contrato se rompe en pedazos. La red falla. Los relojes no están sincronizados. Los nodos se caen y vuelven a subir. Y mientras tanto, la aplicación tiene que seguir funcionando. Entender los sistemas distribuidos no es opcional para quien diseña esquemas modernos: es la diferencia entre un modelo que escala y uno que se convierte en el cuello de botella de toda la organización."*

---

## Tabla de contenidos

1. [Por qué los sistemas distribuidos cambian todo](#1-por-qué-los-sistemas-distribuidos-cambian-todo)
2. [El teorema CAP: lo que realmente dice y lo que no dice](#2-el-teorema-cap-lo-que-realmente-dice-y-lo-que-no-dice)
3. [El teorema PACELC: una visión más completa](#3-el-teorema-pacelc-una-visión-más-completa)
4. [El espectro de consistencia](#4-el-espectro-de-consistencia)
5. [Replicación: el mecanismo base de la distribución](#5-replicación-el-mecanismo-base-de-la-distribución)
6. [El problema del split-brain](#6-el-problema-del-split-brain)
7. [CRDTs: estructuras de datos que se fusionan sin conflicto](#7-crdts-estructuras-de-datos-que-se-fusionan-sin-conflicto)
8. [Sharding: particionamiento horizontal entre nodos](#8-sharding-particionamiento-horizontal-entre-nodos)
9. [Transacciones distribuidas: el problema más difícil](#9-transacciones-distribuidas-el-problema-más-difícil)
10. [El patrón Saga para transacciones distribuidas](#10-el-patrón-saga-para-transacciones-distribuidas)
11. [Joins distribuidos: el costo más subestimado](#11-joins-distribuidos-el-costo-más-subestimado)
12. [Modelado para evitar joins distribuidos](#12-modelado-para-evitar-joins-distribuidos)
13. [Patrones de modelado específicos para sistemas distribuidos](#13-patrones-de-modelado-específicos-para-sistemas-distribuidos)
14. [Resumen y conexión con el resto del curso](#14-resumen-y-conexión-con-el-resto-del-curso)
15. [Ejercicios de comprensión](#15-ejercicios-de-comprensión)

---

## 1. Por qué los sistemas distribuidos cambian todo

### 1.1 El modelo mental de una sola máquina

Todo lo que hemos estudiado hasta ahora en este curso asumía, explícita o implícitamente, que los datos viven en una **única máquina**. Esta suposición es tan fundamental que casi nunca se enuncia, porque en ese contexto:

- **El tiempo es lineal:** todos los eventos tienen un orden absoluto. Primero ocurrió A, luego B, luego C. No hay ambigüedad.
- **La memoria es coherente:** si un proceso escribe un valor y otro proceso lo lee inmediatamente después, leerá el valor escrito. No hay demora.
- **Las operaciones son atómicas o no son:** el disco puede fallar, pero el sistema de archivos garantiza que un `fsync` completo llega a disco o no llega. No hay estado intermedio observable.
- **El "ahora" es único:** todos los componentes del sistema comparten el mismo reloj. El tiempo de la transacción es unívoco.

Bajo estas suposiciones, las propiedades ACID del módulo 8 son implementables de forma eficiente. El módulo 7 mostró cómo el query planner puede razonar sobre toda la base de datos como un grafo de operadores. El módulo 9 mostró cómo los índices permiten acceso eficiente a cualquier parte del dato.

### 1.2 Las tres razones por las que se distribuye

Cuando se dice que un sistema es "distribuido", se significa que los datos y/o el cómputo viven en **múltiples máquinas conectadas por una red**. Hay exactamente tres razones para hacer esto:

**1. Escalabilidad:** Una sola máquina tiene límites físicos. Una base de datos con 100 TB de datos no cabe en un solo disco. Un sistema que necesita atender 1,000,000 de requests por segundo no puede ser procesado por un solo CPU. La distribución permite escalar más allá de los límites de una máquina.

**2. Alta disponibilidad:** Si los datos existen en una sola máquina y esa máquina falla, el sistema se cae completamente. Distribuir los datos en múltiples máquinas (replicación) permite que el sistema continúe operando cuando un nodo falla.

**3. Proximidad geográfica:** Un usuario en Tokio que accede a datos almacenados en un servidor en Nueva York experimenta ~150ms de latencia de red. Con réplicas en Asia, esa latencia cae a ~5ms. Para ciertas aplicaciones (juegos en tiempo real, sistemas financieros de alta frecuencia) esto es crítico.

El problema es que ninguna de estas tres razones es gratuita. Cada una introduce una complejidad que no existe en sistemas de una sola máquina. Y esa complejidad afecta directamente cómo se debe diseñar el modelo de datos.

### 1.3 Las falacias de la computación distribuida

En 1994, Peter Deutsch de Sun Microsystems enumeró lo que llamó las **Falacias de la computación distribuida**: suposiciones incorrectas que los desarrolladores novatos (y a veces experimentados) hacen sobre las redes:

1. La red es confiable.
2. La latencia es cero.
3. El ancho de banda es infinito.
4. La red es segura.
5. La topología no cambia.
6. Hay un solo administrador.
7. El costo de transporte es cero.
8. La red es homogénea.

Cada una de estas falacias, cuando se asume incorrectamente, lleva a diseños de sistemas que fallan de formas impredecibles en producción. La más importante para el diseño de datos es la primera: **la red no es confiable**. Los paquetes se pierden. Las conexiones se interrumpen. Los nodos se tornan inalcanzables temporalmente. Esto ocurre en cualquier red real, incluidas las redes de alta calidad dentro de un mismo datacenter.

### 1.4 La consecuencia fundamental: el tiempo no es lineal en sistemas distribuidos

En una sola máquina, la línea de tiempo de los eventos es una línea recta. En un sistema distribuido, **el tiempo se vuelve multidimensional**. Dos eventos en nodos distintos no tienen un orden absoluto natural, a menos que exista comunicación entre ellos que establezca ese orden.

Considera dos nodos A y B, ambos con sus propios relojes:

```
Nodo A: ─────── escribe X=1 (09:00:00.001) ─────────────────────
         (red con 50ms de latencia)
Nodo B: ─────────────────────────────── lee X (09:00:00.010) ────
```

El nodo B lee X a las 09:00:00.010, 9ms después de que el nodo A escribió X=1 a las 09:00:00.001. ¿Lee el valor X=1?

No necesariamente. La replicación del valor desde A hacia B tarda al menos 50ms (la latencia de la red). Cuando B lee a los 9ms, A no ha tenido tiempo de replicar el valor. B podría leer X=0 (el valor anterior), incluso aunque "cronológicamente" (según los relojes de ambas máquinas) B lee después de que A escribió.

Este fenómeno tiene un nombre: **consistencia eventual**. Y es una de las ideas más importantes y contraintuitivas de los sistemas distribuidos.

---

## 2. El teorema CAP: lo que realmente dice y lo que no dice

### 2.1 Los tres conceptos

En el año 2000, Eric Brewer presentó la conjetura CAP, que en 2002 fue demostrada formalmente por Gilbert y Lynch. El teorema dice:

> **Es imposible para un sistema de datos distribuido garantizar simultáneamente las tres siguientes propiedades: Consistencia, Disponibilidad y Tolerancia a Particiones.**

Las tres propiedades son:

**Consistency (Consistencia — C):** Cualquier lectura recibe el resultado de la escritura más reciente o un error. Todos los nodos del sistema ven los mismos datos al mismo tiempo. Si escribo X=1 en el nodo A, cualquier lectura posterior de X en cualquier nodo debe devolver 1. Esta es **consistencia linealizable** o **linearizability**, no la "C" de ACID (que es algo diferente).

> ⚠️ **Trampa terminológica crítica:** La "C" de ACID (módulo 8) significa que una transacción lleva la base de datos de un estado consistente a otro estado consistente, preservando las invariantes del modelo. La "C" de CAP significa que todos los nodos ven el mismo valor para cualquier dato en cualquier momento. Son propiedades completamente distintas que comparten la misma letra por coincidencia histórica. Un sistema puede tener la "C" de ACID sin tener la "C" de CAP, y viceversa.

**Availability (Disponibilidad — A):** Cada request recibe una respuesta (no un error). El sistema siempre responde, aunque no garantice que la respuesta sea el valor más reciente. Un sistema disponible nunca dice "no puedo responder ahora"; siempre devuelve algo.

**Partition Tolerance (Tolerancia a Particiones — P):** El sistema continúa operando incluso cuando los mensajes entre nodos se pierden o se retrasan arbitrariamente. Una **partición de red** es cuando una falla de red divide los nodos del sistema en grupos que no pueden comunicarse entre sí.

### 2.2 Por qué P no es opcional

La comprensión más común del teorema CAP es: "elige dos de las tres". Pero esta formulación es engañosa, porque **la tolerancia a particiones (P) no es opcional** en ningún sistema distribuido real.

Las particiones de red **ocurren**. No se pueden prevenir. Se pueden reducir su frecuencia con infraestructura de alta calidad, pero nunca se pueden eliminar completamente. Incluso dentro de un datacenter, la red puede fallar. Incluso con hardware de grado militar, un cable puede cortarse, un switch puede reiniciarse, una falla de firmware puede aislar un nodo.

Si diseñas un sistema que no tolera particiones, lo que estás diciendo es: "cuando la red falle, mi sistema fallará completamente". Eso es una decisión válida solo si el sistema opera en una única máquina, lo que no es distribución.

La consecuencia real del teorema CAP es: **cuando ocurre una partición de red (que ocurrirá), tienes que elegir entre Consistencia o Disponibilidad**. No ambas.

```
                    ┌─────────────────────────────────────────┐
                    │      Ocurre una partición de red        │
                    └──────────────────┬──────────────────────┘
                                       │
                    ┌──────────────────┴──────────────────────┐
                    │                                         │
         ┌──────────▼──────────┐               ┌─────────────▼──────────┐
         │  Elige CONSISTENCIA │               │  Elige DISPONIBILIDAD  │
         │   (CP system)       │               │     (AP system)        │
         └──────────┬──────────┘               └─────────────┬──────────┘
                    │                                         │
  "Rechazo requests hasta        "Acepto todos los requests,
   que la partición se           pero algunos nodos pueden
   resuelva. El sistema          devolver datos desactualizados
   puede no responder."          o conflictivos entre sí."
```

### 2.3 Sistemas CP y AP: ejemplos reales

**Sistemas CP (Consistencia + Tolerancia a Particiones):**
- **HBase, Zookeeper, Etcd, Consul:** Priorizan consistencia. Durante una partición, el nodo que no tiene quórum rechaza requests de escritura. El sistema prefiere ser inaccesible a ser inconsistente.
- **PostgreSQL en configuración synchronous_commit=on con replicación síncrona:** Una escritura no se confirma hasta que todos los nodos síncronos la confirmen. Si un nodo falla, la escritura bloquea.

**Sistemas AP (Disponibilidad + Tolerancia a Particiones):**
- **DynamoDB (con consistencia eventual), Cassandra, CouchDB:** Priorizan disponibilidad. Durante una partición, cada nodo acepta escrituras de forma independiente. Cuando la partición se resuelve, los conflictos se reconcilian (con resolución de conflictos configurable).
- **DNS:** El sistema de nombres de internet es un ejemplo clásico. Los cambios de DNS pueden tardar horas en propagarse por todo el sistema, pero el sistema siempre responde aunque con datos desactualizados.

### 2.4 Lo que CAP no captura

El teorema CAP es valioso pero tiene limitaciones importantes:

1. **Solo aplica durante particiones:** Si no hay partición de red, no hay que sacrificar ni consistencia ni disponibilidad. La elección CP vs AP solo importa cuando algo sale mal.

2. **Consistency y Availability son espectros, no binarios:** No existe "consistencia total" ni "disponibilidad total". Hay grados de consistencia (linealizable, secuencial, causal, eventual) y grados de disponibilidad (sin downtime, con degradación graceful, con latencia aceptable).

3. **No habla de latencia:** Un sistema CP puede ser consistente pero extremadamente lento durante operación normal si exige coordinación entre muchos nodos para cada escritura.

Estas limitaciones llevaron a un refinamiento del modelo: el teorema PACELC.

---

## 3. El teorema PACELC: una visión más completa

### 3.1 La extensión de Daniel Abadi

En 2012, Daniel Abadi propuso PACELC como extensión de CAP. El nombre es un acrónimo que se lee así:

- **PA/ELC:** en caso de **P**artición (Partition), elige entre **A**vailability y **C**onsistency. Else (en ausencia de partición), elige entre **L**atency y **C**onsistency.

La segunda parte (**ELC**) es el aporte clave: captura un trade-off que CAP ignora completamente. Incluso cuando no hay partición, un sistema distribuido enfrenta una elección entre:

- **Latencia baja:** responder inmediatamente desde el nodo local, aunque la respuesta no refleje las últimas escrituras de otros nodos.
- **Consistencia fuerte:** coordinar con otros nodos antes de responder, garantizando que la respuesta es la más reciente, a costa de mayor latencia.

```
┌────────────────────────────────────────────────────────────────────┐
│                          PACELC                                    │
├─────────────────────────┬──────────────────────────────────────────┤
│  Durante Partición (P)  │       Sin Partición (E)                 │
│  Elige A o C            │       Elige L o C                       │
├─────────────────────────┼──────────────────────────────────────────┤
│  Cassandra: PA/EL       │  Alta disponibilidad, baja latencia      │
│  (prioriza A y L)       │  Eventual consistency siempre            │
├─────────────────────────┼──────────────────────────────────────────┤
│  CRDB/Spanner: PC/EC    │  Siempre consistente, mayor latencia     │
│  (prioriza C)           │  Paga coordinación siempre               │
├─────────────────────────┼──────────────────────────────────────────┤
│  DynamoDB: PA/EL        │  Lectura local por defecto               │
│  (con configuración     │  Eventual consistency default            │
│  eventual)              │  Strongly consistent es opt-in (+$$$)   │
├─────────────────────────┼──────────────────────────────────────────┤
│  MongoDB con WriteConcern│ Configurable: puede ser PC/EC o PA/EL  │
│  configurable           │  según la configuración elegida          │
└─────────────────────────┴──────────────────────────────────────────┘
```

### 3.2 Implicaciones para el diseño del esquema

PACELC revela que la elección no es solo "¿qué pasa cuando falla la red?" sino "¿cuánta latencia está dispuesto a pagar el sistema por garantías de consistencia en operación normal?".

Esta pregunta tiene una respuesta diferente para distintas partes del mismo sistema:

- **Inventario de un e-commerce:** ¿Cuánto importa leer el stock exacto al milisegundo? Si hay 1000 unidades de un producto, no es crítico que todos los nodos sepan exactamente cuántas hay en cada instante. Disponibilidad y baja latencia son más importantes. Un sistema PA/EL es apropiado.

- **Saldo de una cuenta bancaria:** ¿Cuánto importa que una operación de débito vea el saldo exactamente correcto? Mucho. Un saldo incorrecto puede resultar en sobregiro. Un sistema PC/EC es necesario, aunque sea más lento.

La conclusión práctica: **el modelo de consistencia que elige para un sistema debería ser diferente para distintos tipos de datos, según la criticidad de cada uno**.

---

## 4. El espectro de consistencia

### 4.1 Más allá del binario "consistente / eventual"

La frase "consistencia eventual" se usa tan frecuentemente que parece ser un estado binario: o el sistema es eventualmente consistente, o es fuertemente consistente. En realidad, hay un espectro continuo de modelos de consistencia, ordenados de más fuerte a más débil:

```
MÁS FUERTE (más garantías, más costoso)
     │
     ▼
Linearizability (Consistencia linealizable)
     │
Sequential Consistency (Consistencia secuencial)
     │
Causal Consistency (Consistencia causal)
     │
Read-Your-Writes Consistency
     │
Monotonic Read Consistency
     │
Eventual Consistency (Consistencia eventual)
     ▼
MÁS DÉBIL (menos garantías, menos costoso)
```

### 4.2 Linearizability (Consistencia linealizable)

**Qué garantiza:** Las operaciones aparecen como si ocurrieran en un único punto instantáneo en el tiempo, y ese punto es consistente con el orden real en que ocurrieron. Si una operación A termina antes de que empiece la operación B, entonces A aparece antes que B en cualquier observador.

**Analogía:** Es como si hubiera un único hilo de ejecución compartido por todos los nodos. Cada operación ocupa exactamente un punto en ese hilo.

**Cuándo es necesaria:** Cuando la correctitud del sistema depende de que todos los observadores vean el mismo estado en el mismo momento. Ejemplo: un sistema de elección de líder (solo un nodo puede ser el líder en cada momento). Otro ejemplo: un contador atómico que debe ser incrementado exactamente una vez por evento.

**Costo:** Requiere coordinación entre nodos para cada operación. En la práctica, esto significa consenso distribuido (Paxos, Raft), lo cual tiene latencia de al menos una ronda de comunicación entre nodos.

### 4.3 Consistencia causal

**Qué garantiza:** Si la operación B depende causalmente de la operación A (es decir, A ocurrió antes que B y B "sabe" sobre A), entonces cualquier nodo que ve B también debe ver A primero. Las operaciones sin relación causal pueden verse en distinto orden en distintos nodos.

**Ejemplo concreto:**
```
Alice escribe: "¿Les gusta el nuevo diseño?" (Post A)
Bob escribe en respuesta a A: "¡Me encanta!" (Post B, causalmente dependiente de A)

Con consistencia causal garantizada:
- Cualquier usuario que ve "¡Me encanta!" también verá "¿Les gusta el nuevo diseño?" primero.
- Nunca verá la respuesta sin el post original.

Sin consistencia causal:
- Un usuario podría ver "¡Me encanta!" sin ver el post al que responde.
```

**Cuándo es suficiente:** Para la mayoría de las aplicaciones sociales, de mensajería, y de colaboración. Es más débil que linearizability pero captura las relaciones que a los usuarios les importan.

**Implementación:** Se implementa con **vectores de reloj** (vector clocks) o **relojes de versión** (version vectors). Cada escritura lleva un vector que encode su historia causal.

### 4.4 Consistencia eventual

**Qué garantiza:** Si no hay nuevas escrituras, eventualmente todos los nodos convergerán al mismo valor. "Eventualmente" puede ser milisegundos o horas, dependiendo del sistema.

**Lo que NO garantiza:**
- No garantiza cuándo convergerá.
- No garantiza que todos los nodos vean las escrituras en el mismo orden.
- No garantiza que una lectura vea la última escritura, incluso si esa escritura ocurrió hace mucho tiempo.

**El problema de la anomalía:** Con consistencia eventual, una aplicación puede mostrar estados que son inconsistentes desde la perspectiva del usuario. El ejemplo clásico es el carrito de compras de Amazon (el famoso paper de Werner Vogels): un usuario añade un artículo, lo elimina, y luego el artículo reaparece porque la réplica que procesó la eliminación aún no recibió la adición, y cuando la recibe, los resuelve en el orden incorrecto.

**Cuándo es aceptable:** Cuando las anomalías son tolerables para el negocio y la disponibilidad es más crítica que la precisión. Ejemplos: número de "likes" en una publicación (no importa si hay ±5 durante unos segundos), número de visualizaciones de un video, stock no crítico.

### 4.5 Read-Your-Writes y Monotonic Reads

Dos garantías que son más débiles que linearizability pero más fuertes que consistency eventual pura, y que son muy relevantes para la experiencia de usuario:

**Read-Your-Writes:** Si escribo un valor, mis lecturas posteriores siempre verán ese valor (o uno más reciente). No necesariamente lo ven otros usuarios inmediatamente, pero yo sí.

**Por qué importa:** Sin esta garantía, puedo actualizar mi perfil y al refrescar la página ver el perfil antiguo. Esto es confuso y parece un bug aunque técnicamente no lo sea.

**Monotonic Reads:** Si en un momento T leo el valor V1, en cualquier momento posterior T' ≥ T leeré V1 o un valor más reciente, nunca uno más antiguo.

**Por qué importa:** Sin esta garantía, si leo los mensajes de mi bandeja de entrada y veo 50 mensajes, podría refrescar y ver 48 mensajes (porque el segundo request fue a una réplica más atrasada). Los mensajes "desaparecen" temporalmente. Esto destruye la confianza del usuario.

**Implementación práctica:** Estas dos garantías suelen implementarse con **sticky sessions** (el mismo usuario siempre va al mismo nodo) o **session tokens** (el cliente envía su último timestamp visto, y el servidor garantiza responder con datos al menos tan recientes).

---

## 5. Replicación: el mecanismo base de la distribución

### 5.1 Qué es la replicación y por qué existe

La **replicación** es el proceso de mantener copias de los mismos datos en múltiples nodos. Es la técnica fundamental que habilita tanto la alta disponibilidad como la escalabilidad de lecturas.

Existen dos arquitecturas fundamentales:

**Replicación primary-replica (también llamada master-slave o leader-follower):**
- Un nodo es el **primary** (primario): acepta todas las escrituras.
- Uno o más nodos son **réplicas** (replicas): reciben copias de los datos desde el primary y pueden atender lecturas.
- Si el primary falla, una réplica puede ser promovida a primary (failover).

```
                        ┌──────────────┐
     Escrituras ───────▶│   PRIMARY    │
     Lecturas   ───────▶│  (Nodo 1)   │
                        └──────┬───────┘
                               │ replica stream
                 ┌─────────────┼─────────────┐
                 ▼             ▼             ▼
           ┌─────────┐   ┌─────────┐   ┌─────────┐
           │Réplica 1│   │Réplica 2│   │Réplica 3│
           │(Nodo 2) │   │(Nodo 3) │   │(Nodo 4) │
           └─────────┘   └─────────┘   └─────────┘
            ▲                               ▲
            │ Lecturas                      │ Lecturas
```

**Replicación multi-primary (también llamada multi-master):**
- Múltiples nodos aceptan escrituras simultáneamente.
- Los cambios se propagan entre todos los nodos.
- Los conflictos (cuando dos nodos modifican el mismo dato) deben resolverse.

La replicación primary-replica es más simple y más común. La replicación multi-primary ofrece mayor disponibilidad para escrituras pero introduce complejidad en la resolución de conflictos.

### 5.2 Replicación síncrona vs asíncrona

Esta es la decisión más importante de la replicación, y es directamente el trade-off del teorema CAP en acción:

**Replicación síncrona:**

```
Cliente  ──── escribe ────▶  PRIMARY
                              │
                              ├──── replica ────▶  Réplica A (espera ACK)
                              │
                              └──── replica ────▶  Réplica B (espera ACK)
                              │
                    (solo cuando TODAS las réplicas confirman)
                              │
Cliente  ◀──── ACK ──────────┘
```

- El primary solo confirma la escritura al cliente cuando **todas** las réplicas designadas han confirmado que han recibido y aplicado el cambio.
- **Garantía:** Si el primary falla inmediatamente después de confirmar la escritura, cualquier réplica que se promueva a primary tendrá los datos actualizados. **Zero data loss**.
- **Costo:** La latencia de escritura es al menos tan larga como el tiempo del viaje de ida y vuelta hacia la réplica más lenta. Si una réplica es lenta, todas las escrituras se vuelven lentas.

**Replicación asíncrona:**

```
Cliente  ──── escribe ────▶  PRIMARY
                              │
                   (confirma inmediatamente al cliente)
                              │
Cliente  ◀──── ACK ───────────┘
                              │
                              ├──── replica (eventualmente) ────▶  Réplica A
                              │
                              └──── replica (eventualmente) ────▶  Réplica B
```

- El primary confirma la escritura al cliente inmediatamente, sin esperar las réplicas.
- Las réplicas reciben los cambios de forma asíncrona, con un retraso llamado **replication lag**.
- **Garantía:** Baja latencia de escritura. Alta disponibilidad (el primary no depende de las réplicas para responder).
- **Costo:** Si el primary falla antes de que la réplica reciba el cambio, esos datos se pierden. **Posible data loss**.

**Replicación semi-síncrona (PostgreSQL: synchronous_commit):**
PostgreSQL ofrece una solución intermedia. Con `synchronous_commit = remote_write`, el primary espera que al menos una réplica confirme haber recibido los datos (no necesariamente aplicarlos). Con `synchronous_commit = remote_apply`, espera que al menos una réplica los aplique. Esto balancea la latencia con el riesgo de pérdida de datos.

### 5.3 El replication lag y sus consecuencias en el modelo

El **replication lag** es el retraso entre el momento en que el primary aplica una escritura y el momento en que las réplicas la reflejan. En sistemas bien configurados puede ser milisegundos; bajo carga pesada o con redes lentas puede ser segundos o incluso minutos.

El replication lag tiene consecuencias directas en el diseño de la aplicación y el esquema:

**Lectura desde réplica después de escritura (stale read):**
```
t1: Usuario actualiza su email (escritura va al primary)
t2: La aplicación redirige al usuario a su perfil (lectura viene de una réplica)
t3: La réplica tiene lag de 500ms → el perfil muestra el email antiguo
t4: El usuario piensa que la actualización falló y la vuelve a intentar
```

**Implicación de diseño:** Si usas réplicas para lectura, necesitas decidir qué lecturas pueden tolerar datos ligeramente desactualizados y qué lecturas deben ir siempre al primary. Esto es una decisión de **routing** que afecta el diseño de tu capa de acceso a datos.

**Patrones comunes:**
- Lecturas críticas (saldo bancario, stock de inventario) → siempre al primary.
- Lecturas no críticas (lista de productos, posts de blog) → puede ir a réplicas.
- Lectura inmediatamente después de escritura → al primary, o usar session-aware routing.

---

## 6. El problema del split-brain

### 6.1 Qué es el split-brain

El **split-brain** es uno de los escenarios más peligrosos en sistemas distribuidos. Ocurre cuando una partición de red divide los nodos del sistema en dos grupos (o más) que no pueden comunicarse entre sí, y **cada grupo cree que el otro grupo está caído** y se promueve a sí mismo como el "primario válido".

```
ANTES DE LA PARTICIÓN:

        ┌─────────┐       ┌─────────┐       ┌─────────┐
        │  Nodo A │───────│  Nodo B │───────│  Nodo C │
        │(Primary)│       │(Réplica)│       │(Réplica)│
        └─────────┘       └─────────┘       └─────────┘


DURANTE LA PARTICIÓN (la red entre A y {B,C} falla):

        ┌─────────┐   ╳╳╳╳╳╳╳╳╳╳╳╳╳   ┌─────────┐       ┌─────────┐
        │  Nodo A │   ╳ RED PARTIDA ╳   │  Nodo B │───────│  Nodo C │
        │(Primary)│   ╳╳╳╳╳╳╳╳╳╳╳╳╳   │(Réplica)│       │(Réplica)│
        └─────────┘                     └─────────┘       └─────────┘

Nodo A cree que B y C están caídos → sigue siendo primary.
Nodos B y C creen que A está caído → B se promueve a primary (failover automático).
```

Ahora hay **dos nodos que creen ser el primary** y ambos aceptan escrituras. Los datos en A y en {B, C} divergen. Cuando la partición se resuelve, el sistema tiene dos historias incompatibles de los mismos datos.

### 6.2 Por qué es tan destructivo

Considera una base de datos de cuentas bancarias con split-brain:
- En el nodo A (primary A): la cuenta X tiene $1,000 y se procesan 3 retiros de $300 cada uno → saldo $100.
- En el nodo B (primary B): la cuenta X tiene $1,000 y se procesan 2 retiros de $400 cada uno → saldo $200.

Cuando la partición se resuelve, ¿cuál es el saldo correcto? Ninguno de los dos es correcto desde la perspectiva del negocio. Se procesaron transacciones inválidas en ambos lados.

### 6.3 El quórum: la solución matemática

La solución estándar al split-brain es el **quórum**: un nodo solo puede actuar como primary (o ejecutar escrituras) si cuenta con el respaldo de **mayoría + 1** de los nodos del sistema.

Si hay 5 nodos (N=5), el quórum es 3 (mayoría de 5). En una partición:
- El grupo con 3 o más nodos puede alcanzar quórum → sigue operando.
- El grupo con 2 o menos nodos no puede alcanzar quórum → se vuelve de solo lectura o se detiene.

```
Partición con N=5:
- Grupo A: {Nodo 1, Nodo 2} → 2 nodos < quórum (3) → se detiene o solo lectura
- Grupo B: {Nodo 3, Nodo 4, Nodo 5} → 3 nodos = quórum → continúa operando
```

Esto garantiza que nunca haya dos grupos que operen como primario simultáneamente, porque es matemáticamente imposible que dos grupos distintos alcancen quórum si el total de nodos es N y el quórum es ⌈(N+1)/2⌉.

El quórum es la base de algoritmos de consenso como **Paxos** y **Raft**, que se usan en sistemas como etcd (Kubernetes), ZooKeeper, CockroachDB, y los sistemas de replicación modernos de PostgreSQL.

### 6.4 Cómo el diseño del modelo mitiga el split-brain

Más allá de la infraestructura, el **diseño del modelo de datos** puede mitigar los daños del split-brain:

**1. Operaciones idempotentes:**
Una operación es **idempotente** si ejecutarla múltiples veces produce el mismo resultado que ejecutarla una vez. En el contexto del split-brain, si el mismo request puede llegar a múltiples nodos, la idempotencia garantiza que no importa cuántas veces se procese.

```sql
-- NO idempotente: aplicar dos veces duplica el efecto
UPDATE cuentas SET saldo = saldo + 100 WHERE id = 42;

-- Idempotente: aplicar dos veces produce el mismo resultado
UPDATE cuentas SET saldo = 1100 WHERE id = 42 AND saldo = 1000;
-- (Con optimistic locking basado en versión)

-- Patrón idempotente con identificador único de operación:
INSERT INTO transferencias (operacion_id, cuenta_id, monto)
VALUES ('op-abc-123', 42, 100)
ON CONFLICT (operacion_id) DO NOTHING;
```

**2. Columna de versión para detectar conflictos:**
Añadir una columna `version` o `updated_at` a las entidades críticas permite detectar cuando dos nodos intentaron modificar la misma fila simultáneamente.

```sql
CREATE TABLE productos (
    id          BIGSERIAL PRIMARY KEY,
    nombre      TEXT NOT NULL,
    precio      NUMERIC(10,2) NOT NULL,
    version     BIGINT NOT NULL DEFAULT 1  -- incrementa en cada UPDATE
);

-- UPDATE con optimistic locking:
UPDATE productos
SET precio = 29.99, version = version + 1
WHERE id = 101 AND version = 5;  -- solo aplica si nadie más modificó desde version=5

-- Si retorna 0 filas → alguien más modificó → conflicto detectado
```

**3. Diseño append-only:**
Los modelos de **event sourcing** (módulo 6) son naturalmente más resilientes al split-brain porque solo añaden registros, nunca los modifican. Un conflicto entre dos nodos se manifiesta como dos events en el log, y la lógica de negocio puede decidir cuál prevalece o cómo combinarlos.

---

## 7. CRDTs: estructuras de datos que se fusionan sin conflicto

### 7.1 El problema que los CRDTs resuelven

La replicación eventual enfrenta un problema fundamental: cuando dos réplicas reciben escrituras distintas para el mismo dato, y luego necesitan sincronizarse, ¿cómo deciden cuál escritura prevalece?

Las estrategias naïve son insatisfactorias:
- **Last-Write-Wins (LWW):** gana el timestamp más reciente. Problema: los relojes distribuidos no están sincronizados. Una máquina ligeramente adelantada siempre gana.
- **Pregunta al usuario:** interrumpe la experiencia y no escala.
- **Uno siempre gana:** simplemente incorrecto para la mayoría de los casos de uso.

Los **CRDTs (Conflict-free Replicated Data Types)** son estructuras de datos diseñadas matemáticamente para que **siempre exista una operación de fusión que produzca un resultado determinístico y correcto**, independientemente del orden en que se reciban las actualizaciones de distintas réplicas.

El nombre completo es "Conflict-free Replicated Data Types" o "Convergent Replicated Data Types". Ambos nombres capturan la propiedad clave: **los conflictos no existen** porque la estructura de datos está diseñada para que cualquier combinación de actualizaciones pueda fusionarse sin ambigüedad.

### 7.2 La matemática detrás: semirretículos y monoides

Para que un CRDT funcione, su operación de fusión (merge) debe satisfacer tres propiedades matemáticas:

- **Conmutatividad:** `merge(A, B) = merge(B, A)` — el orden en que se fusionan los estados no importa.
- **Asociatividad:** `merge(merge(A, B), C) = merge(A, merge(B, C))` — el orden de agrupación no importa.
- **Idempotencia:** `merge(A, A) = A` — fusionar el mismo estado consigo mismo no cambia nada (recibir la misma actualización dos veces no causa problema).

Estas tres propiedades forman lo que en álgebra se llama un **semirretículo conmutativo**, o en términos más prácticos: no importa cuántas veces, en qué orden, o desde qué nodos lleguen las actualizaciones; el resultado final siempre será el mismo.

### 7.3 Los CRDTs más importantes

**G-Counter (Grow-Only Counter — Contador solo de incremento):**

Problema: un contador global que puede ser incrementado desde múltiples nodos. Si el nodo A incrementa el contador en +3 y el nodo B lo incrementa en +5, el valor final debe ser 8, independientemente del orden en que lleguen los incrementos.

Solución: cada nodo mantiene su propio sub-contador. El valor global es la suma de todos los sub-contadores.

```
Estado en Nodo A: {A: 3, B: 0, C: 0}  → valor = 3
Estado en Nodo B: {A: 0, B: 5, C: 0}  → valor = 5

Fusión: {A: max(3,0), B: max(0,5), C: max(0,0)} = {A: 3, B: 5, C: 0} → valor = 8
```

La fusión toma el máximo de cada sub-contador. Esto es correcto porque cada nodo solo incrementa su propio sub-contador, y el máximo de dos valores del mismo sub-contador siempre será el mayor de los dos (es decir, el que tuvo más incrementos).

**PN-Counter (Positive-Negative Counter — Contador que puede decrementar):**

Extiende el G-Counter con dos G-Counters: uno para incrementos (P) y uno para decrementos (N). El valor es P - N.

```
Nodo A: P={A:5, B:0}, N={A:2, B:0}  → valor = 5 - 2 = 3
Nodo B: P={A:0, B:3}, N={A:0, B:1}  → valor = 3 - 1 = 2

Fusión P: {A: max(5,0), B: max(0,3)} = {A:5, B:3}
Fusión N: {A: max(2,0), B: max(0,1)} = {A:2, B:1}
Valor fusionado: (5+3) - (2+1) = 5
```

**Casos de uso reales:** número de visitas a una página, número de items en un carrito de compras, número de likes de una publicación. Todos estos son contadores que múltiples nodos pueden incrementar o decrementar de forma independiente.

**G-Set (Grow-Only Set — Conjunto de solo adición):**

Un conjunto donde solo se pueden añadir elementos, nunca eliminar. La fusión es simplemente la unión de los dos conjuntos.

```
Nodo A: {elemento_1, elemento_2}
Nodo B: {elemento_2, elemento_3}

Fusión: {elemento_1, elemento_2} ∪ {elemento_2, elemento_3} = {elemento_1, elemento_2, elemento_3}
```

**2P-Set (Two-Phase Set — Conjunto con adición y eliminación):**

Extiende G-Set con un segundo conjunto para las eliminaciones. Un elemento es "efectivamente presente" si está en el conjunto de adiciones y no en el de eliminaciones. Una vez eliminado, no puede ser re-añadido (restricción importante).

**LWW-Element-Set (Last-Write-Wins Element Set):**

Cada elemento lleva un timestamp. La adición o eliminación más reciente (por timestamp) prevalece. Requiere relojes razonablemente sincronizados, pero es suficientemente bueno para muchas aplicaciones.

**OR-Set (Observed-Remove Set — Conjunto con eliminación observada):**

Soluciona un problema de los 2P-Set: permite re-añadir elementos. Cada adición lleva un identificador único. Una eliminación elimina todas las adiciones vistas hasta ese momento. Si el mismo elemento se añade de nuevo después, su nuevo identificador no está en el conjunto de eliminaciones.

```
Caso de problema sin OR-Set:
- Nodo A añade elemento E (id=1)
- Concurrentemente, Nodo B elimina E (basado en id=1) y Nodo A re-añade E (id=2)
- Al fusionar: la eliminación de Nodo B elimina id=1, pero id=2 persiste
- Resultado correcto: E está presente (la re-adición es posterior a la eliminación observada)
```

**LWW-Register (Last-Write-Wins Register):**

El más simple: un valor con un timestamp. Al fusionar, gana el valor con el timestamp más reciente. Adecuado para campos como "nombre de usuario actual" o "última URL de foto de perfil".

**MV-Register (Multi-Value Register):**

Más sofisticado que LWW-Register: si dos nodos modificaron el valor concurrentemente, ambos valores se preservan (como versiones en conflicto que el usuario puede resolver). Este es el modelo que usa Amazon S3 con sus "concurrent updates" y el que usaba el carrito de compras de Amazon.

### 7.4 CRDTs en la práctica: cuándo usarlos

Los CRDTs son la solución correcta cuando:

1. **El dato es intrínsecamente un contador, un conjunto, o un registro** que múltiples nodos necesitan modificar de forma independiente.
2. **La disponibilidad es más crítica que la consistencia estricta**: los CRDTs trabajan con consistencia eventual, sin coordinación entre nodos.
3. **Los conflictos son frecuentes** y la resolución manual no escala.

Sistemas que los usan en producción: Riak (sus tipos de datos), Redis (con la extensión Redis Cluster para ciertos tipos), Cassandra (counters, aunque con limitaciones), y frameworks como Automerge (para edición colaborativa de documentos, similar a Google Docs).

**Limitación fundamental:** Los CRDTs funcionan para tipos de datos específicos (contadores, conjuntos, secuencias) pero **no para invariantes arbitrarias de negocio**. Si la invariante es "el saldo nunca puede ser negativo", no hay un CRDT que la exprese directamente. Ese tipo de invariante requiere coordinación.

---

## 8. Sharding: particionamiento horizontal entre nodos

### 8.1 Qué es el sharding y por qué existe

La replicación resuelve disponibilidad y escalabilidad de lecturas. Pero ¿qué pasa cuando los datos son demasiados para caber en una sola máquina, o cuando el volumen de escrituras supera la capacidad de un solo nodo primario?

El **sharding** (también llamado **particionamiento horizontal**) es la técnica de dividir los datos de una misma tabla o colección entre múltiples nodos, de modo que cada nodo almacena y procesa solo una **porción (shard)** del total.

```
SIN SHARDING:
┌─────────────────────────────────────┐
│   Tabla usuarios (100M filas)       │
│   Nodo único:                       │
│   usuarios con id 1 a 100,000,000   │
└─────────────────────────────────────┘

CON SHARDING (4 shards):
┌──────────────────┐  ┌──────────────────┐
│  Shard 1 (Nodo A)│  │  Shard 2 (Nodo B)│
│  id: 1 a 25M     │  │  id: 25M a 50M   │
└──────────────────┘  └──────────────────┘
┌──────────────────┐  ┌──────────────────┐
│  Shard 3 (Nodo C)│  │  Shard 4 (Nodo D)│
│  id: 50M a 75M   │  │  id: 75M a 100M  │
└──────────────────┘  └──────────────────┘
```

### 8.2 Sharding por rango (Range-based sharding)

En el **sharding por rango**, las filas se dividen en shards según el valor de la clave de sharding. Los rangos son continuos.

```
Clave de sharding: user_id (BIGINT)

Shard 1: user_id [1,         25,000,000)   → Nodo A
Shard 2: user_id [25,000,000, 50,000,000)  → Nodo B
Shard 3: user_id [50,000,000, 75,000,000)  → Nodo C
Shard 4: user_id [75,000,000, 100,000,000) → Nodo D
```

**Ventajas:**

- **Range scans eficientes:** Una query como `WHERE user_id BETWEEN 1000 AND 2000` va exactamente al shard correcto sin tocar los demás.
- **Locality de datos relacionados:** Si usamos `created_at` como clave de sharding, todos los datos de un período temporal están en el mismo shard, lo que es ideal para queries históricas.
- **Balanceo manual sencillo:** Se puede mover exactamente un rango específico a otro nodo si hay hotspots.

**Desventajas:**

- **Hotspots por acceso secuencial:** Si la aplicación genera IDs secuenciales y las escrituras siempre recaen en el "último" shard (el de los IDs más recientes), ese shard recibe toda la carga de escritura mientras los demás están ociosos. Este patrón es extremadamente común y extremadamente dañino.

```
Problema del hotspot por escrituras secuenciales:

Tiempo →

Shard 1 (id 1-25M):    ██ (escrituras antiguas, ahora mayormente lecturas)
Shard 2 (id 25M-50M):  ████ (escrituras de hace un tiempo)
Shard 3 (id 50M-75M):  ███████ (escrituras más recientes)
Shard 4 (id 75M-100M): ████████████ (TODAS las escrituras nuevas van aquí)
```

- **Rebalanceo complejo:** Si un shard crece demasiado, hay que dividirlo y mover datos, lo que es una operación costosa en tiempo y recursos.

### 8.3 Sharding por hash (Hash-based sharding)

En el **sharding por hash**, se aplica una función de hash a la clave de sharding, y el resultado (módulo el número de shards) determina a qué shard va la fila.

```
Función de sharding:
  shard_id = hash(user_id) % número_de_shards

Ejemplo con 4 shards:
  hash(user_id=1001) % 4 = 2  → Shard 2 (Nodo B)
  hash(user_id=1002) % 4 = 0  → Shard 0 (Nodo A)
  hash(user_id=1003) % 4 = 3  → Shard 3 (Nodo D)
  hash(user_id=1004) % 4 = 1  → Shard 1 (Nodo C)
```

**Ventajas:**

- **Distribución uniforme:** Una buena función de hash distribuye las claves de forma aproximadamente uniforme entre los shards, eliminando los hotspots por acceso secuencial.
- **Predecible:** Para cualquier clave, el shard al que pertenece es calculable directamente, sin necesidad de consultar un directorio.

**Desventajas:**

- **Range scans imposibles:** Una query como `WHERE user_id BETWEEN 1000 AND 2000` tiene que ir a **todos los shards**, porque los user_ids en ese rango están distribuidos por toda la tabla hasheada. El hash destruye la localidad de los datos.

```
Problema del range scan con hash sharding:

Query: WHERE user_id BETWEEN 1000 AND 2000

Shard 0 (Nodo A): ¿hay algún user_id [1000,2000] aquí? → hay que consultar
Shard 1 (Nodo B): ¿hay algún user_id [1000,2000] aquí? → hay que consultar
Shard 2 (Nodo C): ¿hay algún user_id [1000,2000] aquí? → hay que consultar
Shard 3 (Nodo D): ¿hay algún user_id [1000,2000] aquí? → hay que consultar

Todos los shards deben ser consultados y el resultado se combina (scatter-gather).
```

- **Rebalanceo catastrófico:** Si añades un nuevo shard (de 4 a 5 nodos), `hash(key) % 5` produce distribuciones completamente distintas a `hash(key) % 4`. Casi todos los datos tendrían que moverse a nuevos shards. Esto es impracticable.

### 8.4 Consistent Hashing: solución al rebalanceo

El **consistent hashing** es una técnica que resuelve el problema del rebalanceo al cambiar el número de shards. En lugar de usar `hash(key) % N`, se mapean tanto las claves como los nodos en un "anillo" de hash circular.

```
          Anillo de hash (valores 0 a 2^32):

                    Nodo A (hash: 100)
                   /
         0 ──────/────── Nodo B (hash: 200)
                  \
                   \──── Nodo C (hash: 300)
                   (continúa hasta 2^32 y vuelve a 0)

Asignación de claves: una clave se asigna al primer nodo
cuyo hash sea >= al hash de la clave (en sentido horario en el anillo).
```

Cuando se añade un nuevo nodo, solo las claves que "caen" entre el nodo anterior y el nuevo nodo necesitan moverse. En promedio, solo `1/N` de las claves se mueve (donde N es el número de nodos).

El consistent hashing es usado en sistemas como Apache Cassandra, Amazon DynamoDB, y muchos sistemas de caché distribuida.

### 8.5 La clave de sharding: la decisión más crítica del diseño

La elección de la **clave de sharding** (la columna o columnas que determinan en qué shard va cada fila) es probablemente la **decisión de diseño más crítica e irreversible** en un sistema sharded. Una clave de sharding incorrecta es extremadamente costosa de cambiar después.

Los criterios para elegir una buena clave de sharding:

**1. Alta cardinalidad:** La clave debe tener muchos valores distintos. Una clave con pocos valores (como `status` con valores 'activo' / 'inactivo') solo puede crear tantos shards como valores distintos.

**2. Distribución uniforme:** Las filas deben distribuirse uniformemente entre los valores de la clave. Si el 90% de las filas tienen el mismo valor de clave, el 90% de los datos van al mismo shard.

**3. Sin hotspots de escritura:** Con hash sharding, esto está resuelto automáticamente. Con range sharding, hay que evitar claves monotónicas (como timestamps o IDs auto-incrementales).

**4. Colocalización de datos relacionados:** La clave debe agrupar en el mismo shard los datos que típicamente se consultan juntos. Esto minimiza los cross-shard queries.

```
Ejemplo: sistema de e-commerce multi-tenant (múltiples empresas)

Opción A - Shard key: user_id (hash)
  Pro: distribución uniforme
  Con: una orden y su usuario pueden estar en shards distintos.
       La query "dame todas las órdenes del usuario X" necesita 
       ir a dos shards: el del usuario y el de la orden.

Opción B - Shard key: tenant_id (la empresa)
  Pro: todos los datos de una empresa (usuarios, órdenes, productos)
       están en el mismo shard. Las queries son siempre single-shard.
  Con: si hay tenants grandes y tenants pequeños, la distribución
       puede ser desigual (hotspot por tenant grande).

Opción C - Shard key: (tenant_id, user_id) compuesta
  Colocaliza por tenant para queries multi-tabla, y la combinación
  ofrece mejor granularidad para distribución.
```

**5. Inmutabilidad del valor de la clave:** Una vez que una fila está en un shard, cambiar su clave de sharding significa moverla a otro shard. Si la clave de sharding es un atributo que cambia (como la región geográfica de un cliente), cada vez que el cliente cambia de región, su fila se mueve de shard. Esto es muy costoso.

**Regla de oro:** La clave de sharding debe ser un atributo que identifique el "dominio natural" de los datos (la entidad raíz del contexto) y que no cambie con el tiempo. En sistemas multi-tenant, suele ser el `tenant_id`. En sistemas orientados al usuario, suele ser el `user_id`. En sistemas de eventos, puede ser el `aggregate_id` del evento.

---

## 9. Transacciones distribuidas: el problema más difícil

### 9.1 Por qué las transacciones ACID no funcionan de forma nativa en sistemas distribuidos

En el módulo 8 vimos que las transacciones ACID garantizan que un conjunto de operaciones se aplica atómicamente. Esto es relativamente sencillo en una sola máquina: el log de transacciones (WAL) registra el intento, y el motor puede hacer rollback o redo al recuperarse de una falla.

En un sistema distribuido, la atomicidad entre múltiples nodos es fundamentalmente más difícil porque:

1. **Los nodos pueden fallar independientemente:** El nodo A puede confirmar su parte de la transacción, pero el nodo B puede fallar justo antes de confirmar la suya. La transacción queda en un estado parcialmente aplicado.

2. **La red puede fallar:** Después de que el nodo A confirma, el mensaje al nodo B puede perderse. ¿Debe B asumir que debe confirmar o hacer rollback?

3. **No hay un árbitro global:** No hay una entidad única que sepa el estado de todos los nodos simultáneamente.

### 9.2 Two-Phase Commit (2PC): el protocolo clásico y sus problemas

El protocolo de **confirmación en dos fases** (Two-Phase Commit, 2PC) es la solución clásica para transacciones distribuidas. Involucra un **coordinador** y múltiples **participantes**:

**Fase 1: Preparación (Voting Phase)**
```
Coordinador:
  1. Escribe "inicio de transacción" en su log.
  2. Envía PREPARE a todos los participantes.

Participante (para cada uno):
  1. Ejecuta las operaciones localmente.
  2. Escribe "prepared" en su log (garantiza que puede confirmar).
  3. Responde VOTE-COMMIT (si puede confirmar) o VOTE-ABORT (si no puede).
```

**Fase 2: Decisión (Commit Phase)**
```
Coordinador:
  Si TODOS los participantes votaron VOTE-COMMIT:
    1. Escribe "commit" en su log.
    2. Envía COMMIT a todos los participantes.
  Si ALGÚN participante votó VOTE-ABORT:
    1. Escribe "abort" en su log.
    2. Envía ABORT a todos los participantes.

Participante (para cada uno):
  1. Aplica el COMMIT o el ABORT.
  2. Responde ACK al coordinador.
```

**El problema: el coordinador es un punto único de falla y puede bloquear todo el sistema:**

```
Estado problemático: el coordinador falla DESPUÉS de enviar PREPARE
pero ANTES de enviar la decisión (COMMIT o ABORT).

Participante A: recibió PREPARE, preparó su parte, está esperando la decisión.
Participante B: recibió PREPARE, preparó su parte, está esperando la decisión.

En este estado, A y B:
  - No pueden confirmar: no saben si el coordinador decidió commit.
  - No pueden hacer abort: no saben si el coordinador decidió commit y C ya lo recibió.
  - Solo pueden BLOQUEAR hasta que el coordinador se recupere.
```

Este estado es el **estado de incertidumbre** del 2PC, y es su debilidad fundamental. Si el coordinador falla en el momento exacto entre las dos fases, todos los participantes quedan bloqueados indefinidamente (o hasta que el coordinador se recupere). Sus recursos (bloqueos, memoria, conexiones) están retenidos.

En un sistema de alta disponibilidad que necesita procesar miles de transacciones por segundo, tener transacciones bloqueadas durante minutos o horas es inaceptable.

### 9.3 Three-Phase Commit (3PC): teóricamente mejor, prácticamente raro

El 3PC intenta resolver el bloqueo del 2PC añadiendo una fase intermedia que permite a los participantes conocer la decisión del coordinador sin necesidad de que el coordinador esté disponible. Pero introduce su propia complejidad y sigue siendo susceptible a fallas de partición de red. En la práctica, casi ningún sistema de producción usa 3PC.

### 9.4 El problema fundamental: FLP Impossibility

En 1985, Fischer, Lynch y Paterson demostraron el **Teorema de imposibilidad FLP**: en un sistema distribuido asíncrono donde puede fallar al menos un nodo, es **imposible** para el sistema alcanzar consenso de forma garantizada en tiempo finito.

"Sistema asíncrono" significa que no hay límites conocidos para la latencia de los mensajes ni para la velocidad de los nodos. Esta es la realidad de cualquier red real.

La consecuencia práctica: cualquier protocolo de transacción distribuida tiene que aceptar uno de estos compromisos:
- **Puede bloquearse** (como el 2PC): en condiciones de falla, puede no terminar nunca.
- **Puede ser incorrecto**: en condiciones de falla, puede devolver resultados incorrectos.
- **Asume límites de tiempo** (timeouts): que no son garantías formales.

Este resultado es la razón de fondo por la que los sistemas modernos prefieren alternativas al 2PC: la saga (descrita a continuación), consistencia eventual con CRDTs, o coordinación mediante Paxos/Raft que proveen un modelo más práctico aunque con sus propias limitaciones.

---

## 10. El patrón Saga para transacciones distribuidas

### 10.1 La idea central

El patrón **Saga** fue propuesto originalmente por Hector Garcia-Molina y Kenneth Salem en 1987 para bases de datos, y fue adaptado al contexto de microservicios por Chris Richardson. La idea central es:

> En lugar de una transacción distribuida que requiere coordinación síncrona entre múltiples servicios, dividir la operación en una secuencia de **transacciones locales**, cada una aplicada en un servicio individual, con **transacciones compensatorias** que se ejecutan si alguna parte de la saga falla.

Una transacción compensatoria es la operación que deshace el efecto de una transacción local ya confirmada. No es un rollback en el sentido de ACID (que nunca llegó a confirmar); es una operación de negocio que invierte el efecto.

```
Ejemplo: Reserva de viaje
  Paso 1: Reservar vuelo (servicio Vuelos)
  Paso 2: Reservar hotel (servicio Hoteles)
  Paso 3: Reservar auto (servicio Autos)
  Paso 4: Cobrar tarjeta (servicio Pagos)

Si el paso 3 falla:
  Compensación 2: Cancelar reserva de hotel
  Compensación 1: Cancelar reserva de vuelo
  (El paso 4 nunca se ejecutó, no necesita compensación)
```

### 10.2 Dos estilos de implementación: Coreografía y Orquestación

**Coreografía (Choreography):**

Cada servicio publica eventos cuando termina su tarea, y otros servicios reaccionan a esos eventos. No hay coordinador central; el flujo emerge de las reacciones de cada servicio.

```
                  Evento: ReservaIniciadaEvent
                        ↓
             ┌──────────────────────┐
             │ Servicio Vuelos       │
             │ - Reserva vuelo       │
             │ - Publica: VueloReservadoEvent │
             └──────────────────────┘
                        ↓
             ┌──────────────────────┐
             │ Servicio Hoteles     │
             │ - Reserva hotel      │
             │ - Publica: HotelReservadoEvent │
             └──────────────────────┘
                        ↓
             ┌──────────────────────┐
             │ Servicio Autos       │
             │ - Si falla, publica: AutoFallóEvent │
             └──────────────────────┘
                        ↓
             ┌──────────────────────┐
             │ Servicio Hoteles     │
             │ - Reacciona a AutoFallóEvent │
             │ - Cancela hotel (compensación) │
             └──────────────────────┘
```

**Pro:** Simple de implementar. Sin punto único de falla. Bajo acoplamiento.
**Con:** Difícil de razonar sobre el flujo completo. Difícil de debuggear. Puede haber ciclos de eventos inesperados.

**Orquestación (Orchestration):**

Un orquestador central conoce el flujo completo y dirige a cada servicio qué hacer en cada paso. Los servicios son participantes que responden al orquestador.

```
             ┌──────────────────────────────────────┐
             │         ORQUESTADOR (Saga)           │
             │                                      │
             │  Estado: [Paso 1] → [Paso 2] → [Paso 3] │
             │                                      │
             │  1. Envía comando: ReservarVuelo     │────▶ Servicio Vuelos
             │  2. Recibe: VueloReservado            │◀────
             │  3. Envía comando: ReservarHotel     │────▶ Servicio Hoteles
             │  4. Recibe: HotelReservado            │◀────
             │  5. Envía comando: ReservarAuto      │────▶ Servicio Autos
             │  6. Recibe: AutoFalló (error)         │◀────
             │  7. Envía comando: CancelarHotel     │────▶ Servicio Hoteles
             │  8. Envía comando: CancelarVuelo     │────▶ Servicio Vuelos
             └──────────────────────────────────────┘
```

**Pro:** Flujo explícito y visible. Más fácil de debuggear. El estado de la saga es explícito.
**Con:** El orquestador es un componente adicional que mantener. Puede convertirse en un objeto "dios" que conoce demasiado sobre otros servicios.

### 10.3 El modelo de datos de una Saga

Para implementar una Saga correctamente, se necesita persistir su estado. El patrón típico incluye una tabla de estado de saga:

```sql
-- Tabla que registra el estado de cada instancia de saga
CREATE TABLE saga_reserva_viaje (
    saga_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    estado          TEXT NOT NULL,  -- 'iniciada', 'vuelo_reservado', 
                                    -- 'hotel_reservado', 'completada',
                                    -- 'compensando', 'compensada', 'fallida'
    reservation_id  TEXT,           -- ID del sistema externo (legado)
    vuelo_id        TEXT,           -- Para compensación: cancelar este vuelo
    hotel_id        TEXT,           -- Para compensación: cancelar este hotel
    auto_id         TEXT,           -- Null si no se llegó a reservar
    usuario_id      BIGINT NOT NULL REFERENCES usuarios(id),
    datos_request   JSONB NOT NULL, -- Datos originales del request
    motivo_falla    TEXT,           -- Si hubo falla, qué fue
    creada_en       TIMESTAMPTZ NOT NULL DEFAULT now(),
    actualizada_en  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Tabla de eventos de la saga (para auditoría y debugging)
CREATE TABLE saga_reserva_viaje_eventos (
    id          BIGSERIAL PRIMARY KEY,
    saga_id     UUID NOT NULL REFERENCES saga_reserva_viaje(saga_id),
    paso        TEXT NOT NULL,     -- 'reservar_vuelo', 'compensar_hotel', etc.
    accion      TEXT NOT NULL,     -- 'ejecutado', 'compensado', 'fallido'
    datos       JSONB,             -- Datos del paso
    ocurrido_en TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 10.4 El problema de la consistencia eventual en Sagas: anomalías

Las Sagas tienen consistencia eventual, no ACID. Entre el inicio y el fin de una saga, el sistema puede estar en un estado inconsistente desde la perspectiva del negocio. Esto se llama **dirty reads**: otro proceso puede leer datos creados por la saga antes de que la saga complete.

Ejemplo del problema:
```
t1: Saga de reserva comienza. Reserva vuelo (saldo disponible: $1000 → reservado: $1000).
t2: Otro proceso lee el saldo disponible del usuario → ve $0 disponible.
t3: Otro proceso intenta cobrar $100 al usuario → rechazado porque "no tiene saldo disponible".
t4: La saga de reserva falla. Compensa → libera $1000 (saldo disponible vuelve a $1000).
t5: El cobro de $100 que debería haber pasado fue rechazado incorrectamente.
```

Las soluciones a este problema incluyen:

**Countermeasures de Sagas (Chris Richardson):** Técnicas como "semantic lock" (marcar un recurso como "en proceso de saga" para que otros procesos lo traten de forma especial), "commutative updates" (operaciones que se pueden reordenar sin cambiar el resultado), y "pessimistic view" (la secuencia de pasos se ordena para que los estados sucios sean lo más cortos posibles).

---

## 11. Joins distribuidos: el costo más subestimado

### 11.1 Por qué los joins son caros en sistemas distribuidos

En una base de datos de un solo nodo, un JOIN es una operación local: ambas tablas están en el mismo disco, en el mismo servidor. El motor puede elegir entre nested loop join, hash join, o merge join (módulo 7) con costos en el orden de milisegundos a segundos.

En un sistema distribuido sharded, cuando las dos tablas del JOIN están en nodos distintos, el motor tiene que:

1. **Identificar qué shards participan:** ¿Qué subconjunto de shards de la tabla A y qué subconjunto de la tabla B son necesarios para el JOIN?
2. **Enviar los datos relevantes por la red:** Mover filas de uno o más shards al nodo coordinador (o usar map-reduce).
3. **Ejecutar el JOIN localmente** en el nodo que recibió los datos.
4. **Devolver los resultados.**

Los pasos 1 y 2 tienen costos que no existen en un sistema de un solo nodo:
- **Latencia de red:** Incluso en una red de 1 Gbps, enviar 10 millones de filas de 100 bytes cada una significa transferir 1 GB de datos. A 1 Gbps, eso tarda ~8 segundos solo de transferencia, sin contar el procesamiento.
- **Amplificación del ancho de banda:** Si el JOIN une 3 tablas distribuidas, pueden necesitarse múltiples rondas de transferencia de datos.
- **Consumo de memoria en el coordinador:** El nodo que materializa el JOIN necesita memoria para mantener los datos temporales.

### 11.2 Los tres patrones de JOIN distribuido

**Patrón 1: Broadcast (Difusión):**

Si una de las tablas del JOIN es pequeña, se puede **difundir** (broadcast) la tabla pequeña a todos los shards de la tabla grande, y cada shard hace su JOIN local.

```
Tabla grande: orders (sharded por customer_id en 10 nodos)
Tabla pequeña: product_categories (100 filas, una réplica en cada nodo)

Broadcast join:
  Cada nodo tiene product_categories completa.
  Cada shard de orders hace JOIN local con la réplica local de product_categories.
  No hay transferencia de datos entre nodos para el JOIN.
  
Resultado: 10 JOINs locales → combinar resultados (solo los resultados, no los datos fuente)
```

Este patrón funciona bien cuando una tabla es suficientemente pequeña para replicarse en todos los nodos (típicamente < 100 MB). Es el patrón que usan Spark y Hive con el concepto de "broadcast join".

**Patrón 2: Co-located JOIN (JOIN colocado):**

Si ambas tablas están sharded por la misma clave y de la misma manera, las filas que participan en el JOIN están en el mismo shard por construcción.

```
Tabla orders: sharded por customer_id (hash)
Tabla customers: sharded por customer_id (hash)

JOIN: orders.customer_id = customers.customer_id

Como ambas están sharded por customer_id con el mismo hash,
para cualquier valor de customer_id, tanto la orden como el cliente
están en el MISMO shard.

El JOIN se puede ejecutar completamente en cada shard sin mover datos entre nodos.
```

Este es el patrón de diseño más eficiente y la razón principal por la que **la elección de la clave de sharding tiene que considerar los JOINs que la aplicación necesita**.

**Patrón 3: Shuffle JOIN (Redistribución):**

Si las tablas no están co-locadas y ninguna es suficientemente pequeña para broadcast, el motor tiene que **redistribuir** los datos: re-particionar temporalmente una o ambas tablas por la clave de JOIN.

```
Tabla orders: sharded por order_date (range)
Tabla customers: sharded por customer_id (hash)

JOIN: orders.customer_id = customers.customer_id

Para hacer este JOIN:
  Opción A: Re-distribuir orders por customer_id (shuffle)
    → Enviar todas las filas de orders a los shards correspondientes según customer_id
    → Luego hacer co-located join
  Opción B: Re-distribuir customers por order_date (shuffle)
    → Enviar todas las filas de customers a los shards de orders
  
Ambas opciones requieren mover posiblemente millones de filas por la red.
```

El shuffle JOIN es la opción de último recurso: costosa en red, memoria y tiempo. Es lo que frameworks como Spark usan cuando no hay otra opción, y es la razón por la que el rendimiento de ciertos queries en Spark "explota" al superar ciertos umbrales de datos.

### 11.3 El costo real en números

Para hacer tangible el costo, consideremos un ejemplo concreto:

```
Escenario: sistema de e-commerce con sharding

Tabla orders:   500 millones de filas, 200 bytes por fila = 100 GB
Tabla customers: 50 millones de filas, 500 bytes por fila = 25 GB
Clave de JOIN: customer_id

Si las tablas NO están co-locadas (sharded por claves distintas):

Shuffle de orders (redistribuir por customer_id):
  - Transferir 100 GB de datos por la red
  - Red interna de datacenter: 10 Gbps → ~80 segundos solo de transferencia
  - En AWS, 100 GB de datos entre instancias en la misma VPC: ~$0.01/GB → $1 por query
  - Con 100 queries/hora de este tipo → $2,400/mes solo en transferencia de datos

Si las tablas SÍ están co-locadas (mismo shard key):
  - 0 bytes de datos transferidos para el JOIN
  - 0 costo de red para el JOIN
  - Latencia: decenas de milisegundos en lugar de decenas de segundos
```

---

## 12. Modelado para evitar joins distribuidos

### 12.1 La estrategia principal: diseñar para co-localización

La implicación del análisis de joins distribuidos es directa: **el modelo de datos en un sistema distribuido debe diseñarse para que los JOINs más frecuentes y críticos sean siempre co-locados**.

Esto se logra eligiendo la **misma clave de sharding** para todas las tablas que frecuentemente se consultan juntas.

**Ejemplo: sistema de gestión de pedidos multi-tenant**

```sql
-- Diseño SIN co-localización (malo para distribución)
-- Cada tabla con su propia clave de sharding natural
CREATE TABLE tenants (
    tenant_id BIGINT PRIMARY KEY  -- shard key: tenant_id
);

CREATE TABLE users (
    user_id   BIGINT PRIMARY KEY,  -- shard key: user_id
    tenant_id BIGINT NOT NULL REFERENCES tenants(tenant_id),
    ...
);

CREATE TABLE orders (
    order_id  BIGINT PRIMARY KEY,  -- shard key: order_id
    user_id   BIGINT NOT NULL REFERENCES users(user_id),
    ...
);

-- Problema: orders está sharded por order_id, users por user_id.
-- La query "dame todas las órdenes del usuario X" requiere cross-shard JOIN.
```

```sql
-- Diseño CON co-localización (correcto para distribución)
-- Todas las tablas sharded por tenant_id

CREATE TABLE users (
    tenant_id BIGINT NOT NULL,  -- shard key: tenant_id
    user_id   BIGINT NOT NULL,
    ...
    PRIMARY KEY (tenant_id, user_id)  -- la shard key es parte de la PK
);

CREATE TABLE orders (
    tenant_id BIGINT NOT NULL,  -- misma shard key: tenant_id
    order_id  BIGINT NOT NULL,
    user_id   BIGINT NOT NULL,
    ...
    PRIMARY KEY (tenant_id, order_id),
    FOREIGN KEY (tenant_id, user_id) REFERENCES users(tenant_id, user_id)
);

CREATE TABLE order_items (
    tenant_id  BIGINT NOT NULL,  -- misma shard key: tenant_id
    order_id   BIGINT NOT NULL,
    item_id    BIGINT NOT NULL,
    ...
    PRIMARY KEY (tenant_id, order_id, item_id),
    FOREIGN KEY (tenant_id, order_id) REFERENCES orders(tenant_id, order_id)
);

-- Ahora: todas las queries dentro de un tenant son siempre co-locadas.
-- La query "dame todas las órdenes del usuario X del tenant T" 
-- va exactamente a UN shard, porque tenant_id está en todas las tablas.
```

### 12.2 Desnormalización intencional para evitar cross-shard queries

En un sistema de un solo nodo, la desnormalización es generalmente una deuda técnica a evitar (rompe las formas normales del módulo 2, introduce anomalías de actualización). En sistemas distribuidos, la **desnormalización es a veces la única forma práctica de evitar joins distribuidos**.

**Ejemplo:**

En un sistema OLTP normalizado, para mostrar la lista de pedidos con el nombre del cliente:
```sql
SELECT o.order_id, c.nombre, o.total
FROM orders o
JOIN customers c ON c.customer_id = o.customer_id
WHERE o.customer_id = 12345;
```

Si `orders` y `customers` están en shards distintos, este JOIN es distribuido.

**Solución con desnormalización:**
```sql
CREATE TABLE orders (
    order_id        BIGINT PRIMARY KEY,
    customer_id     BIGINT NOT NULL,
    customer_nombre TEXT NOT NULL,   -- Denormalizado: copia del nombre
    customer_email  TEXT NOT NULL,   -- Denormalizado: copia del email
    total           NUMERIC(10,2) NOT NULL,
    ...
);

-- Ahora la query es:
SELECT order_id, customer_nombre, total
FROM orders
WHERE customer_id = 12345;
-- Sin JOIN. Sin cross-shard.
```

**El costo:** Si el nombre del cliente cambia, hay que actualizar tanto la tabla `customers` como todas las filas de `orders` que tienen ese cliente. Esto se gestiona con un evento de dominio: cuando `customers.nombre` cambia, se publica un `CustomerNombreActualizadoEvent` que desencadena la actualización de `orders`.

Este patrón de **consistencia eventual entre entidades desnormalizadas** es muy común en sistemas distribuidos modernos.

### 12.3 El modelo de datos orientado a queries (CQRS-inspired)

En sistemas distribuidos, a veces es más pragmático diseñar el modelo de datos explícitamente para soportar los patrones de acceso más frecuentes, en lugar de derivar el modelo de datos desde el dominio y luego intentar que las queries funcionen.

Esta idea está relacionada con el patrón **CQRS (Command Query Responsibility Segregation)**: separar el modelo de escritura (el modelo "verdadero" de los datos) del modelo de lectura (optimizado para los patrones de consulta).

```
MODELO DE ESCRITURA (normalizado, single source of truth):
  customers: id, nombre, email, region
  orders:    id, customer_id, fecha, estado
  order_items: order_id, product_id, cantidad, precio_unitario

MODELOS DE LECTURA (desnormalizados, optimizados por query):
  
  Para "lista de pedidos del cliente X":
  ┌────────────────────────────────────────────────────┐
  │  orders_by_customer (shard key: customer_id)       │
  │  customer_id | order_id | fecha | total | estado  │
  │  customer_nombre | customer_email                  │
  └────────────────────────────────────────────────────┘
  
  Para "dashboard de ventas por región":
  ┌────────────────────────────────────────────────────┐
  │  sales_by_region (shard key: region)               │
  │  region | fecha | total_ventas | num_ordenes       │
  └────────────────────────────────────────────────────┘
  
  Para "historial de un producto":
  ┌────────────────────────────────────────────────────┐
  │  orders_by_product (shard key: product_id)         │
  │  product_id | order_id | fecha | cantidad | precio │
  └────────────────────────────────────────────────────┘
```

Los modelos de lectura se mantienen actualizados mediante eventos: cuando una orden se crea o se modifica en el modelo de escritura, un evento se procesa y actualiza todos los modelos de lectura relevantes.

Este enfoque tiene un costo: mantener múltiples representaciones de los mismos datos es complejo. Pero el beneficio es que cada query lee exactamente de la representación optimizada para ella, sin necesidad de JOINs en lectura.

---

## 13. Patrones de modelado específicos para sistemas distribuidos

### 13.1 Outbox Pattern: garantizando que el evento se publique

Uno de los problemas más sutiles en sistemas distribuidos es la **publicación confiable de eventos**. Cuando un servicio necesita tanto actualizar su base de datos como publicar un evento en un message broker (como Kafka), hay un riesgo de inconsistencia:

```
// CÓDIGO PROBLEMÁTICO
BEGIN TRANSACTION;
  UPDATE orders SET estado = 'confirmada' WHERE id = 123;
COMMIT;  -- ← aquí la transacción termina

// Si el servicio cae AQUÍ (entre la escritura y la publicación del evento):
publishEvent(new OrderConfirmedEvent(orderId=123));  // ← NUNCA OCURRE
```

Si el servicio cae entre el `COMMIT` y el `publishEvent`, la orden está confirmada en la base de datos pero el evento nunca llegó a Kafka. Los demás servicios que dependen de ese evento (el servicio de inventario, el servicio de notificaciones) nunca saben que la orden fue confirmada.

**El Outbox Pattern** resuelve esto con una tabla adicional:

```sql
-- Tabla de outbox: los eventos que deben publicarse
CREATE TABLE outbox (
    id          BIGSERIAL PRIMARY KEY,
    topic       TEXT NOT NULL,          -- el topic de Kafka donde va el evento
    evento_tipo TEXT NOT NULL,          -- 'OrderConfirmed', 'OrderCancelled', etc.
    payload     JSONB NOT NULL,         -- los datos del evento
    publicado   BOOLEAN NOT NULL DEFAULT FALSE,
    creado_en   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- La transacción de negocio incluye la escritura en outbox:
BEGIN;
  UPDATE orders SET estado = 'confirmada' WHERE id = 123;
  
  INSERT INTO outbox (topic, evento_tipo, payload)
  VALUES ('orders', 'OrderConfirmed', '{"order_id": 123, "customer_id": 456}');
COMMIT;
```

Un proceso independiente (el **Outbox Publisher** o **Relay**) lee periódicamente las filas de `outbox` donde `publicado = FALSE`, las publica en Kafka, y luego marca `publicado = TRUE`.

La garantía: tanto la actualización del pedido como la entrada en outbox están en la **misma transacción local**. Si el servicio cae antes del COMMIT, ninguna de las dos se aplica. Si el COMMIT ocurre, ambas ocurren, y el Outbox Publisher eventualmente publicará el evento.

Este patrón garantiza **at-least-once delivery**: el evento puede publicarse más de una vez (si el Outbox Publisher cae después de publicar pero antes de marcar como publicado), pero nunca se pierde. Los consumidores del evento deben ser idempotentes para manejar duplicados.

### 13.2 Inbox Pattern (Idempotent Consumer)

El complemento del Outbox Pattern: garantizar que **recibir el mismo evento múltiples veces no cause efectos duplicados**.

```sql
-- Tabla de inbox: registra qué mensajes ya fueron procesados
CREATE TABLE inbox (
    mensaje_id  TEXT PRIMARY KEY,       -- ID único del mensaje (del broker)
    procesado_en TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Al procesar un evento:
BEGIN;
  -- Intenta insertar el mensaje_id. Si ya existe, ON CONFLICT lo ignora.
  INSERT INTO inbox (mensaje_id) VALUES ($mensaje_id)
  ON CONFLICT (mensaje_id) DO NOTHING;
  
  -- Si se insertó (GET DIAGNOSTICS rows_affected):
  IF rows_affected = 1 THEN
    -- Procesar el evento (es la primera vez que lo vemos)
    UPDATE inventory SET stock = stock - 1 WHERE product_id = $product_id;
  END IF;
  -- Si rows_affected = 0: el mensaje ya fue procesado, ignorar (idempotencia)
COMMIT;
```

### 13.3 Versioning de APIs y esquemas

En un sistema de una sola aplicación monolítica, cambiar el esquema de la base de datos implica una migración y un deploy sincronizado. En un sistema distribuido con múltiples servicios, los servicios son desplegados de forma independiente.

Esto crea el problema de **schema compatibility**: cuando el Servicio A envía un evento con un esquema determinado, el Servicio B (que puede estar en una versión antigua o nueva) debe poder leerlo.

Las estrategias son:

**Forward compatibility (compatibilidad hacia adelante):** Los mensajes nuevos pueden ser leídos por consumidores viejos. Logrado añadiendo solo campos opcionales (nunca eliminando campos requeridos).

**Backward compatibility (compatibilidad hacia atrás):** Los mensajes viejos pueden ser leídos por consumidores nuevos. Logrado asignando valores por defecto a los campos nuevos.

**El contrato de datos en el modelo:**
```json
// Versión 1 del evento OrderCreated
{
  "order_id": 123,
  "customer_id": 456,
  "total": 99.99
}

// Versión 2 (backward AND forward compatible)
{
  "version": 2,           // campo nuevo, optional
  "order_id": 123,
  "customer_id": 456,
  "total": 99.99,
  "currency": "USD",      // campo nuevo, con default implícito
  "source_channel": "web" // campo nuevo, con default implícito
}
```

Herramientas como **Apache Avro** y **Protocol Buffers** tienen sistemas de evolución de esquema incorporados que hacen cumplir estas reglas de compatibilidad automáticamente.

### 13.4 Timestamp híbrido y causalidad

Un problema específico del diseño de esquemas distribuidos es la **ordenación de eventos**. Los relojes de distintos servidores no están perfectamente sincronizados. Un evento en el Servidor A con timestamp `T=100` puede haber ocurrido antes o después que un evento en el Servidor B con timestamp `T=99`, dependiendo de la deriva de los relojes.

**Vector Clocks (Relojes vectoriales):**

Cada nodo mantiene un vector de contadores, uno por nodo. Al enviar un mensaje, el nodo incrementa su propio contador y envía el vector completo. Al recibir un mensaje, el nodo toma el máximo de cada componente.

```
Nodo A: vector [A:1, B:0, C:0]
Nodo B: vector [A:0, B:1, C:0]

A envía mensaje a B:
  A: [A:1, B:0, C:0] → enviado con el mensaje
  B al recibir: max([A:0, B:1, C:0], [A:1, B:0, C:0]) = [A:1, B:1, C:0]
  B incrementa su contador: [A:1, B:2, C:0]
```

Dos eventos con relojes vectoriales tienen un orden causal si y solo si un vector es estrictamente mayor que el otro en todas las componentes. Si no, son concurrentes (ninguno causó al otro).

**HLC (Hybrid Logical Clocks — Relojes Lógicos Híbridos):**

Propuesto por Kulkarni et al. (2014), combinan la intuición de los relojes físicos (los timestamps que los usuarios entienden) con las garantías causales de los relojes lógicos. Son usados en CockroachDB y algunos sistemas de DynamoDB.

```sql
-- Columna con HLC en un sistema distribuido
CREATE TABLE eventos (
    id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    tipo        TEXT NOT NULL,
    datos       JSONB NOT NULL,
    hlc_ts      BYTEA NOT NULL,  -- timestamp HLC (16 bytes: 8 físico + 8 lógico)
    nodo_origen TEXT NOT NULL
);
```

---

## 14. Resumen y conexión con el resto del curso

### Los conceptos clave del módulo

| Concepto | Lo que significa para el diseño |
|---|---|
| **CAP Theorem** | Elegir entre C y A durante particiones; P no es opcional |
| **PACELC** | También elegir entre latencia y consistencia en operación normal |
| **Consistencia linealizable** | Todos ven el mismo valor; costosa pero necesaria para invariantes globales |
| **Consistencia eventual** | Convergencia garantizada eventualmente; no orden ni timing garantizados |
| **Replicación síncrona** | Zero data loss; mayor latencia de escritura |
| **Replicación asíncrona** | Posible data loss; menor latencia de escritura |
| **Split-brain** | Dos primarios simultáneos; quórum como solución |
| **CRDTs** | Estructuras que se fusionan sin conflicto; para contadores, conjuntos |
| **Sharding por rango** | Range scans eficientes; hotspots por claves monotónicas |
| **Sharding por hash** | Distribución uniforme; range scans ineficientes |
| **Clave de sharding** | Decisión más crítica e irreversible del modelo distribuido |
| **Co-localización** | JOINs en el mismo shard; mismo shard key para tablas relacionadas |
| **Two-Phase Commit** | Atomicidad distribuida; bloqueante si el coordinador falla |
| **Saga Pattern** | Alternativa a 2PC; compensaciones en lugar de rollback |
| **Outbox Pattern** | Publicación confiable de eventos; evita la dualidad escritura/publicación |

### La meta-lección del módulo

El sistema distribuido no es una extensión del modelo relacional de una sola máquina: es un modelo de computación fundamentalmente diferente, con compromisos radicalmente distintos.

El error más común es diseñar un modelo de datos para un sistema de una sola máquina y luego "distribuirlo" esperando que funcione igual de bien. No funciona. Las abstracciones que permiten ignorar la distribución (transacciones globales, JOINs arbitrarios, consistencia fuerte gratuita) o son imposibles, o tienen costos de rendimiento que las hacen inviables en sistemas de escala.

El diseñador que entiende los sistemas distribuidos diseña el modelo sabiendo que:

1. **La co-localización es la primera pregunta**, no la última. Antes de elegir tipos de datos o índices, hay que decidir la clave de sharding y garantizar que las entidades relacionadas queden co-locadas.

2. **La consistencia es un espectro**, y la consistencia fuerte tiene un costo real en latencia y disponibilidad. Solo se paga ese costo donde genuinamente se necesita.

3. **Las transacciones distribuidas son costosas**. El modelo debe estar diseñado para que la mayoría de las transacciones sean locales a un shard. Las que inevitablemente cruzan shards se resuelven con Sagas, no con 2PC.

4. **La desnormalización es a veces la respuesta correcta**, no una deuda técnica. En sistemas distribuidos, evitar un join distribuido justifica mantener datos duplicados y gestionarlos con eventos.

5. **La idempotencia es un principio de diseño**, no un detalle de implementación. Toda operación que puede reintentarse (en caso de falla de red, de nodo, de partición) debe ser idempotente. Esto afecta el diseño de los identificadores, los campos de versión, y los patrones de escritura.

### Conexión con módulos futuros

- **Módulo 14 (Patrones de Fowler y la capa de acceso):** Los patrones de Fowler (Unit of Work, Repository, Identity Map) son difíciles de implementar en sistemas distribuidos. Un Repository que "parece una colección en memoria" oculta el hecho de que las operaciones van a distintos nodos. Entender los sistemas distribuidos ayuda a saber cuándo estas abstracciones son seguras y cuándo son peligrosas.

- **Módulo 15 (ORMs e impedancia objeto-relacional):** Los ORMs están diseñados para sistemas de una sola base de datos. En sistemas distribuidos, los ORMs típicamente no conocen el concepto de shard key, no pueden co-locar entidades automáticamente, y generan JOINs que en un sistema sharded son cross-shard sin que el desarrollador lo sepa. Saber cuándo el ORM es la herramienta incorrecta es conocimiento que viene del módulo 13.

- **Módulo 16 (El oficio):** El refactoring de un sistema monolítico a distribuido es uno de los proyectos más complejos en ingeniería de software. Expand/contract, backfill progresivo, blue/green — todas estas técnicas del módulo 16 adquieren una complejidad adicional cuando el sistema destino es distribuido.

---

## 15. Ejercicios de comprensión

**Ejercicio 1.** Una empresa de pagos quiere diseñar un sistema que procese transferencias entre cuentas. Las cuentas son de usuarios de 50 países distintos. El sistema debe:
- Procesar 100,000 transferencias por segundo en horas pico.
- Tener disponibilidad de 99.99% (máximo ~52 minutos de downtime por año).
- Garantizar que una transferencia nunca crea dinero de la nada (la suma total de todos los saldos es constante).

a) Analiza este sistema con el teorema CAP. ¿Qué propiedad sacrificarías durante una partición de red y por qué? ¿Cuáles son las consecuencias de esa elección?

b) Diseña la clave de sharding. Considera al menos tres opciones y evalúa cada una con respecto a: distribución uniforme, co-localización de datos, hotspots, y capacidad de hacer range scans.

c) La invariante "la suma de todos los saldos es constante" es una invariante global (abarca todas las filas de todas las cuentas). ¿Por qué es imposible garantizarla en tiempo real en un sistema distribuido con consistencia eventual? ¿Qué garantía alternativa ofrecerías y cómo la implementarías?

d) Diseña el esquema completo para el modelo de escritura, incluyendo la tabla `cuentas`, la tabla `transferencias`, y el patrón de Outbox para publicar eventos de transferencia.

---

**Ejercicio 2.** Un sistema de e-commerce quiere migrar de un PostgreSQL de un solo nodo (que tiene 2TB de datos y ya no puede escalar verticalmente) a un sistema sharded. Las tablas principales son:
- `tenants` (1,000 empresas que usan el sistema)
- `customers` (50M filas, distribuidos entre los tenants)
- `orders` (500M filas)
- `order_items` (2,000M filas)
- `products` (10M filas, relativamente estables)

Las queries más frecuentes son:
1. `GET /tenants/:tid/customers/:cid/orders` — órdenes de un cliente específico.
2. `GET /tenants/:tid/orders/:oid` — detalle de una orden con sus items.
3. `GET /tenants/:tid/products` — catálogo de productos de un tenant.
4. `POST /tenants/:tid/orders` — crear una nueva orden (escribe en `orders` y en `order_items`).

a) Propón la clave de sharding. Justifica con respecto a las queries listadas y el balance entre co-localización y distribución uniforme.

b) Para la query #4 (crear una orden), que escribe en dos tablas (`orders` y `order_items`), ¿es necesario un protocolo de transacción distribuida? Justifica basándote en tu elección de shard key del inciso (a).

c) `products` es una tabla pequeña (10M filas × 500 bytes = 5 GB) que es referenciada en `order_items`. ¿Qué estrategia de join usarías para la query `GET /tenants/:tid/orders/:oid` que necesita datos de `order_items` y `products`?

d) Diseña el esquema DDL de las cuatro tablas principales, incluyendo la clave de sharding en la clave primaria.

---

**Ejercicio 3.** Un sistema de redes sociales permite a los usuarios dar "me gusta" a publicaciones. La tabla tiene la estructura:
```sql
CREATE TABLE likes (
    post_id  BIGINT NOT NULL,
    user_id  BIGINT NOT NULL,
    PRIMARY KEY (post_id, user_id)
);
```
Y un contador:
```sql
CREATE TABLE post_stats (
    post_id     BIGINT PRIMARY KEY,
    like_count  BIGINT NOT NULL DEFAULT 0
);
```

El sistema está replicado en 5 centros de datos en distintos continentes. Una publicación viral puede recibir 100,000 likes por segundo desde múltiples continentes simultáneamente.

a) ¿Por qué la operación `UPDATE post_stats SET like_count = like_count + 1 WHERE post_id = ?` es problemática en este sistema distribuido?

b) Diseña una solución usando un PN-Counter como CRDT. Especifica la estructura de datos que cada nodo mantiene y el algoritmo de fusión.

c) ¿Cuándo (en qué condiciones) el conteo que ve un usuario puede ser incorrecto? ¿Cuánto tiempo puede durar esa inconsistencia?

d) ¿Cambiaría tu respuesta si la operación fuera "reservar una de las 1000 entradas disponibles para un concierto" en lugar de "dar like"? ¿Por qué?

---

**Ejercicio 4.** Un sistema de microservicios tiene los siguientes servicios:
- **Servicio de Pedidos:** gestiona la creación y estado de pedidos.
- **Servicio de Inventario:** gestiona el stock de productos.
- **Servicio de Pagos:** procesa cobros a tarjetas de crédito.
- **Servicio de Notificaciones:** envía emails y push notifications.

Cuando un usuario confirma su carrito, el flujo debe:
1. Reservar el stock (Inventario).
2. Procesar el pago (Pagos).
3. Crear el pedido confirmado (Pedidos).
4. Enviar notificación de confirmación (Notificaciones).

a) Diseña una Saga usando el estilo de orquestación para este flujo. Incluye el diagrama de estados del orquestador y las transacciones compensatorias para cada paso.

b) Diseña el esquema de la tabla `saga_checkout` que persiste el estado de la saga.

c) Identifica al menos dos anomalías de "dirty read" que pueden ocurrir durante la ejecución de esta saga (estados que otro proceso puede leer que son "incoherentes" desde la perspectiva del negocio). Para cada una, propón una countermeasure.

d) El Servicio de Notificaciones no necesita ser parte de la saga estrictamente (si falla el envío de email, el pedido ya está hecho). ¿Cómo lo desacoplarías de la saga principal sin perder la garantía de que la notificación eventualmente se envíe?

---

**Ejercicio 5.** Considerando los siguientes modelos de consistencia: **linearizability**, **causal consistency**, **read-your-writes**, y **eventual consistency**:

Para cada uno de los siguientes sistemas, indica qué modelo de consistencia es el **mínimo necesario** y justifica:

a) Un sistema de saldo de cuentas bancarias donde el saldo visible al cliente siempre debe ser el más reciente.

b) Un sistema de comentarios en redes sociales donde una respuesta siempre debe mostrar el comentario al que responde (pero dos comentarios sin relación pueden mostrarse en distinto orden en distintos usuarios).

c) Un sistema donde un usuario actualiza su nombre de perfil y espera verlo actualizado inmediatamente en su propio cliente, aunque otros usuarios puedan ver el nombre antiguo por algunos segundos.

d) Un sistema de contador de visitas para una página de noticias viral (precisión del ±5% es aceptable).

e) Un sistema de registro distribuido donde múltiples instancias del servicio pueden publicar mensajes, y cualquier lector debe ver los mensajes en el mismo orden que cualquier otro lector.

---

*Próximo módulo: Patrones de Fowler y la capa de acceso a datos — donde exploraremos cómo el código de aplicación interactúa con el modelo de datos a través de patrones como Active Record, Data Mapper, Repository, Unit of Work e Identity Map, y cuándo cada uno es la elección correcta.*
