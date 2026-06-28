# 🧠 LifeTrack OS

> Plataforma personal y familiar de productividad construida con arquitectura de microservicios empresarial real.

**Stack:** NestJS · gRPC · NATS JetStream · PostgreSQL · MongoDB · DynamoDB · Redis · S3 · Next.js · Docker · Kubernetes · AWS · Terraform · GitHub Actions · Jenkins · SonarQube · Prometheus · Grafana · OpenTelemetry

---

## 📌 Descripción

LifeTrack OS es una plataforma que centraliza tareas, finanzas, archivos, postulaciones laborales, bóveda segura de contraseñas y más — construida con el stack que usan empresas tecnológicas reales. El objetivo es aprender haciendo: cada módulo enseña una tecnología distinta que la industria exige.

**Problemática:** Las personas manejan su vida digital dispersa en múltiples apps sin integración. LifeTrack unifica todo en un solo sistema seguro y personalizable.

**Usuario objetivo:** Personas y familias que quieren organizar su vida digital con privacidad real y acceso desde cualquier dispositivo.

---

## 📚 Documentación — Índice General

| Documento | Descripción |
|-----------|-------------|
| [README principal](./README.md) | Este archivo — visión general + diagramas |
| [Arquitectura](./ARCHITECTURE.md) | Principios, módulos, decisiones de diseño |
| [Backend](./BACKEND.md) | Microservicios, gRPC, NATS, hexagonal, testing |
| [Frontend](./FRONTEND.md) | Microfrontend, Next.js, React Query, Vault UI |
| [DevOps & Infra](./DEVOPS.md) | Docker, Kubernetes, AWS, Terraform, Prometheus |
| [CI/CD & Calidad](./CICD.md) | GitHub Actions, Jenkins, SonarQube, TDD, BDD |

---

## 🏗️ Arquitectura General

```mermaid
graph TB
    subgraph Clients["Clientes"]
        WEB[Web / Next.js]
        MOB[Mobile App]
    end

    subgraph GW["API Gateway"]
        GATEWAY[api-gateway\nREST · JWT · Rate Limit · Swagger]
    end

    subgraph Core["Microservicios"]
        AUTH[auth-service\nPostgreSQL + OAuth]
        USER[user-service\nPostgreSQL]
        FAMILY[family / group\nPostgreSQL]
        SPACE[space + task\nPostgreSQL + MongoDB]
        VAULT[vault-service\nAES-256-GCM]
        FILE[file + media\nPostgreSQL + S3]
        FIN[finance + career\nPostgreSQL + MongoDB]
        BIZ[business-service\nPostgreSQL]
        SCHED[schedule-service\nPostgreSQL + Redis]
    end

    subgraph MSG["Mensajería"]
        NATS[NATS JetStream\nEventos · ACK · Retry · DLQ]
    end

    subgraph Workers["Workers Asíncronos"]
        NOTIF[notification\nMongoDB · FCM · SMTP]
        AUDIT[audit\nDynamoDB]
        REPORT[report\nMongoDB + Redis]
    end

    subgraph Obs["Observabilidad"]
        PROM[Prometheus + Grafana]
        OTEL[OpenTelemetry]
    end

    WEB & MOB -->|HTTPS + REST| GATEWAY
    GATEWAY -->|gRPC| AUTH & USER & FAMILY & SPACE & VAULT & FILE & FIN & BIZ & SCHED
    AUTH & USER & SPACE & VAULT & FILE & FIN & SCHED -->|domain events| NATS
    NATS -->|consume| NOTIF & AUDIT & REPORT
    Core --> PROM
    GATEWAY --> OTEL
```

---

## 🔄 Flujo de un Request

```mermaid
sequenceDiagram
    participant C as Cliente
    participant GW as API Gateway
    participant AUTH as auth-service
    participant TASK as task-service
    participant NATS as NATS JetStream
    participant NOTIF as notification
    participant AUDIT as audit

    C->>GW: POST /tasks (REST + JWT)
    GW->>AUTH: ValidateToken (gRPC)
    AUTH-->>GW: user_id, roles
    GW->>TASK: CreateTask (gRPC)
    TASK->>TASK: DB write + Outbox
    TASK-->>GW: task_id, status
    GW-->>C: 201 Created
    Note over TASK,NATS: proceso asíncrono
    TASK->>NATS: task.created.v1
    NATS->>NOTIF: push notification
    NATS->>AUDIT: append DynamoDB
```

---

## ☁️ Despliegue AWS

