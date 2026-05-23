# Microservices Deployment - Cloud Virtual Machine, Dedicated Hosts and Instances

Antes de contenedores y Kubernetes, la forma estándar de desplegar microservicios era directamente sobre **máquinas virtuales (VMs)** en la nube. Entender este modelo es fundamental porque muchas organizaciones aún lo usan, y porque los patrones de aislamiento y tenencia que introduce siguen vigentes incluso en arquitecturas modernas.

**Español:** Desplegar microservicios en la nube mediante VMs implica elegir un modelo de tenencia: ¿comparto el hardware físico con otros clientes (multi-tenant), o tengo hardware dedicado exclusivamente para mí (dedicated)? Esta decisión impacta el costo, el rendimiento, el aislamiento y el cumplimiento normativo.

**English:** Deploying microservices on cloud VMs involves choosing a tenancy model: do I share physical hardware with other customers (multi-tenant), or do I have hardware exclusively for myself (dedicated)? This choice impacts cost, performance, isolation, and regulatory compliance.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              CLOUD VM TENANCY MODELS — OVERVIEW                         │
│                                                                         │
│  ┌─────────────────────────────┐   ┌─────────────────────────────────┐  │
│  │    MULTI-TENANT VM          │   │    DEDICATED HOST / INSTANCE    │  │
│  │                             │   │                                 │  │
│  │  Physical Server            │   │  Physical Server                │  │
│  │  ┌───────┬───────┬───────┐  │   │  ┌───────────────────────────┐ │  │
│  │  │ VM-A  │ VM-B  │ VM-C  │  │   │  │       YOUR VMs ONLY       │ │  │
│  │  │(yours)│(other)│(other)│  │   │  │  ┌────────┬────────────┐  │ │  │
│  │  └───────┴───────┴───────┘  │   │  │  │  VM-A  │   VM-B     │  │ │  │
│  │  Hypervisor                 │   │  │  │(yours) │  (yours)   │  │ │  │
│  │                             │   │  │  └────────┴────────────┘  │ │  │
│  │  • Shared CPU/RAM/Network   │   │  └───────────────────────────┘ │  │
│  │  • Cheaper                  │   │                                 │  │
│  │  • "Noisy Neighbor" risk    │   │  • Exclusive hardware           │  │
│  │  • Standard compliance      │   │  • Higher cost                  │  │
│  │                             │   │  • BYOL, HIPAA, PCI-DSS         │  │
│  └─────────────────────────────┘   └─────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Multi-tenant Virtual Machine deployment

**Español:** En el modelo **multi-tenant**, el proveedor de nube (AWS, GCP, Azure) ejecuta las VMs de múltiples clientes sobre el mismo servidor físico. El hipervisor (Xen, KVM, Hyper-V) es responsable de aislar cada VM, garantizando que un cliente no pueda leer la memoria de otro. Es el modelo por defecto de todas las nubes públicas y el más económico.

**English:** In the **multi-tenant** model, the cloud provider (AWS, GCP, Azure) runs VMs from multiple customers on the same physical server. The hypervisor (Xen, KVM, Hyper-V) isolates each VM, ensuring one customer cannot read another's memory. It is the default model for all public clouds and the most economical.

### Cómo funciona / How it works

```
┌─────────────────────────────────────────────────────────────────────────┐
│              MULTI-TENANT VM — ARQUITECTURA INTERNA                     │
│                                                                         │
│  Physical Server (e.g., AWS c6i.metal — 128 vCPU, 256 GB RAM)          │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                         HYPERVISOR (KVM)                          │  │
│  │                                                                   │  │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐ │  │
│  │  │  VM: ShopFast │  │  VM: Cliente B│  │  VM: Cliente C        │ │  │
│  │  │  (orders-svc) │  │  (app X)      │  │  (app Y)              │ │  │
│  │  │               │  │               │  │                       │ │  │
│  │  │  4 vCPU       │  │  8 vCPU       │  │  2 vCPU               │ │  │
│  │  │  16 GB RAM    │  │  32 GB RAM    │  │  8 GB RAM             │ │  │
│  │  │  OS: Ubuntu   │  │  OS: RHEL     │  │  OS: Windows          │ │  │
│  │  └───────────────┘  └───────────────┘  └───────────────────────┘ │  │
│  │                                                                   │  │
│  │  Recursos compartidos: CPU cores, RAM, NIC, NVMe storage          │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ⚠ "Noisy Neighbor": Si Cliente B hace CPU-intensive work,              │
│     ShopFast puede ver latencia elevada aunque no esté bajo carga.      │
└─────────────────────────────────────────────────────────────────────────┘
```

