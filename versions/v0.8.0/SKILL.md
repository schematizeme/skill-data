---
name: schematize-data
metadata:
  version: 0.8.0
description: >-
  Engenharia de dados: dado como contrato, pipelines ETL/ELT, eventos/CDC/outbox, qualidade, lineage, PII/LGPD. Use ao desenhar pipeline, DLQ ou backfill. DDL: schematize-database.
---

# Engenharia de dados da casa (schematize-data)

Disciplina normativa, **agnóstica de linguagem**, que trata **o dado como contrato**. Um schema,
um evento, um dataset, uma tabela — cada um é uma **promessa de forma** para quem consome. Quebrar
essa promessa sem cerimônia (renomear campo, mudar tipo, dropar coluna, publicar dado sujo)
**quebra o consumidor** — muitas vezes em silêncio, longe no tempo e no espaço. Esta skill rege
como se **modela, versiona, move, valida, governa e reprocessa** dado sem corromper nem vazar.

É a contraparte de **dados** da `schematize-engineering`: a base dá os pisos (eventos imutáveis e
versionados, Outbox, migrations reversíveis, LGPD, observabilidade); esta skill os **desdobra na
disciplina de engenharia de dados** — contratos evolutivos, pipelines batch/streaming, CDC,
qualidade com quarentena, lineage, lakehouse e governança de PII — pra rodar como corpo próprio.

**Versão:** skill `schematize-data` v0.8.0. Changelog em `CHANGELOG.md`.

## Comandos (Claude Code)

Digite `/data-help` pra ver todos. Em resumo:

| Comando | O que faz |
|---|---|
| `/data-help` | lista todos os comandos do schematize-data |
| `/data-contract` | define/audita o **contrato de dados**: schema versionado, compatibilidade (back/forward/full), registry, política de evolução; trava breaking change sem nova versão |
| `/data-pipeline` | desenha/audita o **pipeline** (ETL/ELT batch, streaming, CDC/event sourcing): idempotência, entrega via Outbox, replay/reprocessamento, backpressure, DLQ |
| `/data-quality` | monta/audita o **portão de qualidade**: validação na borda, testes de dados/expectations, quarentena e dead-letter, SLA de dados; dado ruim não passa pro downstream |
| `/data-load` | carrega à força TODO o corpo normativo (contratos, pipelines, qualidade, armazenamento, governança) e passa a aplicá-lo |
| `/data-claude` | cria ou mescla o `CLAUDE.md` sempre-on de dados na raiz do repo |
| `/data-cc` | context compact: gera handoff no archive e roda `/compact` |
| `/data-handoff` | gera o handoff (context.md + checklist.md) sem compactar |

Os comandos ficam em `assets/commands/` e são instalados em `.claude/commands/`.

## Como usar esta skill

1. **Contrato primeiro** (`/data-contract`, `references/contratos.md`): antes de mover um byte,
   fixe **o schema e a política de evolução**. Todo dado que cruza fronteira (serviço, tópico,
   tabela pública, arquivo do lake) tem contrato versionado. Sem contrato, o pipeline propaga
   forma arbitrária — e o consumidor quebra na próxima mudança.
2. **Pipeline idempotente** (`/data-pipeline`, `references/pipelines.md`): ETL/ELT, streaming, CDC
   e event sourcing seguem os pisos de entrega (Outbox, sem dual-write), **idempotência** e
   **reprocessamento** determinístico. Rodar 2x não corrompe; dar replay reconstrói o mesmo estado.
3. **Qualidade com quarentena** (`/data-quality`, `references/qualidade.md`): valide na borda,
   teste o dado (não só o código), e **quarentene** o que reprova. Dado ruim vai pra
   quarentena/DLQ com trilha — nunca contamina o downstream, nunca some em silêncio.
4. **Armazene e migre com segurança** (`references/armazenamento.md`): particionamento,
   warehouse/lakehouse e **migração/backfill de dados** reversíveis; o DDL do schema (expand-contract) é da
   `schematize-database → references/migrations.md`.
5. **Governe** (`references/governanca.md`): **deny-by-default** no acesso; PII só com **base
   legal + retenção**; PII fora de log/URL; **lineage & catálogo** pra todo dataset ser rastreável.
6. **Não trabalhe de memória** — as regras, os cruzamentos e o que conta como "seguro" estão nos
   references. Aplique os pisos abaixo independentemente do reference carregado.

Mapa de references — leia o que casa com a tarefa:

