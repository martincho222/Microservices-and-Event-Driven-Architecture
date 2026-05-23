# Microservices & Event-Driven Architecture
## Introducción y Definiciones / Introduction & Definitions

> Ver también / See also:
> - [Beneficios](benefits.md)
> - [Desafíos, Patrones y Herramientas](challenges.md)

---

## ¿Por qué Microservicios? / Motivation for Microservices

Es la arquitectura moderna y popular en la industria. En este estilo arquitectónico, todo el sistema se organiza como una colección de servicios independientes. Cada uno tiene su responsabilidad y es propiedad de un equipo independiente de desarrolladores. Este tipo de arquitecturas permite crear sistemas altamente escalables que llegan a millones de usuarios.

---

## 1. Arquitectura Monolítica / Monolithic Architecture

**Español:**
En una arquitectura monolítica, toda la aplicación se construye y despliega como **una sola unidad**. Todos los módulos (autenticación, pagos, inventario, notificaciones, etc.) comparten el mismo proceso, la misma base de datos y el mismo código. Es el estilo más tradicional y el punto de partida natural de la mayoría de los sistemas.

**English:**
In a monolithic architecture, the entire application is built and deployed as a **single unit**. All modules (authentication, payments, inventory, notifications, etc.) share the same process, the same database, and the same codebase. It is the most traditional style and the natural starting point for most systems.

```
┌─────────────────────────────────────────────┐
│              MONOLITH / MONOLITO             │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │   Auth   │  │  Orders  │  │ Products │  │
│  ├──────────┤  ├──────────┤  ├──────────┤  │
│  │ Payments │  │ Shipping │  │  Users   │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│                                             │
│           Base de datos compartida          │
│              Shared Database                │
└─────────────────────────────────────────────┘
```

### Tipos de Monolitos / Types of Monoliths

| Tipo / Type | Español | English |
|---|---|---|
| **Monolito tradicional** | Una sola app sin separación interna clara | Single app with no clear internal separation |
| **Monolito modular** | Una sola app con módulos bien definidos internamente | Single app with well-defined internal modules |
| **Monolito distribuido** | Múltiples servicios fuertemente acoplados (lo peor de ambos mundos) | Multiple tightly coupled services (worst of both worlds) |

---

## 2. Microservicios / Microservices

**Español:**
Los microservicios son un estilo arquitectónico donde una aplicación se divide en servicios **pequeños, independientes y desplegables de forma autónoma**. Cada servicio es responsable de una sola funcionalidad de negocio, tiene su propia base de datos y se comunica con otros servicios mediante APIs o eventos. Cada servicio puede ser desarrollado, desplegado y escalado de forma totalmente independiente.

**English:**
Microservices is an architectural style where an application is divided into **small, independent, and autonomously deployable** services. Each service is responsible for a single business functionality, has its own database, and communicates with other services via APIs or events. Each service can be developed, deployed, and scaled completely independently.

```
┌──────────────────────────────────────────────────────────────┐
│                  MICROSERVICES / MICROSERVICIOS               │
│                                                              │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐ │
│  │  User    │   │  Order   │   │ Payment  │   │ Product  │ │
│  │ Service  │   │ Service  │   │ Service  │   │ Service  │ │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘   └────┬─────┘ │
│       │              │              │              │        │
│  ┌────┴──┐      ┌────┴──┐      ┌────┴──┐      ┌────┴──┐   │
│  │  DB   │      │  DB   │      │  DB   │      │  DB   │   │
│  └───────┘      └───────┘      └───────┘      └───────┘   │
│                                                              │
│              Comunicación via API / Events                   │
└──────────────────────────────────────────────────────────────┘
```

---

## 3. Arquitectura Orientada a Eventos / Event-Driven Architecture (EDA)

**Español:**
La arquitectura orientada a eventos es un patrón de diseño donde los componentes del sistema se comunican mediante la **producción, detección y consumo de eventos**. Un evento es un registro de algo que ya ocurrió (ej. "PedidoCreado", "PagoAprobado"). Los servicios están completamente desacoplados: el productor no sabe quién va a consumir el evento ni cuándo. La comunicación es **asíncrona** y pasa a través de un broker de mensajes como Kafka o RabbitMQ.

**English:**
Event-driven architecture is a design pattern where system components communicate through the **production, detection, and consumption of events**. An event is a record of something that already happened (e.g., "OrderCreated", "PaymentApproved"). Services are completely decoupled — the producer does not know who will consume the event or when. Communication is **asynchronous** and passes through a message broker like Kafka or RabbitMQ.

