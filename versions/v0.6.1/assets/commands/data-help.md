---
description: schematize-data — lista todos os comandos disponíveis e o que cada um faz
---

Liste os comandos do **schematize-data** instalados (`/data-*`), com 1 linha cada:

- `/data-help` — esta lista.
- `/data-contract` — define/audita o **contrato de dados**: schema versionado, compatibilidade (back/forward/full), schema registry e política de evolução; **trava** breaking change (remover/renomear/mudar tipo/mudar semântica) sem nova versão + janela de coexistência.
- `/data-pipeline` — desenha/audita o **pipeline** (ETL/ELT batch, streaming, CDC/event sourcing): **idempotência**, entrega via **Outbox** (dual-write vetado), replay/reprocessamento determinístico, backpressure e **DLQ**; efeito colateral dedupado por `idempotency-key`.
- `/data-quality` — monta/audita o **portão de qualidade**: validação na borda, testes de dados/expectations, **quarentena** e dead-letter (dado ruim não contamina o downstream, nem some em silêncio), SLA de dados e alerta.
- `/data-load` — carrega à força TODO o corpo normativo (contratos, pipelines, qualidade, armazenamento, governança) e passa a aplicá-lo.
- `/data-claude` — cria ou mescla o `CLAUDE.md` sempre-on de dados na raiz do repo.
- `/data-cc` — context compact: gera handoff no archive e roda `/compact`.
- `/data-handoff` — gera o handoff (context.md + checklist.md) sem compactar.

Depois da lista, lembre a **regra de ouro**: *o dado é um contrato.* Quebra de forma exige nova
versão com janela; dado ruim vai pra **quarentena** (nunca contamina o downstream); acesso é
**deny-by-default**; e **nada de PII sem base legal e retenção**. Detalhe normativo em
`references/` da skill `schematize-data`; a base (dados-eventos, segurança/LGPD, observabilidade,
archive, DoD) é a `schematize-engineering`; a implementação por stack fica nas skills irmãs
(go/rust/elixir/c#/zig/ruby).
