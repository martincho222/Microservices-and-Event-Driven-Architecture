# Bases de datos en Arquitectura de Microservicios / Databases in Microservices Architecture

---

## 1. Motivación para la BD por microservicio / Motivation for Database per Microservice

**Español:**
En un monolito, todos los módulos comparten una única base de datos. A primera vista esto parece eficiente, pero es una de las fuentes más peligrosas de **acoplamiento oculto**: si un módulo cambia el esquema de una tabla, todos los demás módulos que leen esa tabla se ven afectados. Esto hace que los equipos tengan que coordinarse para cada migración de esquema y que sea imposible desplegar un módulo de forma independiente.

**English:**
In a monolith, all modules share a single database. At first glance this seems efficient, but it is one of the most dangerous sources of **hidden coupling**: if one module changes a table's schema, every other module reading that table is affected. This forces teams to coordinate on every schema migration and makes it impossible to deploy a module independently.

```
Problema: DB compartida en el monolito / Problem: Shared DB in the monolith

  ┌──────────────────────────────────────────────────────────────┐
  │                        Monolito                              │
  │                                                              │
  │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌──────────┐ │
  │  │  Orders   │  │  Catalog  │  │  Shipping │  │  Users   │ │
  │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └────┬─────┘ │
  │        │              │              │              │        │
  └────────┼──────────────┼──────────────┼──────────────┼────────┘
           │              │              │              │
           └──────────────┴──────┬───────┴──────────────┘
                                 ▼
                        ┌─────────────────┐
                        │   PostgreSQL DB  │
                        │  (única, global) │
                        └─────────────────┘

  Consecuencias / Consequences:
  ✗ Orders cambia tabla "products" → Catalog y Shipping se rompen
  ✗ Imposible desplegar Orders sin coordinar con otros equipos
  ✗ Imposible escalar solo la DB de Orders bajo alta carga
  ✗ Imposible usar una BD diferente (ej. Redis) para un módulo específico
```

**Español:**
La motivación central del principio de **base de datos por microservicio** es romper este acoplamiento a nivel de datos: si cada servicio es el único dueño de sus datos, los equipos pueden evolucionar sus esquemas, elegir su tecnología de almacenamiento y desplegar de forma completamente independiente.

**English:**
The core motivation of the **database-per-microservice** principle is to break this coupling at the data layer: if each service is the sole owner of its data, teams can evolve their schemas, choose their storage technology, and deploy completely independently.

---

## 2. Beneficios del principio DB por microservicio / Benefits of the Database per Microservice Principle

**Español:**
Cuando cada servicio tiene su propia base de datos aislada, se desbloquean cuatro ventajas fundamentales:

**English:**
When each service has its own isolated database, four fundamental advantages are unlocked:

```
Beneficios / Benefits:

  ┌─────────────────────────────────────────────────────────────────┐
  │  1. Desacoplamiento total / Full decoupling                     │
  │     → Catalog cambia su esquema → Orders no se entera           │
  │     → Cada equipo dueño de su DB, sin coordinación externa      │
  ├─────────────────────────────────────────────────────────────────┤
  │  2. Libertad tecnológica / Technology freedom                   │
  │     → Cada servicio elige la DB más adecuada para su dominio    │
  │     → Search Service → Elasticsearch                            │
  │     → Session Service → Redis (key-value)                       │
  │     → Order Service → PostgreSQL (relacional, transaccional)    │
  │     → Catalog Service → MongoDB (documentos flexibles)          │
  ├─────────────────────────────────────────────────────────────────┤
  │  3. Escalado independiente / Independent scalability            │
  │     → Search recibe 10x más tráfico que Payments en Black Friday│
  │     → Se escala solo la DB de Search, sin tocar las demás       │
  │     → Costos optimizados por servicio                           │
  ├─────────────────────────────────────────────────────────────────┤
  │  4. Aislamiento de fallos / Fault isolation                     │
  │     → La DB de Notifications cae → Orders sigue funcionando     │
  │     → Ningún servicio tiene una dependencia crítica en la DB    │
  │       de otro servicio                                          │
  └─────────────────────────────────────────────────────────────────┘
```

