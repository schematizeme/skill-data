# Armazenamento — particionamento, warehouse/lakehouse e migrations seguras

> Onde o dado **pousa** e como ele **muda de forma sem quebrar**. Este reference cobre a escolha do
> store, o **particionamento**, a arquitetura de **warehouse/lakehouse** (bronze/silver/gold) e o
> piso mais perigoso do dia a dia de dados: **migração e backfill de dado vivo são planejados e
> reversíveis**. O DDL/expand-contract do schema é da `schematize-database`.

Casa com a base (`schematize-engineering` §10 banco): migrations versionadas e **reversíveis**,
schema por serviço, sem join cross-service, timestamps **UTC**, IDs **UUIDv7/ULID**, query
**parametrizada**.

## 1. Escolha do store (por caso)

Herda a matriz da base (§10) e estende pro mundo analítico:

| Caso | Store |
|---|---|
| Transacional / relacional (OLTP) | **PostgreSQL** |
| Cache | **Redis** (TTL sempre explícito, §11) |
| Busca textual | **OpenSearch** |
| Analytics / eventos (OLAP colunar) | **ClickHouse** |
| Lakehouse (arquivo aberto + tabela transacional) | **Iceberg / Delta / Hudi** sobre object storage (S3/GCS) |
| Streaming / log | **Kafka** (retenção/replay) / **NATS** (leve) |

Piso: **OLTP e OLAP são separados.** Não rode analytics pesado no banco transacional de produção
— alimente o warehouse/lake por **CDC/ELT** (`references/pipelines.md`). Um `GROUP BY` de relatório
não pode competir com o checkout.

## 2. Particionamento — a unidade de escala e de reprocessamento

- **Particione por tempo** (dia/hora) e/ou **chave** (tenant, região) quando o crescimento é alto e
  previsível (§10 SHOULD). A partição é também a **unidade de reprocessamento** (`pipelines.md` §2)
  e de **retenção** (`governanca.md`): expira-se uma partição inteira, barato.
- **Evite skew e small-files:** partição por chave de altíssima cardinalidade gera milhões de
  arquivinhos no lake (mata performance); por chave enviesada, concentra carga. Escolha a chave
  pela distribuição real, não pela intuição.
- **Cluster/sort dentro da partição** (ClickHouse `ORDER BY`, Iceberg sort order) pra os predicados
  quentes — é o que faz o `WHERE` pular dado (partition pruning + data skipping).
- **Tenant no particionamento** quando multi-tenant, alinhado ao isolamento (§15) — facilita
  deletar/exportar por tenant (LGPD) e evita varredura cross-tenant.

## 3. Lakehouse — bronze / silver / gold (medalhão)

Padrão ELT em camadas, cada uma um contrato:

| Camada | O que é | Contrato |
|---|---|---|
| **Bronze** | dado **cru**, como chegou (append, imutável) | fiel à origem; guarda pra **reprocessar** (`pipelines.md`) |
| **Silver** | limpo, validado, deduplicado, conformado | passou a **qualidade** (`qualidade.md`); schema estável |
| **Gold** | agregado/modelado pro consumo (BI, feature, API) | **contrato público** versionado (`contratos.md`) |

- **Bronze é imutável e reprocessável:** nunca se edita bronze; corrige-se reprocessando silver/gold
  a partir dele. É o que torna backfill/replay possíveis.
- **Formato aberto e colunar:** Parquet/ORC como base; **tabela transacional** (Iceberg/Delta/Hudi)
  pra ganhar `MERGE`/upsert idempotente, **time-travel** (ler snapshot antigo — ótimo pra auditoria
  e replay) e **schema evolution** controlada.
- **Compaction/manutenção:** compacte small-files, expire snapshots antigos conforme retenção, colete
  metadados. Lake sem manutenção degrada até parar.

## 4. Migração e backfill de DADO vivo

O **DDL** do schema (expand → migrate → contract, constraints, `down` da migration, índice sem downtime) é da `schematize-database → references/migrations.md`; esta skill não o repete. Aqui é o **dado** que se move:

- **Backfill grande é operação planejada:** em **lotes** (não um `UPDATE` de milhões que trava a tabela), com throttle, idempotente, monitorado, **reversível** (como desfazer/reprocessar) e registrado (plano no archive).
- **Só contrai quando ninguém lê:** o **lineage/catálogo** (`governanca.md`) confirma que todo leitor migrou antes de o campo/dataset velho sair; compat de contrato em `contratos.md`.
- **Dual-write só dentro do mesmo banco e da mesma transação.** O dual-write vetado (piso 2) é escrever em **dois sistemas** (banco + broker) sem transação que cubra os dois.
- **Zero-downtime como default:** deploy gradual (blue/green, rolling) sem janela de manutenção.

## 5. Segurança e multi-tenancy no store (herdado, §13-15)

- **Deny-by-default e tenant no `WHERE`** (`references/governanca.md` §4, *"Acesso — deny-by-default e least-privilege"*): toda query filtra por
  `tenant_id`; **RLS** no Postgres quando couber. Nunca confiar em `tenant_id` do cliente.
- **Query parametrizada, sempre** (§10/§37): input externo nunca concatenado em SQL.
- **Cripto em repouso** pra dado pessoal (§32); cache de resposta autenticada segmentado por
  usuário+tenant (§11).
- **Sem join cross-service no banco** (§10): a fronteira entre serviços é o **contrato/evento**, não
  um `JOIN` no schema alheio.

## 6. Onde registrar (archive)

DDL/schema, migrations (com `down`), planos de backfill e decisões de particionamento/camada moram
em `<projeto>_archive/data/` (§28), versionados. A migration é código com history — nunca um
comando digitado à mão em produção sem rastro.

> Regra de bolso: **em dado vivo, você adiciona antes de tirar, backfilla no meio, e só corta o
> velho quando o lineage prova que ninguém lê mais — cada passo reversível.**