```
┌─────────────┐    evento     ┌─────────────┐    evento     ┌─────────────┐
│   Order     │ ────────────► │   Message   │ ────────────► │  Payment    │
│   Service   │  OrderCreated │    Broker   │  OrderCreated │   Service   │
│ (Producer)  │               │  (Kafka)    │               │ (Consumer)  │
└─────────────┘               └─────────────┘               └─────────────┘
                                                                     │
                                                              también escucha
                                                                     │
                                                             ┌────────┴────┐
                                                             │  Inventory  │
                                                             │   Service   │
                                                             │ (Consumer)  │
                                                             └─────────────┘
```

---

## 4. Conceptos clave / Key Concepts

| Término / Term | Español | English |
|---|---|---|
| **Service / Servicio** | Unidad independiente con una responsabilidad específica de negocio | Independent unit with a specific business responsibility |
| **Event / Evento** | Notificación inmutable de algo que ocurrió en el sistema | Immutable notification of something that happened in the system |
| **Producer / Productor** | Servicio que genera y publica eventos al broker | Service that generates and publishes events to the broker |
| **Consumer / Consumidor** | Servicio que se suscribe y procesa eventos del broker | Service that subscribes to and processes events from the broker |
| **Message Broker** | Middleware que almacena y enruta mensajes entre servicios (Kafka, RabbitMQ) | Middleware that stores and routes messages between services (Kafka, RabbitMQ) |
| **API Gateway** | Punto de entrada único que enruta solicitudes de clientes a los microservicios | Single entry point that routes client requests to microservices |
| **Service Discovery** | Mecanismo para que los servicios se encuentren dinámicamente en la red | Mechanism for services to find each other dynamically on the network |
| **Loose Coupling** | Los servicios son independientes entre sí; un cambio en uno no afecta a otros | Services are independent; a change in one does not affect others |
| **High Cohesion** | Cada servicio agrupa toda la lógica relacionada a su dominio | Each service groups all logic related to its domain |
| **Scalability** | Capacidad de crecer horizontalmente añadiendo más instancias de un servicio | Ability to grow horizontally by adding more instances of a service |
| **Idempotency** | Una operación produce el mismo resultado si se ejecuta una o varias veces | An operation produces the same result whether executed once or multiple times |
| **Eventual Consistency** | Los datos convergen a un estado consistente con el tiempo, no inmediatamente | Data converges to a consistent state over time, not immediately |

---

## 5. Comparación / Comparison

| Aspecto | Monolítico / Monolithic | Microservicios / Microservices |
|---|---|---|
| **Despliegue** | Todo el sistema se despliega junto | Cada servicio se despliega de forma independiente |
| **Escalabilidad** | Se escala toda la aplicación | Se escala solo el servicio que lo necesita |
| **Tecnología** | Un solo stack tecnológico | Cada servicio puede usar su propio lenguaje/DB |
| **Equipo** | Un equipo grande trabaja en el mismo código | Equipos pequeños dueños de servicios específicos |
| **Fallo** | Un bug puede tumbar todo el sistema | Un fallo se aísla en un solo servicio |
| **Complejidad inicial** | Baja — simple de empezar | Alta — requiere infraestructura distribuida |
| **Transacciones** | ACID nativas entre módulos | Requiere patrón Saga y consistencia eventual |
| **Debugging** | Stack trace lineal y sencillo | Requiere tracing distribuido (Jaeger, Zipkin) |
| **Latencia interna** | Llamadas en memoria (microsegundos) | Llamadas de red (milisegundos) |
| **CI/CD** | Un solo pipeline para toda la app | Un pipeline independiente por servicio |
| **Costo infraestructura** | Bajo | Alto (Kubernetes, brokers, gateways, monitoring) |
| **Ideal para** | Startups, MVPs, equipos pequeños | Empresas grandes, alto tráfico, múltiples equipos |

---

## 6. ¿Cuándo usar cada uno? / When to Use Each?

| Usa Monolito si... / Use Monolith if... | Usa Microservicios si... / Use Microservices if... |
|---|---|
| Estás construyendo un MVP o nuevo producto | El sistema tiene dominios de negocio bien separados |
| El equipo tiene menos de 5-8 personas | Múltiples equipos trabajan en el mismo sistema |
| El dominio de negocio no está completamente definido | Necesitas escalar módulos de forma independiente |
| El presupuesto de infraestructura es limitado | Los despliegues son lentos, arriesgados o bloqueantes |
| La velocidad de iteración es la prioridad | El tiempo de build del monolito supera los 10-15 min |
| No hay experiencia con sistemas distribuidos | Alta disponibilidad y resiliencia son críticas |