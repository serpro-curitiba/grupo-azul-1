<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Catálogo de Regras de Negócio — SIFAP Legado

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **business-rules-catalog**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Registre aqui todas as regras de negócio extraídas do código Natural/Adabas.
> Cada regra precisa ter rastreabilidade até o código-fonte.
>
> **REGRA DURA:** linhas com `Programa Fonte` vazio são **inválidas** e não contam para o gate do Estágio 2. Use o formato `01-arqueologia/legado-sifap/natural-programs/ARQUIVO.NSN#L<inicio>-L<fim>` sempre que possível. Mínimo aceito: nome do arquivo .NSN.

## Como pensar em "regra de negócio"

O que conta:

- Um `IF` que decide algo no domínio (ex.: _"se a UF é do Nordeste e o programa é Seca, valor base × 1.2"_)
- Uma constante numérica sem explicação (ex.: `0.075` num cálculo de imposto)
- Uma transição de status com regra (ex.: _"só de A para S, nunca de I para A"_)
- Um tratamento especial para um caso (ex.: _"se o CPF começa com 999, é teste"_)

O que NÃO conta: paginação de relatório, formatação de saída, manipulação de cursor Adabas, abertura de arquivo. Ignore esses detalhes de implementação.

## Níveis de Risco

| Nível       | Descrição                                                     |
| ----------- | ------------------------------------------------------------- |
| **CRÍTICO** | Regra financeira ou de segurança — erro causa prejuízo direto |
| **ALTO**    | Regra de negócio central — afeta fluxo principal              |
| **MÉDIO**   | Regra de validação ou formatação — afeta qualidade dos dados  |
| **BAIXO**   | Regra de apresentação ou conveniência — impacto limitado      |

## Regras Encontradas

