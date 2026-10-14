# Pipelines — ETL/ELT, streaming, CDC, event sourcing, idempotência e replay

> Como o dado **se move** sem se corromper. Um pipeline é uma cadeia de transformações entre uma
> origem e um destino; a promessa é que rodar essa cadeia — uma vez, duas vezes, ou de novo sobre
> uma janela antiga — produza **o mesmo estado correto**. Este reference rege os padrões de
> movimento (batch/streaming), a captura de mudança (CDC/event sourcing), e os dois pisos que
> separam pipeline de gambiarra: **idempotência** e **reprocessamento determinístico**.

Casa com a base (`schematize-engineering` §9 eventos, §18 resiliência, §19 jobs): eventos
imutáveis/versionados, **Transactional Outbox**, DLQ, retry limitado, jobs idempotentes.

## 1. ETL vs ELT — e onde vive a transformação

| Padrão | Fluxo | Quando |
|---|---|---|
| **ETL** | extrai → **transforma** → carrega | destino caro/rígido; transformação pesada antes de gravar; PII removida **antes** de pousar no destino |
| **ELT** | extrai → **carrega cru** → transforma no destino | warehouse/lakehouse elástico (bronze→silver→gold, `references/armazenamento.md`); mantém o cru pra reprocessar |

Piso: **preserve o dado cru** (camada bronze) quando o destino aguenta — reprocessar exige a
origem. Transformação que descarta o cru é **irreversível**; só faça isso na borda por razão de
governança (ex.: dropar PII que não pode nem pousar — `references/governanca.md`).

## 2. Batch — janelas, watermark e reprocessamento

- **Particione por tempo/chave** e processe por **janela** (dia, hora, partição). A janela é a
  unidade de reprocessamento: dar replay de `2026-08-14` reprocessa **só** aquela partição.
- **Watermark/estado idempotente:** o job registra até onde processou. Rodar de novo a mesma janela
  **substitui** (idempotent write / `MERGE`/upsert por chave), não **duplica** (append cego). Um
  pipeline batch que só faz `INSERT` sem chave duplica em cada re-run — VETADO.
- **Backfill é operação de primeira classe:** reprocessar histórico (novo campo, correção de bug de
  cálculo) tem **plano** (janelas, ordem, custo, impacto no downstream) e é **reversível** — mesma
  disciplina de migration (`references/armazenamento.md`). Backfill sem plano é incidente agendado.
- **Checkpoint em jobs longos** (§19): divida em chunks, persista progresso, retome de onde parou;
  cancelamento gracioso (respeita context/signal).

## 3. Streaming — entrega, ordem, backpressure, DLQ

- **Semântica de entrega:** assuma **at-least-once** do broker e alcance **exactly-once no efeito**
  via **idempotência** (§4) — não confie em "exactly-once" mágico do transporte. Duplicata vai
  acontecer; o consumidor tem que sobreviver a ela.
- **Ordem:** garantida só **por chave de partição**. Se a ordem importa (eventos da mesma conta),
  particione por essa chave. Não assuma ordem global.
- **Backpressure** (§9): consumidor com limite explícito de concorrência e prefetch; produtor faz
  throttle quando o broker sinaliza pressão. Sem limite, um pico derruba o consumidor.
- **DLQ obrigatória:** mensagem que falha após o **retry limitado** (com backoff+jitter; **retry
  infinito é VETADO**, §18) vai pra **dead-letter** com o motivo e a origem — nunca é descartada em
  silêncio, nunca trava a fila atrás dela. DLQ é revisada (alerta, §16), não é cemitério.

## 4. Idempotência — o piso que torna o replay seguro (§19)

Um consumidor/job é **idempotente** quando processá-lo N vezes tem o **mesmo efeito** que
processá-lo uma. Sem isso, retry e replay corrompem. Como se consegue:

