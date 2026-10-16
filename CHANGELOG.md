# Changelog — schematize-data

Todas as mudanças relevantes deste pacote, no formato [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/),
com versionamento [SemVer](https://semver.org/lang/pt-BR/).


## [0.8.0] — 2026-10-15
### Alterado
- Relação com outras skills e `references/qualidade.md` §3 apontam para `schematize-regression` (regressão
  de dados: expectation do defeito corrigido entra vista vermelha). Expectations e quarentena seguem aqui.
- Porquê: fronteira explícita, sem duplicar conteúdo.

## [0.7.0] — 2026-10-14
Integração com as 8 skills novas (skill-dev, iam, api, payments, postgres, incident, analytics, git) e saneamento pelo gate novo.

### Alterado
- Blocos herdados `engineering/precedencia:dominio` e `engineering/orquestracao:curto` (SKILL.md e assets/CLAUDE.md), exigidos pelo gate da `schematize-skill-dev`.
- Description reduzida a ≤190 caracteres para caber no orçamento do catálogo (45 skills).
- Ponteiro para `schematize-analytics` (evento de produto na origem, consentimento) em `references/governanca.md` e na relação com as skills; piso de PII fica aqui, o desenho do tracking é de lá.

## [0.6.1] — 2026-10-14
Reduz a description para o orçamento de listagem.

### Alterado
- Description reduzida (≤235 chars) para caber no orçamento de listagem de skills (soma ≤8300); mesmo território, fronteiras com as irmãs preservadas.

## [0.6.0] — 2026-10-14
Fronteira data x database decidida.

### Corrigido
- `references/armazenamento.md` §4 e `SKILL.md` (item 4, piso 5): o expand-contract do DDL é da `schematize-database → references/migrations.md`; aqui ficou migração/backfill de dados (lotes, reversível, só contrai sem leitor via lineage).
- `references/governanca.md`: ponteiro para a marca da coluna PII no DDL (`schematize-database → references/seguranca-pii.md`); base legal, retenção e LGPD ficam aqui.
- `description` em `>-` (YAML antes inválido) com 350 caracteres ou menos.
- **Description** reduzida (~270 chars) para caber no orçamento de listagem (soma das 31 skills ≤ 8500; o Claude Code corta a listagem acima disso).

### Alterado
- `scripts/check-external-effects{,.test}.sh` viram arquivos herdados da engineering (cabeçalho "herdado de schematize-engineering — não edite aqui"); conteúdo inalterado.

## [0.5.0] — 2026-08-21

### Adicionado
- **`pipelines.md` §7.1 — Orquestração.** A skill exigia backfill com plano, checkpoint e replay determinístico **sem dizer em que isso roda** — e sem orquestrador declarado o pipeline vira `cron` + script, com as três exigências existindo só na cabeça de quem escreveu. Entraram **o que a casa exige do orquestrador** (estado durável das execuções — *sem ele não existe replay determinístico, existe rodar de novo e torcer*; idempotência por **janela**, não pelo relógio; retry com teto e DLQ; **dependência explícita** em vez de encadeamento por horário; backfill de primeira classe com limite de concorrência; observabilidade com SLA de dado) e o rol com o encaixe de cada um (**Dagster**, **Airflow** — com a cautela de que *DAG é código que roda no import* —, **Prefect**, **Temporal**, e **cron + script** para tarefa única, assumindo que **não há replay**). Fecha com a fronteira que mais confunde: *Temporal resolve durabilidade de execução de **negócio**; Dagster/Airflow/Prefect resolvem orquestração de **dados***.

## [0.4.0] — 2026-08-21
Segunda leva do saneamento: as contradições e a lacuna de ferramenta do inventário da vistoria.

### Corrigido
- **"Backward permite remover campo" × "remover campo é breaking change"** — a contradição era de **pergunta**, não de regra: a matriz de `contratos.md` §4 responde *"o schema novo lê o dado velho?"* (e o registry aprova a remoção, porque o **leitor** não pede mais o campo); o piso 1 responde *"algum consumidor ainda lê isso?"* — e o registry **não sabe**. Ficou escrito que **`BACKWARD` é condição necessária, não suficiente**: remover exige **inventário de consumidores medido**, janela de depreciação e `v2` se alguém não migrou. *"Achei que ninguém usava" é a frase que precede o incidente de dados.*
- **`dual-write` deixou de significar duas coisas.** O piso 2 passou a dizer **"dual-write ENTRE SISTEMAS é VETADO"** (banco + broker sem transação comum), e o `armazenamento.md` diz **"dual-write na mesma transação, no mesmo banco"** na fase Migrate — *que é o oposto do problema, e é por isso que o expand/contract funciona*.

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
