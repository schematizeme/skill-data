# Governança — PII/LGPD, acesso deny-by-default, retenção, lineage & catálogo

> Quem pode ver o dado, por quanto tempo ele vive, e de onde ele veio. Este reference rege a
> **governança**: **deny-by-default** no acesso, **PII só com base legal e retenção**, mascaramento/
> anonimização, e o **catálogo + lineage** que torna todo dataset rastreável. Aqui o piso é dos
> mais duros da casa: dado pessoal é passivo jurídico, não ativo — trate-o como material perigoso.

Casa com a base (`schematize-engineering` §13-15 segurança/auth/multi-tenancy, §16.1/16.6 logs e
trilha de auditoria, §32 LGPD).

## 1. Classificação — antes de governar, rotular

Todo dado é classificado (§32). Sem rótulo, não há política aplicável:

| Classe | Exemplo | Regra dominante |
|---|---|---|
| Público | catálogo de produto | livre |
| Interno | métrica agregada | acesso interno controlado |
| Confidencial | contrato, preço negociado | least-privilege, auditado |
| **Pessoal (PII)** | nome, e-mail, CPF, telefone, IP | **base legal + retenção + minimização** |
| **Pessoal sensível** | saúde, biometria, origem, financeiro | tudo do PII **+** cripto forte, acesso restrito, DPIA |

A classificação **viaja com o dado** (metadado no catálogo): uma coluna PII continua PII quando
copiada pro lake, pro warehouse, pro export. Perder o rótulo na cópia é como perde-se o controle.

## 2. PII — os pisos inegociáveis (§16.1 / §32)

- **Nada de PII sem base legal registrada.** Consentimento, contrato, obrigação legal, legítimo
  interesse — a **base** existe e está documentada **antes** de coletar. Coletar "porque pode ser
  útil" é ilegal e é dívida.
- **Minimização:** colete só o necessário pro propósito declarado. Campo pessoal que ninguém usa é
  risco puro — não colete, ou drope na borda (`references/qualidade.md`).
- **PII nunca em log** (§16.1): senha, token, JWT, CPF, e-mail, telefone, dado financeiro,
  payload de pagamento — **mascarados** ou fora do log. Logar body/request cru "pra debugar" é
  VETADO (§37).
- **PII nunca em URL/query string** (§32/§37): acaba em log de acesso, histórico, referer.
- **PII em cache só cifrada** (§11); em repouso, **cripto** (§13); em trânsito, **TLS 1.2+**.
- **Nunca PII de produção em ambiente de teste/dev sem anonimizar.** Dump de prod pra dev é
  vazamento agendado — mascare/gere sintético.

## 3. Retenção — o dado tem prazo de validade

- **Toda classe de dado tem política de retenção** documentada (§32): por quanto tempo vive, e o
  que acontece no fim (apagar ou **anonimizar** irreversivelmente).
- **Retenção é aplicada, não só escrita:** um job expira/apaga/anonimiza no prazo (particionar por
  tempo torna isso barato — `references/armazenamento.md`). Política de retenção sem enforcement é
  ficção.
- **Direitos do titular implementáveis** (§32): acesso, correção, **eliminação**, portabilidade. Se
  o titular pede exclusão, o sistema **sabe onde** o dado dele está (é aqui que o **lineage** paga a
  conta) e consegue apagá-lo em **todas** as cópias (lake, warehouse, backup conforme política,
  índices). "A gente não sabe onde tá o dado do cliente" reprova LGPD.
- **Backups e derivados também retêm:** exclusão que não alcança a cópia no lake/warehouse é
  exclusão incompleta. O plano de retenção cobre todas as camadas.

## 4. Acesso — deny-by-default e least-privilege

- **Negado por padrão.** Acesso a dataset/tabela/tópico é **fechado**; concede-se por **necessidade
  explícita** (least-privilege), nunca "todo mundo lê tudo". O default é `deny`; cada grant é uma
  decisão.
- **Autorização server-side, derivada do token** (§15): quem pode ler o quê é decidido **no
  servidor**, a partir da identidade autenticada — nunca de um `role`/`tenant_id` que o cliente
  mandou. Confiar no cliente é VETADO.
- **Isolamento por tenant** (§15): `WHERE tenant_id` em toda query; **RLS** no Postgres quando
  couber; `tenant_id` propagado em contexto/logs/traces. Teste de **cross-tenant leak** em CI
  (§15 SHOULD; a `schematize-pentest` ataca isso).
- **Colunas sensíveis com acesso mais estreito** que o resto da tabela (column-level / views
  mascaradas): quem lê a tabela não necessariamente lê a coluna de CPF.
- **Trilha de auditoria de acesso a dado sensível** (§16.6): acesso/exportação de PII em massa é
  operação sensível — registrada em trilha **append-only** imutável (`actor_id`, `tenant_id`,
  `action`, `resource`, `timestamp`, `ip`, `result`).

## 5. Mascaramento e anonimização

- **Mascaramento** (reversível/parcial) pro que precisa ser exibido parcialmente (`****1234`).
- **Anonimização** (irreversível) pro que sai do domínio ou é retido além do prazo pessoal: remover
  identificadores **e** cuidar de **reidentificação** por combinação (quase-identificadores —
  k-anonimato/agregação). Trocar o nome por um id não anonimiza se o resto ainda identifica.
- **Pseudonimização** (tokenizar PII, guardar o mapa em cofre separado) pra manter utilidade
  analítica sem espalhar o dado bruto.
- **Dado sintético** pra teste/dev/demo — nunca prod mascarado "às pressas".

## 6. Lineage & catálogo — nenhum dataset órfão (o piso da rastreabilidade)

- **Catálogo de dados:** todo dataset tem ficha — **nome, dono, classificação, schema/contrato,
  SLA, localização**. Dataset que não está no catálogo é dado fantasma: ninguém sabe o que é, quem
  cuida, nem se pode usar.
- **Lineage (linhagem):** de onde o dado **vem** e pra onde **vai** — origem → transformações →
  destinos (o mapa do pipeline, `references/pipelines.md`, alimenta isto). O lineage responde as
  perguntas que salvam o dia:
  - *"Se eu mudar este campo, quem quebra?"* → os consumidores a jusante (contrato, `contratos.md`).
  - *"O titular pediu exclusão — onde está o dado dele?"* → todas as cópias a jusante (§3).
  - *"Este número no dashboard está errado — de onde veio?"* → a cadeia a montante até a origem.
- **Dono explícito por dataset:** responde pelo contrato, pela qualidade (SLA) e pela governança.
  Dado sem dono é dado sem responsável — apodrece.
- **Automatize onde der:** lineage derivado do código do pipeline/CDC e dos schemas (OpenLineage,
  Marquez, catálogos do warehouse) é mais confiável que planilha à mão — mas o **piso** é *existir*
  e estar **rastreável até a origem**, não a ferramenta.

## 7. Onde registrar (archive)

Classificação, políticas de retenção, matriz de acesso (grants), catálogo e mapa de lineage moram
em `<projeto>_archive/data/governance/` (§28), versionados. Sanitize: o registro **descreve** o
dado sensível, nunca **contém** amostra real de PII. Segredo/credencial encontrado no rastro →
sinalize **rotação**.

> Regra de bolso: **deny-by-default no acesso; PII só com base legal e prazo; toda cópia rastreável
> pelo lineage — se você não sabe quem lê nem onde o dado do cliente está, você não governa, você
> torce.**