- **Dedupe por id do evento / chave natural.** Mantenha o id processado (tabela de dedupe, chave do
  upsert, `ON CONFLICT DO NOTHING`); reentrada com o mesmo id é no-op.
- **Escrita idempotente no destino:** `MERGE`/upsert por chave, não `INSERT` cego. O estado final
  depende do **conteúdo**, não de quantas vezes chegou.
- **Efeito colateral externo dedupado por `Idempotency-Key`** (§12): cobrar cartão, mandar e-mail,
  chamar terceiro — carregue uma chave idempotente e **implemente-a de fato** (aceitar o header e
  ignorar é VETADO, §37). Um replay não pode cobrar o cliente duas vezes.
- **Transições explícitas:** aplicar "pago" sobre "pago" é no-op; aplicar sobre estado incompatível
  é rejeição registrada, não corrupção.

## 5. Entrega — Outbox, sem dual-write (o piso da consistência)

O erro clássico: gravar no banco **e** publicar no broker no mesmo fluxo, sem transação
(**dual-write**). Um dos dois falha e o sistema fica inconsistente (gravou e não publicou, ou
publicou e não gravou). **VETADO** (§37). Padrões corretos:

- **Transactional Outbox:** a mudança de estado **e** o registro do evento vão na **mesma
  transação** do banco (tabela `outbox`); um relay lê a outbox e publica no broker, marcando como
  enviado (com dedupe no consumidor pra a entrega at-least-once). Atômico por construção.
- **CDC (Change Data Capture):** em vez de a aplicação publicar, um conector lê o **log de
  replicação** do banco (WAL/binlog — Debezium & cia.) e emite o evento de mudança. O evento é
  derivado do commit real — sem dual-write possível. Bom pra integrar sistemas que você não
  controla ou pra alimentar o lake.

> Escolha: **Outbox** quando a aplicação é dona do evento de domínio (semântico:
> `invoice.paid`); **CDC** quando você quer o *stream de mudança* de tabelas (replicação,
> alimentação de warehouse, integração com legado). Os dois eliminam o dual-write.

## 6. Event sourcing — o log como fonte da verdade

Quando o **histórico de mudanças** é o ativo (auditoria, reconstrução, temporalidade):

- **O log de eventos é a fonte da verdade**; o estado atual é uma **projeção** (fold dos eventos).
  Eventos são **imutáveis e versionados** (§9) — nunca se edita um evento passado; corrige-se com
  um **evento compensatório** novo.
- **Projeções são reconstruíveis:** apagar e reprojetar a partir do log dá o mesmo estado. Isso é o
  replay do §7. Uma projeção que não se reconstrói do log virou fonte da verdade paralela — bug.
- **Snapshots** pra não refazer o fold inteiro toda vez (checkpoint do estado a cada N eventos),
  sempre reconstruíveis do log.
- **Versão do evento na projeção:** o projetor entende `v1` e `v2` durante a coexistência
  (`references/contratos.md`).

## 7. Reprocessamento e replay — reconstruir sem incidente

Reprocessar (novo consumidor, correção de bug, backfill de campo, reconstrução de projeção) é
**operação normal**, não emergência — desde que:

- **Determinístico:** a mesma entrada + o mesmo código = a mesma saída. Nada de `now()`/random/
  ordem não-determinística dentro da transformação; o tempo do evento vem do **evento**, não do
  relógio de quando reprocessou.
- **Idempotente no destino** (§4): reprojetar por cima **converge** ao mesmo estado; não duplica.
- **Isolável:** reprocessar em um destino/tabela sombra e **trocar** (blue/green de dados) quando
  couber, pra não servir estado meio-reconstruído.
- **Com plano e reversível:** janela, ordem, custo, impacto e **rollback** — igual backfill (§2).
- **Efeitos colaterais desligados ou dedupados:** replay **não** re-dispara e-mail/cobrança (§4).
  Reprocessar dado ≠ re-executar side effects. Confundir os dois é o jeito clássico de transformar
  um replay em incidente de produção.

