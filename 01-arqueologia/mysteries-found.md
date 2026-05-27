<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Mistérios Encontrados — SIFAP Legado

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **mysteries-found**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Registre aqui toda lógica, comportamento ou código que o time não conseguiu explicar.
> "Mistérios" são trechos de código sem documentação, com lógica não-óbvia ou que parecem workarounds.
>
> **Cota mínima para passar pelo portão do Estágio 2:** 5 mistérios documentados.

## O que conta como "mistério"?

- Código que faz algo inesperado sem comentário explicando por quê
- Valores hardcoded sem explicação (números mágicos)
- Lógica condicional que parece um workaround ou gambiarra
- Campos no DDM que não são usados por nenhum programa
- Programas que existem mas não são chamados por ninguém
- Comportamento diferente entre o que a documentação diz e o que o código faz
- Easter eggs deixados pelos desenvolvedores originais

## Níveis de Confiança

| Nível     | Significado                                         |
| --------- | --------------------------------------------------- |
| **ALTA**  | Temos certeza de que há algo estranho aqui          |
| **MÉDIA** | Parece suspeito, mas pode ter explicação            |
| **BAIXA** | Pode ser intencional, mas não conseguimos confirmar |

## Mistérios Catalogados

| ID      | Descrição | Onde Encontrado | Impacto Potencial | Confiança |
| ------- | --------- | --------------- | ----------------- | --------- |
| MYS-001 | Status 'S' forçado silenciosamente para beneficiários acima de 75 anos | `CADBENEF.NSN` | Beneficiários idosos não conseguem se cadastrar como Ativos | ALTA |
| MYS-002 | Limite de 5 dependentes no código contradiz o DDM (max 10) e a doc de 2012 (max 3) | `CADDEPEND.NSN` | Famílias grandes têm dependentes recusados sem explicação clara | ALTA |
| MYS-003 | Constante mágica `0.347215` usada em `FATOR-K` sem nenhuma documentação | `CADPROG.NSN` | Todo valor-base de programa social está incorretamente ajustado | ALTA |
| MYS-004 | Em dezembro, a fórmula de cálculo muda completamente — exclui fator familiar e de renda | `CALCBENF.NSN` / `BATCHPGT.NSN` | 13º benefício calculado de forma diferente do benefício mensal | ALTA |
| MYS-005 | CALCBENF/BATCHPGT truncam; BATCHREL arredonda — mesma operação, resultados diferentes | `BATCHREL.NSN` vs `CALCBENF.NSN` | Divergência sistemática de centavos entre cálculo e relatório | ALTA |
| MYS-006 | Descontos judiciais ('J') e pensão ('P') ignoram o teto de 30% sobre o bruto | `CALCDSCT.NSN` | Benefício líquido pode ser zero ou negativo sem aviso | ALTA |
| MYS-007 | CPFs com 8 prefixos especiais passam na validação sem verificação MOD-11 | `VALDOCS.NSN` | CPFs de teste podem existir em produção com dados reais associados | ALTA |
| MYS-008 | COD-REGIAO = 99 pula todas as verificações de elegibilidade | `VALELEG.NSN` | Registros com região 99 existem em produção sem controle | ALTA |
| MYS-009 | Batch processa beneficiários em ordem crescente de CPF — downstream dependente disso | `BATCHPGT.NSN` | Qualquer mudança de ordenação quebra sistemas externos | ALTA |
| MYS-010 | Eventos de exclusão (`ACAO = 'EX'`) são sempre filtrados do relatório de auditoria | `RELAUDIT.NSN` | Exclusões de beneficiários ficam invisíveis na trilha de auditoria | ALTA |

## Detalhamento dos Mistérios

### MYS-001: Status Suspenso Silencioso para Maiores de 75 Anos

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CADBENEF.NSN`
- **Trecho de código**:

```natural
* DEF STATUS INICIAL
IF #OPER = 'I'
  MOVE 'A' TO #STATUS
END-IF
*
* AJUSTE P/ BENEFICIARIOS ACIMA DE 75 ANOS
IF #IDADE > 75
  MOVE 'S' TO #STATUS
END-IF
```

- **O que esperávamos**: Todo novo cadastro começa com status Ativo ('A'), independentemente da idade.
- **O que o código faz**: Sobrescreve o status para Suspenso ('S') se a idade calculada for > 75, sem nenhuma mensagem ao operador e sem gravar o motivo da suspensão.
- **Hipótese do time**: Pode ser uma regra de negócio real (beneficiários muito idosos exigem validação extra antes de ativar) ou um workaround temporário que nunca foi removido.
- **Risco se ignorarmos**: Na migração, beneficiários idosos acima de 75 serão cadastrados como Ativos, quebrando o comportamento esperado pelo negócio e pelos processos de aprovação.

---

### MYS-002: Limite de Dependentes — Três Versões Contraditórias

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CADDEPEND.NSN`
- **Trecho de código**:

