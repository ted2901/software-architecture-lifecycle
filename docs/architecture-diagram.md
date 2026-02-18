# Diagrama de Arquitectura del Sistema

## Resumen Ejecutivo

Este documento describe la arquitectura del sistema basada en microservicios, diseñada para escalabilidad, confiabilidad y mantenibilidad. La arquitectura sigue principios de Domain-Driven Design y utiliza patrones cloud-native.

---

## 1. Arquitectura de Alto Nivel

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CLIENTE (Web/Mobile)                         │
└────────────────────────────┬────────────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   API GATEWAY (Kong/Nginx)                           │
│        ┌─────────────┬──────────────┬──────────────┐                │
│        │ Rate Control│ Auth/JWT     │ Load Balance │                │
│        │ CORS        │ Logging      │ SSL/TLS      │                │
│        └──────────────────────────────────────────┘                │
└────────────┬─────────────────┬──────────────────────┬────────────────┘
             │                 │                      │
    ┌────────▼─────┐  ┌────────▼──────┐  ┌──────────▼──────┐
    │ User Service │  │Product Service│  │Order Service    │
    ├──────────────┤  ├───────────────┤  ├─────────────────┤
    │• Auth        │  │• CRUD Product │  │• Orders         │
    │• Profiles    │  │• Categories   │  │• Tracking       │
    │• Roles       │  │• Inventory    │  │• Notifications  │
    └────────┬─────┘  └───────┬───────┘  └────────┬────────┘
             │                │                   │
    ┌────────▼────────────────▼───────────────────▼─────────────┐
    │            MESSAGE BROKER (RabbitMQ/Kafka)                 │
    │   ┌────────────┬──────────────┬────────────────────┐       │
    │   │User Events │Product Events│Order Events        │       │
    │   └────────────┴──────────────┴────────────────────┘       │
    └────────────────────────┬────────────────────────────────────┘
             │
    ┌────────▼──────────────────────────────────┐
    │      DATABASES                            │
    │  ┌─────────┐  ┌──────────┐  ┌─────────┐  │
    │  │User DB  │  │Product DB│  │Order DB │  │
    │  │(MySQL)  │  │ (MySQL)  │  │(MySQL)  │  │
    │  └─────────┘  └──────────┘  └─────────┘  │
    └───────────────────────────────────────────┘

    ┌──────────────────────────────────┐
    │   EXTERNAL SERVICES              │
    │  ┌──────────┬──────────────────┐ │
    │  │Payment   │Email/SMS         │ │
    │  │Provider  │Notification      │ │
    │  └──────────┴──────────────────┘ │
    └──────────────────────────────────┘

    ┌──────────────────────────────────┐
    │   MONITORING & LOGGING           │
    │  ┌──────────┬──────────────────┐ │
    │  │Prometheus│ELK Stack         │ │
    │  │Grafana   │Distributed Trace │ │
    │  └──────────┴──────────────────┘ │
    └──────────────────────────────────┘
```

---

## 2. Componentes Principales

### 2.1 API Gateway
- **Responsabilidad:** Punto de entrada único para todas las solicitudes
- **Funcionalidades:**
  - Enrutamiento inteligente de solicitudes
  - Autenticación y autorización (JWT)
  - Rate limiting y throttling
  - Transformación de solicitudes/respuestas
  - Logging centralizado
- **Tecnología:** Kong o Nginx
- **Puerto:** 8080 (HTTP), 8443 (HTTPS)

### 2.2 User Service
- **Puerto:** 3001
- **Base de Datos:** MySQL (user_db)
- **Responsabilidades:**
  - Gestión de autenticación
  - Perfiles de usuario
  - Control de roles y permisos
- **Endpoints:**
  - POST /auth/register
  - POST /auth/login
  - GET /users/{id}
  - PUT /users/{id}
- **Eventos Publicados:**
  - user.registered
  - user.updated
  - user.deleted

### 2.3 Product Service
- **Puerto:** 3002
- **Base de Datos:** MySQL (product_db)
- **Responsabilidades:**
  - Gestión de catálogo de productos
  - Administración de categorías
  - Control de inventario
- **Endpoints:**
  - GET /products
  - POST /products
  - PUT /products/{id}
  - DELETE /products/{id}
- **Eventos Publicados:**
  - product.created
  - product.updated
  - product.stock_updated
  - product.deleted

### 2.4 Order Service
- **Puerto:** 3003
- **Base de Datos:** MySQL (order_db)
- **Responsabilidades:**
  - Creación y gestión de órdenes
  - Seguimiento de envíos
  - Generación de notificaciones
- **Endpoints:**
  - GET /orders
  - POST /orders
  - GET /orders/{id}
  - PUT /orders/{id}/status
- **Eventos Publicados:**
  - order.created
  - order.payment_received
  - order.shipped
  - order.delivered
  - order.cancelled
- **Eventos Consumidos:**
  - product.stock_updated

---

## 3. Flujos de Comunicación

### 3.1 Flujo de Crear Orden

```
┌─────────────┐
│   Cliente   │
└──────┬──────┘
       │ POST /orders
       ▼
