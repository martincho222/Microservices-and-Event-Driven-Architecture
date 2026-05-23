# API Management for Microservices Architecture

---

## 1. El problema de gestionar APIs en arquitectura de microservicios

**Español:**
En una arquitectura monolítica existe un único punto de entrada para los clientes: una URL, un puerto, un proceso. Cuando migramos a microservicios, ese único punto se fragmenta en decenas de servicios independientes, cada uno con su propia URL, puerto y protocolo. Gestionar esa exposición de forma segura, eficiente y coherente se convierte en uno de los retos más prácticos del día a día.

**English:**
In a monolithic architecture there is a single entry point for clients: one URL, one port, one process. When we migrate to microservices, that single point splinters into dozens of independent services, each with its own URL, port, and protocol. Managing that exposure securely, efficiently, and consistently becomes one of the most practical day-to-day challenges.

### ShopFast — antes del API Gateway

```
          ┌──────────────────┐
          │    Mobile App    │
          └────────┬─────────┘
                   │  necesita renderizar una página de producto
                   │
     ┌─────────────┼──────────────────────────────┐
     │             │             │         │       │
     ▼             ▼             ▼         ▼       ▼
catalog:3001  pricing:3002  reviews:3003  recs:3004  cart:3005
```

**Español:**
Para renderizar una sola página de producto, la app móvil debe hacer **5 llamadas** a **5 servicios distintos** con **5 URLs distintas**. Esto es solo para una página — en producción ShopFast tiene más de 15 microservicios expuestos.

**English:**
To render a single product page, the mobile app must make **5 calls** to **5 different services** at **5 different URLs**. This is just one page — in production ShopFast has more than 15 microservices exposed.

### Consecuencias concretas

```
┌─────────────────────────────────────────────────────────────────┐
│              PROBLEMAS SIN API GATEWAY                          │
├─────────────────────────────────────────────────────────────────┤
│  1. Acoplamiento cliente–servicio: el cliente conoce la         │
│     topología interna. Si Orders se mueve de :3001 a :4001,     │
│     hay que actualizar TODOS los clientes.                      │
│                                                                 │
│  2. Cross-cutting concerns duplicados: autenticación, rate      │
│     limiting, logging y CORS deben implementarse en CADA        │
│     microservicio (N servicios × misma lógica = N puntos        │
│     de fallo y divergencia).                                    │
│                                                                 │
│  3. N llamadas de red por pantalla: cada llamada adicional      │
│     añade latencia. En redes móviles 3G/4G, 5 llamadas          │
│     serializadas pueden tardar >2 segundos.                     │
│                                                                 │
│  4. Gestión de protocolos heterogéneos: Catalog usa REST,       │
│     Recommendations usa gRPC, Notifications usa WebSocket.      │
│     El cliente necesita hablar los 3 protocolos.                │
│                                                                 │
│  5. Sin punto único de aplicación de seguridad: una             │
│     vulnerabilidad en un servicio puede exponerse antes de      │
│     detectarse; no existe un WAF centralizado.                  │
│                                                                 │
│  6. Difícil evolución de la API: añadir versioning (v1/v2)      │
│     debe coordinarse en todos los servicios simultáneamente.    │
└─────────────────────────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "Every microservice should be an implementation detail hidden from the client. The client should talk to one door, not to the entire building."

---

## 2. El patrón API Gateway

**Español:**
El API Gateway es un **proxy reverso inteligente** que actúa como único punto de entrada para todos los clientes externos. No es solo un router: es la capa donde se centralizan todas las responsabilidades transversales que de otra forma se duplicarían en cada servicio.

**English:**
The API Gateway is an **intelligent reverse proxy** that acts as the single entry point for all external clients. It is not just a router: it is the layer where all cross-cutting concerns are centralized — concerns that would otherwise be duplicated across every service.

### Arquitectura con API Gateway

```
                     ┌──────────────────────────────────────────────┐
  ┌──────────────┐   │                 API GATEWAY                  │
  │  Mobile App  │──▶│                                             │
  └──────────────┘   │  ┌──────────────────────────────────────┐    │
                     │  │       Cross-cutting concerns         │    │
  ┌──────────────┐   │  │  • Auth/AuthZ  (JWT / OAuth 2.0)     │    │
  │ Web Browser  │──▶│ │  • Rate limiting (100 req/min/user)  │   │
  └──────────────┘   │  │  • SSL termination (HTTPS → HTTP)    │   │
                     │  │  • Request logging & tracing         │   │
  ┌──────────────┐   │  │  • CORS headers                      │   │
  │  3rd Party   │──▶│  │  • Request/Response transformation   │   │
  │     API      │   │  │  • Caching                           │   │
  └──────────────┘   │  │  • Circuit breaker                   │   │
                     │  └──────────────────┬───────────────────┘   │
                     │                     │ routing                │
                     └─────────────────────┼────────────────────────┘
                                           │
              ┌────────────────────────────┼──────────────────────────┐
              │                            │                          │
              ▼                            ▼                          ▼
    ┌──────────────────┐        ┌──────────────────┐       ┌──────────────────┐
    │  Catalog Service │        │  Orders Service  │       │ Reviews Service  │
    │    :3001         │        │    :3002         │       │    :3003         │
    └──────────────────┘        └──────────────────┘       └──────────────────┘
