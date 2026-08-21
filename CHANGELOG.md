# Changelog — schematize-data

Todas as mudanças relevantes deste pacote, no formato [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/),
com versionamento [SemVer](https://semver.org/lang/pt-BR/).


## [0.3.0] — 2026-08-21
Saneamento do catálogo conforme a vistoria de 2026-08-21.

### Mudado
- `scripts/check-external-effects.sh` (+ teste) atualizado para a versão distribuída do gate de efeito externo — **md5 idêntico nas 17 skills** que o carregam (ADR-0008).
- `install.sh` regenerado do template único: **exclui `*.zip`** do pacote instalado, **poda comando removido** e instala hooks.
- README com a contagem de pisos corrigida (9 → 10, conferida contra o `SKILL.md`) — hoje verificada pela regra `contagem` do lint do catálogo.

## [0.2.0] — 2026-08-20

Propagação do piso **"efeito externo NUNCA sai de não-produção"** no recorte de **DADOS**: o
gatilho do incidente que originou o piso (`schematize-engineering` →
`references/efeitos-externos.md`) foi **o dataset**, não o código — uma base de hml com endereços
que **existiam** virou **>5.000 e-mails reais** no primeiro laço.

### Adicionado
- **SKILL.md** — piso inegociável **9** ("Cópia de prd para não-prd NUNCA leva e-mail (nem contato)
  real"): reescrita do endereço para o domínio de teste em rota nula, determinismo por HMAC,
  reescrita na saída de prd, fixtures/factories só no domínio de teste, expectation obrigatória;
  menção ao piso na `description` do frontmatter (pisos renumerados: archive passa a ser o 10).
- **references/governanca.md** §5.1 — "Endereço de contato: a cópia prd → hml/dev REESCREVE o
  e-mail (rota nula)": por que **mascarar parcialmente não basta** (`j***@gmail.com` continua no
  domínio real e volta a ser entregável); fórmula determinística
  `user+hex(HMAC-SHA256(k_snapshot, email))[0:12]@test.<domain>` preservando **unicidade e joins**;
  por que **hash sem chave** é reidentificação por dicionário (o espaço de e-mails é enumerável) e
  por que a chave é **por snapshot, no cofre e descartável**; extensão a telefone/push token/
  webhook do cliente/token de PSP; a reescrita acontece **na saída de prd**; geradores de fixture/
  factory/seed/persona/carga só emitem `<papel>+<run-id>-<n>@test.<domain>` (com a lista de vetados);
  a coluna reescrita **continua marcada como PII** no catálogo do destino.
- **references/qualidade.md** §3 — expectation obrigatória em não-produção: **0 linhas com domínio
  de contato fora da allowlist de teste**; o que reprova vai pra **quarentena** e o refresh
  **falha** (nunca warning).
- **assets/CLAUDE.md** — piso sempre-on **9** com a regra de reescrita e os vetados.

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
