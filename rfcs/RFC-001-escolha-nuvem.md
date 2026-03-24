# RFC-001: Escolha da Nuvem - Oracle Cloud Infrastructure

**Status:** Aceito
**Data:** 2026-03-15
**Autor:** Equipe Tech Challenge

## Resumo

Este documento descreve a decisao de utilizar Oracle Cloud Infrastructure (OCI) como provedor de nuvem para o projeto Tech Challenge.

## Contexto

O projeto necessita de infraestrutura de nuvem para hospedar:
- Cluster Kubernetes para a aplicacao
- Banco de dados MySQL gerenciado
- Functions serverless para autenticacao
- API Gateway

## Opcoes Consideradas

### 1. AWS (Amazon Web Services)
**Pros:**
- Maior ecossistema de servicos
- Documentacao extensa
- Ampla adocao no mercado

**Contras:**
- Custos mais elevados para projetos academicos
- Free Tier limitado (12 meses)

### 2. GCP (Google Cloud Platform)
**Pros:**
- GKE com excelente integracao Kubernetes
- Precos competitivos
- Cloud Run para serverless

**Contras:**
- Free Tier limitado
- Custos imprevisı́veis

### 3. Azure
**Pros:**
- Integracao com ferramentas Microsoft
- AKS robusto
- Creditos para estudantes

**Contras:**
- Interface complexa
- Curva de aprendizado

### 4. Oracle Cloud Infrastructure (OCI) - ESCOLHIDA
**Pros:**
- **Always Free Tier generoso e permanente:**
  - 4 OCPUs ARM (VM.Standard.A1.Flex)
  - 24GB RAM total
  - MySQL Database Service incluido
  - Oracle Functions (5M invocacoes/mes)
  - 200GB Object Storage
- MySQL nativo (HeatWave)
- Oracle Functions com Fn Project (open-source)
- Precos competitivos apos Free Tier
- OKE (Oracle Kubernetes Engine) compativel

**Contras:**
- Menor comunidade comparada a AWS
- Documentacao menos extensa
- Menos recursos educacionais

## Decisao

Escolhemos **Oracle Cloud Infrastructure (OCI)** devido ao:

1. **Always Free Tier Permanente:** Diferente de AWS/GCP/Azure, o Free Tier da OCI nao expira, permitindo manter o projeto ativo indefinidamente para demonstracoes.

2. **Recursos ARM Generosos:** 4 OCPUs e 24GB RAM permitem executar cluster Kubernetes completo gratuitamente.

3. **MySQL Nativo:** OCI MySQL HeatWave e um MySQL gerenciado de primeira classe, sem necessidade de RDS ou Cloud SQL.

4. **Oracle Functions:** Baseado no Fn Project open-source, compativel com padres do mercado.

## Consequencias

### Positivas
- Zero custo para ambiente de staging e demonstracao
- MySQL gerenciado com alta disponibilidade
- Kubernetes nativo (OKE)
- Functions serverless integradas

### Negativas
- Necessidade de aprender SDK/CLI OCI especifico
- Menos exemplos e tutoriais disponveis online
- Menor portabilidade para outras nuvens

## Referencias

- [OCI Always Free Resources](https://www.oracle.com/cloud/free/)
- [OCI MySQL HeatWave](https://www.oracle.com/mysql/)
- [Oracle Kubernetes Engine](https://www.oracle.com/cloud/compute/container-engine-kubernetes/)