### Pros y contras de cada tecnología de BD / Pros and Cons of Each DB Technology

#### PostgreSQL — Order Service

| | |
|---|---|
| **✅ Pros** | **❌ Contras** |
| Transacciones ACID completas (crítico para pedidos y pagos) | Escalado horizontal complejo (requiere sharding manual) |
| Integridad referencial con foreign keys | Esquema rígido — migraciones costosas si el modelo cambia mucho |
| Soporte maduro para queries complejas con JOINs | Mayor overhead de escritura vs bases de datos NoSQL |
| Ideal para datos financieros que requieren consistencia total | No es óptimo para estructuras de datos jerárquicas o flexibles |

#### MongoDB — Catalog Service

| | |
|---|---|
| **✅ Pros** | **❌ Contras** |
| Esquema flexible — un producto puede tener atributos distintos (ropa vs electrónica) | Sin transacciones ACID multi-documento por defecto (en versiones antiguas) |
| Documentos anidados evitan JOINs (producto + variantes + imágenes en un doc) | Consistencia eventual en clusters — lecturas pueden devolver datos obsoletos |
| Escalado horizontal nativo con sharding | Duplicación de datos frecuente (desnormalización intencional) |
| Alto rendimiento en lecturas de catálogos grandes | Consultas ad-hoc y reporting complejos son más difíciles que en SQL |

#### Elasticsearch — Search Service

| | |
|---|---|
| **✅ Pros** | **❌ Contras** |
| Búsqueda full-text con relevancia y ranking nativo | No es una DB primaria — los datos deben sincronizarse desde la fuente |
| Filtros facetados (por precio, categoría, valoración) en tiempo real | Consistencia eventual — un producto nuevo puede tardar segundos en aparecer |
| Altamente escalable horizontalmente (shards + replicas) | Operacionalmente complejo: índices, mappings, reindexación |
| Soporte para búsqueda fuzzy, sinónimos y autocomplete | No soporta transacciones — no apto para datos que requieran ACID |

#### Redis — Session Service

| | |
|---|---|
| **✅ Pros** | **❌ Contras** |
| Latencia sub-milisegundo — ideal para sesiones y caché | Datos en memoria — capacidad limitada por RAM disponible |
| TTL (time-to-live) nativo para expirar sesiones automáticamente | Persistencia opcional — en modo caché puro, datos se pierden si el nodo cae |
| Estructuras de datos ricas: strings, hashes, sets, sorted sets | No apto para datos relacionales o consultas complejas |
| Pub/Sub integrado para notificaciones en tiempo real | Sin transacciones ACID completas entre múltiples keys |

```
Arquitectura correcta / Correct architecture (ShopFast):

                          API Gateway
                              │
         ┌────────────────────┼────────────────────┐
         ▼                    ▼                    ▼
  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
  │Order Service│     │Catalog Svc  │     │Search Svc   │
  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘
         │                   │                   │
         ▼                   ▼                   ▼
  ┌────────────┐     ┌─────────────┐     ┌──────────────┐
  │ PostgreSQL │     │  MongoDB    │     │Elasticsearch │
  │ (orders)   │     │ (catalog)   │     │  (search)    │
  └────────────┘     └─────────────┘     └──────────────┘

  Cada servicio es el único que accede a su propia DB.
  Ningún servicio puede leer ni escribir directamente en la DB de otro.
```

> **ES:** La regla es absoluta: **ningún servicio puede acceder directamente a la base de datos de otro servicio**. Si necesita datos de otro servicio, debe pedírselos vía API o recibirlos a través de eventos.
>
> **EN:** The rule is absolute: **no service may directly access another service's database**. If it needs data from another service, it must request it via API or receive it through events.

---

