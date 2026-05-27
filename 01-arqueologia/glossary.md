<!-- markdownlint-disable MD013 MD025 MD026 MD028 MD029 MD034 MD040 MD051 MD060 -->

# Glossário do SIFAP Legado

![ESTÁGIO 01 Arqueologia](https://img.shields.io/badge/ESTÁGIO-01%20Arqueologia-F25022?style=for-the-badge) ![TIPO Worksheet](https://img.shields.io/badge/TIPO-Worksheet-1A1A1A?style=for-the-badge) ![PREENCHA Durante S1](https://img.shields.io/badge/PREENCHA-Durante%20S1-737373?style=for-the-badge)

> 🗺 **Você está aqui:** [Kit PT-BR](../README.md) → [Estágio 1](README.md) → **glossary**

> **Para quem é isto?** Este é um **artefato preenchido pelo time** durante o Estágio 1 (Arqueologia).
>
> **O que você terá ao final do estágio:**
>
> 1. Este documento totalmente preenchido com os dados reais do legado SIFAP
> 2. Rastreabilidade para `01-arqueologia/legado-sifap/` (programas `.NSN` e DDMs)
> 3. Base de evidência usada nas EARS do Estágio 2 (`source_legacy:`)
>
> 📘 **Guia passo a passo:** [`GUIDE.md`](GUIDE.md).


> Preencha esta tabela com todos os termos, abreviações e siglas encontrados no código Natural/Adabas.
> **Meta: no mínimo 30 termos.**

## Por que isso importa

Sistemas legados têm vocabulário próprio que ninguém documenta em lugar nenhum — só está no nome das variáveis. Se o time do Estágio 2 não souber o que `DSCT`, `BENF`, `PE` ou `CTC` significam, vai escrever uma spec sobre o que ele _acha_ que isso significa. Glossário é o que evita esse desencontro.

## Como preencher

- **Termo**: a abreviação ou sigla exatamente como aparece no código
- **Expansão**: o significado completo do termo
- **Programa**: em qual arquivo `.NSN` ou `.ddm` o termo foi encontrado
- **Contexto**: breve explicação de como/onde o termo é usado

## Dica de extração

Prompt útil no Copilot Chat (cole o conteúdo de 2–3 arquivos `.NSN` no chat antes):

> _"Liste todas as abreviações e siglas usadas neste código Natural. Para cada uma, sugira a expansão e marque com 'CONFIRMADO' ou 'HIPÓTESE'."_

## Termos encontrados

| #   | Termo | Expansão | Programa | Contexto |
| --- | ----- | -------- | -------- | -------- |
| 1   | `SIFAP` | Sistema de Fiscalização e Administração de Pagamentos | Todos os `.NSN` | Nome do sistema legado; aparece no cabeçalho de todos os programas |
| 2   | `BENEF` / `BENEFICIARIO` | Beneficiário | `CADBENEF.NSN`, `BENEFICIARIO.ddm` | Pessoa cadastrada que recebe o benefício social |
| 3   | `CPF` | Cadastro de Pessoas Físicas | Todos os `.NSN` | Identificador principal do beneficiário; 11 dígitos com MOD-11 |
| 4   | `NIS` | Número de Identificação Social | `CADBENEF.NSN`, `CONSBENF.NSN` | Identificador alternativo ao CPF para busca de beneficiário |
| 5   | `DDM` | Data Definition Module | `*.ddm` | Estrutura de dados do Adabas — equivalente a um schema de tabela |
| 6   | `PE` | Periodic Group (Grupo Periódico) | `BENEFICIARIO.ddm`, `CALCDSCT.NSN` | Campo que pode ter múltiplas ocorrências em um registro Adabas (ex: lista de dependentes) |
| 7   | `MU` | Multiple Value Field (Campo Multi-Valor) | `BENEFICIARIO.ddm`, `PROGRAMA-SOCIAL.ddm` | Campo Adabas que armazena múltiplos valores em uma única ocorrência |
| 8   | `FNR` | File Number (Número do Arquivo) | Comentários dos `.NSN` | Identificador numérico do arquivo Adabas. FNR 150=BENEFICIARIO, 151=PROGRAMA-SOCIAL, 152=PAGAMENTO, 153=AUDITORIA |
| 9   | `ARQ 150` | Arquivo 150 — BENEFICIARIO | Comentários dos `.NSN` | Referência ao arquivo Adabas de beneficiários (nomenclatura interna nos comentários, equivale a FNR 150) |
| 10  | `ARQ 160` | Arquivo 160 — PAGAMENTO | Comentários dos `.NSN` | Referência ao arquivo de pagamentos (equivale a FNR 152; divergência de numeração nos comentários) |
| 11  | `DSCT` | Desconto | `CALCDSCT.NSN`, `PAGAMENTO.ddm` | Dedução aplicada sobre o valor bruto. Tipos: C=Contribuição, I=Imposto, J=Judicial, S=Sindical, P=Pensão, A=Administrativo |
| 12  | `VLR` | Valor | Todos os `.NSN` | Prefixo usado em variáveis monetárias (ex: `VLR-BRUTO`, `VLR-LIQUIDO`, `VLR-DESCONTO`) |
| 13  | `VLR-BRUTO` | Valor Bruto | `CALCBENF.NSN`, `PAGAMENTO.ddm` | Valor do benefício antes da aplicação de descontos |
| 14  | `VLR-LIQ` / `VLR-LIQUIDO` | Valor Líquido | `CALCBENF.NSN`, `PAGAMENTO.ddm` | Valor efetivamente creditado ao beneficiário após descontos |
| 15  | `PGTO` | Pagamento | `BATCHPGT.NSN`, `PAGAMENTO.ddm` | Registro de um crédito gerado para um beneficiário em uma competência |
| 16  | `COMPETENCIA` | Período de Referência (AAAAMM) | `BATCHPGT.NSN`, `PAGAMENTO.ddm` | Ano e mês de referência do pagamento no formato 6 dígitos (ex: 202605 = maio 2026) |
| 17  | `FATOR-REG` | Fator Regional | `CALCBENF.NSN`, `BATCHPGT.NSN` | Multiplicador do benefício por UF (1.00 a 1.40); armazenado na tabela `#TAB-REG(27)` |
| 18  | `FATOR-FAM` | Fator Familiar | `CALCBENF.NSN` | Multiplicador por número de dependentes; aumenta progressivamente com retorno decrescente |
| 19  | `FATOR-RND` | Fator de Renda | `CALCBENF.NSN` | Multiplicador baseado na renda familiar declarada; varia de 0.40 a 1.00 |
| 20  | `FATOR-K` | Fator K (constante misteriosa) | `CADPROG.NSN` | Constante `0.347215` usada para ajustar VLR-BASE no cadastro de programas; origem não documentada |
| 21  | `FATOR-REAJ` | Fator de Reajuste | `PROGRAMA-SOCIAL.ddm`, `CALCBENF.NSN` | Percentual anual de reajuste aplicado sobre o benefício calculado |
| 22  | `COD-REGIAO` | Código de Região | `BENEFICIARIO.ddm`, `VALELEG.NSN` | 01-25 = UF específica; 99 = bypass total de elegibilidade (Internacional/Diplomático) |
| 23  | `COD-PROG` | Código de Programa | `BENEFICIARIO.ddm`, `PROGRAMA-SOCIAL.ddm` | Identificador de 4 dígitos do programa social ao qual o beneficiário está vinculado |
| 24  | `STATUS` / `SIT` | Status / Situação | Todos os `.NSN` | A=Ativo, S=Suspenso, C=Cancelado, I=Inativo, D=Desligado (beneficiário); G=Gerado, P=Pago, D=Devolvido, E=Estornado, X=Cancelado (pagamento) |
| 25  | `TIPO-PGTO` | Tipo de Pagamento | `CALCBENF.NSN`, `PAGAMENTO.ddm` | N=Normal (mensal), D=Décimo terceiro (dezembro), T=Terceiro (não implementado) |
| 26  | `ACAO` | Ação de Auditoria | `AUDITORIA.ddm`, `RELAUDIT.NSN` | IN=Inclusão, AL=Alteração, EX=Exclusão, CO=Conciliação, CN=Consulta, DV=Divergência |
| 27  | `CNAB 240` | Centro Nacional de Automação Bancária — layout de 240 bytes | `BATCHCON.NSN` | Padrão de arquivo bancário para transferência de dados de pagamentos entre o SIFAP e os bancos |
| 28  | `CALLNAT` | Call Natural (chamada a sub-programa) | N/A — não utilizado | Instrução Natural para chamar outro programa; BATCHPGT não usa CALLNAT para CALCBENF — replica lógica inline |
| 29  | `PERFORM` | Executar sub-rotina | Todos os `.NSN` | Chamada a uma `DEFINE SUBROUTINE` local — equivalente a um método privado |
| 30  | `FIND` / `READ` | Busca no Adabas | Todos os `.NSN` | FIND = busca por descritor (índice); READ = leitura sequencial por ISN ou descritor ordenado |
| 31  | `STORE` | Gravar novo registro Adabas | `CADBENEF.NSN`, `BATCHPGT.NSN` | Equivalente a `INSERT` em SQL |
| 32  | `UPDATE` | Atualizar registro Adabas | `BATCHCON.NSN`, `CALCCORR.NSN` | Equivalente a `UPDATE` em SQL; requer `END TRANSACTION` para confirmar |
| 33  | `ESCAPE ROUTINE` | Sair da sub-rotina | `VALELEG.NSN`, `CALCDSCT.NSN` | Equivalente a `return` — sai imediatamente da rotina atual |
| 34  | `ESCAPE TOP` | Voltar ao início do loop | Vários `.NSN` | Equivalente a `continue` — pula para a próxima iteração do loop |
| 35  | `ESCAPE BOTTOM` | Sair do loop | Vários `.NSN` | Equivalente a `break` — encerra o loop atual |
| 36  | `IPCA` | Índice Nacional de Preços ao Consumidor Amplo | `CALCCORR.NSN` | Índice de inflação do IBGE usado para correção retroativa de pagamentos; tabela hardcoded até 2014 |
| 37  | `SIAFI` | Sistema Integrado de Administração Financeira do Governo Federal | Comentários de DDMs | Sistema de gestão financeira federal ao qual o SIFAP deve conciliar pagamentos |
| 38  | `SUPDE` / `DESIF` | Superintendência de Desenvolvimento / Departamento de Sistemas Fiscais | `BENEFICIARIO.ddm` | Unidades organizacionais responsáveis pelo SIFAP |
| 39  | `TCU` | Tribunal de Contas da União | `AUDITORIA.ddm`, `RELAUDIT.NSN` | Órgão de controle externo; IN-TCU 63/2010 exige trilha de auditoria completa e imutável |
| 40  | `ISN` | Internal Sequence Number | DDMs Adabas | Identificador interno de registro no Adabas; equivalente ao `rowid` do Oracle |

> Adicione mais linhas conforme necessário. Não se limite a 30!

## Exemplo de linha bem preenchida

| #   | Termo  | Expansão | Programa                        | Contexto                                                                                                         |
| --- | ------ | -------- | ------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| 1   | `DSCT` | Desconto | `CALCDSCT.NSN`, `PAGAMENTO.ddm` | Tipo de dedução aplicada sobre valor bruto do pagamento. Tipos: 'J' (judicial), 'I' (imposto), 'T' (trabalhista) |

## Observações

- Anote aqui qualquer padrão de nomenclatura que o time identificou:
- Convenções de prefixo/sufixo encontradas:
- Termos ambíguos que precisam de validação com especialista:

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
<a href="business-rules-catalog.md"><strong>business-rules-catalog.md</strong></a><br/>
<sub>Catálogo de regras.</sub>
</td>
</tr>
</table>

<sub>↑ <a href="README.md">Voltar ao Kit PT-BR</a></sub>

