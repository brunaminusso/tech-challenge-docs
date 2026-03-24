# ADR-001: Uso de Horizontal Pod Autoscaler (HPA)

**Status:** Aceito
**Data:** 2026-03-15

## Contexto

A aplicacao pode receber picos de demanda (ex: segundas-feiras, inicio de mes) que excedem a capacidade configurada estaticamente.

## Decisao

Implementar **Horizontal Pod Autoscaler (HPA)** para escalar automaticamente os pods da aplicacao baseado em metricas de CPU e memoria.

### Configuracao

| Ambiente | Min Replicas | Max Replicas | CPU Target | Memory Target |
|----------|--------------|--------------|------------|---------------|
| Dev | 2 | 5 | 70% | 80% |
| Staging | 3 | 8 | 70% | 75% |
| Production | 4 | 20 | 60% | 70% |

### Behavior

**Scale Up:**
- Stabilization Window: 60s
- Max: 4 pods ou 100% de aumento por periodo

**Scale Down:**
- Stabilization Window: 300s (5 min)
- Max: 1 pod por periodo (conservador)

## Consequencias

### Positivas
- Escalabilidade automatica sem intervencao manual
- Reducao de custos quando demanda e baixa
- Resiliencia contra picos inesperados

### Negativas
- Metrics Server obrigatorio
- Cold start ao escalar (mitigado por min replicas)
- Complexidade adicional de monitoramento

## Alternativas Rejeitadas

1. **Replicas Estaticas:** Desperdicio de recursos ou indisponibilidade
2. **KEDA:** Complexidade excessiva para o escopo atual
3. **Cluster Autoscaler:** Limitado pelo Free Tier
