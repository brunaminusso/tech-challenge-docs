# RFC-003: Estrategia de Autenticacao - JWT via CPF

**Status:** Aceito
**Data:** 2026-03-15
**Autor:** Equipe Tech Challenge

## Resumo

Este documento descreve a estrategia de autenticacao implementada, utilizando CPF como identificador e JWT para sessoes.

## Contexto

O sistema requer:
- Autenticacao de clientes para acesso a APIs protegidas
- Validacao de identidade usando documento brasileiro (CPF)
- Sessoes stateless para escalabilidade
- Seguranca contra ataques comuns

## Opcoes Consideradas

### 1. Session-based (Cookies)
**Pros:**
- Facil invalidacao
- Controle total do servidor

**Contras:**
- Stateful (requer storage)
- Problemas com CORS/mobile
- Escalabilidade limitada

### 2. OAuth 2.0 com provedor externo
**Pros:**
- Delegacao de autenticacao
- SSO potencial
- Reducao de responsabilidade

**Contras:**
- Complexidade adicional
- Dependencia externa
- Nao adequado para login por CPF

### 3. JWT (JSON Web Token) - ESCOLHIDO
**Pros:**
- **Stateless:** Nenhum storage de sessao necessario
- **Escalavel:** Qualquer instancia pode validar
- **Padrao:** RFC 7519, amplamente suportado
- **Flexivel:** Claims customizaveis

**Contras:**
- Tokens nao podem ser invalidados antes da expiracao
- Tamanho maior que session ID

## Decisao

Implementamos autenticacao **JWT com login por CPF** usando:

1. **Oracle Function (Serverless):** Processa autenticacao isoladamente
2. **Validacao Completa de CPF:** Algoritmo com digitos verificadores
3. **API Key para Function:** Protecao adicional do endpoint
4. **JWT HS256:** Assinatura simetrica com secret

### Fluxo de Autenticacao

```
Cliente -> Kong -> Oracle Function -> MySQL -> JWT -> Cliente
   |                    |              |        |
   |  POST /auth/cpf    |              |        |
   |  x-api-key: xxx    |              |        |
   |  {cpf: "..."}      |              |        |
   |                    |              |        |
   |                Valida API Key     |        |
   |                Valida CPF         |        |
   |                    |----SELECT--->|        |
   |                    |<---Client----|        |
   |                    |              |        |
   |                    |----jwt.sign--------->|
   |<-----------------200 {token, ...}---------|
```

### Estrutura do JWT

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload:**
```json
{
  "iss": "TechChallenge",
  "sub": 1,
  "cpf": "52998224725",
  "name": "Cliente Demo",
  "role": "client",
  "iat": 1711296000,
  "exp": 1711299600
}
```

### Configuracoes de Seguranca

| Parametro | Valor | Motivo |
|-----------|-------|--------|
| Algoritmo | HS256 | Simplicidade, seguranca adequada |
| Expiracao | 3600s (1h) | Balanco seguranca/UX |
| API Key | Required | Protecao da function |

## Seguranca Implementada

### 1. Validacao de CPF
```javascript
// Algoritmo completo
- Remove caracteres nao-numericos
- Verifica 11 digitos
- Rejeita sequencias repetidas (111.111.111-11)
- Calcula e valida 2 digitos verificadores
```

### 2. Protecao contra Timing Attacks
```javascript
function constantTimeEquals(a, b) {
  if (a.length !== b.length) return false;
  let result = 0;
  for (let i = 0; i < a.length; i++) {
    result |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }
  return result === 0;
}
```

### 3. Rate Limiting (Kong)
- 100 requisicoes/minuto por IP
- Protecao contra brute-force

### 4. CORS Configurado
- Access-Control-Allow-Origin: * (dev)
- Access-Control-Allow-Methods: POST, OPTIONS
- Access-Control-Allow-Headers: Content-Type, Authorization

## Consequencias

### Positivas
- Autenticacao stateless e escalavel
- CPF como identificador natural do cliente
- Funcao serverless isolada (surface attack reduzida)
- Padrao JWT amplamente suportado

### Negativas
- JWT nao pode ser revogado antes da expiracao
- Necessidade de refresh token para sessoes longas (futuro)

## Referencias

- [RFC 7519 - JSON Web Token](https://tools.ietf.org/html/rfc7519)
- [OWASP JWT Security](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet_for_Java.html)
- [Algoritmo CPF](https://www.macoratti.net/alg_cpf.htm)
