# CLAUDE.md — Engenharia de Dados da Casa (sempre on)

> Copie para a **raiz do repositório** e ajuste `<project>`. Fica pinado no contexto de toda tarefa
> e garante o piso mesmo quando a skill `schematize-data` não dispara sozinha. Em repo multi-skill,
> use **junto** com os `CLAUDE.md` das skills de engenharia (rode `/data-claude` que mescla, sem
> sobrescrever os outros blocos).

## Regra mestre

Neste repo, **o dado é um contrato** — uma promessa de forma para quem consome. Mover, transformar,
armazenar ou expor dado sem honrar essa promessa quebra o consumidor, corrompe o estado ou vaza o
titular. A skill `schematize-data` rege como se faz isso sem incidente. Em conflito entre "joga no
banco e segue" e este piso, **o piso vence**. Consulte o reference antes de agir — não trabalhe de
memória.

## Pisos inegociáveis (VETADO — sem exceção)

1. **Dado é contrato — breaking change exige NOVA VERSÃO.** Todo dado que cruza fronteira (evento,
   tópico, tabela/view pública, arquivo do lake, API de dados) tem **schema versionado**. Adicionar
   campo **opcional com default** é evolutivo; **remover/renomear/mudar tipo/mudar semântica** é
   quebra → `v2` + **janela de coexistência**. Consumidor **ignora campo desconhecido**.
2. **Entrega via Outbox — dual-write é VETADO.** Gravar no banco e publicar no broker no mesmo fluxo
   sem transação corrompe consistência. Use **Transactional Outbox** ou **CDC**. Todo consumidor tem
   **DLQ**; retry é limitado com backoff — **retry infinito é proibido**.
3. **Idempotência e reprocessamento seguros.** Consumidor/job **idempotente** (rodar 2x não corrompe
   — dedupe por id/chave). **Replay determinístico** reconstrói o mesmo estado. Efeito colateral
   dedupado por `Idempotency-Key` — replay **não** re-cobra nem re-envia.
4. **Dado ruim NÃO contamina o downstream.** Validação **na borda**; o que reprova o contrato/
   expectativa vai pra **quarentena** (com motivo + origem), **alerta**, e é reprocessável — nunca
   entra no dataset bom, nunca some em silêncio. Sem coerção silenciosa.
5. **Migration de dado é expand-contract e reversível.** **expand → migrate (backfill) → contract**
   (corta o velho só quando ninguém lê). Cada passo com **`down`** testado. `DROP`/rename destrutivo
   num passo só é VETADO. Timestamps **UTC**; IDs **UUIDv7/ULID**; query **parametrizada**.
6. **Deny-by-default no acesso.** Acesso a dataset/tabela/tópico **negado por padrão**, concedido
   por necessidade; decisão **server-side** derivada do token; **tenant no `WHERE`** (RLS quando
   couber). Confiar em `tenant_id`/role do cliente é VETADO.
7. **Nada de PII sem base legal E retenção.** Dado pessoal só com **base legal registrada** e
   **retenção** aplicada (expira e é apagado/anonimizado). PII **nunca** em log, URL/query string,
   nem cache sem cripto. Direito de eliminação/portabilidade implementável.
8. **Lineage rastreável — nenhum dataset órfão.** Todo dataset tem **origem, dono e destino** no
   catálogo/lineage. Dataset sem linhagem é dívida — ninguém sabe o que quebra se ele mudar, nem
   onde está o dado do titular.
