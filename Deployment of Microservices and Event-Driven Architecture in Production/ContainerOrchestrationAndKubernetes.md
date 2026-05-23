# Container Orchestration and Kubernetes for Microservices Architecture

**Español:** Una vez que adoptamos contenedores para nuestros microservicios, resolvimos el problema del empaquetado y la paridad de entornos. Pero surge inmediatamente el siguiente problema: ¿cómo operamos decenas o cientos de contenedores en múltiples servidores en producción? La respuesta es la **orquestación de contenedores**, y Kubernetes es el estándar de la industria para implementarla.

**English:** Once we adopted containers for our microservices, we solved the packaging and environment parity problem. But the next problem immediately arises: how do we operate dozens or hundreds of containers across multiple servers in production? The answer is **container orchestration**, and Kubernetes is the industry standard for implementing it.

---

## Introduction to Container Orchestration

**Español:** La **orquestación de contenedores** es la automatización del despliegue, gestión, escalado, networking y ciclo de vida de contenedores en producción. Sin un orquestador, un ingeniero tendría que hacer manualmente todo lo que hace el orquestador: decidir en qué servidor corre cada contenedor, reiniciar los que fallan, escalar ante picos de tráfico, balancear la carga entre réplicas, y gestionar actualizaciones sin downtime.

**English:** **Container orchestration** is the automation of deployment, management, scaling, networking, and lifecycle of containers in production. Without an orchestrator, an engineer would have to manually do everything the orchestrator does: decide which server each container runs on, restart failing ones, scale during traffic spikes, load-balance between replicas, and manage updates without downtime.

### El problema que resuelve / The problem it solves

```
┌─────────────────────────────────────────────────────────────────────────┐
│         SIN ORQUESTACIÓN — SHOPFAST EN PRODUCCIÓN (BLACK FRIDAY)        │
│                                                                         │
│  23:58 — Tráfico 10× normal. El equipo de Ops recibe alertas:           │
│                                                                         │
│  🔴 orders-service en server-3 murió (OOMKilled)                        │
│     → ¿En qué servidor tengo capacidad para relanzarlo?                 │
│     → ¿Cuánta RAM y CPU necesita?                                       │
│     → ¿Cómo lo conecto al load balancer?                                │
│                                                                         │
│  🔴 catalog-service necesita 5 réplicas más (tráfico ×10)               │
│     → ¿Dónde tengo espacio? Server-1 tiene 2GB libres, server-5 tiene   │
│        8GB... ¿cómo distribuyo las 5 réplicas manualmente?              │
│                                                                         │
│  🔴 Nueva versión de payments-service lista para deploy                 │
│     → ¿Cómo hago rolling update sin downtime a las 00:00?               │
│     → ¿Cómo hago rollback si falla?                                     │
│                                                                         │
│  CON ORQUESTADOR (Kubernetes):                                          │
│  → orders-service reiniciado automáticamente en 10 segundos ✅          │
│  → catalog-service escalado a 8 réplicas en 30 segundos ✅              │
│  → payments deploy: rolling update automático, rollback con 1 cmd ✅    │
└─────────────────────────────────────────────────────────────────────────┘
```

### Responsabilidades del orquestador / Orchestrator responsibilities

