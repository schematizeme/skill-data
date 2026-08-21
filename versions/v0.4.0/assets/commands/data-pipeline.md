---
description: schematize-data — desenha/audita o pipeline (ETL/ELT batch, streaming, CDC, event sourcing): idempotência, Outbox (sem dual-write), replay/reprocessamento, backpressure e DLQ
argument-hint: "[pipeline/fluxo alvo]"
---

Desenhe ou audite o **pipeline** `${ARGUMENTS:-<fluxo alvo>}` (`references/pipelines.md`). A
promessa: rodar a cadeia — 1x, 2x, ou de novo sobre uma janela antiga — produz **o mesmo estado
correto**.

## 1. Padrão de movimento (§1-3)
- **ETL vs ELT:** transforma antes ou depois de pousar? **Preserve o cru** (bronze) quando o
  destino aguenta — reprocessar exige a origem.
- **Batch:** particione por tempo/chave; **watermark** idempotente; re-run **substitui** (MERGE/
  upsert por chave), não duplica (append cego); **backfill** é operação planejada e reversível.
- **Streaming:** assuma **at-least-once**; ordem só **por chave de partição**; **backpressure**
  (concorrência/prefetch); **DLQ** obrigatória (retry limitado com backoff — **infinito é vetado**).

## 2. Idempotência (§4) — o piso que torna o replay seguro
Dedupe por id do evento/chave natural; escrita idempotente no destino (MERGE, `ON CONFLICT`);
**efeito colateral externo** (cobrança/e-mail/terceiro) dedupado por `Idempotency-Key` **de fato**.
Rodar 2x = mesmo efeito que 1x, senão retry/replay corrompem.

## 3. Entrega — Outbox, sem dual-write (§5)
Gravar no banco **e** publicar no broker no mesmo fluxo sem transação (**dual-write**) é **VETADO**.
Use **Transactional Outbox** (evento + estado na mesma transação; relay publica) ou **CDC** (lê o
WAL/binlog — evento derivado do commit). Consumidor dedupa (entrega at-least-once).

## 4. Event sourcing / CDC (§6)
Se o histórico é o ativo: **log de eventos é a fonte da verdade**; estado é **projeção
reconstruível** (fold); eventos **imutáveis/versionados** (corrige com compensatório, nunca edita o
passado); snapshots reconstruíveis; projetor entende `v1`/`v2` na coexistência.

## 5. Reprocessamento/replay (§7) — sem incidente
**Determinístico** (tempo vem do evento, não do relógio; sem random/ordem instável); **idempotente
no destino** (converge, não duplica); isolável (destino sombra + troca); **com plano e reversível**;
**efeitos colaterais desligados/dedupados** (replay **não** re-cobra nem re-envia).

## 6. Observabilidade e gate (§8-9)
Instrumente **lag, throughput, erro, tamanho da DLQ/quarentena, freshness, volume por janela**;
alerta no SLO de dados; logs sem PII (§16.1). **Pronto (§35)** só se: rodar 2x não corrompe, replay
reconstrói, e o efeito colateral não repete. Grave o mapa (origem→destino, contrato por aresta) +
plano de backfill/replay em `<projeto>_archive/data/pipelines/` — alimenta o **lineage**.
