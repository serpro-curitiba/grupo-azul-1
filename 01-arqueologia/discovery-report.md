<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Relatório de Descoberta — Estágio 1: Arqueologia Digital

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **discovery-report**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Este documento consolida todas as descobertas do Estágio 1.
> Preencha cada seção com as conclusões do time. **Este é o input principal do Estágio 2** — sem ele, a especificação vira chute.

**Time**: grupo-azul-1
**Data**: 27/05/2026
**Edição**: 1.0 (final Estágio 1)
**Participantes**: Par 1 (PO + RE), Par 2 (EA + SA), Par 3 (TL + Dev), Par 4 (DBA + QA), Par 5 (DevOps + TW)

---

## 1. Sumário Executivo

> Em 3 a 5 frases, resuma o que o time descobriu sobre o SIFAP legado.
> O que é este sistema? Qual sua criticidade? Qual o estado do código?

O SIFAP (Sistema de Fiscalização e Administração de Pagamentos) é um sistema crítico de missão nacional mantido há 29 anos em Natural 6.3.12/Adabas 7.4.3, responsável por gerir **4,2 milhões de beneficiários** e gerar pagamentos mensais de benefícios sociais do governo federal. O código é funcional e estabilizado, porém contém **10 regras de negócio escondidas**, uma constante mágica não documentada (`FATOR-K=0.347215`), e uma tabela de correção IPCA desatualizada desde 2014 — todas acumuladas em 29 anos de evolução orgânica. O risco mais sério é a lógica de cálculo replicada inline em BATCHPGT sem CALLNAT, o que pode ter gerado divergência silenciosa entre pagamentos em lote e cálculos avulsos. A migração é urgentíssima: o suporte ao Natural 6.x e ao Adabas 7.x encerra em 2026.

### 2.1 Propósito do SIFAP

O SIFAP é o motor financeiro de distribuição de benefícios sociais do governo federal. Sua função principal é calcular o valor mensal do benefício de cada família cadastrada (aplicando fórmula multiplicativa com 5 fatores), aplicar descontos compulsórios, gerar um arquivo CNAB 240 para os bancos pagadores e conciliar o retorno. O sistema também mantém a trilha de auditoria exigida pela IN-TCU 63/2010.

### 2.2 Arquitetura Legada

O SIFAP é composto por **15 programas Natural** divididos em 4 módulos funcionais e **4 DDMs Adabas** que servem como único mecanismo de integração entre os programas (não há CALLNAT entre módulos — toda comunicação passa pelo banco Adabas). O fluxo principal é linear: **Cadastro** (CADBENEF → CADDEPEND → CADPROG) → **Validação** (VALDOCS → VALBENEF → VALELEG) → **Cálculo** (CALCBENF → CALCDSCT → CALCCORR) → **Batch** (BATCHPGT → BATCHCON → BATCHREL) → **Consulta/Audit** (CONSBENF, RELPGT, RELAUDIT). Os 4 DDMs são: `BENEFICIARIO` (FNR 150, ~4,2M registros), `PROGRAMA-SOCIAL` (FNR 151, ~45 programas), `PAGAMENTO` (FNR 152, ~180M registros), `AUDITORIA` (FNR 153, ~25M eventos).

### 2.3 Usuários e Perfis

Três perfis identificados nos comentários dos programas:
- **Operador de Cadastro** — acessa CADBENEF, CADDEPEND, CADPROG via terminal 3270 para registrar beneficiários e programas
- **Analista/Auditor** — acessa CONSBENF, RELPGT, RELAUDIT para consultas e relatórios
- **Operador de Batch/TI** — agenda e monitora BATCHPGT, BATCHCON, BATCHREL via JES2 no 1º dia útil de cada mês

---

## 3. Principais Descobertas

### 3.1 Regras de Negócio Críticas

> Liste as 5 regras de negócio mais importantes encontradas.

1. **BR-001** — Fórmula central: `BRUTO = BASE × F_reg × F_fam × F_renda × F_idade × (1 + F_reaj)` — toda a lógica financeira deriva desta fórmula de 5 fatores. `CALCBENF.NSN#L215-L230`
2. **BR-009** — Teto de 30% para descontos — exceto tipos 'J' (Judicial) e 'P' (Pensão) que não têm teto, podendo zerar o benefício. `CALCDSCT.NSN#L105-L125`
3. **BR-018** — Batch ordenado por CPF crescente — sistemas downstream do governo federal dependem desta ordem de processamento. `BATCHPGT.NSN#L180-L190`
4. **BR-014** — COD-REGIAO=99 bypassa completamente todas as validações de elegibilidade. `VALELEG.NSN#L90-L96`
5. **BR-015** — Eventos de exclusão ('EX') nunca aparecem na trilha de auditoria — viola IN-TCU 63/2010. `RELAUDIT.NSN#L90-L95`

