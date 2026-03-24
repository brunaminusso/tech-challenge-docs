# Visao Geral da Arquitetura

## Introducao

O Tech Challenge e um sistema de gerenciamento de ordens de servico para uma oficina mecanica multi-unidades. O sistema foi projetado para suportar alta disponibilidade, escalabilidade e observabilidade completa.

## Requisitos Atendidos

| Requisito | Implementacao |
|-----------|---------------|
| API Gateway | Kong (DB-less) com rate limiting e CORS |
| Autenticacao CPF | Oracle Function + JWT |
| Banco Gerenciado | OCI MySQL HeatWave |
| Kubernetes Escalavel | OKE com HPA |
| IaC | Terraform para toda infraestrutura |
| CI/CD | GitHub Actions em todos os repositorios |
| Monitoramento | Datadog (APM, Logs, Metricas, Alertas) |

## Componentes Principais

### 1. Camada de Entrada

```
Internet -> Kong API Gateway -> Rotas
                |
                +-> /api/v1/auth/cpf  -> Oracle Function (Serverless)
                +-> /api/v1/*         -> PHP Application (Kubernetes)
                +-> /docs             -> Swagger UI
```

**Kong Gateway:**
- Rate Limiting: 100 req/min
- CORS: Habilitado para todas origens
- Modo: DB-less (configuracao declarativa)

### 2. Camada de Autenticacao

A autenticacao por CPF e tratada por uma Function Serverless isolada:

1. Cliente envia POST /api/v1/auth/cpf com CPF
2. Kong roteia para Oracle Function
3. Function valida API Key (constant-time comparison)
4. Function valida CPF (algoritmo completo com digitos verificadores)
5. Function consulta MySQL para verificar cliente ativo
6. Function gera JWT com payload: {iss, sub, cpf, name, role, iat, exp}
7. Cliente recebe token para usar nas demais requisicoes

### 3. Camada de Aplicacao

**Arquitetura DDD + Clean Architecture:**

```
Presentation (Controllers)
      |
      v
Application (Use Cases)
      |
      v
Domain (Entities, Enums, Interfaces)
      ^
      |
Infrastructure (Repositories, External Services)
```

**Entidades de Dominio:**
- Client, Vehicle, ServiceOrder, Service, Part
- ServiceOrderService, ServiceOrderPart (associativas)
- ServiceOrderStatusHistory, Notification, PartReservation

### 4. Camada de Dados

**MySQL 8.0 (OCI HeatWave):**
- 11 tabelas normalizadas
- Precos em centavos (INT) para precisao
- Indices otimizados para consultas frequentes
- Charset: utf8mb4_unicode_ci

### 5. Camada de Observabilidade

**Datadog:**
- Agent como DaemonSet no cluster
- APM com instrumentacao automatica PHP
- Logs estruturados em JSON
- Dashboard customizado para Ordens de Servico
- Alertas integrados com PagerDuty

## Fluxo de Deploy

```
Developer -> Push -> GitHub Actions -> Build -> Test -> Push Image -> Terraform Apply -> OKE
                                                              |
                                                              v
                                                     Oracle Container Registry
```

**Ambientes:**
- `develop` branch -> Staging namespace
- `main/master` branch -> Production namespace

## Escalabilidade

**Horizontal Pod Autoscaler (HPA):**

| Ambiente | Min | Max | CPU Target | Memory Target |
|----------|-----|-----|------------|---------------|
| Staging | 3 | 8 | 70% | 75% |
| Production | 4 | 20 | 60% | 70% |

**Node Pool (OKE):**
- Shape: VM.Standard.A1.Flex (ARM)
- 1 node x 1 OCPU x 6GB (Free Tier)
- Namespaces separados para staging/production

## Seguranca

1. **Autenticacao:** JWT com expiracao de 1 hora
2. **API Key:** Protecao adicional para Function
3. **Rate Limiting:** 100 req/min no Kong
4. **Secrets:** Kubernetes Secrets para credenciais
5. **HTTPS:** TLS termination no Load Balancer
6. **Validacao:** CPF validado com algoritmo completo

## Proximos Passos (Roadmap)

1. [ ] Implementar refresh token para sessoes longas
2. [ ] Adicionar Redis para cache de consultas frequentes
3. [ ] Implementar notificacoes push via Oracle Notification Service
4. [ ] Adicionar testes de carga com k6
5. [ ] Implementar circuit breaker com resilience4j
