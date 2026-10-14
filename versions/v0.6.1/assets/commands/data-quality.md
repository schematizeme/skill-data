---
description: schematize-data — monta/audita o portão de qualidade: validação na borda, testes de dados/expectations, quarentena e dead-letter, SLA; dado ruim não passa pro downstream
argument-hint: "[dataset/pipeline alvo]"
---

Monte ou audite o **portão de qualidade** de `${ARGUMENTS:-<dataset alvo>}`
(`references/qualidade.md`). Código passa por teste; **dado, também**. O piso: **dado que reprova
vai pra quarentena — nunca contamina o downstream, nunca some em silêncio.**

## 1. Validar na borda (§1)
Valide **no ponto de ingestão**, contra o **contrato** (`references/contratos.md`): forma, tipos,
obrigatoriedade, faixa, enum, referência. **Rejeição explícita, não coerção silenciosa** — coagir/
truncar/assumir default **fabrica** dado e esconde a origem quebrada. PII que não pode pousar é
dropada/mascarada **aqui** (`references/governanca.md`).

## 2. Cobrir as dimensões (§2)
Declare o **esperado** por dataset: schema/validade, **completude**, **unicidade** (chave sem
duplicata), **faixa/sanidade**, **integridade referencial**, **freshness**, **volume** (queda
súbita = origem quebrada), **distribuição/deriva** (drift).

## 3. Testes de dados — expectations (§3)
Asserções **versionadas** (dbt tests / Great Expectations / Soda / constraints). Dois momentos:
**no CI** (a transformação produz a forma esperada, com fixtures) **e** **em runtime** (as
expectations rodam sobre o **dado real** de cada execução — pega o lixo da origem que o CI não vê).
Dado crítico da DoD (dinheiro/PII/caminho crítico, §35) roda o portão em runtime **sempre**.

## 4. Quarentena e dead-letter (§4) — o piso
O que reprova vai pra **quarentena** com **motivo** (regra que falhou) + **origem** (arquivo/offset/
partição/`trace_id`) + timestamp: (a) **não contamina** o dataset bom; (b) **não derruba** o lote
inteiro (isola o registro); (c) **não some** — **alerta** (§16) e é **revisada**; (d) é
**reprocessável** pelo mesmo pipeline idempotente após corrigir origem/regra.

## 5. SLA e gate (§5-6)
**SLA de dados** por dataset (freshness/completude/erro/disponibilidade) com **dono explícito**
(do catálogo); budget estourado **freia** a promoção pra gold. Anti-padrões **vetados**: coerção
silenciosa, `try/catch` que engole a linha, derrubar o lote por um registro, métrica que ninguém
olha, "a origem é confiável". Grave expectations/SLA + relatórios de qualidade (o que reprovou,
quanto quarentenou, deriva) em `<projeto>_archive/data/quality/` (§28).

> Dado é suspeito até validar; o que reprova vai pra quarentena com motivo e origem, alerta, e é
> reprocessável — nunca contamina o bom, nunca some.
