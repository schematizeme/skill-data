# Changelog — schematize-data

Formato: [Keep a Changelog](https://keepachangelog.com/pt-BR/). Versionamento semântico.

## [0.1.0] — 2026-08-15

Primeira versão da skill de **engenharia de dados** da casa — a disciplina "o dado é um contrato",
agnóstica de linguagem, que desdobra os pisos de dados da `schematize-engineering` (dados-eventos
§9-12, resiliência/jobs §18-19, segurança/LGPD §13-15/§32, observabilidade §16) num corpo próprio.

### Adicionado
- **SKILL.md** com 9 pisos inegociáveis (dado é contrato → breaking change exige nova versão;
  entrega via Outbox, dual-write vetado; idempotência e reprocessamento seguros; dado ruim vai pra
  quarentena; migration expand-contract reversível; deny-by-default no acesso; nada de PII sem base
  legal e retenção; lineage rastreável; saída durável no archive) + mapa de references + relação com
  engineering/skills-irmãs/pentest/web.
- **references/**:
  - `contratos.md` — dado é contrato: onde há fronteira, modelagem, versionamento (`v1`/`v2`),
    matriz de compatibilidade (back/forward/full), schema registry com checagem no CI, política de
    evolução (aditivo vs quebra, incl. mudança silenciosa de semântica).
  - `pipelines.md` — ETL vs ELT, batch (janela/watermark/backfill), streaming (entrega/ordem/
    backpressure/DLQ), CDC e event sourcing, idempotência, entrega via Outbox (sem dual-write),
    reprocessamento/replay determinístico sem re-disparar efeito colateral, observabilidade.
  - `qualidade.md` — validação na borda (rejeição, não coerção), dimensões da qualidade,
    expectations no CI e em runtime, quarentena/dead-letter (dado ruim não contamina, não some),
    SLA de dados, anti-padrões.
  - `armazenamento.md` — escolha do store (OLTP≠OLAP), particionamento, lakehouse bronze/silver/
    gold, migration expand-contract reversível (nunca DROP/rename destrutivo num passo; backfill em
    lotes; `down` testado).
  - `governanca.md` — classificação, PII/LGPD, base legal + retenção aplicada, direitos do titular,
    deny-by-default no acesso (server-side, tenant no `WHERE`/RLS), mascaramento/anonimização,
    lineage & catálogo (nenhum dataset órfão).
- **assets/commands/**: `/data-help`, `/data-contract`, `/data-pipeline`, `/data-quality`,
  `/data-load`, `/data-claude`, `/data-cc`, `/data-handoff`.
- **assets/CLAUDE.md** — regra sempre-on: dado é contrato; dual-write vetado; dado ruim vai pra
  quarentena; deny-by-default; nada de PII sem base legal e retenção.
