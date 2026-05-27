<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# ADR-002 — Persistência com PostgreSQL 16 + JPA/Hibernate

## Status

✅ **Aceito** · 27/05/2026 · Par 2 (EA + SA) + Par 4 (DBA).

## Contexto

O legado usa Adabas 7.4.3 como único mecanismo de armazenamento — um banco NoSQL hierárquico com suporte a grupos periódicos (PE) e campos multi-valor (MU), estruturas sem equivalente direto em SQL relacional. Os 4 DDMs totalizam ~180 milhões de registros de pagamento e ~4,2 milhões de beneficiários.

A migração precisa preservar todas as regras de negócio que dependem de estrutura de dados (BR-012: grupos periódicos de dependentes; BR-018: ordenação por CPF; BR-020: campos de conciliação CNAB). O banco escolhido deve suportar:

- Transações ACID (requisito de auditoria imutável — REQ-AUD-002)
- Ordenação garantida por CPF sem custo extra (BR-018)
- Arrays nativos para substituir grupos periódicos Adabas
- Suporte a esquemas isolados por módulo (ADR-001)

## Opções Consideradas

### Opção 1: PostgreSQL 16 + JPA/Hibernate (escolhida)

- **Prós:** ACID completo; arrays nativos substituem grupos PE do Adabas; schemas por módulo já suportados; equipe tem mais familiaridade; Testcontainers com imagem oficial para testes; Azure Database for PostgreSQL Flexible Server para produção.
- **Contras:** Mapeamento de grupos periódicos Adabas (PE) para arrays PostgreSQL requer atenção; JPA pode esconder N+1 queries se não monitorado.

### Opção 2: MongoDB

- **Prós:** Estrutura de documento mapeia naturalmente grupos periódicos do Adabas.
- **Contras:** Sem suporte a JOINs transacionais entre coleções — crítico para conciliação (REQ-PAY-006) que une Payment e Auditoria. Equipe sem experiência com otimização de queries MongoDB para volumes de 180M documentos.

### Opção 3: Oracle Database

- **Prós:** Amplamente usado em sistemas governamentais; suporte nativo a PL/SQL para lógica de batch.
- **Contras:** Custo de licenciamento incompatível com o orçamento. Lock-in de fornecedor. Nenhuma vantagem técnica sobre PostgreSQL para este caso.

## Decisão

Adotaremos **PostgreSQL 16** com **JPA/Hibernate** (Spring Data JPA) e as seguintes convenções:

**Schemas por módulo (ADR-001):**
```sql
CREATE SCHEMA beneficiary;
CREATE SCHEMA payment;
CREATE SCHEMA audit;
CREATE SCHEMA admin;
```

**Mapeamento de grupos periódicos Adabas → arrays PostgreSQL:**
```java
// BENEFICIARIO.GRP-DEPENDENTE (PE no Adabas) → @ElementCollection em JPA
@ElementCollection
@CollectionTable(schema = "beneficiary", name = "dependents")
private List<Dependent> dependents; // máximo 5 (REQ-BEN-004)
```

**Convenções obrigatórias:**
- `@Transactional` somente na camada `application/` (nunca em `infrastructure/repositories`)
- Migrations via **Flyway** (`V{N}__descricao.sql`) — nunca `ddl-auto=update` em produção
- Queries complexas de batch com `@Query` JPQL ou `JdbcTemplate` (sem concatenação de strings — OWASP A03)
- `@Version` em todas as entidades mutáveis para controle de concorrência otimista
- Auditoria: schema `audit` com GRANT somente INSERT para o usuário da aplicação (REQ-AUD-002)

## Consequências

### Positivas

- ACID garante consistência da trilha de auditoria imutável (REQ-AUD-002).
- Testcontainers com PostgreSQL elimina mocks de banco nos testes de integração.
- Azure Database for PostgreSQL Flexible Server: backup automático, HA e patch managment gerenciados.
- Flyway garante rastreabilidade de schema — auditável pelo TCU.

### Negativas

- Mapeamento de PE/MU do Adabas para SQL requer análise cuidadosa DDM a DDM (Par 4 — DBA).
- JPA pode gerar N+1 queries em relatórios de pagamento — monitorar com `spring.jpa.show-sql` em staging.
- Dados históricos (180M registros de PAGAMENTO) precisam de estratégia de migração em lotes.

## Requisitos Relacionados

- REQ-BEN-004 (limite de dependentes → array)
- REQ-PAY-005 (ordenação por CPF → ORDER BY cpf ASC)
- REQ-AUD-002 (imutabilidade → GRANT apenas INSERT)
- REQ-ADM-002 (IPCA via API — sem tabela hardcoded no banco)

## Quando Revisitar

- Se o volume de PAGAMENTO superar 500M registros e consultas de relatório ultrapassarem SLA de 2s — considerar particionamento por `competencia` ou columnar storage (Citus).
