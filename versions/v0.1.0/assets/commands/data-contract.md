---
description: schematize-data — define/audita o CONTRATO de dados: schema versionado, compatibilidade (back/forward/full), registry e política de evolução; trava breaking change sem nova versão
argument-hint: "[dataset/tópico/tabela/evento alvo]"
---

Defina ou audite o **contrato de dados** de `${ARGUMENTS:-<fronteira alvo>}`
(`references/contratos.md`). O dado que cruza fronteira é uma **promessa de forma** — sem contrato
versionado, o consumidor quebra na próxima mudança.

## 1. Identifique a fronteira (§1)
É contrato? (evento/tópico, tabela/view que **outro** serviço ou o BI lê, arquivo do lake, API de
dados, stream de CDC). Dado **interno** privado não é — pode mudar livre. Proteja a **borda**.

## 2. Modele direito antes de versionar (§2)
Uma unidade semântica por registro/evento; **chave estável** (UUIDv7/ULID) + chave de negócio pra
dedupe/idempotência; tipos honestos (dinheiro inteiro/decimal, **UTC** ISO-8601, enum fechado);
nulabilidade é decisão; **semântica documentada** (o nome não basta). Separe fato de projeção.

## 3. Versione e declare a compatibilidade (§3-4)
- Versão **explícita** (`orders.v1`, `/v1`, namespace). Consumidor sabe o que lê.
- Escolha o **modo** por fronteira e **registre**: **backward** (novo lê antigo), **forward**
  (antigo lê novo — o padrão da casa: consumidor **ignora campo desconhecido**), **full**.
- **SEGURO (evolutivo):** adicionar campo **opcional com default**; afrouxar obrigatoriedade.
- **QUEBRA (exige `v2` + janela de coexistência):** remover/renomear campo, mudar tipo, estreitar,
  tornar obrigatório, **mudar a semântica** (mesmo nome, novo significado — a mais traiçoeira),
  mudar unidade sem renomear.

## 4. Registry como gate (§5)
Schema como **artefato versionado** (Avro/Protobuf/JSON Schema/Iceberg) no repo/registry; **checagem
de compatibilidade no CI** que **falha** a mudança que viola o modo declarado; gerar tipos do
schema nos dois lados (não escrever à mão). Contrato consumer-driven se houver poucos consumidores.

## 5. Gate
Breaking change **sem `v2` + janela documentada** → **não passa**. Consumidores conhecidos (do
**lineage**, `references/governanca.md`) precisam da janela pra migrar. Grave o contrato + história
+ modo de compatibilidade + janelas abertas em `<projeto>_archive/data/contracts/` (§28).

> Se outro serviço lê, é contrato; se é contrato, é versionado; se quebra, é `v2` com janela — o
> consumidor nunca descobre a mudança em produção.