┌───────────────────┐      ┌──────────────────┐
│  API Gateway      │─────▶│  Order Service   │
└───────────────────┘      └────────┬─────────┘
                                    │
                        1. Valida stock
                           ▼
                    ┌──────────────────┐
                    │ Product Service  │
                    └──────┬───────────┘
                           │ OK / FAIL
                           ▼
                    ┌──────────────────┐
                    │  Order Service   │
                    │ (Crea Order DB)  │
                    └────────┬─────────┘
                             │
                    2. Publica evento
                      (order.created)
                             ▼
                    ┌──────────────────┐
                    │ Message Broker   │
                    │  (RabbitMQ)      │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │Email     │  │Payment   │  │Inventory│
        │Service   │  │Service   │  │Service   │
        └──────────┘  └──────────┘  └──────────┘
```

### 3.2 Flujo de Actualización de Stock

```
┌──────────────────┐
│ Product Service  │
│ (Stock update)   │
└────────┬─────────┘
         │
    Publica evento
    (product.stock_updated)
         ▼
   ┌──────────────┐
   │ Message Broker
   └───────┬──────┘
           │
     ┌─────┴─────┐
     ▼           ▼
┌─────────┐  ┌───────────┐
│Order    │  │Notif      │
│Service  │  │Service    │
└─────────┘  └───────────┘
   (Si hay órdenes  (Alerta sobre
    pendientes)      stock bajo)
```

---

## 4. Patrón SAGA para Transacciones Distribuidas

Para garantizar consistencia en operaciones que involucran múltiples servicios:

```
CREAR ORDEN SAGA:
┌───────────────────┐
│ OrchestrationSaga │
│  (Order Service)  │
└────────┬──────────┘
         │
    Step 1: Reservar Stock
         ▼
    ┌─────────────────┐      ┌──────────────────┐
    │ Product Service │─────▶│ Intenta reservar │
    └─────────────────┘      │ stock            │
                             └────────┬─────────┘
                                      │ OK
         ┌────────────────────────────┘
         │
    Step 2: Procesar Pago
         ▼
    ┌─────────────────┐      ┌──────────────────┐
    │ Payment Service │─────▶│ Procesa pago     │
    └─────────────────┘      └────────┬─────────┘
                                      │ OK
         ┌────────────────────────────┘
         │
    Step 3: Crear Orden
         ▼
    ┌─────────────────┐      ┌──────────────────┐
    │ Order Service   │─────▶│ Crea registro    │
    └─────────────────┘      │ y notifica       │
                             └──────────────────┘

EN CASO DE FALLO (Compensating Transaction):
- Si pago falla: Liberar stock reservado
- Si orden no se crea: Reembolsar pago
```

---

## 5. Datos por Servicio (Database per Service)

```yaml
User Service:
  Database: mysql://user-db:3306/users
  Tablas:
    - users (id, email, password_hash, full_name, role, created_at)
    - roles (id, name, permissions)
    - user_roles (user_id, role_id)

Product Service:
  Database: mysql://product-db:3306/products
  Tablas:
    - products (id, name, description, price, category_id, stock)
    - categories (id, name, description)
    - reviews (id, product_id, user_id, rating, comment)

Order Service:
  Database: mysql://order-db:3306/orders
  Tablas:
    - orders (id, user_id, status, total, created_at)
    - order_items (id, order_id, product_id, quantity, price)
    - tracking (id, order_id, carrier, tracking_number)
