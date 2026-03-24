# Tech Challenge - Documentacao Arquitetural

Documentacao arquitetural completa do projeto Tech Challenge FIAP - Sistema de Gerenciamento de Ordens de Servico para Oficina Mecanica.

## Repositorios do Projeto

| Repositorio | Descricao | Status |
|-------------|-----------|--------|
| [tech-challenge-app](https://github.com/brunaminusso/tech-challenge-app) | Aplicacao principal PHP (Slim 4 + DDD) | [![CI/CD](https://github.com/brunaminusso/tech-challenge-app/actions/workflows/ci-cd.yml/badge.svg)](https://github.com/brunaminusso/tech-challenge-app/actions) |
| [tech-challenge-serverless](https://github.com/brunaminusso/tech-challenge-serverless) | Oracle Function para autenticacao por CPF | [![Serverless](https://github.com/brunaminusso/tech-challenge-serverless/actions/workflows/serverless.yml/badge.svg)](https://github.com/brunaminusso/tech-challenge-serverless/actions) |
| [tech-challenge-infra-k8s](https://github.com/brunaminusso/tech-challenge-infra-k8s) | Infraestrutura Kubernetes (Terraform + OKE) | [![Infrastructure](https://github.com/brunaminusso/tech-challenge-infra-k8s/actions/workflows/infrastructure.yml/badge.svg)](https://github.com/brunaminusso/tech-challenge-infra-k8s/actions) |
| [tech-challenge-infra-db](https://github.com/brunaminusso/tech-challenge-infra-db) | Infraestrutura Banco de Dados (Terraform + OCI MySQL) | [![Database](https://github.com/brunaminusso/tech-challenge-infra-db/actions/workflows/database-infra.yml/badge.svg)](https://github.com/brunaminusso/tech-challenge-infra-db/actions) |

---

## Diagramas

### Diagrama de Componentes

Visao geral da arquitetura de nuvem, incluindo APIs, banco de dados e monitoramento.

```mermaid
flowchart TB
    subgraph Internet
        Cliente[Cliente/App Frontend]
    end

    subgraph OCI[Oracle Cloud Infrastructure - eu-paris-1]
        subgraph VCN[VCN 10.0.0.0/16]
            subgraph OKE[OKE Cluster - tech-challenge]
                subgraph NSStaging[Namespace: staging]
                    KongS[Kong Gateway]
                    AppS[PHP App + Nginx]
                    HPAS[HPA 3-8]
                end
                subgraph NSProd[Namespace: production]
                    KongP[Kong Gateway]
                    AppP[PHP App + Nginx]
                    HPAP[HPA 4-20]
                end
                subgraph NSMon[Namespace: monitoring]
                    Datadog[Datadog Agent]
                end
            end
        end

        subgraph Serverless[Oracle Functions]
            AuthFn[auth-cpf\nNode.js 18]
        end

        subgraph Database[OCI MySQL HeatWave]
            MySQL[(MySQL 8.0\n11 tabelas)]
        end
    end

    subgraph External[Servicos Externos]
        DatadogCloud[Datadog Cloud\nDashboards + Alertas]
        GitHub[GitHub Actions\nCI/CD]
    end

    Cliente -->|HTTPS| KongS
    Cliente -->|HTTPS| KongP
    KongS -->|/api/v1/*| AppS
    KongP -->|/api/v1/*| AppP
    KongS -->|/auth/cpf| AuthFn
    KongP -->|/auth/cpf| AuthFn
    AppS -->|3306| MySQL
    AppP -->|3306| MySQL
    AuthFn -->|3306| MySQL
    Datadog -->|metrics/logs| DatadogCloud
    GitHub -->|deploy| OKE
    GitHub -->|terraform| MySQL
```

### Diagrama de Sequencia - Autenticacao por CPF

Fluxo completo de autenticacao usando CPF como identificador.

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant K as Kong Gateway
    participant F as Oracle Function
    participant DB as MySQL
    participant JWT as JWT Library

    C->>K: POST /api/v1/auth/cpf<br/>x-api-key: xxx<br/>{cpf: "529.982.247-25"}

    K->>K: Verifica Rate Limit<br/>(100 req/min)
    K->>F: Invoke Function

    F->>F: validateAPIKey()<br/>(constant-time comparison)

    alt API Key inválida
        F-->>C: 401 Unauthorized<br/>{error: "API Key invalida"}
    end

    F->>F: validateCPF()<br/>- Remove caracteres<br/>- Valida 11 dígitos<br/>- Calcula verificadores

    alt CPF inválido
        F-->>C: 400 Bad Request<br/>{error: "CPF invalido"}
    end

    F->>DB: SELECT id, name, cpf, status<br/>FROM clients<br/>WHERE cpf = ? AND status = 'active'
    DB-->>F: [{id, name, cpf, status}]

    alt Cliente não encontrado
        F-->>C: 404 Not Found<br/>{error: "Cliente nao encontrado"}
    end

    F->>JWT: jwt.sign(payload, secret)<br/>payload: {iss, sub, cpf, name, role}
    JWT-->>F: eyJhbGciOiJIUzI1NiIs...

    F-->>C: 200 OK<br/>{token, type: "Bearer", expires_in: 3600, client}
```

### Diagrama de Sequencia - Ordens de Servico

Fluxo de criacao e transicao de status das ordens.

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant K as Kong
    participant Ctrl as Controller
    participant UC as UseCase
    participant Repo as Repository
    participant DB as MySQL
    participant Log as Logger/Datadog

    rect rgb(232, 245, 233)
        Note over C,Log: FLUXO 1: Criar Ordem de Servico
    end

    C->>K: POST /api/v1/service-orders<br/>Authorization: Bearer JWT
    K->>K: Valida JWT
    K->>Ctrl: Forward Request

    Ctrl->>Ctrl: Valida body<br/>(client_id, vehicle_id, services, parts)

    Ctrl->>Repo: findById(client_id)
    Repo->>DB: SELECT * FROM clients WHERE id = ?
    DB-->>Repo: Client
    Repo-->>Ctrl: Client entity

    Note over Ctrl,Repo: Repete para Vehicle, Services, Parts

    Ctrl->>UC: execute(ServiceOrder)
    UC->>UC: Cria ServiceOrder<br/>status = 'received'<br/>Gera order_number
    UC->>Repo: save(order)
    Repo->>DB: INSERT INTO service_orders...
    DB-->>Repo: OK
    Repo-->>UC: Saved
    UC-->>Ctrl: Order created

    Ctrl->>Log: logger.info('Ordem criada',<br/>{order_id, order_number, client_id})
    Log->>Log: Envia para Datadog

    Ctrl-->>C: 201 Created<br/>{id, order_number, status, ...}

    rect rgb(255, 243, 205)
        Note over C,Log: FLUXO 2: Alterar Status
    end

    C->>Ctrl: PATCH /service-orders/{id}/status<br/>{status: "in_diagnostic"}
    Ctrl->>UC: TransitionStatusUseCase.execute()
    UC->>UC: Valida transicao<br/>(state machine)
    UC->>Repo: save() + saveStatusHistory()
    Repo->>DB: UPDATE + INSERT status_history
    DB-->>Repo: OK

    Ctrl->>Log: logger.info('Status alterado',<br/>{order_id, previous, new})

    Ctrl-->>C: 200 OK {updated order}
```

### Maquina de Estados - OrderStatus

```mermaid
stateDiagram-v2
    [*] --> received: Ordem criada

    received --> in_diagnostic: Iniciar diagnostico
    in_diagnostic --> awaiting_approval: Enviar orcamento

    awaiting_approval --> in_execution: Cliente aprova
    awaiting_approval --> cancelled: Cliente rejeita

    in_execution --> finished: Servico concluido
    finished --> delivered: Veiculo entregue

    delivered --> [*]
    cancelled --> [*]

    note right of received: Status inicial
    note right of awaiting_approval: Aguarda decisao cliente
    note right of cancelled: Fim (rejeitado)
    note right of delivered: Fim (sucesso)
```

### Diagrama Entidade-Relacionamento (ER)

Modelo relacional completo com 11 tabelas.

```mermaid
erDiagram
    clients ||--o{ vehicles : possui
    clients ||--o{ service_orders : solicita
    clients ||--o{ notifications : recebe

    vehicles ||--o{ service_orders : associado

    service_orders ||--o{ service_order_services : contem
    service_orders ||--o{ service_order_parts : contem
    service_orders ||--o{ service_order_status_history : registra
    service_orders ||--o{ notifications : gera
    service_orders ||--o{ part_reservations : reserva

    services ||--o{ service_order_services : incluido
    parts ||--o{ service_order_parts : incluido
    parts ||--o{ part_reservations : reservado

    clients {
        int id PK
        string name
        string cpf UK
        string cnpj UK
        string email
        string phone
        text address
        timestamp created_at
        timestamp updated_at
    }

    vehicles {
        int id PK
        string plate UK
        string brand
        string model
        int year
        string color
        int client_id FK
        timestamp created_at
        timestamp updated_at
    }

    service_orders {
        int id PK
        string order_number UK
        int client_id FK
        int vehicle_id FK
        string status
        text description
        text diagnostic_notes
        int quotation_total_in_cents
        timestamp quotation_approved_at
        boolean cancelled
        timestamp created_at
        timestamp updated_at
    }

    services {
        int id PK
        string name
        text description
        int price_in_cents
        int estimated_duration_minutes
        boolean active
        timestamp created_at
    }

    parts {
        int id PK
        string name
        string code UK
        text description
        int price_in_cents
        int stock_quantity
        int minimum_stock
        boolean active
        timestamp created_at
    }

    service_order_services {
        int id PK
        int service_order_id FK
        int service_id FK
        int quantity
        int unit_price_in_cents
        int subtotal_in_cents
    }

    service_order_parts {
        int id PK
        int service_order_id FK
        int part_id FK
        int quantity
        int unit_price_in_cents
        int subtotal_in_cents
    }

    service_order_status_history {
        int id PK
        int service_order_id FK
        string status
        text notes
        timestamp changed_at
    }

    notifications {
        int id PK
        int client_id FK
        int service_order_id FK
        string type
        string event
        string subject
        text message
        string status
        timestamp sent_at
    }

    part_reservations {
        int id PK
        int service_order_id FK
        int part_id FK
        int quantity
        string status
        timestamp reserved_at
        timestamp expires_at
    }

    users {
        int id PK
        string name
        string email UK
        string password_hash
        string role
        boolean active
        timestamp created_at
    }
```

---

## RFCs (Request for Comments)

Documentos que descrevem decisoes tecnicas relevantes do projeto.

| RFC | Titulo | Status |
|-----|--------|--------|
| [RFC-001](rfcs/RFC-001-escolha-nuvem.md) | Escolha da Nuvem (OCI) | Aceito |
| [RFC-002](rfcs/RFC-002-escolha-banco-dados.md) | Escolha do Banco de Dados (MySQL) | Aceito |
| [RFC-003](rfcs/RFC-003-estrategia-autenticacao.md) | Estrategia de Autenticacao (JWT + CPF) | Aceito |

### Resumo das Decisoes

#### Por que Oracle Cloud Infrastructure?
- **Always Free Tier permanente** com 4 OCPUs ARM e 24GB RAM
- MySQL HeatWave incluso gratuitamente
- Oracle Functions com 5M invocacoes/mes gratis
- OKE (Kubernetes) compativel com padroes

#### Por que MySQL 8.0?
- Incluso no OCI Free Tier
- Compatibilidade nativa com PHP (PDO)
- Modelo relacional adequado para o dominio
- Precos em centavos para evitar problemas de precisao

#### Por que JWT com CPF?
- Autenticacao stateless e escalavel
- CPF como identificador natural brasileiro
- Function serverless isolada (menor surface attack)

---

## ADRs (Architecture Decision Records)

Registros de decisoes arquiteturais permanentes.

| ADR | Titulo | Status |
|-----|--------|--------|
| [ADR-001](adrs/ADR-001-uso-hpa.md) | Uso de Horizontal Pod Autoscaler (HPA) | Aceito |
| [ADR-002](adrs/ADR-002-kong-api-gateway.md) | Kong como API Gateway | Aceito |
| [ADR-003](adrs/ADR-003-arquitetura-ddd.md) | Arquitetura DDD com Clean Architecture | Aceito |
| [ADR-004](adrs/ADR-004-datadog-observabilidade.md) | Datadog para Observabilidade | Aceito |

### Decisoes Arquiteturais Chave

| Decisao | Escolha | Alternativas Rejeitadas |
|---------|---------|-------------------------|
| Autoscaling | HPA (CPU/Memory) | KEDA, Replicas estaticas |
| API Gateway | Kong (DB-less) | Traefik, Istio, AWS API GW |
| Arquitetura | DDD + Clean Architecture | MVC tradicional, Hexagonal |
| Monitoramento | Datadog | Prometheus+Grafana, New Relic |

---

## Arquitetura DDD - Camadas

```mermaid
flowchart TB
    subgraph Presentation[Camada de Apresentacao]
        Controllers[Controllers REST]
        Middleware[Auth Middleware]
    end

    subgraph Application[Camada de Aplicacao]
        UseCases[Use Cases]
        DTOs[DTOs]
    end

    subgraph Domain[Camada de Dominio]
        Entities[Entities]
        Enums[Enums]
        Interfaces[Repository Interfaces]
        DomainServices[Domain Services]
    end

    subgraph Infrastructure[Camada de Infraestrutura]
        Repositories[MySQL Repositories]
        External[External Services]
        Logger[Datadog Logger]
    end

    Presentation --> Application
    Application --> Domain
    Infrastructure --> Domain
    Application -.-> Infrastructure

    style Domain fill:#e1f5fe
    style Application fill:#fff3e0
    style Presentation fill:#e8f5e9
    style Infrastructure fill:#fce4ec
```

---

## Stack Tecnologica

### Aplicacao
| Componente | Tecnologia |
|------------|------------|
| Linguagem | PHP 8.2 |
| Framework | Slim Framework 4 |
| ORM | Doctrine DBAL |
| Arquitetura | DDD + Clean Architecture |

### Infraestrutura
| Componente | Tecnologia |
|------------|------------|
| Cloud | Oracle Cloud Infrastructure (OCI) |
| Kubernetes | OKE (Oracle Kubernetes Engine) |
| IaC | Terraform >= 1.5.0 |
| API Gateway | Kong (DB-less) |

### Dados
| Componente | Tecnologia |
|------------|------------|
| Banco de Dados | MySQL 8.0 (OCI HeatWave) |
| Cache | (Planejado: Redis) |

### Observabilidade
| Componente | Tecnologia |
|------------|------------|
| APM | Datadog |
| Logs | Datadog Logs (JSON estruturado) |
| Metricas | Datadog Metrics |
| Alertas | Datadog Monitors + PagerDuty |

### CI/CD
| Componente | Tecnologia |
|------------|------------|
| Pipelines | GitHub Actions |
| Registry | Oracle Container Registry (OCIR) |
| Deploy | Terraform + kubectl |

---

## Pipeline CI/CD

```mermaid
flowchart LR
    subgraph Developer
        Code[Codigo]
    end

    subgraph GitHub
        Push[Push/PR]
        Actions[GitHub Actions]
    end

    subgraph Pipeline[Pipeline CI/CD]
        Lint[Lint & Test]
        Build[Build Docker]
        Push2[Push OCIR]
        TFPlan[Terraform Plan]
        TFApply[Terraform Apply]
    end

    subgraph OCI[Oracle Cloud]
        OCIR[Container Registry]
        OKE[Kubernetes Cluster]
    end

    Code --> Push
    Push --> Actions
    Actions --> Lint
    Lint --> Build
    Build --> Push2
    Push2 --> OCIR
    Actions --> TFPlan
    TFPlan --> TFApply
    TFApply --> OKE
    OCIR --> OKE
```

---

## Video de Demonstracao

**Link:** [YouTube - Tech Challenge Demo](https://youtube.com/...)

**Conteudo do Video:**
1. Autenticacao com CPF (Oracle Function)
2. Pipeline CI/CD em execucao
3. Deploy automatizado para OKE
4. Consumo das APIs protegidas via Kong
5. Dashboard Datadog com metricas ao vivo
6. Logs e traces em tempo real

---

## Equipe

- **Aluno:** Bruna Minusso
- **Curso:** Pos-Graduacao FIAP
- **Fase:** Tech Challenge

---

## Licenca

Este projeto foi desenvolvido como parte do Tech Challenge FIAP.
