# ADR-003: Arquitetura DDD com Clean Architecture

**Status:** Aceito
**Data:** 2026-03-15

## Contexto

O sistema de ordens de servico possui regras de negocio complexas (state machine, validacao de estoque, notificacoes) que devem ser mantidas independentes de frameworks e infraestrutura.

## Decisao

Adotar **Domain-Driven Design (DDD)** combinado com **Clean Architecture** em 4 camadas.

### Estrutura

```
src/
├── Presentation/     # Controllers, DTOs de entrada/saida
├── Application/      # Use Cases (orquestracao)
├── Domain/           # Entidades, Value Objects, Interfaces
└── Infrastructure/   # Implementacoes concretas (MySQL, HTTP)
```

### Regras de Dependencia

```
Presentation -> Application -> Domain <- Infrastructure
                    |                        |
                    +------------------------+
                    (Application pode usar Infrastructure via interfaces)
```

### Componentes por Camada

**Domain:**
- Entities: Client, Vehicle, ServiceOrder, Part, Service
- Enums: OrderStatus
- Repository Interfaces: ServiceOrderRepositoryInterface
- Domain Services: (se necessario)

**Application:**
- Use Cases: CreateServiceOrderUseCase, TransitionStatusUseCase
- DTOs: CreateOrderRequest, OrderResponse

**Infrastructure:**
- Repositories: MySQLServiceOrderRepository
- External Services: DatadogLogger

**Presentation:**
- Controllers: ServiceOrderController
- Middleware: AuthMiddleware

## Consequencias

### Positivas
- Regras de negocio isoladas e testaveis
- Facil substituicao de infraestrutura (MySQL -> PostgreSQL)
- Codigo expressivo (ubiquitous language)
- Manutencao simplificada

### Negativas
- Mais arquivos e classes
- Curva de aprendizado
- Pode parecer overengineering para CRUD simples

## Exemplo: Fluxo CreateServiceOrder

```php
Controller (Presentation)
    |
    v
CreateServiceOrderUseCase (Application)
    |
    +-> ClientRepository->findById()
    +-> VehicleRepository->findById()
    +-> new ServiceOrder(...) [Domain]
    +-> ServiceOrderRepository->save()
    |
    v
Controller serializa e retorna
```
