<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# ADR-001 — Adotar Monolito Modular em vez de Microsserviços

## Status

✅ **Aceito** · 27/05/2026 · Decisão sob responsabilidade do Par 2 (EA + SA) com revisão do PO.

## Contexto

Estamos modernizando o SIFAP, sistema de 29 anos com **4 domínios bem definidos** (beneficiary, payment, audit, admin) e ~4,2 milhões de beneficiários. Temos janela de um dia de workshop para entregar protótipo funcional.

Restrições conhecidas:

- **Time pequeno** (5 pares) sem experiência prévia com infraestrutura de microsserviços em produção.
- **Domínios fortemente acoplados pela auditoria** — quase toda mutação em beneficiário ou pagamento gera evento de auditoria. Comunicação cross-service via mensageria adicionaria latência e complexidade sem benefício proporcional.
- **Legado sem separação de concerns** — os 15 programas .NSN comunicam-se exclusivamente via DDMs Adabas compartilhados, evidenciando acoplamento de dados (não de serviços).
- **Suporte Natural/Adabas encerra em 2026** — urgência exige entrega rápida e segura, não arquitetura ideal de longo prazo.

## Opções Consideradas

### Opção 1: Monolito Modular (escolhida)

- **Prós:** 1 deployable, 1 pipeline CI/CD, 1 banco PostgreSQL (schemas isolados), fronteiras de domínio visíveis via ArchUnit, migração futura para microsserviços módulo a módulo.
- **Contras:** Escalabilidade horizontal limitada por módulo; risco de acoplamento acidental se fronteiras não forem verificadas.

### Opção 2: Microsserviços desde o dia 1

- **Prós:** Escalabilidade horizontal por domínio; deploy independente por time.
- **Contras:** Service mesh, observabilidade distribuída, contratos OpenAPI versionados e 4 pipelines CI/CD independentes consomem o tempo disponível inteiro. Time não tem skill operacional para 4 serviços em produção simultaneamente. Latência de chamadas síncronas entre audit/payment quebraria SLA de p95 < 200ms.

### Opção 3: Monolito tradicional (sem módulos)

- **Prós:** Mais simples de implementar inicialmente.
- **Contras:** Perde clareza de domínio — os 4 bounded contexts se misturariam em pacotes por camada (`controllers/`, `services/`), exatamente o anti-padrão do legado. Refatoração futura para microsserviços ficaria inviável sem fronteiras claras.

## Decisão

Adotaremos **monolito modular** em Java 21 + Spring Boot 3.3, com a seguinte estrutura de pacotes:

```
src/main/java/br/gov/sifap/
├── beneficiary/
│   ├── domain/         (entidades, value objects, interfaces — puro Java, sem Spring)
│   ├── application/    (services, use cases)
│   └── infrastructure/ (controllers REST, repositories JPA, adapters)
├── payment/
├── audit/
└── admin/
```

**Regras de fronteira verificadas pelo ArchUnit no CI:**
1. Nenhum módulo importa classes de `infrastructure/` de outro módulo.
2. Comunicação inter-módulo somente via interfaces declaradas em `domain/`.
3. Banco compartilhado com schemas PostgreSQL separados: `beneficiary`, `payment`, `audit`, `admin`.
4. Cada módulo só faz JOIN dentro do próprio schema.

**Caminho de evolução:** se um módulo precisar escalar independentemente no futuro, o `audit` será o primeiro candidato à extração (menor acoplamento de escrita).

## Consequências

### Positivas

- Pipeline CI/CD único — menos overhead operacional.
- Fronteiras de domínio documentadas e verificadas automaticamente.
- Facilita debugging e rastreamento de transações cross-módulo em um único trace.
- Banco único simplifica a migração dos dados do Adabas.

### Negativas

- Escalabilidade horizontal é por instância (todo o monolito), não por módulo.
- Risco de acoplamento acidental se ArchUnit não for mantido atualizado.
- Releases envolvem deploy de todos os módulos simultaneamente.

## Requisitos Relacionados

- REQ-BEN-001, REQ-BEN-002, REQ-PAY-001, REQ-AUD-001, REQ-ADM-001
- Ver `SPECIFICATION.md` para lista completa de REQ-IDs.

## Quando Revisitar

- Se o módulo `payment` atingir carga > 10.000 req/s e não puder escalar horizontalmente com o monolito.
- Se times distintos precisarem de ciclos de release independentes por módulo.