| Responsabilidad | Sin orquestador | Con orquestador (Kubernetes) |
|---|---|---|
| **Deployment automation** | Manual: scripts frágiles, coordinar réplicas a mano | Automático: `kubectl apply` / ArgoCD — zero-downtime, rollback en 1 comando |
| **Resource allocation** | Manual: sobreprovisionar por miedo a quedarse sin recursos | Automático: requests/limits por Pod; Scheduler asigna según disponibilidad real |
| **Health monitoring** | Manual: monitoreo externo con alertas reactivas | Integrado: liveness + readiness probes; K8s actúa antes de que el usuario note el fallo |
| **Self-healing** | Manual: detectar caída, SSH, relanzar | Automático: reinicio en <30s, sin intervención |
| **Bin-packing** | Manual: decidir qué corre en qué servidor (ineficiente) | Automático: Scheduler empaqueta Pods maximizando uso de CPU/RAM por Node |
| **Load balancing** | Manual: configurar Nginx/HAProxy | Automático: Services + kube-proxy balancean entre réplicas |
| **Scaling services** | Manual: aprovisionar y conectar réplicas | Automático: HPA basado en CPU/RPS/métricas custom |
| **Container discovery** | Manual: DNS/registro estático hardcodeado | Automático: CoreDNS resuelve nombres de Service dentro del clúster |
| **Network connectivity** | Manual: configurar red, firewall y routing entre hosts | Automático: CNI (Cilium/Calico) crea red overlay; cada Pod tiene IP única en el clúster |
| **Rolling updates** | Manual: coordinar con scripts | Automático: RollingUpdate strategy, maxSurge/maxUnavailable configurables |
| **Config/Secrets** | Manual: gestionar por servidor | Centralizado: ConfigMaps + Secrets montados como env vars o volúmenes |

---

## Container Orchestration Architecture - Kubernetes Example

**Español:** **Kubernetes** (también llamado K8s) es el sistema de orquestación de contenedores open-source originalmente desarrollado por Google, basado en su sistema interno Borg. Fue donado a la CNCF (Cloud Native Computing Foundation) en 2014 y es hoy el estándar de facto para orquestación de contenedores en producción. Kubernetes funciona con una arquitectura **master-worker**: un plano de control (control plane) gestiona el clúster, y los nodos worker ejecutan los contenedores.

**English:** **Kubernetes** (also called K8s) is the open-source container orchestration system originally developed by Google, based on their internal Borg system. It was donated to the CNCF (Cloud Native Computing Foundation) in 2014 and is today the de facto standard for container orchestration in production. Kubernetes operates with a **master-worker** architecture: a control plane manages the cluster, and worker nodes run the containers.

