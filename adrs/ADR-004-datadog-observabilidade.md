# ADR-004: Datadog para Observabilidade

**Status:** Aceito
**Data:** 2026-03-15

## Contexto

O sistema em producao requer observabilidade completa:
- Metricas de infraestrutura (CPU, memoria, rede)
- Metricas de aplicacao (latencia, erros, throughput)
- Logs estruturados com correlacao
- Traces distribuidos (APM)
- Alertas proativos

## Decisao

Utilizar **Datadog** como plataforma unificada de observabilidade.

### Componentes

| Componente | Funcao |
|------------|--------|
| Datadog Agent | Coleta metricas do cluster K8s |
| APM Library | Instrumentacao automatica PHP |
| Log Forwarder | Envio de logs estruturados |
| Dashboards | Visualizacao customizada |
| Monitors | Alertas automaticos |

### Metricas Coletadas

**Infraestrutura:**
- kubernetes.cpu.usage.total
- kubernetes.memory.usage
- kubernetes.pods.running

**Aplicacao:**
- trace.http.request.duration (P50, P95, P99)
- trace.http.request.errors
- app.ordens_servico.criadas
- app.ordens_servico.tempo_execucao

### Dashboard

```
+------------------------------------------+
| Volume Diario OS | Tempo por Status      |
| [grafico barras] | [grafico linha]       |
+------------------------------------------+
| Erros Integracao | Latencia P95          |
| [grafico linha]  | [grafico linha]       |
+------------------------------------------+
| CPU Cluster      | Memoria Cluster       |
| [gauge]          | [gauge]               |
+------------------------------------------+
```

### Alertas

| Alerta | Condicao | Acao |
|--------|----------|------|
| Falha Processamento OS | > 10 erros/5min | PagerDuty |
| Latencia Alta | P95 > 2s | Slack |
| CPU Critico | > 90% por 5min | PagerDuty |

### Logs Estruturados

```json
{
  "timestamp": "2026-03-24T12:00:00Z",
  "level": "info",
  "message": "Ordem de servico criada",
  "context": {
    "order_id": 123,
    "order_number": "OS-2026-0001",
    "client_id": 45,
    "status": "received",
    "trace_id": "abc123..."
  }
}
```

## Consequencias

### Positivas
- Visibilidade total do sistema
- Deteccao proativa de problemas
- Correlacao entre logs, metricas e traces
- SLA mensuravel

### Negativas
- Custo do Datadog (mitigado com Free Trial)
- Dependencia de servico externo
- Volume de dados pode crescer

## Alternativas Rejeitadas

1. **Prometheus + Grafana + Loki:** Mais complexo de operar
2. **New Relic:** Custo elevado
3. **ELK Stack:** Overhead de infraestrutura
4. **CloudWatch:** Vendor lock-in AWS