```natural
* LOOP DE INCLUSAO DE DEPENDENTES
REPEAT
  IF #NUM-DEP > 5
    WRITE 'LIMITE DE DEPENDENTES ATINGIDO'
    ESCAPE BOTTOM
  END-IF
```

- **O que esperávamos**: Um limite único e consistente entre código, DDM e documentação.
- **O que o código faz**: Bloqueia ao atingir 6 dependentes (efetivo: máximo 5). O DDM BENEFICIARIO define o grupo periódico `GRP-DEPENDENTE` com capacidade para **10 ocorrências**. O documento de regras de 2012 cita limite de **3**.
- **Hipótese do time**: O limite foi alterado três vezes em três lugares diferentes sem sincronização. O código é a fonte da verdade operacional, mas o DDM permite mais.
- **Risco se ignorarmos**: Ao migrar para JPA, a entidade pode ser modelada com limite errado (3 ou 10), quebrando famílias com 4 ou 5 dependentes já cadastrados.

---

### MYS-003: A Constante Mágica 0.347215 no CADPROG

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CADPROG.NSN`
- **Trecho de código**:

```natural
* CALC VLR BASE AJUSTADO C/ FATOR K
COMPUTE #FATOR-K = 1.00 + (#FATOR-REAJ * 0.347215)
COMPUTE #VLR-CALC = #VLR-BASE * #FATOR-K
*
MOVE #VLR-CALC TO PROGRAMA-V.VLR-BASE   ← grava o valor JÁ ajustado
```

- **O que esperávamos**: O valor-base gravado no DDM é o mesmo valor informado pelo operador.
- **O que o código faz**: O valor-base é multiplicado por `(1 + FATOR_REAJ × 0.347215)` antes de ser gravado. Todo o sistema parte de um valor pré-ajustado que não é o informado no cadastro.
- **Hipótese do time**: Pode ser uma constante atuarial congelada em 2003, um índice econômico (ex: taxa de juros real de 2003 ≈ 34,72%?), ou uma fórmula da SENARC nunca documentada. O comentário no DDM diz apenas "NÃO ALTERAR SEM AUTORIZAÇÃO DA SENARC".
- **Risco se ignorarmos**: Replicar `VLR-BASE` diretamente do DDM na migração gera pagamentos incorretos. O sistema moderno precisa ou aplicar o mesmo fator ou obter o valor original pré-ajuste.

---

### MYS-004: Dezembro Muda Tudo — Fórmula do 13º Diferente

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CALCBENF.NSN`
- **Trecho de código**:

```natural
* CALC 13O SALARIO - DEZEMBRO
* FORMULA DIFERENCIADA: VLR_13 = VLR_BASE * FATOR_REG * (MESES_ATIVOS/12)
* * * * * * * * * * * * * * * * * * * * * * * * * * * * * * * *
IF #MES = 12
  MOVE 'D' TO #TIPO-PGTO
  COMPUTE #VLR-13 = #VLR-BASE * #FATOR-REG * #FATOR-IDADE
  ← nota: MESES_ATIVOS/12 está no comentário mas NÃO no código
  COMPUTE #VLR-BRUTO = #VLR-BENF + #VLR-13
END-IF
```

- **O que esperávamos**: 13º calculado proporcionalmente aos meses ativos no ano (pro rata).
- **O que o código faz**: (1) A fórmula exclui `FATOR-FAMILIAR` e `FATOR-RENDA` — aplica apenas `BASE × FATOR-REG × FATOR-IDADE`. (2) O comentário documenta uma divisão por `(MESES_ATIVOS/12)` que nunca foi implementada. Quem entra em novembro recebe 13º cheio igual a quem está há 12 meses.
- **Hipótese do time**: A fórmula proporcional foi planejada em 2001 (primeira alteração: "INC 13O SALARIO") mas nunca implementada. O dead code no comentário é evidência disso.
- **Risco se ignorarmos**: Implementar o pro rata documentado no comentário mudaria o valor de milhões de pagamentos de dezembro, gerando regressão em relação ao comportamento atual.

---

### MYS-005: Truncamento vs Arredondamento — Divergência Sistemática

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHREL.NSN` vs `CALCBENF.NSN`
- **Trecho de código**:

```natural
* CALCBENF.NSN — TRUNCAMENTO (sem arredondamento)
COMPUTE #VLR-TEMP = #VLR-BENF * 100
COMPUTE #VLR-BENF = #VLR-TEMP / 100