```mermaid
graph LR
    USER[Usuario] --> CF[CloudFront + S3\nFrontend MFE]
    USER --> ALB[ALB + HTTPS]
    ALB --> GW2[api-gateway]

    subgraph ECS["ECS Fargate / Kubernetes"]
        GW2 --> SVC[Microservicios\n×16 servicios]
        SVC --> NATS2[NATS JetStream]
        NATS2 --> WRK[Workers]
    end

    subgraph DB["Storage"]
        SVC --> RDS[(RDS PostgreSQL)]
        SVC --> DYN[(DynamoDB)]
        WRK --> MONGO[(MongoDB)]
        GW2 --> REDIS[(ElastiCache Redis)]
        SVC --> S3[(S3 Bucket)]
    end

    subgraph MON["Monitoring"]
        ECS --> PROM2[Prometheus]
        PROM2 --> GRAF[Grafana]
        ECS --> CW[CloudWatch]
    end
```

---

## 🔐 Cifrado del Vault

```mermaid
sequenceDiagram
    participant U as Usuario
    participant FE as Frontend (Browser)
    participant BE as vault-service
    participant DB as PostgreSQL

    U->>FE: master password + secreto
    FE->>FE: Argon2id(password + salt) → clave
    FE->>FE: AES-256-GCM(secreto, clave, iv) → blob
    Note over FE: El secreto nunca sale del browser
    FE->>BE: POST { encrypted_blob, salt, iv }
    BE->>DB: guarda blob cifrado
    Note over BE: Backend nunca ve el secreto real
    BE-->>FE: 201 Created
    BE->>BE: vault.secret_created.v1 → audit
```

---

## 🗂️ Módulos del Sistema

| Módulo | Tecnología | Estado |
|--------|-----------|--------|
| auth-service | NestJS + PostgreSQL + OAuth Google/GitHub | 🔧 En desarrollo |
| user-service | NestJS + PostgreSQL + Push devices | 🔧 En desarrollo |
| family-service | NestJS + PostgreSQL | 📋 Planificado |
| group-service | NestJS + PostgreSQL | 📋 Planificado |
| space-service | NestJS + PostgreSQL + MongoDB | 📋 Planificado |
| task-service | NestJS + PostgreSQL | 📋 Planificado |
| schedule-service | NestJS + PostgreSQL + Redis | 📋 Planificado |
| file-service | NestJS + PostgreSQL + S3 | 📋 Planificado |
| media-service | NestJS + MongoDB + S3 | 📋 Planificado |
| finance-service | NestJS + PostgreSQL | 📋 Planificado |
| vault-service | NestJS + PostgreSQL + AES-256-GCM | 📋 Planificado |
| career-service | NestJS + MongoDB | 📋 Planificado |
| business-service | NestJS + PostgreSQL + TypeORM | 📋 Planificado |
| notification-service | NestJS + MongoDB + FCM + SMTP | 📋 Planificado |
| audit-service | NestJS + DynamoDB | 📋 Planificado |
| report-service | NestJS + MongoDB + Redis | 📋 Planificado |

---

## 🚀 Correr el Proyecto Localmente

```bash
# 1. Clonar
git clone https://github.com/tu-usuario/lifetrack-os.git
cd lifetrack-os

# 2. Levantar infraestructura base
cd lifetrack-infra
cp .env.example .env
docker compose up -d
# Levanta: NATS · PostgreSQL · MongoDB · Redis · DynamoDB Local · MinIO · Prometheus · Grafana

# 3. Correr auth-service
cd services/auth-service
cp .env.example .env && npm install
npx prisma migrate dev
npm run start:dev

# 4. Correr api-gateway
cd services/api-gateway
cp .env.example .env && npm install
npm run start:dev

# 5. Verificar
curl http://localhost:3000/health
```

---

## 🤖 IA en el Proyecto

| Herramienta | Modelo | Uso |
|-------------|--------|-----|
| Anthropic Claude API | claude-sonnet-4-6 | Asistente personal, sugerencias de tareas, resúmenes |
| OpenAI API | gpt-4o | Análisis de postulaciones, resúmenes financieros |
| Hugging Face | distilbert | Clasificación automática de prioridades y gastos |

---

## 👤 Equipo

| Nombre | Rol |
|--------|-----|
| [Tu nombre] | Full Stack Developer · DevOps · Arquitecto |

---

## 📦 Release

**Release:** `Prototipo de arquitectura - LifeTrack OS`

Incluye documentación completa, diagramas y estructura base del proyecto.
