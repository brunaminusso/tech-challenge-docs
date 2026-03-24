# RFC-002: Escolha do Banco de Dados - MySQL 8.0

**Status:** Aceito
**Data:** 2026-03-15
**Autor:** Equipe Tech Challenge

## Resumo

Este documento descreve a decisao de utilizar MySQL 8.0 (OCI HeatWave) como banco de dados relacional do projeto.

## Contexto

O sistema de gerenciamento de ordens de servico requer:
- Armazenamento de dados transacionais (clientes, veiculos, ordens)
- Relacionamentos complexos entre entidades
- Consultas frequentes com JOINs
- Suporte a transacoes ACID
- Alta disponibilidade em producao

## Opcoes Consideradas

### 1. PostgreSQL
**Pros:**
- Recursos avancados (JSONB, full-text search)
- Forte suporte a tipos customizados
- Excelente para dados complexos

**Contras:**
- Nao incluido no OCI Free Tier
- Configuracao mais complexa
- Maior consumo de recursos

### 2. MongoDB
**Pros:**
- Schema flexivel
- Performance em leitura
- Escalabilidade horizontal

**Contras:**
- Nao adequado para dados altamente relacionais
- Consistencia eventual
- Nao suporta JOINs nativos

### 3. MySQL 8.0 (OCI HeatWave) - ESCOLHIDO
**Pros:**
- **Incluido no OCI Free Tier**
- Amplamente conhecido e documentado
- Suporte nativo em PHP (PDO, mysqli)
- JSON columns para flexibilidade
- Transacoes ACID completas
- HeatWave para analytics (se necessario)

**Contras:**
- Menos extensivel que PostgreSQL
- Replicacao mais simples

### 4. Oracle Autonomous Database
**Pros:**
- Totalmente gerenciado
- Auto-tuning
- Always Free Tier

**Contras:**
- Oracle SQL especifico
- Menos compativel com ecossistema PHP
- Overhead de aprendizado

## Decisao

Escolhemos **MySQL 8.0 (OCI HeatWave)** devido a:

1. **Inclusao no Free Tier:** Banco de dados gerenciado sem custo adicional.

2. **Compatibilidade PHP:** Integracao nativa e bem documentada com PHP via PDO.

3. **Modelo Relacional:** O dominio de ordens de servico e altamente relacional (clientes -> veiculos -> ordens -> servicos -> pecas).

4. **Simplicidade:** Equipe ja possui experiencia com MySQL.

## Justificativa Tecnica do Modelo

### Entidades Principais

| Entidade | Descricao | Relacionamentos |
|----------|-----------|-----------------|
| clients | Clientes (PF/PJ) | 1:N vehicles, 1:N service_orders, 1:N notifications |
| vehicles | Veiculos dos clientes | N:1 clients, 1:N service_orders |
| service_orders | Ordens de servico | N:1 clients, N:1 vehicles, N:M services, N:M parts |
| services | Catalogo de servicos | N:M service_orders |
| parts | Estoque de pecas | N:M service_orders, 1:N part_reservations |

### Decisoes de Modelagem

1. **Precos em Centavos:** Todos os valores monetarios sao armazenados em centavos (INT) para evitar problemas de precisao com DECIMAL/FLOAT.

2. **Tabelas Associativas:** `service_order_services` e `service_order_parts` armazenam quantidade e preco no momento da inclusao, permitindo historico mesmo com alteracoes de catalogo.

3. **Status History:** Tabela separada `service_order_status_history` para auditoria completa de transicoes.

4. **Part Reservations:** Sistema de reserva de pecas com expiracao para evitar conflitos de estoque.

5. **Soft Delete:** Campo `cancelled` em service_orders em vez de exclusao fisica para manter historico.

### Indices

```sql
-- Performance de busca
CREATE INDEX idx_clients_cpf ON clients(cpf);
CREATE INDEX idx_service_orders_status ON service_orders(status);
CREATE INDEX idx_service_orders_created ON service_orders(created_at);

-- Integridade referencial
CREATE INDEX idx_vehicles_client ON vehicles(client_id);
CREATE INDEX idx_so_client ON service_orders(client_id);
CREATE INDEX idx_so_vehicle ON service_orders(vehicle_id);
```

## Consequencias

### Positivas
- Banco de dados sem custo
- Facil manutencao e backup via OCI
- Performance adequada para o volume esperado
- Compatibilidade com ferramentas existentes

### Negativas
- Limitacao a 50GB no Free Tier (suficiente para o projeto)
- Sem alta disponibilidade no Free Tier

## Referencias

- [OCI MySQL HeatWave Documentation](https://docs.oracle.com/en-us/iaas/mysql-database/)
- [MySQL 8.0 Reference Manual](https://dev.mysql.com/doc/refman/8.0/en/)