| ID     | Regra de Negócio | Programa Fonte | Campos DDM | Nível de Risco | Notas |
| ------ | ---------------- | -------------- | ---------- | -------------- | ----- |
| BR-001 | Fórmula central do benefício: `BRUTO = BASE × F_reg × F_fam × F_renda × F_idade × (1 + F_reaj)` | `CALCBENF.NSN#L215-L230` | `PAGAMENTO.VLR-BRUTO`, `PROGRAMA-SOCIAL.VLR-BASE`, `PROGRAMA-SOCIAL.FATOR-REAJUSTE` | CRÍTICO | Fórmula replicada em BATCHPGT.NSN sem CALLNAT — risco de divergência |
| BR-002 | Fator regional hardcoded por UF (25 valores): Nordeste até 1.40, Sul até 1.03, ref=1.00 | `CALCBENF.NSN#L90-L118` | `BENEFICIARIO.COD-REGIAO` | CRÍTICO | Também duplicado em BATCHPGT.NSN#L127-L153 |
| BR-003 | Fator familiar por dependentes: 0 dep=1.00; 1-2 dep: +5% cada; 3-4 dep: +3% cada; 5+: +2% cada; sem teto | `CALCBENF.NSN#L155-L175` | `BENEFICIARIO.NUM-DEPENDENTES` | CRÍTICO | Sem teto máximo — famílias muito grandes acumulam fator indefinidamente |
| BR-004 | Faixas de renda como fator redutor: até R$300=1.00; até R$600=0.85; até R$1000=0.70; até R$1500=0.55; acima=0.40 | `CALCBENF.NSN#L120-L135` | `BENEFICIARIO.RENDA-FAMILIAR` | CRÍTICO | Faixas em valores absolutos — nunca foram reajustadas desde 2013 |
| BR-005 | Fator de idade: <18 anos=1.05; 18-59=1.00; 60-64=1.10; ≥65=1.15 | `CALCBENF.NSN#L201-L215` | `BENEFICIARIO.DT-NASCIMENTO` | ALTO | Idade calculada por diferença de anos, sem considerar mês/dia exatos |
| BR-006 | Em dezembro (MES=12), o tipo de pagamento é 'D' e o 13º salário usa fórmula diferente: `VLR-13 = BASE × F_reg × F_idade` (sem F_fam e F_renda) | `CALCBENF.NSN#L237-L255` | `PAGAMENTO.TIPO-PGTO`, `PAGAMENTO.VLR-BRUTO` | CRÍTICO | Pro-rata por meses ativos documentado no comentário mas nunca implementado |
| BR-007 | Abono natalino de 15% sobre VLR-BENF exclusivo para programas tipo 'A' (Assistencial) em dezembro | `CALCBENF.NSN#L255-L265` | `PROGRAMA-SOCIAL.TIPO`, `PAGAMENTO.VLR-ABONO` | ALTO | Tipos 'P' (Previdenciário) e 'T' (Trabalho) não recebem abono |
| BR-008 | Contribuição social compulsória por faixas do bruto: até R$500=3%; até R$1000=5%; até R$2000=7%; acima=9% (alíquota sobre o total, não progressiva) | `CALCDSCT.NSN#L40-L80` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-DESCONTO` | CRÍTICO | Tabela regressiva simples — alíquota sobre valor total, não sobre cada faixa |
| BR-009 | Teto de 30% para soma de descontos sobre o bruto — exceto tipos 'J' (Judicial) e 'P' (Pensão Alimentícia) que não têm teto | `CALCDSCT.NSN#L105-L125` | `PAGAMENTO.VLR-BRUTO`, `BENEFICIARIO.TIPO-DSCT` | CRÍTICO | Desconto judicial pode zerar o benefício; valor líquido é forçado a zero se negativo |
| BR-010 | Todos os valores monetários são truncados (não arredondados): `VLR-TEMP = VLR * 100; VLR = VLR-TEMP / 100` | `CALCBENF.NSN#L230-L235` | Todos os campos `VLR-*` de PAGAMENTO | CRÍTICO | BATCHREL usa arredondamento matemático (+0.005), divergindo dos cálculos individuais |
| BR-011 | Beneficiário com idade >75 anos tem status forçado para 'S' (Suspenso) na inclusão — sem mensagem ao operador | `CADBENEF.NSN#L165-L170` | `BENEFICIARIO.STATUS`, `BENEFICIARIO.DT-NASCIMENTO` | ALTO | Regra silenciosa — operador não recebe aviso; motivo não é gravado no registro |
| BR-012 | Limite de 5 dependentes por família em CADDEPEND — DDM suporta 10, documentação 2012 cita 3 | `CADDEPEND.NSN#L55` | `BENEFICIARIO.NUM-DEPENDENTES`, `BENEFICIARIO.GRP-DEPENDENTE(PE)` | ALTO | Três fontes contraditórias: código=5, DDM=10, doc=3 |
| BR-013 | FATOR-K misterioso no cadastro de programas: `FATOR-K = 1.00 + (FATOR_REAJ × 0.347215)`, VLR-BASE é gravado já com fator aplicado | `CADPROG.NSN#L105-L115` | `PROGRAMA-SOCIAL.VLR-BASE`, `PROGRAMA-SOCIAL.FATOR-REAJUSTE` | CRÍTICO | Constante 0.347215 sem documentação — o VLR-BASE no DDM não é o valor informado pelo operador |
| BR-014 | COD-REGIAO=99 bypassa completamente todas as validações de elegibilidade em VALELEG | `VALELEG.NSN#L90-L96` | `BENEFICIARIO.COD-REGIAO` | CRÍTICO | Comentário histórico: "Marcos Antônio não soube explicar". Registros com região 99 existem em produção |
| BR-015 | Eventos de exclusão (ACAO='EX') são sempre filtrados do relatório de auditoria, independente dos filtros informados pelo operador | `RELAUDIT.NSN#L90-L95` | `AUDITORIA.ACAO` | ALTO | Viola IN-TCU 63/2010 — exclusões invisíveis na trilha de auditoria |
| BR-016 | Validação CPF por algoritmo MOD-11 com exceção: CPFs com todos os dígitos iguais são inválidos, exceto os 8 prefixos especiais (000,001,002,010,011,099,100,999) | `VALDOCS.NSN#L60-L120` | `BENEFICIARIO.CPF` | ALTO | Backdoor de testes — CPFs com esses prefixos passam sem verificação MOD-11 |
| BR-017 | Elegibilidade por tipo de programa: Assistencial(A)=renda≤600 sem dependente falha; Previdenciário(P)=idade≥60; Trabalho(T)=16≤idade≤65 | `VALELEG.NSN#L140-L175` | `PROGRAMA-SOCIAL.TIPO`, `BENEFICIARIO.RENDA-FAMILIAR`, `BENEFICIARIO.DT-NASCIMENTO` | ALTO | Regras distintas por tipo — um mesmo beneficiário pode ser elegível para A mas não para P |
| BR-018 | Batch BATCHPGT processa beneficiários ordenados por CPF crescente — sistemas downstream dependem dessa ordem | `BATCHPGT.NSN#L180-L190` | `BENEFICIARIO.CPF` | CRÍTICO | Nota no código: "SISTEMAS DOWNSTREAM DEPENDEM DESTA ORDENACAO" |
| BR-019 | Idempotência do batch: se já existe pagamento para o mesmo CPF+competência, o beneficiário é ignorado (não duplica) | `BATCHPGT.NSN#L200-L215` | `PAGAMENTO.CPF-BENEF`, `PAGAMENTO.COMPETENCIA` | CRÍTICO | Proteção exactly-once; também verifica CPF duplicado consecutivo via #CPF-ANT |
| BR-020 | Conciliação bancária CNAB 240 aceita divergência de até R$0,01 entre valor SIFAP e valor retornado pelo banco | `BATCHCON.NSN#L120-L175` | `PAGAMENTO.VLR-LIQUIDO` | MÉDIO | Posições CNAB hardcoded para layout Banco do Brasil (CPF bytes 44-54, valor bytes 120-134) |
| BR-021 | Correção retroativa por IPCA aplica índice acumulado sobre VLR-BRUTO; tabela hardcoded apenas para 2010-2014; pagamentos já corrigidos (IND-CORRIGIDO='S') são ignorados | `CALCCORR.NSN#L80-L140` | `PAGAMENTO.VLR-CORRECAO`, `PAGAMENTO.IND-CORRIGIDO` | ALTO | Tabela IPCA desatualizada desde 2014 — correções após 2014 retornam índice=1.0 (sem correção) |

