<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Decisões de Escopo — SIFAP 2.0

![ESTÁGIO 02 Spec](https://img.shields.io/badge/ESTÁGIO-02%20Spec-00A4EF?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S2](https://img.shields.io/badge/PREENCHA-Durante%20S2-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 2](README.md) → **Scope Decisions**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 2 (Spec Moderna).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento preenchido para sua feature
> 2. Rastreabilidade `source_legacy:` para cada REQ-ID
> 3. Sign-off do Product Owner antes da passagem H2
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Para cada funcionalidade encontrada no Estágio 1, decida: **Migrar**, **Descartar** ou **Evoluir**.
>
> - **Migrar**: trazer para o SIFAP 2.0 como está (mesma lógica, nova tecnologia)
> - **Descartar**: não trazer — funcionalidade obsoleta ou desnecessária
> - **Evoluir**: trazer E melhorar (nova UX, novo fluxo, nova capacidade)

**Time**: grupo-azul-1
**Data**: 27/05/2026
**Edição**: 1.0
**Par 1 (Product Owner) responsável**: Par 1 do grupo-azul-1

## Por que isso importa

O escopo é o que protege o time de chegar às 17h00 com 12 features pela metade. Se o Par 1 não cortar, o Estágio 3 não fecha. **Decisão difícil é tomada aqui, não no Estágio 3.**

## Como decidir

Pergunte de cada funcionalidade:

1. **Afeta o ciclo mensal de pagamento?** Sim → Migrar. Não → considere descartar.
2. **Tem uso documentado nos últimos 12 meses?** Não → descartar.
3. **Faz parte de um relatório regulatório obrigatório (TCU, CGU, BB)?** Sim → Migrar como está.
4. **Tem uma versão moderna mais barata de implementar?** Sim → Evoluir.

---

## Decisões por Funcionalidade

| #   | Funcionalidade | Decisão | Justificativa | Regra de Negócio | Prioridade |
| --- | -------------- | ------- | ------------- | ----------------- | ---------- |
| 1 | Cadastro de Beneficiários (CADBENEF) | **Migrar** | Fluxo principal de entrada de dados; 4,2M benefíciários dependem | BR-011, BR-012 | Alta |
| 2 | Cadastro de Dependentes (CADDEPEND) | **Migrar** | Dados de dependêntes impactam cálculo de benefício | BR-012 | Alta |
| 3 | Cadastro de Programas Sociais (CADPROG) | **Migrar + Evoluir** | Migrar sem FATOR-K implícito; VLR-BASE gravado como informado | BR-013 | Alta |
| 4 | Validação de CPF (VALDOCS) | **Migrar + Evoluir** | Migrar algoritmo MOD-11; REMOVER backdoor 8 prefixos especiais | BR-016 | Alta |
| 5 | Validação Cadastral (VALBENEF) | **Migrar** | Integridade dos dados de entrada | — | Alta |
| 6 | Validação de Elegibilidade (VALELEG) | **Migrar + Evoluir** | Migrar regras A/P/T; REMOVER bypass COD-REGIAO=99 (tornar explícito) | BR-014, BR-017 | Alta |
| 7 | Cálculo do Benefício (CALCBENF) | **Migrar** | Núcleo financeiro; UMA implementação unificada (eliminar duplicata em BATCHPGT) | BR-001–BR-007, BR-010 | Alta |
| 8 | Cálculo de Descontos (CALCDSCT) | **Migrar** | Teto de 30% e contribuição social compulsória | BR-008, BR-009 | Alta |
| 9 | Correção IPCA (CALCCORR) | **Descartar tabela + Evoluir** | Tabela hardcoded parada em 2014 — substituir por API IBGE | BR-021 | Média |
| 10 | Batch de Pagamentos (BATCHPGT) | **Migrar** | Fluxo crítico; unificar lógica de cálculo com CALCBENF | BR-018, BR-019 | Alta |
| 11 | Conciliação CNAB 240 (BATCHCON) | **Migrar + Evoluir** | Manter tolerância R$0,01; tornar layout bancário configurável | BR-020 | Alta |
| 12 | Relatório Consolidado Batch (BATCHREL) | **Migrar** | Obrigatório para prestação de contas | — | Média |
| 13 | Consulta de Beneficiário (CONSBENF) | **Migrar + Evoluir** | Corrigir bug de máscara de CPF; melhorar UX | — | Média |
| 14 | Relatório de Pagamentos (RELPGT) | **Migrar** | Exigido por analistas e auditores | — | Média |
| 15 | Trilha de Auditoria (RELAUDIT) | **Migrar + Evoluir** | CORRIGIR: tornar exclusões (EX) visíveis; conformar IN-TCU 63/2010 | BR-015 | Alta |

> Adicione linhas para cada funcionalidade identificada no `discovery-report.md` do Estágio 1.

---

## Funcionalidades Novas (não existem no legado)

> Liste funcionalidades que o SIFAP 2.0 deveria ter e que não existem no sistema legado. Cada uma vira REQ-ID com `source_legacy: [GREENFIELD] <justificativa>`.

| #   | Funcionalidade Nova | Justificativa | Prioridade | Complexidade |
| --- | ------------------- | ------------- | ---------- | ------------ |
| N1 | Autenticação OAuth2 / JWT (Azure Entra ID) | Legado usava RACF do mainframe — sem equivalente em API REST. Obrigatório para acesso seguro | Alta | Média |
| N2 | API REST com documentação OpenAPI/Swagger | Legado não tinha API; necessário para integração e auditabilidade | Alta | Baixa |
| N3 | Dashboard web (Next.js 15) | Terminal 3270 substituído por interface moderna acessível via browser | Alta | Alta |
| N4 | Imutabilidade da trilha de auditoria (GRANT apenas INSERT) | Legado não tinha controle; exigência IN-TCU 63/2010 | Alta | Baixa |
| N5 | Fatores regionais e faixas de renda configuráveis via banco | Legado tinha tabelas hardcoded que nunca foram atualizadas | Média | Baixa |

---

## Resumo de Escopo

| Decisão | Quantidade | Percentual |
| --------- | ---------- | ---------- |
| Migrar | 8 | 53% |
| Descartar | 1 | 7% |
| Evoluir (Migrar + melhorar) | 6 | 40% |
| **Total** | **15** | 100% |

## Riscos de Escopo

> Liste os riscos das decisões tomadas:

| Risco | Probabilidade | Impacto | Mitigação |
| ----- | ------------- | ------- | --------- |
| FATOR-K=0.347215 não esclarecido antes da migração | Alta | Alto | BLOQUEAR migração de CADPROG até confirmar com SENARC |
| Divergência entre CALCBENF e BATCHPGT já em produção | Média | Alto | Auditar PAGAMENTO histórico antes do go-live; unificar lógica |
| Tabela IPCA desatualizada gera valores errados até o go-live | Alta | Médio | Corrigir como P1 via API IBGE; notificar SENARC dos valores errados desde 2014 |
| Migração de 180M registros de PAGAMENTO do Adabas | Média | Alto | Migrar em lotes noturnos com validação checksums; Par 4 (DBA) lidera |

## Aprovação

- [ ] Par 1 (Product Owner) aprovou as decisões de escopo
- [ ] Par 2 (Enterprise Architect) validou a viabilidade técnica
- [ ] Par 3 (Technical Lead) confirmou que cabe nas 3 horas do Estágio 3
- [ ] Time concordou com as prioridades

> **Aprovação obrigatória na Passagem #2** (~16:00). Sem ela, o Estágio 3 não começa.

— Paula


---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="GUIDE.md"><strong>GUIDE do Estágio 2</strong></a><br/>
<sub>Passo a passo do estágio.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="ADR-TEMPLATE.md"><strong>ADR-TEMPLATE</strong></a><br/>
<sub>Template de ADR.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="../README.md">Voltar ao Kit PT-BR</a></sub>