### Tipos de instancias multi-tenant comunes / Common multi-tenant instance types

| Familia | Optimizado para | Ejemplo AWS | Uso en microservicios |
|---|---|---|---|
| General Purpose | CPU + RAM balanceados | `t3.medium`, `m6i.large` | APIs REST, servicios stateless |
| Compute Optimized | Alta CPU | `c6i.xlarge` | Procesamiento de pedidos, ML inference |
| Memory Optimized | Alto RAM | `r6i.2xlarge` | Caches en memoria, bases de datos |
| Storage Optimized | Alto I/O disco | `i3.large` | Event stores, bases de datos locales |
| Burstable | CPU baseline + burst | `t3.micro` | Servicios con tráfico irregular |

### ShopFast — Asignación multi-tenant típica

```
┌─────────────────────────────────────────────────────────────────────────┐
│              SHOPFAST — MULTI-TENANT VM DEPLOYMENT                      │
│                                                                         │
│  Servicio          Instancia        vCPU   RAM    Motivo                │
│  ─────────────────────────────────────────────────────────────────────  │
│  orders-service    m6i.large          2    8 GB   General purpose       │
│  catalog-service   m6i.xlarge         4   16 GB   Alto tráfico lectura  │
│  payments-service  c6i.large          2    4 GB   Compute (crypto/TLS)  │
│  search-service    r6i.xlarge         4   32 GB   Elasticsearch heap    │
│  notifications-svc t3.medium          2    4 GB   Burstable (eventos)   │
│  shipping-service  m6i.large          2    8 GB   General purpose       │
│                                                                         │
│  Total estimado: ~$650/mes (on-demand, us-east-1)                       │
│  Con Reserved Instances (1 año): ~$420/mes (-35%)                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pros y Contras — Multi-tenant VM

| Dimensión | ✅ Pro | ❌ Contra |
|---|---|---|
| **Costo** | Más barato; el hardware se amortiza entre muchos clientes | Difícil de predecir con workloads variables |
| **Aislamiento** | Hipervisor provee aislamiento sólido (side-channel ataques son raros) | "Noisy neighbor": otro cliente puede afectar tu latencia |
| **Escalabilidad** | Auto Scaling Groups permiten escalar en minutos | Escalar requiere provisionar nuevas VMs (minutos, no segundos) |
| **Gestión** | El proveedor gestiona hardware físico y firmware | Tú gestionas OS, parches, agentes de monitoreo |
| **Compliance** | Suficiente para SOC2, ISO 27001 en la mayoría de casos | No apto para PCI-DSS nivel 1 o HIPAA sin controles adicionales |
| **Flexibilidad** | Amplio catálogo de tamaños y familias | Sobre-aprovisionamiento frecuente (pagas CPU que no usas) |
| **Portabilidad** | Imágenes (AMI/GCE Image) portables entre regiones | Vendor lock-in de la imagen si usas herramientas propietarias |

> **Regla de Pogrebinsky:** El 80% de las cargas de trabajo de microservicios no necesitan dedicated hosts. Multi-tenant VMs con Reserved Instances correctamente dimensionadas son la opción más rentable para la mayoría de startups y empresas medianas.

---

## Dedicated Host and Instances

**Español:** Cuando multi-tenant no es suficiente — ya sea por razones de compliance, rendimiento determinístico o licenciamiento de software — el proveedor de nube ofrece dos variantes de hardware exclusivo: **Dedicated Hosts** y **Dedicated Instances**. La diferencia clave está en el nivel de control que tienes sobre el hardware físico.

**English:** When multi-tenant is not enough — whether due to compliance, deterministic performance, or software licensing reasons — cloud providers offer two variants of exclusive hardware: **Dedicated Hosts** and **Dedicated Instances**. The key difference lies in the level of control you have over the physical hardware.

### Dedicated Instances vs Dedicated Hosts — Diferencia fundamental

```
┌─────────────────────────────────────────────────────────────────────────┐
│         DEDICATED INSTANCES vs DEDICATED HOSTS                          │
│                                                                         │
│  DEDICATED INSTANCES                  DEDICATED HOSTS                   │
│  ─────────────────────────────────    ──────────────────────────────── │
│  Physical Server                      Physical Server                   │
│  ┌──────────────────────────────┐     ┌──────────────────────────────┐  │
│  │  Solo tus VMs                │     │  Solo tus VMs                │  │
│  │  ┌────────┬────────┐         │     │  ┌───────────────────────┐   │  │
│  │  │  VM-A  │  VM-B  │         │     │  │  VM-A   VM-B   VM-C   │   │  │
│  │  └────────┴────────┘         │     │  └───────────────────────┘   │  │
│  │                              │     │                              │  │
│  │  ✅ Hardware exclusivo        │     │  ✅ Hardware exclusivo        │  │
│  │  ❌ No ves el host ID         │     │  ✅ Ves el host ID físico     │  │
│  │  ❌ No controlas placement    │     │  ✅ Controlas placement       │  │
│  │  ❌ No puedes usar BYOL       │     │  ✅ BYOL (SQL Server, etc.)   │  │
│  │  💲 Por VM (como standard)    │     │  💲 Por host completo         │  │
│  └──────────────────────────────┘     └──────────────────────────────┘  │
│                                                                         │
│  Dedicated Instance = hardware exclusivo, sin visibilidad del host      │
│  Dedicated Host = hardware exclusivo + control total del servidor físico│
└─────────────────────────────────────────────────────────────────────────┘
```

### Dedicated Instances — Detalle

**Español:** Las **Dedicated Instances** garantizan que tus VMs corren en hardware que no comparte ningún otro cliente de AWS/GCP/Azure. Sin embargo, el proveedor puede mover tu instancia entre servidores físicos al hacer stop/start. No tienes visibilidad del host ID ni puedes controlar en qué servidor físico específico aterrizan tus VMs.

**English:** **Dedicated Instances** guarantee that your VMs run on hardware not shared with any other AWS/GCP/Azure customer. However, the provider can move your instance between physical servers on stop/start. You have no visibility of the host ID, nor can you control which specific physical server your VMs land on.

**Cuándo usar Dedicated Instances / When to use:**
- Compliance que exige aislamiento físico pero no requiere control del host (p.ej. algunos perfiles de FedRAMP)
- Workloads de microservicios con datos sensibles (salud, finanzas) donde multi-tenant no es aprobado por el equipo de seguridad
- Migración de on-premises donde las políticas internas exigen "no compartir hardware"

### Dedicated Hosts — Detalle

**Español:** Los **Dedicated Hosts** te dan acceso a un servidor físico completo con visibilidad y control total. Puedes ver el `HostId`, elegir en qué host lanzar cada VM, y el host no cambia entre reinicios. Esto habilita el **Bring Your Own License (BYOL)** para software como Windows Server, SQL Server, Oracle o SAP, cuyos contratos de licencia se basan en núcleos físicos o sockets.

**English:** **Dedicated Hosts** give you access to a complete physical server with full visibility and control. You can see the `HostId`, choose which host to launch each VM on, and the host doesn't change between reboots. This enables **Bring Your Own License (BYOL)** for software like Windows Server, SQL Server, Oracle, or SAP, whose license contracts are based on physical cores or sockets.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              DEDICATED HOST — CONFIGURACIÓN TÍPICA (AWS)                │
│                                                                         │
│  aws ec2 allocate-hosts \                                               │
│    --instance-type m6i.large \                                          │
│    --availability-zone us-east-1a \                                     │
│    --auto-placement on \                                                │
│    --quantity 1                                                         │
│                                                                         │
│  → Retorna: host-id = h-0a1b2c3d4e5f67890                               │
│                                                                         │
│  aws ec2 run-instances \                                                │
│    --instance-type m6i.large \                                          │
│    --placement "HostId=h-0a1b2c3d4e5f67890" \                           │
│    --image-id ami-0abcdef1234567890                                     │
│                                                                         │
│  Capacidad del host m6i.metal (ejemplo):                                │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ Total: 128 vCPU / 512 GB RAM                                       │ │
│  │                                                                    │ │
│  │ Puedes colocar:                                                    │ │
│  │  • 32 × m6i.large  (4 vCPU / 16 GB)                               │ │
│  │  • 16 × m6i.xlarge (8 vCPU / 32 GB)                               │ │
│  │  •  8 × m6i.2xlarge (16 vCPU / 64 GB)                             │ │
│  │  •  4 × m6i.4xlarge (32 vCPU / 128 GB)                            │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### Casos de uso por compliance / Compliance use cases

| Regulación | Requisito de aislamiento | Modelo recomendado |
|---|---|---|
| **PCI-DSS Level 1** | Separación de red y computo para datos de tarjetas | Dedicated Host + VPC privada |
| **HIPAA** | PHI no puede residir en hardware compartido (por política) | Dedicated Instances mínimo |
| **FedRAMP High** | Control total del hardware, auditoría del host | Dedicated Host + GovCloud |
| **GDPR** | No requiere dedicated por defecto; depende del DPA | Multi-tenant suficiente en EU regions |
| **SOC 2 Type II** | No requiere dedicated; controles lógicos son suficientes | Multi-tenant con cifrado |
| **BYOL (SQL Server)** | Licencia por socket/core físico | Dedicated Host obligatorio |

### ShopFast — Cuándo migrar a Dedicated

```
┌─────────────────────────────────────────────────────────────────────────┐
│              SHOPFAST — DECISIÓN DE TENENCIA POR SERVICIO               │
│                                                                         │
│  payments-service    → Dedicated Instance                               │
│    Motivo: PCI-DSS, datos de tarjeta, equipo de seguridad               │
│            exige aislamiento físico para el cardholder data env.        │
│                                                                         │
│  fraud-detection     → Dedicated Host                                   │
│    Motivo: Usa Oracle DB con licencia per-socket (BYOL),                │
│            necesita host ID fijo para el license manager.               │
│                                                                         │
│  orders-service      → Multi-tenant (m6i.large)                         │
│  catalog-service     → Multi-tenant (m6i.xlarge)                        │
│  search-service      → Multi-tenant (r6i.xlarge)                        │
│  notifications-svc   → Multi-tenant (t3.medium)                         │
│                                                                         │
│  Regla: solo los servicios que tocan datos regulados o tienen           │
│  licencias basadas en hardware van a Dedicated.                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pros y Contras — Dedicated Hosts & Instances