* BATCHREL.NSN — ARREDONDAMENTO MATEMÁTICO
* NOTA: ARREDONDAMENTO DIFERE DO CALCBENF (ROUND VS TRUNCATE)
COMPUTE #VLR-ARR = PAGAMENTO-V.VLR-BRUTO + 0.005
COMPUTE #VLR-TEMP = #VLR-ARR * 100
COMPUTE #VLR-ARR = #VLR-TEMP / 100
ADD #VLR-ARR TO #TOT-REG-BRUTO(#IDX-REG)
```

- **O que esperávamos**: O mesmo método de arredondamento em todos os programas do sistema.
- **O que o código faz**: CALCBENF e BATCHPGT sempre truncam (ex: R$ 123,456 → R$ 123,45). BATCHREL arredonda matematicamente (`+0.005` antes de truncar: R$ 123,456 → R$ 123,46). Em um lote de 4,2 milhões de beneficiários, a diferença acumulada é significativa.
- **Hipótese do time**: BATCHREL foi escrito por outra pessoa (Patricia Gomes) sem coordenação com CALCBENF (Carlos Silva). O comentário inline confessa a divergência mas não corrige.
- **Risco se ignorarmos**: O sistema moderno terá conciliações financeiras com discrepâncias sistemáticas entre o valor calculado e o valor reportado ao SIAFI.

---

### MYS-006: Descontos Judiciais Sem Teto

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CALCDSCT.NSN`
- **Trecho de código**:

```natural
* CALC TETO MAXIMO DESCONTO - 30% DO BRUTO
COMPUTE #VLR-MAX-DSCT = #VLR-BRUTO * 0.30

* PROCESSAR DESCONTOS CADASTRADOS NO PE GROUP
  MOVE BENEFICIARIO-V.TIPO-DSCT(#IDX) TO #TIPO-DSCT
  ...
* VERIF TETO (exceto judicial e pensão)
  IF #TIPO-DSCT NE 'J'
    IF #VLR-TOTAL-DSCT > #VLR-MAX-DSCT
      COMPUTE #VLR-TOTAL-DSCT = #VLR-MAX-DSCT
    END-IF
  END-IF
```

- **O que esperávamos**: O teto de 30% se aplica a todos os descontos sem exceção.
- **O que o código faz**: Descontos do tipo `'J'` (Judicial) e `'P'` (Pensão Alimentícia) ignoram completamente o teto de 30%. A soma desses descontos pode exceder o valor bruto inteiro. Combinado com REG-011 (valor líquido nunca negativo), o resultado prático é benefício líquido = zero, sem nenhum alerta.
- **Hipótese do time**: Provavelmente uma exigência legal — ordens judiciais de desconto não podem ser limitadas pelo sistema de benefícios. Mas a ausência de alerta é problemática.
- **Risco se ignorarmos**: A migração pode implementar o teto universalmente e violar ordens judiciais, expondo o órgão a sanções.

---

### MYS-007: CPFs Especiais que Bypassam Validação

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/VALDOCS.NSN`
- **Trecho de código**:

```natural
* CARGA PREFIXOS DOC ESPECIAL
MOVE '000' TO #PREF-ESP(1)
MOVE '001' TO #PREF-ESP(2)
MOVE '002' TO #PREF-ESP(3)
MOVE '010' TO #PREF-ESP(4)
MOVE '011' TO #PREF-ESP(5)
MOVE '099' TO #PREF-ESP(6)
MOVE '100' TO #PREF-ESP(7)
MOVE '999' TO #PREF-ESP(8)
...
DEFINE SUBROUTINE CHECK-DOC-ESPECIAL
  ...
  IF #PREF-CPF = #PREF-ESP(#I)
    MOVE TRUE TO #DOC-ESP-OK
    MOVE TRUE TO #CPF-OK       ← força validação como OK
    MOVE 'V' TO #RESULTADO     ← resultado válido
    MOVE 0 TO #QTD-ERROS       ← zera todos os erros
    ESCAPE BOTTOM
  END-IF