| Tarefa | Reference |
|---|---|
| Modelar dado, definir **contrato**, schema evolutivo, versionamento, compatibilidade (back/forward/full), schema registry, política de mudança | `references/contratos.md` |
| Construir **pipeline** ETL/ELT (batch e streaming), event sourcing / CDC, idempotência, reprocessamento/replay, Outbox, backpressure, DLQ | `references/pipelines.md` |
| **Qualidade** de dados: validação na borda, testes de dados/expectations, quarentena, dead-letter, SLA/contrato de qualidade, deteção de deriva | `references/qualidade.md` |
| **Armazenamento**: particionamento, warehouse/lakehouse (bronze/silver/gold), formato/layout, **migração e backfill de dados** seguros | `references/armazenamento.md` |
| **Governança**: PII/LGPD, base legal, retenção, deny-by-default no acesso, mascaramento/anonimização, **lineage & catálogo** | `references/governanca.md` |

## Pisos inegociáveis (vetam o atalho)

Independente do reference, estes limites nunca são cruzados:

1. **Dado é contrato — breaking change exige NOVA VERSÃO.** Todo dado que cruza fronteira (evento,
   tópico, tabela/view pública, arquivo do lake, resposta de API de dados) tem **schema
   versionado**. Adicionar campo **opcional com default** é evolutivo; **remover/renomear/mudar
   tipo** é quebra — exige `v2` e **janela de coexistência** documentada. Consumidores **ignoram
   campo desconhecido** (forward-compatible). Quebrar contrato sem versão é **VETADO**.
2. **Entrega via Outbox — dual-write ENTRE SISTEMAS é VETADO.** Gravar no banco **e** publicar o
   evento no broker no mesmo fluxo, **sem transação que cubra os dois**, corrompe consistência: um
   dos lados falha e ninguém sabe qual. Use **Transactional Outbox** (ou CDC do log do banco).
   *(O termo é sobrecarregado, e a diferença importa: **escrever nas duas colunas/tabelas do MESMO
   banco, na MESMA transação**, durante uma migração expand/contract — o que `armazenamento.md` e a
   `schematize-database` chamam de "dual-write" na fase Migrate — **é o padrão correto**, não o
   vetado. O que se veta é o dual-write **sem transação comum**, entre sistemas diferentes.)* Todo consumidor tem **DLQ**; retry é limitado, com backoff — **retry
   infinito é proibido** (base §9/§18).
3. **Idempotência e reprocessamento seguros.** Consumidor/job é **idempotente** (rodar 2x não
   corrompe — dedupe por id do evento/chave natural). Todo pipeline suporta **replay
   determinístico**: reprocessar a mesma janela reconstrói **o mesmo** estado. Efeito colateral
   (e-mail, cobrança) é dedupado por `idempotency-key`. Sem isso, um replay vira incidente.
4. **Dado ruim NÃO contamina o downstream.** Validação **na borda** de ingestão; o que reprova o
   contrato/expectativa vai pra **quarentena** (dead-letter de dados) com motivo e origem — nunca
   entra no dataset "bom", nunca some em silêncio. Falha de qualidade **alerta** (§16), não é
   engolida.
5. **Migração/backfill de dados é reversível e em lotes.** O expand-contract do schema é da
   `schematize-database → references/migrations.md`; aqui, backfill sem plano de rollback ou que só
   corta o velho com leitor ainda ativo (lineage) é **VETADO**. Timestamps em **UTC**; IDs **UUIDv7/ULID**.
6. **Deny-by-default no acesso ao dado.** Acesso a dataset/tabela/tópico é **negado por padrão** e
   concedido por necessidade (least-privilege), derivado do token **server-side** e por
   **tenant** (`WHERE tenant_id`, RLS quando couber — base §15). Confiar em `tenant_id`/role vindo
   do cliente é **VETADO**. Query com input externo é **parametrizada** — SQL concatenado é VETADO.
7. **Nada de PII sem base legal E retenção.** Dado pessoal só é coletado/tratado com **base legal
   registrada** e **política de retenção** definida (expira e é apagado/anonimizado no prazo). PII
   **nunca** em log, em URL/query string, nem em cache sem cripto (base §16.1/§32). Mascaramento/
   anonimização no que sai do domínio; direito de eliminação/portabilidade **implementável**.
8. **Lineage rastreável — nenhum dataset órfão.** Todo dataset tem **origem, dono e destino**
   documentados (catálogo + lineage): de onde veio, quem produz, quem consome, qual contrato.
   Dataset sem linhagem é dívida — ninguém sabe o que quebra se ele mudar.