| Dimensión | ✅ Pro | ❌ Contra |
|---|---|---|
| **Aislamiento** | Aislamiento físico real; ningún otro cliente en el mismo hardware | El hipervisor sigue presente; no equivale a bare metal |
| **Compliance** | Habilita PCI-DSS L1, HIPAA, FedRAMP High | Costo de auditoría se incrementa; más superficie a documentar |
| **BYOL** | Dedicated Host permite usar licencias SQL Server/Oracle por socket | Solo disponible en Dedicated Host, no en Dedicated Instance |
| **Rendimiento** | Sin "noisy neighbor"; rendimiento determinístico y predecible | No necesariamente más rápido que multi-tenant bien dimensionado |
| **Control** | Dedicated Host: ves el host ID, controlas placement, controlas affinity | Dedicated Instance: sin visibilidad del host físico |
| **Costo** | — | Significativamente más caro: Dedicated Host paga el servidor completo aunque no lo llene |
| **Escalabilidad** | Puedes pre-aprovisionar hosts para garantizar capacidad | Capacidad fija por host; escalar requiere nuevos hosts |
| **Gestión** | Misma API/consola que VMs normales | Requiere planificación de capacidad del host para no desperdiciar vCPUs |

### Tabla comparativa final / Final comparison table

| Característica | Multi-tenant VM | Dedicated Instance | Dedicated Host |
|---|---|---|---|
| Hardware compartido con otros | ✅ Sí | ❌ No | ❌ No |
| Visibilidad del host físico | ❌ No | ❌ No | ✅ Sí |
| Control de placement | ❌ No | ❌ No | ✅ Sí |
| BYOL soportado | ❌ No | ❌ No | ✅ Sí |
| Costo relativo | 💲 Bajo | 💲💲 Medio | 💲💲💲 Alto |
| PCI-DSS / HIPAA | ⚠️ Con controles extra | ✅ Generalmente sí | ✅ Sí |
| "Noisy neighbor" risk | ✅ Presente | ❌ Eliminado | ❌ Eliminado |
| Mejor para | La mayoría de microservicios | Datos sensibles sin BYOL | Licencias + compliance estricto |

