# ADR-002: Kong como API Gateway

**Status:** Aceito
**Data:** 2026-03-15

## Contexto

O sistema necessita de um API Gateway para:
- Roteamento de requisicoes
- Rate limiting
- Validacao de JWT
- CORS handling
- Monitoramento centralizado

## Decisao

Utilizar **Kong Gateway** em modo **DB-less (Declarative)**.

### Justificativa

1. **Open Source:** Sem custos de licenciamento
2. **DB-less:** Configuracao via YAML, sem banco adicional
3. **Plugins nativos:** rate-limiting, cors, jwt, prometheusu
4. **Kubernetes native:** Ingress Controller disponivel

### Configuracao

```yaml
services:
  - name: tech-challenge-api
    url: http://app-service:80
    routes:
      - paths: [/api/v1/*]
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: cors
```

### Roteamento

| Path | Destino | Autenticacao |
|------|---------|--------------|
| /api/v1/auth/login | App | Nao |
| /api/v1/auth/register | App | Nao |
| /api/v1/auth/cpf | Oracle Function | API Key |
| /api/v1/* | App | JWT |
| /docs | App | Nao |

## Consequencias

### Positivas
- Centralizacao de cross-cutting concerns
- Metricas unificadas
- Facil adicao de novos plugins

### Negativas
- Ponto unico de entrada (mitigar com replicas)
- Latencia adicional (~5-10ms)

## Alternativas Rejeitadas

1. **AWS API Gateway:** Custo, vendor lock-in
2. **Traefik:** Menos plugins out-of-box
3. **Istio:** Overhead excessivo para o projeto
4. **Nginx Ingress:** Menos funcionalidades de gateway