### Arquitectura del clúster / Cluster architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    KUBERNETES CLUSTER ARCHITECTURE                      │
│                                                                         │
│  ┌─────────────────────────── CONTROL PLANE ─────────────────────────┐ │
│  │                                                                    │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │ │
│  │  │  API Server  │  │    etcd      │  │  Controller Manager      │ │ │
│  │  │              │  │              │  │                          │ │ │
│  │  │ Entry point  │  │ Distributed  │  │ Node Controller          │ │ │
│  │  │ for all K8s  │  │ key-value    │  │ ReplicaSet Controller    │ │ │
│  │  │ operations   │  │ store        │  │ Deployment Controller    │ │ │
│  │  │              │  │ (cluster     │  │ Service Controller       │ │ │
│  │  │ kubectl →    │  │  state)      │  │ HPA Controller           │ │ │
│  │  └──────────────┘  └──────────────┘  └──────────────────────────┘ │ │
│  │                                                                    │ │
│  │  ┌──────────────────────────────────────────────────────────────┐ │ │
│  │  │                      Scheduler                               │ │ │
│  │  │  Asigna Pods a Nodes según: recursos disponibles,            │ │ │
│  │  │  affinity/anti-affinity rules, taints/tolerations            │ │ │
│  │  └──────────────────────────────────────────────────────────────┘ │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                              │  API calls                               │
│         ┌────────────────────┼────────────────────┐                    │
│         ▼                    ▼                    ▼                    │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                │
│  │  WORKER     │    │  WORKER     │    │  WORKER     │                │
│  │  NODE 1     │    │  NODE 2     │    │  NODE 3     │                │
│  │             │    │             │    │             │                │
│  │  kubelet    │    │  kubelet    │    │  kubelet    │                │
│  │  kube-proxy │    │  kube-proxy │    │  kube-proxy │                │
│  │  container  │    │  container  │    │  container  │                │
│  │  runtime    │    │  runtime    │    │  runtime    │                │
│  │             │    │             │    │             │                │
│  │ [Pod][Pod]  │    │ [Pod][Pod]  │    │ [Pod][Pod]  │                │
│  │ [Pod]       │    │ [Pod][Pod]  │    │ [Pod]       │                │
│  └─────────────┘    └─────────────┘    └─────────────┘                │
└─────────────────────────────────────────────────────────────────────────┘
```

### Componentes del Control Plane

**Español:** El **Control Plane** es el cerebro del clúster. Todos sus componentes trabajan juntos para mantener el **estado deseado** (desired state) del clúster. El principio fundamental de Kubernetes es declarativo: tú describes lo que quieres (YAML), y Kubernetes trabaja continuamente para que la realidad coincida con esa descripción.

**English:** The **Control Plane** is the brain of the cluster. All its components work together to maintain the **desired state** of the cluster. The fundamental principle of Kubernetes is declarative: you describe what you want (YAML), and Kubernetes continuously works to make reality match that description.

| Componente | Función |
|---|---|
| **API Server** | Punto de entrada único para todas las operaciones (kubectl, CI/CD, HPA). Valida y persiste objetos en etcd. |
| **etcd** | Base de datos distribuida clave-valor que almacena TODO el estado del clúster. Es la fuente de verdad de K8s. |
| **Scheduler** | Decide en qué Node se ejecuta cada Pod nuevo, considerando recursos disponibles, afinidades y restricciones. |
| **Controller Manager** | Ejecuta los controladores (loops de reconciliación) que mantienen el estado deseado: ReplicaSet, Deployment, Node, Service, HPA. |
| **Cloud Controller Manager** | Integra K8s con la API del proveedor de nube (AWS/GCP/Azure) para crear load balancers, volúmenes, etc. |

### Componentes del Worker Node

| Componente | Función |
|---|---|
| **kubelet** | Agente que corre en cada Node. Recibe instrucciones del API Server y gestiona los Pods en su Node. |
| **kube-proxy** | Gestiona las reglas de red (iptables/eBPF) para que los Services funcionen correctamente. |
| **Container Runtime** | Ejecuta los contenedores (containerd, CRI-O). Docker ya no es el runtime por defecto desde K8s 1.24. |

### Objetos fundamentales de Kubernetes / Core Kubernetes objects

```
┌─────────────────────────────────────────────────────────────────────────┐
│              JERARQUÍA DE OBJETOS EN KUBERNETES                         │
│                                                                         │
│  Deployment (describe el estado deseado)                                │
│  └── ReplicaSet (garantiza N réplicas del Pod)                          │
│      └── Pod (unidad mínima de despliegue)                              │
│          └── Container(s) (1 o más contenedores)                        │
│              └── Container Image (orders-service:v2.3.1)                │
│                                                                         │
│  Service (expone Pods con IP/DNS estable y load balancing)              │
│  └── apunta a Pods mediante label selectors                             │
│                                                                         │
│  ConfigMap   (configuración no sensible: URLs, feature flags)           │
│  Secret      (configuración sensible: passwords, API keys, certs)       │
│  Namespace   (aislamiento lógico de recursos dentro del clúster)        │
│  Ingress     (routing HTTP/HTTPS desde exterior al clúster)             │
│  PersistentVolumeClaim (almacenamiento persistente para Pods)           │
│  HorizontalPodAutoscaler (escalado automático basado en métricas)       │
└─────────────────────────────────────────────────────────────────────────┘
```

#### Pod — La unidad mínima

**Español:** Un **Pod** es la unidad mínima de despliegue en Kubernetes. Contiene uno o más contenedores que comparten la misma red (misma IP, mismo `localhost`) y el mismo almacenamiento (volúmenes montados). En la práctica, la mayoría de Pods tienen un solo contenedor principal más, opcionalmente, contenedores auxiliares (**sidecars**: logging, tracing, proxy).

**English:** A **Pod** is the minimal deployment unit in Kubernetes. It contains one or more containers that share the same network (same IP, same `localhost`) and the same storage (mounted volumes). In practice, most Pods have a single main container plus, optionally, auxiliary containers (**sidecars**: logging, tracing, proxy).

```yaml
# Ejemplo: Pod de orders-service con sidecar de logging
apiVersion: v1
kind: Pod
metadata:
  name: orders-service-7d9f8b-xk2p
  labels:
    app: orders-service
    version: v2.3.1