> **Regla de Pogrebinsky:** Dedicated ≠ más seguro por defecto. Un multi-tenant VM con cifrado de disco, cifrado en tránsito, VPC aislada y IAM correctamente configurado es más seguro que un Dedicated Host mal configurado. Usa Dedicated cuando el compliance lo exija explícitamente o cuando tengas licencias BYOL que lo requieran — no por "sentirte más seguro".
---

## Cloud Virtual Machine, Dedicated Hosts and Instances - Solutions

### Multi-Tenant Virtual Machine Cloud Service

**Amazon Elastic Compute Cloud (EC2) Instances** - Provides on-demand, scalable computing capacity using "virtual servers."

Offers different instance types optimized for different workloads such as General Purpose, Memory, Compute, Storage, and more.

**GCP Compute Engine** - Offers secure and customizable virtual machines running on Google's infrastructure.

You can choose from a variety of virtual machines for different architectures (x86, ARM, etc) and workloads such as "Scale Out", general purpose, ultra-high memory, compute-intensive, accelerator-optimized machines, and more.

**Microsoft Azure Virtual Machines** - Allows creating Linux and Windows virtual machines (VMs) in seconds. Includes several series of virtual machines for testing/development, Economical burstable VMs, general-purpose computing, Optimized for in-memory applications, Compute-optimized virtual machines, Memory and storage optimized virtual machines, High-Performance Computing virtual machines, Storage-optimized virtual machines, GPU-enabled virtual machines and more.