9. **Cópia de prd → hml/dev NUNCA leva e-mail (nem contato) real.** Todo dump/snapshot/subset/
   refresh passa pelo mascaramento e nele o **endereço é REESCRITO** — não só ofuscado — pro
   **domínio de teste em ROTA NULA** (`test.<domain>` com **null MX + SPF `-all` + DMARC
   `p=reject`**, ou `.test`/`.invalid`/`.example`), **determinístico por HMAC com chave de
   snapshot** (preserva unicidade e joins; hash sem chave é reidentificável). Reescreve **na saída
   de prd**, nunca depois de chegar. Mesma regra pra telefone, push token, webhook do cliente e
   token de PSP. **Fixture/factory/seed/persona/carga só geram `<papel>+<run-id>-<n>@test.<domain>`**
   — **VETADO** `@gmail.com` e afins, domínio de terceiro, e-mail de pessoa real (inclusive o seu) e
   o domínio de produção. Expectation em não-prd: **0 linhas com domínio fora da allowlist** →
   **quarentena**, refresh reprovado. **Por quê:** dataset com e-mail real em ambiente que envia é
   **incidente esperando o próximo laço** (>5.000 e-mails reais, reputação queimada, **OTP de login**
   de prd derrubado). Normativa: `schematize-engineering` → `references/efeitos-externos.md`;
   detalhe em `references/governanca.md` §5.1.
10. **Saída durável no archive.** Contratos, DDL/migrations, mapas de pipeline, planos de backfill e
   relatórios de qualidade moram em `<project>_archive/data/` (§28), versionados. Nunca só na cabeça.

## Como se trabalha dados aqui

- **Contrato primeiro (`/data-contract`):** fixa schema + versão + modo de compatibilidade antes de
  mover byte. Breaking change sem `v2`+janela não passa.
- **Pipeline idempotente (`/data-pipeline`):** ETL/ELT/streaming/CDC com Outbox (sem dual-write),
  idempotência, DLQ e replay determinístico (efeito colateral dedupado).
- **Qualidade com quarentena (`/data-quality`):** valida na borda, testa o dado (CI + runtime),
  quarentena o que reprova (motivo+origem+alerta), SLA por dataset com dono.
- **Armazena/migra seguro:** OLTP≠OLAP, particiona, lakehouse bronze/silver/gold, migration
  expand-contract reversível.
- **Governa:** classifica, base legal + retenção, deny-by-default, mascara/anonimiza, catálogo +
  lineage rastreável até a origem.

## Relação com as outras skills

- **schematize-engineering** — a base: dados-eventos (§9-12), resiliência/jobs (§18-19), segurança/
  auth/multi-tenancy/LGPD (§13-15/§32), observabilidade (§16), archive (§28), DoD (§35).
- **schematize-go / rust / elixir / c# / zig / ruby** — a implementação: o contrato/pipeline/
  migration é agnóstico; o código roda no gate da linguagem (`/<slug>-review`, testes, lint).
- **schematize-pentest** — o oráculo ofensivo: cross-tenant leak, IDOR em endpoint de dados, PII em
  log/resposta, injeção na borda de ingestão.

## Gestão de contexto (sessões longas)

Ao se aproximar do teto de contexto: **PARE e** gere o handoff em `<project>_archive/context/`
(datasets/contratos/pipelines/migrations em jogo + FEITO vs EM ABERTO) **antes** de compactar
(`/data-cc`).

## Orquestração

<!-- herdado:engineering/orquestracao:curto -->**Orquestrador não desenvolve; subagent barato executa.** O agent principal só planeja, despacha e revisa; ação onerosa vira micro-tasks para o executor mais barato que resolve: `haiku` (mecânico) / `sonnet` (código; falhou → mesmo subagent corrige, até 2 rodadas) → Codex `gpt-5.6-terra` → Cursor → subagent `opus` (motivo) → `park`; **visual vai direto ao Codex**; **imagem nunca pelo Claude: detectar Codex/Gemini e perguntar ao usuário qual usar (Codex é a sugestão), nunca API de imagem paga**; papéis em `agentes.toml`. No overdev, cada item do checklist vai a um subagent e o principal revisa antes do `- [x]`. **Sem frota ociosa:** idle com pendência volta ao trabalho; dependente de outro agent → mata e enfileira com gatilho; terminou → mata (§9.6). Detalhe: `schematize-engineering` → `references/orquestracao.md` §9.<!-- /herdado -->