```

---

## 6. Despliegue (Kubernetes)

```yaml
# Estructura de recursos K8s
├── namespaces/
│   └── production/
├── configmaps/
│   └── app-config
├── secrets/
│   └── db-credentials
├── deployments/
│   ├── api-gateway
│   ├── user-service
│   ├── product-service
│   └── order-service
├── services/
│   ├── api-gateway-svc
│   ├── user-service-svc
│   ├── product-service-svc
│   └── order-service-svc
├── statefulsets/
│   ├── mysql-primary
│   ├── rabbitmq-cluster
│   └── redis-cache
├── ingress/
│   └── api-ingress
└── hpa/
    └── service-autoscaling
```

### Requisitos de Recursos por Servicio
```
API Gateway:
  CPU: 200m - 500m
  Memory: 256Mi - 512Mi
  Replicas: 2-5 (HPA)

Microservices:
  CPU: 100m - 500m
  Memory: 256Mi - 1Gi
  Replicas: 2-3 (HPA)

Databases:
  CPU: 500m - 2000m
  Memory: 1Gi - 4Gi
```

---

## 7. Seguridad

### 7.1 Capas de Seguridad

```
Internet
   │
   ▼
┌──────────────────┐
│ WAF (ModSecurity)│
└────────┬─────────┘
         ▼
┌──────────────────────┐
│ TLS/SSL Termination  │
│ (Nginx Ingress)      │
└────────┬─────────────┘
         ▼
┌──────────────────────┐
│ API Gateway          │
│ (Rate Limit, Auth)   │
└────────┬─────────────┘
         ▼
┌──────────────────────┐
│ Microservices        │
│ (TLS Internal)       │
└────────┬─────────────┘
         ▼
┌──────────────────────┐
│ Database Encryption  │
│ (SSL + Encryption)   │
└──────────────────────┘
```

### 7.2 Autenticación y Autorización

- **JWT Tokens:** Acceso a APIs
  - Header: Authorization: Bearer {token}
  - Tiempo de expiración: 1 hora
  - Refresh token: 7 días

- **API Keys:** Para integraciones externas
  - Headers: X-API-Key

- **mTLS:** Comunicación entre servicios

---

## 8. Monitoreo y Observabilidad

### 8.1 Métricas (Prometheus)

```
Métricas de Negocio:
- http_requests_total
- user_registrations_total
- orders_created_total
- orders_completed_total
- payment_failures_total

Métricas de Sistema:
- container_cpu_usage_seconds_total
- container_memory_usage_bytes
- http_request_duration_seconds
- database_connection_pool_usage
```

### 8.2 Logs (ELK Stack)

```
Elasticsearch:
  - Índices por fecha y servicio
  - Retención: 30 días

Logstash:
  - Parseado de logs JSON
  - Enriquecimiento con metadatos
  - Filtrado de logs sensibles

Kibana:
  - Dashboards por aplicación
  - Alertas basadas en patrones
```

### 8.3 Trazabilidad Distribuida (Jaeger)

```
Flujo: Cliente → API Gateway → Servicio A → DB
       │         │               │          │
       └─────────┴───────────────┴──────────┘
         (Trace ID propagado a través)
```

---

## 9. Escalabilidad y Disponibilidad

### Autoscaling Horizontal
```
HPA Configuration:
  - CPU Target: 70%
  - Memory Target: 80%
  - Min Replicas: 2
  - Max Replicas: 10
  - Scale-up delay: 30s
  - Scale-down delay: 300s
```

### Resiliencia
```
Circuit Breaker:
  - Timeout: 5s
  - Failure threshold: 50% (10 requests)
  - Success threshold: 2 requests
  - Half-open state: Permite 3 requests

Retry Policy:
  - Max retries: 3
  - Backoff exponencial: 100ms, 200ms, 400ms
  - Jitter: ±10%

Bulkhead:
  - Thread pool por servicio
  - Max threads: 50
  - Queue size: 100
```

---

## 10. Disaster Recovery

```
RPO (Recovery Point Objective): 1 hora
RTO (Recovery Time Objective): 15 minutos

Backup Strategy:
  - Backups automáticos cada 6 horas
  - Retención: 30 días
  - Replicación geográfica: Región secundaria
```

---

**Última actualización:** 2025-02-18
**Autor:** Arquitecto de Software
**Estado:** Vigente
**Versión:** 1.0