### Dedicated Hosts and Single Tenant Virtual Machine Cloud Services

**AWS Dedicated EC2 Instances** - Dedicated EC2 Instances are regular EC2 instances that run on hardware that's dedicated to a single customer with an AWS account.

Dedicated EC2 Instances that belong to different AWS accounts are physically isolated at a hardware level.

However, EC2 Dedicated Instances might share hardware with other instances from the same AWS account that are not Dedicated Instances.

**Amazon EC2 Dedicated Hosts** - An Amazon EC2 Dedicated Host is a physical server fully dedicated for your use so you can help address corporate compliance requirements. You can find the pricing of Dedicated Hosts here.

**GCP Sole-Tenant Node** - A Sole-tenant node gives you exclusive access to a physical Compute Engine server that is dedicated to hosting only your project's VMs.

Within a sole-tenant node, you can provision multiple VMs on machine types. Alternatively, you can meet security or compliance requirements with workloads that require physical isolation from other workloads or VMs by not sharing the node with any other project.

**Microsoft Azure Dedicate Hosts** - Azure Dedicated Host offers physical servers designed to host one or more Azure virtual machines. These servers are exclusively dedicated to your organization and its workloads, ensuring that the capacity is not shared with any other customers.

This level of isolation at the host level serves to meet compliance requirements effectively.
