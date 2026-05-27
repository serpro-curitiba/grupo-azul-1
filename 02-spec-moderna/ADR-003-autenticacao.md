<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# ADR-003 — Autenticação e Autorização com OAuth2 / JWT (Spring Security)

## Status

✅ **Aceito** · 27/05/2026 · Par 2 (EA + SA) + Par 5 (DevOps).

## Contexto

O legado SIFAP não tem mecanismo explícito de autenticação documentado nos programas Natural — o controle de acesso era feito pelo terminal 3270 via RACF (Resource Access Control Facility) no mainframe. Na migração para uma API REST + Next.js, é necessário definir um mecanismo moderno de autenticação e autorização compatível com o ambiente Azure do governo federal.

Perfis identificados na arqueologia (discovery-report.md §2.3):
- **OPERATOR** — cadastra beneficiários e programas, gera pagamentos
- **ANALYST** — consulta e exporta relatórios
- **ADMIN** — gerencia tabelas configuráveis (fatores regionais, faixas de renda)
- **BATCH** — conta de serviço para jobs automáticos (sem interação humana)

Requisitos de segurança OWASP Top 10 relevantes: A01 (controle de acesso), A02 (falhas criptográficas), A07 (falhas de identificação e autenticação).

## Opções Consideradas

### Opção 1: OAuth2 + JWT via Azure Entra ID (Microsoft Entra ID) — escolhida

- **Prós:** Integração nativa com o tenant Azure do governo federal; MFA incluído; tokens JWT verificados sem estado no backend; auditoria de login gerenciada pelo Entra; compatível com Spring Security OAuth2 Resource Server; Next.js usa `next-auth` com provider Azure.
- **Contras:** Dependência do Azure Entra ID (aceitável, pois o deploy já é Azure); complexidade inicial de configuração de App Registration.

### Opção 2: Keycloak self-hosted

- **Prós:** Open source; independente de cloud; controle total sobre políticas.
- **Contras:** Requer infraestrutura adicional (mais um serviço para operar, monitorar e fazer HA). No contexto do workshop, dobra o esforço de IaC. Sem benefício concreto sobre o Entra ID já disponível no tenant.

### Opção 3: Autenticação customizada (usuário/senha + sessão no banco)

- **Prós:** Zero dependências externas.
- **Contras:** Viola OWASP A07 (implementação própria de auth é fonte clássica de vulnerabilidades). Sem MFA nativo. Sem SSO com outros sistemas do governo. Proibido pelas regras de segurança do workshop.

## Decisão

Adotaremos **Azure Entra ID como Identity Provider** com **Spring Security OAuth2 Resource Server** no backend e **next-auth** (provider Azure) no frontend.

**Fluxo de autenticação:**
```
Usuário → Next.js (next-auth) → Azure Entra ID → JWT (access_token)
                                                         ↓
Next.js → API sifap-api → Spring Security (verifica JWT)
                               ↓
                          @PreAuthorize("hasRole('OPERATOR')")
```

**Mapeamento de papéis (roles) no JWT:**
```yaml
roles:
  OPERATOR:  acesso a POST/PUT em /api/v1/beneficiaries e /api/v1/payments
  ANALYST:   acesso a GET em todos os endpoints de consulta
  ADMIN:     acesso a /api/v1/admin/**
  BATCH:     client_credentials flow (sem usuário humano)
```

**Convenções obrigatórias:**
- Todos os endpoints protegidos com `@PreAuthorize` — sem endpoint público além de `/actuator/health`
- Tokens com expiração máxima de 1 hora; refresh token via Entra ID
- CORS configurado explicitamente — sem wildcard `*` em produção (OWASP A05)
- Secrets (client_id, client_secret) somente via Azure Key Vault — nunca em `application.properties` ou variáveis de ambiente diretas
- Dados sensíveis (CPF, valores de benefício) mascarados em logs — nunca expostos em stack traces

## Consequências

### Positivas

- MFA nativo via Entra ID sem implementação adicional.
- SSO com outros sistemas do governo federal já integrados ao mesmo tenant Azure.
- Auditoria de autenticação gerenciada pelo Entra (log de logins, falhas, IPs).
- Spring Security OAuth2 Resource Server: verificação de JWT sem chamada ao Entra a cada request (verificação local da assinatura).

### Negativas

- Configuração inicial de App Registration no Entra ID requer acesso de admin ao tenant.
- Testes de integração precisam de mock do Entra ID (usar `spring-security-test` com `@WithMockUser`).
- Conta BATCH precisa de App Registration separada com `client_credentials` flow.

## Requisitos Relacionados

- REQ-AUD-001 (auditoria inclui usuário que executou a ação — vem do JWT `sub`)
- REQ-AUD-002 (imutabilidade — somente role BATCH pode fazer INSERT em audit_events)
- REQ-PAY-005 (job batch usa conta de serviço BATCH com client_credentials)

## Quando Revisitar

- Se o governo federal adotar um Identity Provider federal centralizado diferente do Azure Entra ID.
- Se a política de tokens mudar para exigir verificação stateful (ex: revogação imediata).