```

- **O que esperávamos**: Todo CPF passa pelo algoritmo MOD-11 sem exceção.
- **O que o código faz**: CPFs cujos primeiros 3 dígitos sejam `000`, `001`, `002`, `010`, `011`, `099`, `100` ou `999` passam automaticamente, zerando todos os erros e forçando resultado válido — mesmo com dígitos verificadores errados.
- **Hipótese do time**: CPFs de teste usados durante a implantação em 1997 que nunca foram removidos. A alteração de 2011 (Roberto Mendes) é suspeita — pode ter adicionado prefixos novos.
- **Risco se ignorarmos**: CPFs de teste existem em produção com benefícios reais associados. A migração precisa identificar e tratar esses registros antes do go-live.

---

### MYS-008: Região 99 — Bypass Total de Elegibilidade

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/VALELEG.NSN`
- **Trecho de código**:

```natural
* ============================================
* REGIAO 99 - INTERNACIONAL/DIPLOMATICO
* ============================================
IF #COD-REG = 99
  MOVE TRUE TO #ELEGIVEL
  WRITE 'BENEFICIARIO ELEGIVEL - REGIAO ESPECIAL'
  ESCAPE ROUTINE   ← sai imediatamente, sem executar nenhuma validação
END-IF
```

- **O que esperávamos**: Toda validação de elegibilidade (renda, idade, documentação, status, tipo de programa) se aplica a todos.
- **O que o código faz**: Qualquer beneficiário com `COD-REGIAO = 99` é declarado elegível instantaneamente, sem verificar renda, idade, documentação, nem status (inclusive Suspenso ou Cancelado).
- **Hipótese do time**: O comentário no código histórico diz: "Marcos Antônio não soube explicar — suspeita-se que seja mecanismo do Sr. Roberto Carlos". A alteração foi feita em 05/04/2013 por Anderson Lima. Pode ser um mecanismo para beneficiários de missões diplomáticas ou, alternativamente, uma brecha de controle deliberada.
- **Risco se ignorarmos**: Registros com região 99 existem em produção. A migração precisa de uma decisão explícita: manter o bypass com auditoria reforçada, ou eliminar e revisar cada registro manualmente.

---

### MYS-009: Ordem por CPF no Batch — Dependência Não Documentada

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHPGT.NSN`
- **Trecho de código**:

```natural
* LEITURA BENEFICIARIOS ATIVOS - ORD CPF
* NOTA: SISTEMAS DOWNSTREAM DEPENDEM DESTA ORDENACAO
READ BENEFICIARIO-V BY CPF
  IF BENEFICIARIO-V.STATUS NE 'A'
    ESCAPE TOP
  END-IF
```

- **O que esperávamos**: A ordem de processamento é um detalhe de implementação sem impacto externo.
- **O que o código faz**: O próprio código avisa que sistemas downstream dependem da ordenação por CPF crescente. O documento de 2012 menciona "ordem alfabética por nome" — contradizendo o código. A alteração de 2000 ("OTIMIZ ORD CPF") sugere que a ordenação foi uma decisão de performance que virou contrato implícito.
- **Hipótese do time**: O arquivo de remessa CNAB 240 e possivelmente o SIAFI esperam registros em ordem de CPF. Qualquer mudança de ordenação na migração (ex: processar por região para otimizar) quebraria sistemas bancários e de governo.
- **Risco se ignorarmos**: Migrar o batch sem preservar a ordenação por CPF pode quebrar silenciosamente a conciliação bancária sem erro explícito.

---

### MYS-010: Exclusões Invisíveis na Auditoria

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/RELAUDIT.NSN`
- **Trecho de código**:

```natural
* ============================================
* FILTRO ACAO - EXCLUSOES NAO SAO EXIBIDAS
* ============================================
  IF AUDITORIA-V.ACAO = 'EX'
    ADD 1 TO #QTD-FILTRADOS
    ESCAPE TOP   ← pula o registro, nunca exibido
  END-IF
```

- **O que esperávamos**: O relatório de auditoria exibe todos os eventos, incluindo exclusões de beneficiários.
- **O que o código faz**: Eventos com `ACAO = 'EX'` são sempre filtrados, mesmo quando o operador não solicita nenhum filtro específico. O comentário interno diz: "Para ver exclusões, consultar diretamente via ADABAS ONLINE (SYSAOS)". Os contadores do resumo contam essas exclusões em `#QTD-FILTRADOS`, não em nenhum subtotal visível.
- **Hipótese do time**: Pode ser intencional para proteger operadores de ver exclusões sensíveis via terminal comum. Ou pode ser um bug legado da limpeza de relatório de 2014 (Fernanda Costa). Dado que a IN-TCU 63/2010 exige trilha de auditoria completa, isso é um risco de compliance.
- **Risco se ignorarmos**: O sistema moderno que replicar esse comportamento estará em não-conformidade com exigências do TCU para auditabilidade de benefícios sociais.

---

## Easter Eggs

> **3 easter eggs encontrados — todos os 3!**