### 3.2 Dependências Complexas

> Quais programas estão mais acoplados? Onde há risco de efeito cascata?

[Descreva]

### 3.3 Dívida Técnica Identificada

> Que problemas no código legado vão complicar a migração?

- [ ] Fator K = 0.347215 sem explicação (MYS-001) — é preciso consultar SENARC antes de migrar
- [ ] Tabela IPCA hardcoded parou em 2014 (MYS-003) — correções após 2014 são silenciosamente nulas
- [ ] Lógica de cálculo duplicada entre CALCBENF e BATCHPGT (MYS-009) — possível divergência já em produção
- [ ] Trilha de auditoria incompleta (BR-015) — exclusões não aparecem; provável não-conformidade com TCU
- [ ] Backdoor de CPF (BR-016) — 8 prefixos especiais contornam o MOD-11; origem desconhecida

### 3.4 Gaps de Documentação

> O que a documentação existente NÃO cobre?

Treze dos 15 programas não possuem comentários de cabeçalho com autor/data. A documentação de 2012 cita limites que divergem do código (ex: limite de dependentes = 3 no documento, 5 no código, 10 no DDM). O COD-REGIAO=99 não é mencionado em nenhuma documentação — descoberto apenas por leitura do código. A constante `0.347215` não tem nenhuma referência documental.

---

## 4. Mistérios e Riscos

### 4.1 Mistérios Não Resolvidos

> Resuma os mistérios do arquivo `mysteries-found.md` que permanecem sem explicação.

| ID  | Descrição | Risco para Migração |
| --- | --------- | ------------------- |
| MYS-001 | FATOR-K = 0.347215 sem documentação | ALTO — VLR-BASE gravado já com fator aplicado; recalcular incorretamente muda todos os benefícios |
| MYS-002 | STATUS forçado para 'S' aos 75 anos sem aviso | MÉDIO — comportamento silencioso que deve ser preservado ou tornando explícito |
| MYS-003 | IPCA hardcoded e parado em 2014 | ALTO — todas as correções desde 2014 são silenciosamente zero; migrar sem corrigir perpetua o erro |
| MYS-004 | Banco Real com códigos CNAB não-padrão | BAIXO — layout exclusivo para BB; Banco Real foi adquirido; códigos obsoletos |
| MYS-005 | Limite de dependentes: 3 (doc) vs 5 (código) vs 10 (DDM) | MÉDIO — regra real é 5; DDM pode ser redimensionado |
| MYS-006 | COD-REGIAO=99 bypassa elegibilidade | CRÍTICO — benefícios sem verificação; origem desconhecida; registros com 99 existem em produção |
| MYS-007 | Backdoor de 8 prefixos de CPF | ALTO — possivelmente para testes; deve ser removido em produção |
| MYS-008 | Trilha de auditoria filtra exclusões | CRÍTICO — viola IN-TCU 63/2010; risco de sancão do tribunal |
| MYS-009 | BATCHPGT duplica CALCBENF sem CALLNAT | ALTO — possível divergência já existente entre cálculos avulsos e pagamentos em lote |
| MYS-010 | Pró-rata do 13º nunca implementado | MÉDIO — comentado no código mas não implementado; beneficiários com menos de 12 meses recebem 13º cheio |

### 4.2 Riscos para o Estágio 2

> O que o time de especificação precisa saber antes de começar?

1. **Constante FATOR-K = 0.347215** — consultar urgente a SENARC antes de migrar; qualquer suposição errada muda o valor de 4,2M benefícios
2. **Divergência CALCBENF vs BATCHPGT** — verificar se já existe diferença entre pagamentos batch e cálculos avulsos auditando o histórico de PAGAMENTO
3. **Trilha de auditoria com exclusões suprimidas** — o sistema moderno deve iniciar com auditoria completa para evitar não-conformidade TCU desde o dia 1
4. **Ordenação por CPF no batch** — os sistemas downstream que dependem desta ordenação precisam ser identificados antes de alterar o formato de saída CNAB

---

## 5. Recomendações

### 5.1 O que migrar primeiro

> Com base na priorização do Par 1 (Product Owner), quais funcionalidades devem ser migradas primeiro?