```

### Responsabilidades del API Gateway

| Responsabilidad | Descripción | Ejemplo en ShopFast |
|-----------------|-------------|---------------------|
| **Routing** | Enruta `/api/catalog/*` a Catalog Service, `/api/orders/*` a Orders Service | `GET /api/catalog/123` → `catalog:3001/products/123` |
| **Autenticación** | Valida JWT antes de reenviar la request; los servicios confían en el header `X-User-Id` | El gateway verifica el token de sesión antes de pasar a Orders |
| **Rate Limiting** | Limita a N requests/min por usuario/IP para prevenir abuso | API pública: 60 req/min; API interna: sin límite |
| **SSL Termination** | El gateway habla HTTPS con el exterior; internamente HTTP sin overhead TLS | Reduce la carga de CPU en cada microservicio (~15% típicamente) |
| **Request Aggregation** | Fan-out a múltiples servicios + fan-in de respuestas en una sola | Una llamada del cliente → gateway llama a Catalog + Pricing + Reviews en paralelo |
| **Transformación** | Adapta el formato de request/response entre cliente y servicio | Convierte `page=2&size=10` a `offset=10&limit=10` |
| **Circuit Breaker** | Si Reviews falla, devuelve un fallback vacío en vez de error 500 | Página de producto carga sin reviews en lugar de romperse |
| **Observabilidad** | Inyecta `X-Request-Id` en cada request para trazabilidad distribuida | Todos los logs de un flujo comparten el mismo `requestId` |

### Request Aggregation — ejemplo ShopFast

```
  Cliente solicita: GET /api/product-page/123
                              │
                              ▼
                       ┌─────────────┐
                       │ API Gateway │
                       └──────┬──────┘
                              │  fan-out (en paralelo)
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
    Catalog :3001       Pricing :3002       Reviews :3003
  GET /products/123   GET /price/123    GET /reviews/123
           │                  │                  │
           └──────────────────┴──────────────────┘
                              │  fan-in
                              ▼
              { product, price, reviews }  ◀── 1 sola respuesta al cliente
```

**Español:**
En lugar de que el cliente haga 3 llamadas serializadas (~450ms), el gateway las lanza **en paralelo** y espera el resultado más lento (~150ms). El cliente recibe **un único JSON** con todo lo que necesita.

**English:**
Instead of the client making 3 sequential calls (~450ms), the gateway fires them **in parallel** and waits for the slowest one (~150ms). The client receives **a single JSON** with everything it needs.

### Herramientas reales

```
┌─────────────────────────────────────────────────────────────────┐
│                 API GATEWAY — OPCIONES REALES                   │
├────────────────────┬────────────────────────────────────────────┤
│  AWS API Gateway   │  Managed, pay-per-request, integra con     │
│                    │  Lambda y servicios AWS nativamente         │
├────────────────────┼────────────────────────────────────────────┤
│  Kong              │  Open-source, plugin ecosystem, on-prem    │
│                    │  o cloud; muy usado en empresas             │
├────────────────────┼────────────────────────────────────────────┤
│  Nginx / OpenResty │  Alta performance, configuración en Lua,   │
│                    │  base de muchas soluciones enterprise       │
├────────────────────┼────────────────────────────────────────────┤
│  Traefik           │  Cloud-native, autodescubrimiento en K8s,  │
│                    │  configuración dinámica                     │
├────────────────────┼────────────────────────────────────────────┤
│  Azure API Mgmt.   │  Managed, portal de developer, policies    │
│                    │  declarativas                               │
├────────────────────┼────────────────────────────────────────────┤
│  Envoy Proxy       │  Base de Istio/service mesh, L7 proxy de   │
│                    │  alto performance                           │
└────────────────────┴────────────────────────────────────────────┘
```

> **Regla de Pogrebinsky:** "The API Gateway should handle cross-cutting concerns only — it should not contain business logic. The moment you put business logic in the gateway, you've created a new monolith."

---

## 3. Load Balancer vs API Gateway

**Español:**
Es uno de los puntos de confusión más frecuentes en equipos que empiezan con microservicios. Ambos componentes reciben tráfico y lo distribuyen, pero operan en capas distintas con propósitos distintos. No son intercambiables — son **complementarios**.

**English:**
This is one of the most common points of confusion in teams new to microservices. Both components receive traffic and distribute it, but they operate at different layers with different purposes. They are not interchangeable — they are **complementary**.

### Diferencia conceptual

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         LOAD BALANCER                                   │
│                                                                         │
│  Pregunta: ¿A cuál INSTANCIA del MISMO servicio envío esta request?     │
│                                                                         │
│  Client ──▶ Load Balancer ──▶ Orders-instance-1  (round robin)         │
│                          ├──▶ Orders-instance-2                        │
│                          └──▶ Orders-instance-3                        │
│                                                                         │
│  Capa L4 (TCP/UDP) o L7 (HTTP). No entiende de negocio.                │
│  No autentica. No transforma. No limita rate.                           │
│  Solo distribuye carga para escalabilidad y alta disponibilidad.        │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY                                     │
│                                                                         │
│  Pregunta: ¿A cuál SERVICIO (diferente) envío esta request?             │
│                                                                         │
│  Client ──▶ API Gateway ──▶ /api/orders/*   ──▶ Orders Service         │
│                         ├──▶ /api/catalog/* ──▶ Catalog Service        │
│                         └──▶ /api/reviews/* ──▶ Reviews Service        │
│                                                                         │
│  Capa L7 exclusivamente. Entiende HTTP, headers, JWT, rutas.            │
│  Autentica. Transforma. Limita rate. Agrega. Cachea.                    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Tabla comparativa

| Dimensión | Load Balancer | API Gateway |
|-----------|--------------|-------------|
| **Propósito principal** | Distribuir carga entre instancias del mismo servicio | Enrutar y gestionar acceso entre servicios distintos |
| **Capa OSI** | L4 (TCP/UDP) o L7 (HTTP) | L7 (HTTP/HTTPS/WebSocket/gRPC) |
| **¿Conoce el negocio?** | No — solo IPs y puertos | Sí — rutas, headers, JWT, versioning |
| **Autenticación** | No | Sí — valida tokens antes de reenviar |
| **Rate limiting** | No | Sí — por usuario, IP o API key |
| **Transformación de request** | No | Sí — headers, payload, protocolos |
| **Agregación de respuestas** | No | Sí — fan-out / fan-in |
| **Circuit breaker** | No (solo health checks básicos) | Sí — fallback configurable por ruta |
| **Casos de uso** | Alta disponibilidad, escalado horizontal | Exposición segura de APIs, BFF |
| **Ejemplos** | AWS ELB/ALB, HAProxy, Nginx (modo LB) | Kong, AWS API GW, Traefik, Azure APIM |

### Coexistencia en producción — ShopFast

```
         Internet
             │
             ▼
      ┌─────────────┐
      │ API Gateway │  ◀── autenticación, rate limiting, routing, SSL termination
      └──────┬──────┘
             │
      ┌──────┴────────────────────┐
      │                           │
      ▼                           ▼
┌──────────────┐           ┌──────────────┐
│ Load Balancer│           │ Load Balancer│  ◀── uno por servicio (en K8s: Service)
│  (Orders)    │           │  (Catalog)   │
└──────┬───────┘           └──────┬───────┘
       │                          │
  ┌────┴────┐                ┌────┴────┐
  │         │                │         │
Orders-1  Orders-2        Catalog-1  Catalog-2   ◀── instancias escaladas horizontalmente
```

**Español:**
En producción los dos componentes **siempre coexisten**. El API Gateway es el portero del edificio — decide qué piso visitas y si tienes permiso. El Load Balancer es el ascensor — una vez que se decide el piso correcto, lleva al huésped a la cabina disponible más rápida.

**English:**
In production both components **always coexist**. The API Gateway is the building's doorman — it decides which floor you visit and whether you're allowed in. The Load Balancer is the elevator — once the right floor is decided, it takes you to the fastest available cab.

### Backend For Frontend (BFF) — variante del API Gateway

**Español:**
Cuando los clientes son muy distintos (web vs mobile vs TV), un único API Gateway puede volverse un cuello de botella con lógicas contradictorias. El patrón **BFF (Backend For Frontend)** crea un gateway especializado por tipo de cliente.

**English:**
When clients are very different (web vs mobile vs TV), a single API Gateway can become a bottleneck with conflicting logic. The **BFF (Backend For Frontend)** pattern creates a specialized gateway per client type.

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│   Web BFF   │       │ Mobile BFF  │       │   TV BFF    │
│  (React)    │       │ iOS/Android │       │  (SmartTV)  │
└──────┬──────┘       └──────┬──────┘       └──────┬──────┘
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             │
               ┌─────────────┼──────────────┐
               ▼             ▼              ▼
           Catalog         Orders         Reviews
           Service         Service        Service
```

**Español:**
El BFF de mobile devuelve imágenes en baja resolución y payloads mínimos. El BFF web devuelve datos completos. Cada BFF agrega solo los campos que su cliente necesita — sin lógica de negocio, solo orquestación y transformación de respuestas.

**English:**
The mobile BFF returns low-resolution images and minimal payloads. The web BFF returns full data. Each BFF aggregates only the fields its client needs — no business logic, just orchestration and response transformation.

> **Regla de Pogrebinsky:** "Use BFF when different client types have fundamentally different data needs. Don't create a BFF per team — create one per client type."


