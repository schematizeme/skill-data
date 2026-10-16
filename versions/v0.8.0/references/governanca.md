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

A **marca da coluna PII no DDL** (comentário, mecanismo de expurgo, PII fora de índice) é da `schematize-database → references/seguranca-pii.md`; a classificação, a base legal e a retenção são daqui.

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
- **Evento de analytics de produto é coleta na origem:** o evento nasce com finalidade, propriedades
  fechadas e base legal; consentimento e modo sem cookie são decididos **antes** do SDK disparar, não
  limpos depois no warehouse. Piso aqui; o desenho (tracking plan, consentimento, revogação) é da
  `schematize-analytics → references/consentimento-lgpd.md` e `references/plano-de-tracking.md`.

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

### 5.1 Endereço de contato: a cópia prd → hml/dev REESCREVE o e-mail (rota nula)

Recorte de dados do piso **"efeito externo nunca sai de não-produção"** (normativa:
`schematize-engineering` → `references/efeitos-externos.md`; DNS/rede é a `schematize-infra`).
Aqui o ponto é só um, e é de dados: **um dataset com e-mail real num ambiente que envia é um
incidente esperando o próximo laço.** O laço que originou o piso disparou **>5.000 e-mails reais**
porque a base de hml carregava endereços que **existiam** — reputação de IP/domínio queimada, o
transacional de **produção** (inclusive o **OTP de login**) parando de chegar, semanas de warm-up.
Não tem undo, e o gatilho foi **o dado**, não o código.

- **A cópia de prd para hml/dev NUNCA leva e-mail real.** Vale para dump/restore, snapshot,
  export pro lake, subset "só uns clientes pra testar" e refresh de ambiente — todos passam pelo
  **mesmo** pipeline de mascaramento (§5), e nele o **endereço é REESCRITO**, junto do resto da PII.
- **Reescrever ≠ mascarar parcialmente.** `j***@gmail.com` continua apontando pro **domínio real**
  e volta a ser entregável assim que alguém "descobre" o padrão. O que importa não é ofuscar o
  caractere: é **trocar o domínio por um em rota nula** — `test.<domain>` (null MX + SPF `-all` +
  DMARC `p=reject`) ou TLD reservado (`.test`/`.invalid`/`.example`).
- **Determinístico por hash com chave, pra preservar unicidade e joins:**

  ```
  email_mascarado = "user+" || hex(HMAC-SHA256(k_snapshot, lower(trim(email_original))))[0:12]
                           || "@test.<domain>"
  ```

  - **Determinístico** ⇒ o mesmo endereço vira o mesmo pseudônimo em **todas** as tabelas e
    datasets do snapshot: `UNIQUE` continua valendo, e o join por e-mail (quando existir) continua
    fechando. Um `uuid()` aleatório por linha quebraria os dois.
  - **Com chave (HMAC), não hash puro.** O conjunto de e-mails do mundo é **enumerável**: `sha256`
    sem chave é reidentificação por dicionário em minutos — pseudonimização de mentira. A chave
    `k_snapshot` é **por snapshot**, mora no **cofre** e é **descartada** quando a re-linkagem com
    prd não for necessária (aí a pseudonimização vira anonimização de fato).
  - **Colisão** é tratada como qualquer chave derivada: 12 hex bytes bastam pro volume típico; se o
    dataset for grande, aumente o corte em vez de "desempatar" com o endereço original.
- **Mesma disciplina pro resto do contato**, que é o mesmo bug em outra porta: **telefone** →
  faixa/magic number de teste do provedor (nunca o número real, nunca "trocando o último dígito");
  **push token** → descartado; **webhook/callback URL do cliente** → reescrita pro sink interno;
  **cartão/token de PSP** → token de teste. Reescrever o e-mail e deixar o telefone real é meia
  cópia limpa.
- **A reescrita acontece na SAÍDA de prd** — no job de export/dump, antes do dado atravessar a
  fronteira do ambiente. "Copia agora e mascara depois que chegar em hml" já é **vazamento** (§2) e
  já deixou a munição carregada no ambiente que envia.
- **Geradores de fixture, factory, seed, persona, demo e teste de carga produzem SÓ o domínio de
  teste**, na forma canônica `<papel>+<run-id>-<n>@test.<domain>` — o `+tag` mantém unicidade por
  execução e deixa o rastro de qual run gerou aquilo. **VETADO** em qualquer artefato de dados:
  `@gmail.com`/`@hotmail.com`/`@outlook.com` e afins, domínio do cliente/parceiro, e-mail de
  **pessoa real (inclusive o seu)** e o **domínio de produção** da casa.
- **É expectation, não boa intenção** (`references/qualidade.md` §3): em dataset de não-produção,
  a asserção **"0 linhas com domínio de e-mail fora da allowlist de teste"** roda no portão; o que
  reprova vai pra **quarentena** e o refresh **falha** — dado de contato real em ambiente que envia
  é **quarentena, não warning**. O mesmo grep trava no CI — e ele tem nome e caminho: **`scripts/check-external-effects.sh`**
  (distribuído nesta skill), que varre `seed*/`, `fixtures/`, `test*/`, `*.sql` e `.env*` atrás
  de `gmail|hotmail|outlook|yahoo|icloud` e afins. *Dizer "o mesmo grep trava no CI" sem
  apontar qual era o achado D2 da vistoria de 2026-08-21: piso sem endereço não é gate.*
- **A classificação viaja com a cópia** (§1): a coluna reescrita **continua marcada como PII** no
  catálogo do ambiente de destino — pseudônimo não é dado anônimo enquanto a chave existir.

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