| Prioridade | Funcionalidade | Justificativa |
| ---------- | -------------- | ------------- |
| 1 | Geração de pagamentos mensais (CALCBENF + CALCDSCT + BATCHPGT) | Fluxo de caixa principal; 4,2M famílias dependem de precisão |
| 2 | Cadastro e validação de beneficiários (CADBENEF + VALDOCS + VALBENEF + VALELEG) | Base de dados que alimenta tudo; sem cadastro correto, cálculo é inválido |
| 3 | Conciliação bancária (BATCHCON) | Conformidade financeira; garante que o dinheiro enviado ao banco batem com os registros |
| 4 | Trilha de auditoria completa (RELAUDIT) | Conformidade legal TCU — deve ser corrigida durante a migração |
| 5 | Correção retroativa IPCA (CALCCORR) | Deve ser reescrita integrando API do IBGE; tabela hardcoded obsoleta desde 2014 |

### 5.2 O que descartar

> Funcionalidades que provavelmente não precisam ser migradas:

- **Tabela IPCA hardcoded (CALCCORR.NSN):** Descartar a tabela — migrar integrando API pública do IBGE em tempo real
- **Códigos bancários do Banco Real (EGG-003 em BATCHCON.NSN):** O Banco Real foi adquirido pelo Santander. Os códigos exclusivos não-padrão CNAB devem ser descartados
- **Lookup de regiões por nome literal (EGG-001 em CALCBENF.NSN):** Substituir por tabela configurável no banco de dados PostgreSQL

### 5.3 O que evoluir

> Funcionalidades que devem ser migradas E melhoradas:

- **Trilha de auditoria (RELAUDIT):** Migrar E corrigir — tornar eventos de exclusão visíveis, adicionar timestamp do ator responsável, conformar com IN-TCU 63/2010
- **Validação de CPF (VALDOCS):** Migrar E remover backdoor dos 8 prefixos especiais (ou criar mecanismo explícito de ambiente de teste)
- **13º benefício (CALCBENF):** Migrar E implementar o pro-rata por meses ativos (comentado no código mas nunca codificado)
- **Limite de dependentes:** Migrar E unificar o limite para 5 (regra real do código) com campo configurável por programa social

---

## 6. Métricas do Estágio

| Métrica                       | Valor        |
| ----------------------------- | ------------ |
| Programas analisados          | 15 / 15      |
| DDMs mapeados                 | 4 / 4        |
| Regras de negócio encontradas | 21           |
| Regras escondidas encontradas | 10 / 10      |
| Easter eggs encontrados       | 3 / 3        |
| Termos no glossário           | 40           |
| Mistérios catalogados         | 10           |
| Tempo total gasto             | ~4 horas     |

---

## 7. Notas para o Próximo Estágio

> Deixe aqui mensagens para o time no Estágio 2 (Especificação Moderna):

**Time Estágio 2 (Especificação EARS):**

1. **CONSULTE SENARC sobre o FATOR-K antes de escrever qualquer EARS de cálculo.** A constante `0.347215` está no coração do VLR-BASE gravado em `PROGRAMA-SOCIAL`. Se a suposição estiver errada, toda a especificação de cálculo estará errada.
2. **Cada EARS de cálculo financeiro deve referenciar exatamente uma das duas fontes** (CALCBENF.NSN ou BATCHPGT.NSN) — documente qual é a fonte da verdade; a outra deve ser descontinuada.
3. **A ordenação por CPF no batch é um contrato com o mundo externo** — identifique os sistemas downstream antes de alterar qualquer coisa no formato do arquivo de saída.
4. **A trilha de auditoria deve ser corrigida desde o primeiro commit** — não é opcional; não herde o bug de RELAUDIT.
5. **As faixas de renda (BR-004) e os fatores regionais (BR-002) estão defasados** — são valores de 2013. Inclua uma EARS para torná-los configuráveis no banco de dados.

---

## Definição de Pronto deste relatório

- [ ] Todas as seções acima preenchidas (sem placeholders).
- [ ] Pelo menos 5 regras críticas listadas em §3.1, cada uma referenciando uma `BR-XXX` do catálogo.
- [ ] Decisões de migrar/descartar/evoluir em §5 cobrem as 8+ funcionalidades principais.
- [ ] Métricas de §6 conferem com os outros artefatos (glossary.md, business-rules-catalog.md, mysteries-found.md).

— Paula


---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="mysteries-found.md"><strong>mysteries-found.md</strong></a><br/>
<sub>Lista de mistérios.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="../02-spec-moderna/GUIDE.md"><strong>Estágio 2 — Spec</strong></a><br/>
<sub>Próximo estágio: spec moderna.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="../README.md">Voltar ao Kit PT-BR</a></sub>

