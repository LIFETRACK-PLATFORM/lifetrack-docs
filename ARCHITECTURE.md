# 🏗️ Arquitectura General — LifeTrack OS

> Para ver el contexto completo ir al [README principal](./README.md)

---

## Principios de Diseño

| Principio | Cómo se aplica |
|-----------|---------------|
| Arquitectura Hexagonal | `domain/` no importa NestJS, Prisma ni NATS. Solo entidades, puertos y reglas de negocio. |
| DB per Service | Cada microservicio tiene su propia base de datos. Nadie accede tablas ajenas directamente. |
| API Gateway único | Ningún frontend llama directo a microservicios. Todo pasa por el Gateway. |
| Contratos versionados | Protobuf `.proto` y schemas de eventos viven en `lifetrack-contracts`. |
| Outbox Pattern | Eventos se guardan en la misma transacción del cambio. Worker los publica a NATS. |
| Correlation ID | Todo request lleva un ID único que viaja por gRPC metadata y eventos NATS. |
| Módulos opt-in | Cada módulo (`tasks`, `rehab`, `finance`, etc.) se activa por usuario en `user_modules` (user-service). Ningún módulo nuevo se habilita solo. |

---

## Reglas de Comunicación

| Canal | Cuándo | Ejemplo |
|-------|--------|---------|
| REST/HTTPS | Entrada desde clientes web/móvil hacia Gateway | `POST /tasks`, `GET /spaces` |
| gRPC + Protobuf | Comunicación interna síncrona entre servicios | `gateway → task-service.CreateTask` |
| NATS JetStream | Eventos de dominio asíncronos con persistencia | `task.created.v1`, `vault.secret_accessed.v1` |
| Redis | Cache, rate limit, locks, sesiones efímeras | Rate limit por usuario, lock de scheduler |
| WebSocket (futuro) | Notificaciones en tiempo real al frontend | Estado de tareas en vivo, alerts |

---

## Mapa de Microservicios

```mermaid
graph LR
    GW[api-gateway] --> AUTH[auth-service]
    GW --> USER[user-service]
    GW --> REHAB[rehab-service]
    GW --> VAULT[vault-service]
    GW --> FIN[finance-service]
    GW --> MEDIA[media-service]
    GW --> TASK[task-service]
    GW --> SOCIAL[social-service]
    GW --> SCHED[schedule-service]
    GW --> CAREER[career-service]
    GW --> NOTIF[notification-service]
    GW --> AUDIT[audit-service]
    GW --> REPORT[report-service]
```

> `social-service` fusiona `family` + `group` (mismo modelo de datos). `space-service` se fusionó dentro de `task-service`. `business-service` y `file-service` se eliminaron del diseño — ver [README → Módulos del Sistema](./README.md#-módulos-del-sistema).

---

## Bases de Datos por Servicio

| DB | Servicios | ORM | Por qué |
|----|-----------|-----|---------|
| PostgreSQL | auth, user, social, task, schedule, finance, vault, rehab | Prisma / TypeORM | ACID, relacional, migraciones tipadas |
| MongoDB | task (plantillas), career, report, notification, media | Mongoose | Esquemas flexibles, buen para read models |
| DynamoDB | audit-service | AWS SDK v3 | Append-only, sin servidor, escala automática |
| Redis | api-gateway, schedule, vault (cache) | ioredis | In-memory, rate limit, locks |
| S3 / MinIO | rehab-service, media-service | AWS SDK v3 | Object storage, presigned URLs — sin `file-service` intermedio, cada dominio sube directo |

---

## Fases de Desarrollo

Orden de construcción real (no todo el diagrama se implementa de una vez — cada servicio se construye solo cuando tiene un consumidor real, ver [README → Módulos del Sistema](./README.md#-módulos-del-sistema)):

| Fase | Nombre | Entregable | Estado |
|------|--------|-----------|--------|
| 0 | Fundación | contracts + infra local + NATS + Postgres + Redis + MinIO | ✅ Hecho |
| 1 | Identidad | auth-service + user-service + api-gateway + OAuth | ✅ Hecho |
| 2 | Rehab | rehab-service — caso de uso real que motiva el proyecto | 🔧 Actual |
| 3 | Seguridad y Finanzas | vault-service + finance-service | 📋 Siguiente |
| 4 | Productividad y Media | media-service + task-service (opcional) | 📋 Planificado |
| 5 | Social y Extras | social-service (family+group) + schedule-service + career-service — casos de uso reales pero de menor prioridad | 📋 Planificado |
| — | Pausado sin fecha | notification-service, report-service, audit-service — sin necesidad concreta identificada aún | ⏸️ Pausado |
| 6 | Frontend | Next.js shell + microfrontends (o monolito modular al inicio) + React Query + Zustand | 📋 Planificado |
| 7 | CI/CD y Testing | GitHub Actions + Jenkins + SonarQube + BDD + TDD + E2E | 🔧 Parcial (auth/user en Jenkins) |
| 8 | Observabilidad | Prometheus + Grafana + OpenTelemetry (solo si se retoman notification/audit/report) | 📋 Planificado |
| 9 | Cloud Real | EC2 → ECS Fargate → Kubernetes (EKS) | 📋 Planificado |

---

> Detalles de implementación → [Backend](./BACKEND.md) · [Frontend](./FRONTEND.md) · [DevOps](./DEVOPS.md) · [CI/CD](./CICD.md)