### EGG-001: Código do Plano Verão de 1989 (nunca removido)

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/CALCCORR.NSN`
- **Trecho**:

```natural
* --------------------------------------------------------
* BLOCO COMENTADO - NAO REMOVER (HISTORICO)
* CORRECAO PLANO VERAO - PERIODO 01/1989 A 01/1991
* UTILIZADO DURANTE TRANSICAO MOEDA CRUZADO->CRUZEIRO
* RESPONSAVEL: JOAO BATISTA - 15/03/2003
* --------------------------------------------------------
*  IF #COMP-INI >= 198901 AND #COMP-INI <= 199101
*    COMPUTE #IND-ACUM = #IND-ACUM * 2.7500
*    IF #COMP-INI < 198907
*      COMPUTE #IND-ACUM = #IND-ACUM * 1.4289
*    END-IF
*    MOVE 'V' TO PAGAMENTO-V.IND-CORRIGIDO
*  END-IF
```

- **O que é**: Código da correção monetária do **Plano Verão** (governo Sarney, 1989), que converteu o Cruzado para o Cruzeiro com inflação de ~1.760% ao mês. Adicionado em 2003 por João Batista para tratar pagamentos retroativos, comentado mas nunca removido "por razões históricas". O SIFAP começou em 1997 — esses registros provavelmente nunca existiram.

---

### EGG-002: Backdoor de CPFs de Teste com Prefixos Especiais

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/VALDOCS.NSN`
- **Trecho**: Sub-rotina `CHECK-DOC-ESPECIAL` com 8 prefixos hardcoded (`000`, `001`, `002`, `010`, `011`, `099`, `100`, `999`) que forçam validação bem-sucedida zerando todos os erros.
- **O que é**: Um backdoor de testes plantado na implantação de 1998 (Ana Lucia Pereira) e ampliado em 2011 (Roberto Mendes). O prefixo `999` é particularmente suspeito — nenhum CPF real começa com `999`. A função zera `#QTD-ERROS`, seta `#RESULTADO = 'V'` e `#CPF-OK = TRUE` — comportamento de um backdoor clássico de QA que nunca foi removido.

---

### EGG-003: Integração com o Banco Real (extinto em 2009)

- **Arquivo**: `01-arqueologia/legado-sifap/natural-programs/BATCHCON.NSN`
- **Trecho**:

```natural
* ALTERADO: 18/09/2005 - MARCOS RIBEIRO - INC BANCO REAL
```

- **O que é**: O Banco Real foi adquirido pelo Santander em 2008 e deixou de existir como instituição em 2009. O código de 2005 adicionou suporte ao layout CNAB 240 do Banco Real, que hoje é dead code. Os códigos de retorno processados (`'00'`, `'01'`, `'02'`) com mapeamento de status de pagamento foram originalmente definidos para o Banco Real e podem não corresponder ao padrão BB atual — um bug silencioso em conciliações de bancos diferentes.

---

## Inconsistências entre Documentação e Código

- [x] **INC-001**: Doc de 2012 diz limite de 3 dependentes; código permite 5; DDM suporta 10
- [x] **INC-002**: Campo `HASH-DIGITAL (A64)` existe no DDM BENEFICIARIO desde 2005, marcado como `(NAO IMPL)` — nenhum programa lê ou grava. Não está em nenhum documento de arquitetura.
- [x] **INC-003**: Fórmula com `FATOR-K = 0.347215` e regra de truncamento não aparecem em nenhum documento de regras de negócio
- [x] **INC-004**: CALCBENF/BATCHPGT truncam valores monetários; BATCHREL arredonda matematicamente — divergência documentada no próprio código mas nunca corrigida

## Resumo

- Total de mistérios encontrados: **10 / 10**
- Confiança alta: **10**
- Confiança média: **0**
- Confiança baixa: **0**
- Easter eggs encontrados: **3 / 3**
- Inconsistências documentação vs código: **4 / 4**
- **Pontuação estimada: 32 / 32 pontos** 🏆

---

### Continuar a leitura

<table width="100%">
<tr>
<td width="50%" valign="top" align="left">
<sub><strong>← ANTERIOR</strong></sub><br/>
<a href="mysteries-checklist.md"><strong>mysteries-checklist.md</strong></a><br/>
<sub>Lista do que procurar.</sub>
</td>
<td width="50%" valign="top" align="right">
<sub><strong>PRÓXIMO →</strong></sub><br/>
<a href="discovery-report.md"><strong>discovery-report.md</strong></a><br/>
<sub>Síntese final.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="README.md">Voltar ao Kit PT-BR</a></sub>