spec:
  containers:
  - name: orders-service          # contenedor principal
    image: shopfast/orders:v2.3.1
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /actuator/health/liveness
        port: 8080
      initialDelaySeconds: 30
    readinessProbe:
      httpGet:
        path: /actuator/health/readiness
        port: 8080
  - name: fluent-bit              # sidecar de logging
    image: fluent/fluent-bit:3.0
```

#### Deployment — Gestión declarativa de réplicas

**Español:** Un **Deployment** describe el estado deseado de un conjunto de Pods: cuántas réplicas quiero, qué imagen usar, y cómo hacer actualizaciones. Kubernetes garantiza continuamente que la realidad coincida con esta descripción (reconciliation loop).

**English:** A **Deployment** describes the desired state of a set of Pods: how many replicas I want, which image to use, and how to perform updates. Kubernetes continuously guarantees that reality matches this description (reconciliation loop).

```yaml
# Ejemplo: Deployment de orders-service con rolling update
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders-service
  namespace: shopfast-prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: orders-service
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # máx 1 Pod extra durante el update
      maxUnavailable: 0  # 0 Pods fuera de servicio (zero-downtime)
  template:
    metadata:
      labels:
        app: orders-service
    spec:
      containers:
      - name: orders-service
        image: shopfast/orders:v2.3.1