9. **Cópia de prd para não-prd NUNCA leva e-mail (nem contato) REAL.** Recorte de dados do piso
   "efeito externo nunca sai de não-produção" (`schematize-engineering` →
   `references/efeitos-externos.md`). Todo dump/snapshot/subset/refresh que sai de produção passa
   pelo mascaramento, e nele **o endereço é REESCRITO** — não parcialmente ofuscado — para o
   **domínio de teste em rota nula** (`test.<domain>` com null MX + SPF `-all` + DMARC `p=reject`,
   ou TLD reservado `.test`/`.invalid`/`.example`), **determinístico por HMAC com chave de
   snapshot** pra preservar unicidade e joins (hash **sem chave** é reidentificável por dicionário).
   A reescrita acontece **na saída de prd**, nunca "depois que chegar em hml". Vale igual pra
   telefone, push token, webhook do cliente e token de PSP. **Geradores de fixture/factory/seed/
   persona/carga só produzem `<papel>+<run-id>-<n>@test.<domain>`** — **VETADO** `@gmail.com` e
   afins, domínio de terceiro, e-mail de pessoa real (inclusive o seu) e o domínio de produção.
   **Expectation obrigatória** em não-prd: 0 linhas com domínio fora da allowlist de teste →
   **quarentena** e refresh reprovado. **Por quê:** um dataset com e-mail real num ambiente que
   envia é **incidente esperando o próximo laço** — foi assim que um laço disparou **>5.000
   e-mails reais**, queimou reputação de IP/domínio e derrubaria o **OTP de login** de produção.
   Detalhe: `references/governanca.md` §5.1 e `references/qualidade.md` §3.

   **O gate de máquina:** `scripts/check-external-effects.sh` (distribuído idêntico nesta skill — não é ponteiro para outro repo). Rode-o no CI: ele reprova endereço de caixa real em seed/fixture/persona, chave de provedor não-sandbox em `.env` de não-prd (fail-closed quando o ambiente não está declarado) e domínio de teste sem null MX. O vermelho dele está provado em `scripts/check-external-effects.test.sh`.

10. **Saída durável no archive.** Contratos, DDL/migrations, mapas de pipeline, planos de backfill
   e relatórios de qualidade moram em `<projeto>_archive/data/` (§28), versionados — nunca só na
   cabeça de quem rodou. Sem archive, a decisão de dados não aconteceu.

## Precedência e herança (leia antes de divergir)

<!-- herdado:engineering/precedencia:dominio intro="Esta skill é o recorte **Engenharia de dados (contrato, pipeline, qualidade, lineage)** da `schematize-engineering`: **disciplina de domínio**, não linguagem de backend." cmd="data" -->Esta skill é o recorte **Engenharia de dados (contrato, pipeline, qualidade, lineage)** da `schematize-engineering`: **disciplina de domínio**, não linguagem de backend.

1. **Onde esta skill divergir da base, a BASE MANDA.** Aqui mora a especialização do domínio ou da
   ferramenta — o como, o padrão, o gate. Divergência de *piso* é defeito desta skill, não variante local.
2. **O que não está repetido aqui é HERDADO, não dispensado.** Esta skill herda os pisos da base e
   os comandos dela cobrem o **domínio**, não o piso geral. Valem integralmente, sem cópia local:
   - **§28 Archive** — `<projeto>/<projeto>_archive/` é **repositório git próprio, PRIVADO e
     obrigatório**, criticidade 0 (`schematize-archive`).
   - **§35 Definition of Done** e a lista de anti-padrões (citada por **título**, nunca por número).
   - **§39 Índice/MAPA** — enumeração exaustiva (uma entrada por unidade chamável, `M == N`) e grafo
     com arestas em ASCII (`A -> B`), nunca a seta unicode.
   - **IAM** (`schematize-engineering` → `references/iam.md`): identidade ≠ email, ≥2 fatores, ReBAC
     multi-tenant, argon2id-only para senha nova, sessão longa e logout irreversível.
   - **Efeito externo** nunca sai de não-produção (`schematize-engineering` →
     `references/efeitos-externos.md`).
   - **Orquestração** (`schematize-engineering` → `references/orquestracao.md` §9): o principal não
     desenvolve (é Claude Opus); executor mais barato que resolve — `haiku`/`sonnet` → Codex Terra → Cursor → `opus` só após falha; visual direto ao Codex; imagem nunca pelo Claude: detecta Codex/Gemini e pergunta ao usuário qual usar (Codex é a sugestão, não automática), nunca API paga.