## 3. Desventajas del principio DB por microservicio / Downsides of the Database per Microservice Principle

**Español:**
El principio de BD por microservicio introduce complejidad real que debe gestionarse conscientemente. Pogrebinsky es explícito al respecto: los beneficios superan a los costos, pero los costos existen y no deben ignorarse.

**English:**
The database-per-microservice principle introduces real complexity that must be consciously managed. Pogrebinsky is explicit about this: the benefits outweigh the costs, but the costs exist and must not be ignored.

```
Desventajas / Downsides:

  ┌─────────────────────────────────────────────────────────────────┐
  │  1. Sin transacciones ACID entre servicios                      │
  │     No ACID transactions across services                        │
  │                                                                 │
  │     Monolito: una transacción abarca Orders + Inventory + Pay   │
  │     → commit o rollback atómico                                 │
  │                                                                 │
  │     Microservicios: cada servicio tiene su propia DB            │
  │     → no hay forma de hacer un commit atómico entre 3 DBs       │
  │     → se necesita consistencia eventual + patrones como Saga    │
  ├─────────────────────────────────────────────────────────────────┤
  │  2. Consultas entre servicios son complejas                     │
  │     Cross-service queries are complex                           │
  │                                                                 │
  │     Monolito: SELECT o.*, p.name FROM orders o JOIN products p  │
  │     → una sola query SQL                                        │
  │                                                                 │
  │     Microservicios: Order Service llama a Catalog Service API   │
  │     → múltiples round-trips de red, mayor latencia              │
  │     → necesita API Composition o patrón CQRS                    │
  ├─────────────────────────────────────────────────────────────────┤
  │  3. Duplicación de datos / Data duplication                     │
  │                                                                 │
  │     Order Service necesita el nombre del producto en su DB      │
  │     → copia desnormalizada de datos de Catalog                  │
  │     → si el nombre cambia en Catalog, Order tiene dato obsoleto │
  │     → hay que sincronizar vía eventos (eventual consistency)    │
  ├─────────────────────────────────────────────────────────────────┤
  │  4. Sobrecarga operacional / Operational overhead               │
  │                                                                 │
  │     ShopFast monolito: 1 DB que operar, hacer backup, monitorear│
  │     ShopFast microservicios: 8+ DBs (PostgreSQL, MongoDB,       │
  │     Redis, Elasticsearch...)                                    │
  │     → más instancias, más costos, más conocimiento del equipo   │
  └─────────────────────────────────────────────────────────────────┘
```

### Tabla resumen / Summary Table

| Aspecto | Monolito (DB compartida) | Microservicios (DB por servicio) |
|---|---|---|
| Transacciones ACID | ✅ Nativas, simples | ❌ Requieren patrón Saga |
| Queries entre módulos | ✅ SQL JOIN directo | ❌ API calls / CQRS |
| Despliegue independiente | ❌ Coordinación requerida | ✅ Cada equipo independiente |
| Elección de tecnología | ❌ Una sola DB para todos | ✅ Best-fit por servicio |
| Escalado granular | ❌ Toda la DB o nada | ✅ Por servicio |
| Aislamiento de fallos | ❌ Fallo global | ✅ Fallo localizado |
| Complejidad operacional | ✅ Baja (1 DB) | ❌ Alta (N DBs) |

> **ES:** La complejidad de las transacciones distribuidas y las queries entre servicios es el precio de la autonomía. Pogrebinsky enseña que este precio vale la pena para sistemas que necesitan escalar equipos e infraestructura de forma independiente, pero no para todos los sistemas.
>
> **EN:** The complexity of distributed transactions and cross-service queries is the price of autonomy. Pogrebinsky teaches that this price is worth paying for systems that need to scale teams and infrastructure independently — but not for every system.

Database per microservices principle
Motivation:
    sharing a database thightly couples:
        codebases
        teams
    eliminated when each microservice owns its data
drawbacks of database per microservices:
    worse performance
    complex join operation
    no transactions