```

#### Service — Networking estable entre microservicios

**Español:** Un **Service** expone un conjunto de Pods con una IP virtual estable y un nombre DNS. Mientras que los Pods tienen IPs efímeras que cambian al reiniciarse, el Service siempre tiene la misma IP/DNS. Los microservicios se comunican entre sí usando el nombre del Service como hostname.

**English:** A **Service** exposes a set of Pods with a stable virtual IP and a DNS name. While Pods have ephemeral IPs that change on restart, the Service always has the same IP/DNS. Microservices communicate with each other using the Service name as the hostname.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              SERVICE DISCOVERY EN KUBERNETES                            │
│                                                                         │
│  orders-service llama a payments-service:                               │
│                                                                         │
│  http://payments-service.shopfast-prod.svc.cluster.local:8080/charge    │
│         └──────────────────────────────────────────┘                   │
│         DNS resuelto por CoreDNS → ClusterIP del Service                │
│                                                                         │
│  payments-service (ClusterIP: 10.96.45.23)                              │
│  ├── Pod payments-7f9d-abc1  (10.244.1.5)  ← kube-proxy balancea        │
│  ├── Pod payments-7f9d-abc2  (10.244.2.8)  ← entre las 3 réplicas       │
│  └── Pod payments-7f9d-abc3  (10.244.3.2)                               │
│                                                                         │
│  Tipos de Service:                                                      │
│  • ClusterIP   — solo accesible dentro del clúster (por defecto)        │
│  • NodePort    — expone un puerto en cada Node (dev/testing)            │
│  • LoadBalancer— crea un LB en el proveedor de nube (producción)        │
│  • ExternalName— alias DNS para servicios externos al clúster           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Escalado automático / Automatic Scaling

**Español:** Kubernetes ofrece tres mecanismos de escalado automático:

**English:** Kubernetes offers three automatic scaling mechanisms:

```
┌─────────────────────────────────────────────────────────────────────────┐
│              TRES TIPOS DE ESCALADO EN KUBERNETES                       │
│                                                                         │
│  1. HPA — Horizontal Pod Autoscaler                                     │
│     Escala el NÚMERO DE PODS basándose en métricas (CPU, RPS, custom)   │
│                                                                         │
│     catalog-service: 3 réplicas → tráfico ×5 → 15 réplicas             │
│     [Pod][Pod][Pod] ──────────────────────────────────────────────►     │
│     [Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod]   │
│                                                                         │
│  2. VPA — Vertical Pod Autoscaler                                       │
│     Ajusta los RECURSOS (CPU/RAM requests/limits) de los Pods           │
│     Útil para workloads que no escalan bien horizontalmente             │
│                                                                         │
│  3. Cluster Autoscaler                                                  │
│     Escala el NÚMERO DE NODES en el clúster                             │
│     Si no hay nodos con capacidad para nuevos Pods → agrega Nodes       │
│     Si hay Nodes infrautilizados → los elimina y migra los Pods         │
│                                                                         │
│  SHOPFAST Black Friday — escalado en cascada:                           │
│  Tráfico ×10 → HPA escala Pods → Nodes llenos → Cluster Autoscaler     │
│  agrega 5 Nodes en AWS en ~3 minutos → HPA coloca 50 nuevos Pods       │
└─────────────────────────────────────────────────────────────────────────┘
```

### Self-healing — Reconciliation loop

**Español:** El mecanismo más poderoso de Kubernetes es el **bucle de reconciliación**: continuamente compara el estado deseado (lo que declaraste en el YAML) contra el estado real (lo que corre en el clúster) y actúa para cerrar la brecha. Este loop corre decenas de veces por segundo.

**English:** Kubernetes' most powerful mechanism is the **reconciliation loop**: it continuously compares desired state (what you declared in YAML) against actual state (what is running in the cluster) and acts to close the gap. This loop runs dozens of times per second.

```
┌─────────────────────────────────────────────────────────────────────────┐
│              RECONCILIATION LOOP — SELF-HEALING                         │
│                                                                         │
│  Estado deseado (etcd):  orders-service → 3 réplicas                   │
│                                                                         │
│  T=0:  Estado real: [Pod1 ✅][Pod2 ✅][Pod3 ✅]  → OK, nada que hacer  │
│                                                                         │
│  T=5m: Pod3 crashea (OOMKilled)                                         │
│        Estado real: [Pod1 ✅][Pod2 ✅][Pod3 ❌]                          │
│        Controller Manager: "Faltan réplicas. Crear Pod nuevo."          │
│                                                                         │
│  T=5m+15s: [Pod1 ✅][Pod2 ✅][Pod4 ✅]  → Reconciliado ✅               │
│                                                                         │
│  Sin intervención humana. Sin alertas de on-call a las 3 AM.            │
└─────────────────────────────────────────────────────────────────────────┘
```

### ShopFast — Deployment completo en Kubernetes

```
┌─────────────────────────────────────────────────────────────────────────┐
│              SHOPFAST — ARQUITECTURA COMPLETA EN K8S                    │
│                                                                         │
│  Internet                                                               │
│     │                                                                   │
│     ▼                                                                   │
│  [Ingress Controller (nginx/traefik)]                                   │
│     │  /api/orders → orders-service                                     │
│     │  /api/catalog → catalog-service                                   │
│     │  /api/payments → payments-service                                 │
│     │                                                                   │
│     ▼                                                                   │
│  Namespace: shopfast-prod                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  [orders-svc ×3] ──→ [payments-svc ×2] ──→ [fraud-svc ×2]      │   │
│  │       │                                                          │   │
│  │       ▼                                                          │   │
│  │  [catalog-svc ×5] ──→ [search-svc ×3]                           │   │
│  │       │                                                          │   │
│  │       ▼                                                          │   │
│  │  [notifications-svc ×2] [shipping-svc ×2] [inventory-svc ×3]   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  Namespace: shopfast-infra                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  [kafka ×3 brokers]  [redis ×3]  [prometheus]  [grafana]        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  HPA configurado en: orders, catalog, search (escala por CPU/RPS)       │
│  Cluster Autoscaler: escala de 5 a 20 Nodes en Black Friday             │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pros y Contras de Kubernetes

