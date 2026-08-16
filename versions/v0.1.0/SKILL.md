---
name: schematize-data
metadata:
  version: 0.1.0
description: Engenharia de dados da casa — a disciplina, agnóstica de linguagem, que trata o dado como CONTRATO e não como efeito colateral. Rege modelagem e contratos de dados (schema evolutivo, versionamento, compatibilidade back/forward/full, schema registry), pipelines ETL/ELT batch e streaming, event sourcing / CDC (casando com a schematize-engineering: eventos imutáveis, versionados, Transactional Outbox, sem dual-write), qualidade de dados (validação na borda, testes de dados/expectations, quarentena e dead-letter — dado ruim não contamina o downstream), lineage & catálogo (todo dataset rastreável até a origem, nada órfão), particionamento / warehouse / lakehouse, migrations de dados seguras (expand-contract, reversível, nunca drop destrutivo), idempotência e reprocessamento (consumidor idempotente, replay determinístico, exactly-once no efeito) e PII/governança/retention/LGPD (deny-by-default no acesso, base legal + retenção obrigatórias, PII fora de log/URL, mascaramento). Saída durável no `<projeto>_archive/`. Use SEMPRE que for modelar dados, desenhar schema/tabela/tópico/evento, construir ou revisar pipeline (ETL/ELT/streaming/CDC), definir contrato de dados, escrever migration de dados, planejar warehouse/lakehouse/particionamento, tratar qualidade/quarentena/DLQ, lidar com PII/retenção/LGPD, ou reprocessar/dar replay — mesmo sem citar "dados". Contém pisos inegociáveis (dado é contrato — breaking change exige nova versão; deny-by-default no acesso; nada de PII sem base legal e retention; idempotência e reprocessamento seguros; migration expand-contract reversível; entrega via Outbox, dual-write VETADO; dado ruim vai pra quarentena/DLQ, nunca contamina o downstream; lineage rastreável) que vetam atalhos que corrompem ou vazam dado. Pareia com a schematize-engineering (a BASE: dados-eventos §9-12/§18-19, segurança/LGPD §13-15/§32, observabilidade §16, archive §28, DoD §35); a implementação por stack fica nas skills irmãs (go/rust/elixir/c#/zig/ruby); segurança ofensiva na schematize-pentest.
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

**Versão:** skill `schematize-data` v0.1.0. Changelog em `CHANGELOG.md`.

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
   warehouse/lakehouse e **migrations expand-contract** reversíveis — nunca um `DROP`/rename
   destrutivo num passo só.
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
| **Armazenamento**: particionamento, warehouse/lakehouse (bronze/silver/gold), formato/layout, **migrations de dados** expand-contract seguras | `references/armazenamento.md` |
| **Governança**: PII/LGPD, base legal, retenção, deny-by-default no acesso, mascaramento/anonimização, **lineage & catálogo** | `references/governanca.md` |

## Pisos inegociáveis (vetam o atalho)

Independente do reference, estes limites nunca são cruzados:

1. **Dado é contrato — breaking change exige NOVA VERSÃO.** Todo dado que cruza fronteira (evento,
   tópico, tabela/view pública, arquivo do lake, resposta de API de dados) tem **schema
   versionado**. Adicionar campo **opcional com default** é evolutivo; **remover/renomear/mudar
   tipo** é quebra — exige `v2` e **janela de coexistência** documentada. Consumidores **ignoram
   campo desconhecido** (forward-compatible). Quebrar contrato sem versão é **VETADO**.
2. **Entrega via Outbox — dual-write é VETADO.** Publicar evento/mudança e gravar no banco **no
   mesmo fluxo** sem transação (dual-write) corrompe consistência. Use **Transactional Outbox**
   (ou CDC do log do banco). Todo consumidor tem **DLQ**; retry é limitado, com backoff — **retry
   infinito é proibido** (base §9/§18).
3. **Idempotência e reprocessamento seguros.** Consumidor/job é **idempotente** (rodar 2x não
   corrompe — dedupe por id do evento/chave natural). Todo pipeline suporta **replay
   determinístico**: reprocessar a mesma janela reconstrói **o mesmo** estado. Efeito colateral
   (e-mail, cobrança) é dedupado por `idempotency-key`. Sem isso, um replay vira incidente.
4. **Dado ruim NÃO contamina o downstream.** Validação **na borda** de ingestão; o que reprova o
   contrato/expectativa vai pra **quarentena** (dead-letter de dados) com motivo e origem — nunca
   entra no dataset "bom", nunca some em silêncio. Falha de qualidade **alerta** (§16), não é
   engolida.
5. **Migration de dados é expand-contract e reversível.** Mudança de schema em dado vivo segue
   **expand → migrate → contract** (adiciona o novo, backfill, corta o velho só quando ninguém lê),
   cada passo com **`down`** testado (aplica e reverte). `DROP`/rename destrutivo num passo só, ou
   backfill sem plano de rollback, são **VETADOS**. Timestamps em **UTC**; IDs **UUIDv7/ULID**.
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
9. **Saída durável no archive.** Contratos, DDL/migrations, mapas de pipeline, planos de backfill
   e relatórios de qualidade moram em `<projeto>_archive/data/` (§28), versionados — nunca só na
   cabeça de quem rodou. Sem archive, a decisão de dados não aconteceu.

## Relação com as outras skills

- **schematize-engineering** — a **BASE**. Esta skill desdobra os pisos de dados da engenharia:
  **eventos/mensageria, banco, cache, APIs** (§9-12), **resiliência e jobs** (§18-19),
  **segurança/auth/multi-tenancy/LGPD** (§13-15/§32), **observabilidade** (§16), **archive** (§28)
  e **DoD** (§35). O que lá é piso, aqui vira disciplina de engenharia de dados.
- **schematize-go / rust / elixir / c# / zig / ruby** — a **implementação**. Esta skill é
  agnóstica: define o contrato, o pipeline, a migration e a governança; **o código** (o consumidor,
  o job, o worker de CDC, a migration executável) roda no gate daquela linguagem
  (`/<slug>-review`, testes, lint). O piso de dados é o mesmo em qualquer stack.
- **schematize-pentest** — o **oráculo ofensivo** do dado: cross-tenant leak no dataset/RAG,
  IDOR/BOLA em endpoint de dados, PII vazando em log/resposta, injeção na borda de ingestão. A
  disciplina de dados desenha o deny-by-default; a pentest tenta furá-lo.
- **schematize-web** — a borda de consumo: dado que chega ao frontend segue o contrato; **segredo
  nunca no bundle**, PII nunca em `localStorage`, decisão de acesso **server-side**.
