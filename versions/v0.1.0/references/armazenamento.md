# Armazenamento — particionamento, warehouse/lakehouse e migrations seguras

> Onde o dado **pousa** e como ele **muda de forma sem quebrar**. Este reference cobre a escolha do
> store, o **particionamento**, a arquitetura de **warehouse/lakehouse** (bronze/silver/gold) e o
> piso mais perigoso do dia a dia de dados: **migration de dado vivo é expand-contract e
> reversível** — nunca um `DROP`/rename destrutivo num passo só.

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

## 4. Migration de dado vivo — expand-contract (o piso)

Mudar o schema de uma tabela/dataset que **está sendo lido e escrito** não pode parar o serviço nem
perder dado. Nunca faça `ALTER ... DROP COLUMN`/rename num passo só sobre dado vivo — o deploy que
ainda não subiu (ou o que já está de saída) quebra. Faça **expand → migrate → contract**:

1. **Expand (aditivo, compatível):** adicione o novo (coluna/tabela/campo) **opcional, com default**
   — nada quebra, código velho e novo coexistem (`references/contratos.md`, compat backward).
2. **Migrate (backfill + dual-write):** popule o novo a partir do velho (**backfill** em janelas,
   idempotente, `pipelines.md`); o código passa a **escrever nos dois** durante a transição.
3. **Contract (corta o velho — só quando ninguém lê):** quando **todo** leitor migrou (o
   **lineage/catálogo** confirma quem lê, `governanca.md`) e o backfill fechou, remova o antigo.
   Só aqui o `DROP` acontece — em passo separado, deployado depois, reversível.

Regras da migration:

- **Reversível:** todo passo tem **`down`** testado (aplica **e** reverte, §10). Migration só com
  `up` é VETADA. "Rodei em dev uma vez" não é teste de rollback.
- **Rename = adiciona novo + backfill + dual-write + corta velho.** Nunca `RENAME` direto em dado
  vivo com consumidor — é uma quebra de contrato disfarçada.
- **Mudança de tipo = nova coluna + backfill convertido + corta.** Alterar tipo in-place trava a
  tabela e/ou quebra o leitor.
- **Backfill grande é operação planejada:** em **lotes** (não um `UPDATE` de milhões que trava a
  tabela), com throttle, idempotente, monitorado, **reversível** e registrado (plano no archive).
- **Índice sem downtime:** crie concorrente (`CREATE INDEX CONCURRENTLY`), valide, depois use.
- **Zero-downtime como default:** a ordem expand-contract existe justamente pra o deploy poder ser
  gradual (blue/green, rolling) sem janela de manutenção.

## 5. Segurança e multi-tenancy no store (herdado, §13-15)

- **Deny-by-default e tenant no `WHERE`** (`references/governanca.md`, §15): toda query filtra por
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