| Dimensión | ✅ Pro | ❌ Contra |
|---|---|---|
| **Self-healing** | Reinicio automático de Pods caídos, sin intervención humana | — |
| **Escalado** | HPA + Cluster Autoscaler: escala Pods y Nodes automáticamente | Latencia de 1-3 min para provisionar nuevos Nodes en la nube |
| **Rolling updates** | Zero-downtime deployments + rollback con un comando | Requiere readiness probes bien configuradas para funcionar correctamente |
| **Service discovery** | DNS automático para todos los Services del clúster | Debugging de red en K8s puede ser complejo (CNI, kube-proxy, DNS) |
| **Portabilidad** | Corre igual en AWS EKS, GCP GKE, Azure AKS, on-prem | Diferencias entre distribuciones managed (EKS vs GKE vs AKS) |
| **Ecosistema** | CNCF: Helm, Istio, ArgoCD, Prometheus, Fluentd, etc. | Ecosistema enorme → parálisis de elecciones para equipos nuevos |
| **Resource efficiency** | Bin-packing: maximiza uso de CPU/RAM en cada Node | Overhead del Control Plane (~3 instancias para HA): ~$200-400/mes |
| **Curva de aprendizaje** | — | Alta: YAML, kubectl, Pods, Services, Deployments, RBAC, Networking... |
| **Complejidad operacional** | — | Gestionar el propio clúster (updates, etcd backups, certs) es trabajo full-time |
| **Managed K8s** | EKS/GKE/AKS reducen la complejidad operacional significativamente | Costo adicional del servicio managed (~$0.10/hora en EKS) |

> **Regla de Pogrebinsky:** Kubernetes no es para todos. Un equipo de 3 personas con 5 microservicios probablemente está mejor con AWS ECS, Google Cloud Run o Railway. Kubernetes se justifica cuando tienes suficiente escala y complejidad: muchos servicios, múltiples equipos, necesidad de control granular sobre el scheduling, o requisitos de compliance que exigen on-premises. La complejidad de K8s solo se justifica cuando los beneficios superan claramente ese costo.

### Managed Kubernetes — Opciones principales

| Proveedor | Servicio | Control Plane | Destacado |
|---|---|---|---|
| **AWS** | EKS (Elastic Kubernetes Service) | Managed (pago por hora) | Integración nativa con IAM, ALB, EBS, ECR |
| **Google Cloud** | GKE (Google Kubernetes Engine) | Managed (Autopilot: gratis) | El más maduro; Autopilot gestiona Nodes también |
| **Microsoft Azure** | AKS (Azure Kubernetes Service) | Managed (gratis) | Integración con Azure AD, ACR, Azure Monitor |
| **DigitalOcean** | DOKS | Managed (gratis) | Más simple, ideal para equipos pequeños |
| **On-premises** | kubeadm / Rancher / OpenShift | Self-managed | Control total; máxima complejidad operacional |

### Alternativas a Kubernetes / Kubernetes alternatives

| Herramienta | Mejor para | Complejidad | Cuándo elegirlo |
|---|---|---|---|
| **Amazon ECS** | Workloads en AWS sin querer K8s | Baja | Equipo pequeño, solo AWS, sin necesidad de portabilidad |
| **Google Cloud Run** | Contenedores serverless (scale-to-zero) | Muy baja | Servicios HTTP stateless con tráfico variable |
| **Azure Container Apps** | Microservicios en Azure (Dapr integrado) | Baja | Ecosistema Azure, EDA con Dapr |
| **Docker Swarm** | K8s simplificado | Muy baja | Equipos con conocimiento de Docker, escala pequeña |
| **Nomad (HashiCorp)** | Orquestación mixta (containers + VMs + JARs) | Media | Infraestructura heterogénea, ya usan Vault/Consul |
| **Kubernetes** | Escala grande, multi-cloud, control granular | Alta | Muchos servicios, múltiples equipos, requisitos complejos |