3. **QA, ops, IAM e índice não têm comando próprio aqui:** vêm da base `/eng-*`. Para trabalho
   contínuo use `/eng-overdev`; para planejar `/eng-plan`; para varrer a base `/eng-scan`; para
   revisão geral `/eng-review` (e `/eng-ops`, `/eng-iam`, `/eng-index`). Os comandos
   `/data-*` cobrem o domínio desta skill; o piso geral é o da base.<!-- /herdado -->

## Orquestração

<!-- herdado:engineering/orquestracao:curto -->**Orquestrador não desenvolve; subagent barato executa.** O agent principal só planeja, despacha e revisa; ação onerosa vira micro-tasks para o executor mais barato que resolve: `haiku` (mecânico) / `sonnet` (código; falhou → mesmo subagent corrige, até 2 rodadas) → Codex `gpt-5.6-terra` → Cursor → subagent `opus` (motivo) → `park`; **visual vai direto ao Codex**; **imagem nunca pelo Claude: detectar Codex/Gemini e perguntar ao usuário qual usar (Codex é a sugestão), nunca API de imagem paga**; papéis em `agentes.toml`. No overdev, cada item do checklist vai a um subagent e o principal revisa antes do `- [x]`. **Sem frota ociosa:** idle com pendência volta ao trabalho; dependente de outro agent → mata e enfileira com gatilho; terminou → mata (§9.6). Detalhe: `schematize-engineering` → `references/orquestracao.md` §9.<!-- /herdado -->

## Relação com as outras skills

- **schematize-engineering** — a **BASE**. Esta skill desdobra os pisos de dados da engenharia:
  **eventos/mensageria, banco, cache, APIs** (§9-12), **resiliência e jobs** (§18-19),
  **segurança/auth/multi-tenancy/LGPD** (§13-15/§32), **observabilidade** (§16), **archive** (§28)
  e **DoD** (§35). O que lá é piso, aqui vira disciplina de engenharia de dados.
- **schematize-go / rust / elixir / c# / zig / ruby** — a **implementação**. Esta skill é
  agnóstica: define o contrato, o pipeline, a migration e a governança; **o código** (o consumidor,
  o job, o worker de CDC, a migration executável) roda no gate daquela linguagem
  (`/<slug>-review`, testes, lint). O piso de dados é o mesmo em qualquer stack.
- **schematize-regression** — a **regressão de dados**: dataset/contrato que voltou a quebrar, backfill
  que reintroduz erro já corrigido. As expectations e a quarentena são desta skill; o teste que
  impede o MESMO defeito de dado de voltar segue o ciclo de `schematize-regression` →
  `references/contrato-dados-ia.md`.
- **schematize-analytics** — o **evento de produto** na origem: tracking plan como contrato,
  consentimento e modo sem cookie (`schematize-analytics → references/consentimento-lgpd.md`,
  `references/plano-de-tracking.md`). Esta skill leva o evento depois que ele nasceu certo (contrato,
  pipeline, qualidade, retenção); decidir *se e o quê* coletar é de lá.
- **schematize-pentest** — o **oráculo ofensivo** do dado: cross-tenant leak no dataset/RAG,
  IDOR/BOLA em endpoint de dados, PII vazando em log/resposta, injeção na borda de ingestão. A
  disciplina de dados desenha o deny-by-default; a pentest tenta furá-lo.
- **schematize-web** — a borda de consumo: dado que chega ao frontend segue o contrato; **segredo
  nunca no bundle**, PII nunca em `localStorage`, decisão de acesso **server-side**.
- **schematize-database** — a **fronteira mais fácil de errar**, e ela é bilateral. A
  `schematize-database` é dona do **DESENHO do schema relacional**: entidades → tabelas,
  normalização 1FN–3FN, **chaves** (PK surrogate ULID/UUIDv7 + chave natural `UNIQUE`), tipo por
  coluna, `CHECK`/`FOREIGN KEY` com `ON DELETE` consciente, **índices** (compostos, ordem das
  colunas, parciais) e a migration expand-contract do schema. Esta skill é dona do **dado em
  movimento**: contrato e versionamento, pipeline, CDC/outbox, qualidade/quarentena, lineage,
  retenção e PII.
  Regra prática: **"como a tabela é" é `schematize-database`; "como o dado entra, sai, evolui e
  quem pode vê-lo" é `schematize-data`.** Quem entra por `/data-*` e vai modelar tabela **carrega
  a `schematize-database` junto** — pipeline novo que cria tabela sem o piso de chaves e índices é
  dívida no dia 1, e é exatamente onde os dois lados se encontram (a migration de dados desta
  skill **executa** o schema desenhado lá).