> Adicione mais linhas conforme necessário. Lembre-se: existem **10 regras escondidas** no código!

## Exemplo de linha bem preenchida

| ID     | Regra de Negócio                                                                        | Programa Fonte                                   | Campos DDM                                                               | Nível de Risco | Notas                                      |
| ------ | --------------------------------------------------------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------ | -------------- | ------------------------------------------ |
| BR-013 | Desconto total não pode exceder 30% do valor bruto, exceto descontos judiciais (tipo J) | `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN#L142-L148` | `PAGAMENTO.VLR-BRUTO`, `PAGAMENTO.VLR-TOTAL-DSCT`, `PAGAMENTO.TIPO-DSCT` | CRÍTICO        | Regra financeira. Tipo 'J' = exceção legal |

## Regras por Categoria

### Cálculos Financeiros

- **BR-001** — Fórmula central multiplicativa (5 fatores)
- **BR-002** — Tabela de fatores regionais por UF (25 valores hardcoded)
- **BR-003** — Fator familiar com retorno marginal decrescente (sem teto)
- **BR-004** — Faixas de renda como fator redutor (5 faixas)
- **BR-005** — Fator de idade para idosos e menores
- **BR-006** — 13º benefício em dezembro com fórmula diferente
- **BR-007** — Abono natalino 15% exclusivo para programas tipo 'A'
- **BR-008** — Contribuição social compulsória por faixas (3%/5%/7%/9%)
- **BR-009** — Teto de 30% para descontos (exceto judicial e pensão)
- **BR-010** — Truncamento em vez de arredondamento em todos os valores
- **BR-013** — FATOR-K = 0.347215 aplicado silenciosamente ao VLR-BASE
- **BR-020** — Tolerância de R$0,01 na conciliação CNAB 240
- **BR-021** — Correção retroativa por IPCA (tabela parada em 2014)

### Validações de Status

- **BR-011** — Status 'S' forçado silenciosamente para beneficiários >75 anos no cadastro
- **BR-015** — Eventos de exclusão ('EX') invisíveis no relatório de auditoria
- **BR-019** — Idempotência do batch — ignora CPF+competência já processados

### Regras de Autorização e Bypass

- **BR-014** — COD-REGIAO=99 bypassa todas as validações de elegibilidade
- **BR-016** — 8 prefixos de CPF especiais bypassam validação MOD-11
- **BR-017** — Elegibilidade diferenciada por tipo de programa (A/P/T)

### Regras de Negócio Temporais

- **BR-006** — Lógica especial de dezembro (13º salário)
- **BR-018** — Batch executado no 1º dia útil de cada mês, ordenado por CPF
- **BR-012** — Limite de 5 dependentes (hard limit no código)

## Resumo Estatístico

- Total de regras encontradas: **21**
- Regras críticas: **10** (BR-001, BR-002, BR-003, BR-004, BR-008, BR-009, BR-010, BR-013, BR-014, BR-018, BR-019)
- Regras com duplicação entre programas: **2** (BR-001 e BR-002 replicadas em BATCHPGT e CALCBENF)
- Regras sem documentação (escondidas/mistérios): **8** (BR-011, BR-012, BR-013, BR-014, BR-015, BR-016, BR-019, BR-020)

---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="GUIDE.md"><strong>GUIDE do Estágio 1</strong></a><br/>
<sub>Passo a passo do estágio.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="dependency-map.md"><strong>dependency-map.md</strong></a><br/>
<sub>Mapa de quem chama quem.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="README.md">Voltar ao Kit PT-BR</a></sub>