## 7.1 Orquestração — quem dispara, retenta e sabe o que já rodou

A skill exige **backfill com plano, checkpoint e replay determinístico** (§2, §7) e não dizia **em
que** isso roda. Sem orquestrador declarado, o pipeline vira `cron` + script, e as três coisas que a
skill exige passam a existir só na cabeça de quem escreveu.

**O que a casa exige do orquestrador** (isto é o piso; a ferramenta é escolha com ADR):

- **Estado durável das execuções:** quais rodaram, com que parâmetro, em que janela, com que
  resultado. Sem isso não existe "replay determinístico" — existe rodar de novo e torcer.
- **Idempotência por unidade de trabalho** (dag-run/task-run identificado pela **janela**, não pelo
  relógio de quando disparou). É o que faz `backfill` e `retry` não duplicarem.
- **Retry com backoff e teto** — e **DLQ/alerta** quando estoura (§5). Retry infinito é VETADO na
  base.
- **Dependência explícita entre tarefas** (grafo), não encadeamento por horário: *"roda às 3h porque
  o outro roda às 2h" é acoplamento por relógio, e ele quebra no dia em que o de cima atrasa.*
- **Backfill de primeira classe:** disparar uma janela histórica **sem** editar código, com limite
  de concorrência (senão o backfill derruba a fonte).
- **Observabilidade:** duração, atraso (*lag*) e taxa de falha por tarefa, com alerta de **SLA de
  dado** (§8) — não só "o job falhou".

**O rol, e onde cada um encaixa** (a decisão é ADR, e o critério é o formato do problema):

| Ferramenta | Encaixa quando | Cautela |
|---|---|---|
| **Dagster** | o pipeline é sobre **assets de dados** (tabela, modelo, arquivo) e você quer linhagem e materialização por asset | opinativo; vale quando o time compra o modelo mental de asset |
| **Airflow** | grafo de tarefas clássico, muita integração pronta, time que já o conhece | estado no scheduler; DAG é código Python que **roda no import** — lógica pesada no topo do arquivo derruba o scheduler |
| **Prefect** | fluxo em Python que quer ser código normal, com retry/observabilidade acoplados | menos ecossistema de operador pronto |
| **Temporal** | **workflow de negócio** durável (dias/semanas, com espera humana, saga, compensação) | **não é orquestrador de dados**: usá-lo para ETL em lote é caro e desconfortável |
| **cron + script** | tarefa **única**, sem dependência, sem backfill | *sem* estado durável: assume-se que **não há** replay — e isso vira dívida no primeiro incidente |

**A fronteira que confunde:** **Temporal** resolve *durabilidade de execução de negócio*;
**Dagster/Airflow/Prefect** resolvem *orquestração de pipeline de dados*. Escolher pelo hype é como
o time acaba com um workflow engine transportando parquet — ou com um DAG de dados tentando esperar
a aprovação de um humano por três dias.

## 8. Observabilidade do pipeline (§16)

Todo pipeline nasce instrumentado: **lag** do consumidor, throughput, taxa de erro, tamanho da
**DLQ/quarentena**, **freshness** (quão atrasado está o dado), e volume por janela (queda súbita =
origem quebrada). Logs estruturados com `trace_id`/`tenant_id`, **sem PII** (§16.1). Alerta quando
lag/DLQ/freshness cruzam o SLO de dados (`references/qualidade.md`). Pipeline mudo é pipeline que
falha em silêncio.

## 9. Onde registrar (archive)

O **mapa do pipeline** (origem → transformações → destino, com contrato de cada aresta), o plano de
backfill/replay e as decisões de particionamento moram em `<projeto>_archive/data/pipelines/`
(§28). O mapa alimenta o **lineage** (`references/governanca.md`).

> Regra de bolso: **rodar 2x não corrompe; dar replay reconstrói; o efeito colateral não repete.**
> Se qualquer uma das três não vale, o pipeline não está pronto (§35).
