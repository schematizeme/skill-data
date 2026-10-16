# Qualidade de dados — validar, testar, quarentenar (dado ruim não passa)

> Código passa por teste; **dado, também**. Um pipeline pode estar perfeito e mesmo assim entregar
> lixo, porque a **origem** mandou lixo. Este reference rege como se **valida na borda**, se
> **testa o dado** (não só a lógica), e o piso que separa engenharia de dados de "joga no banco e
> reza": **dado que reprova vai pra quarentena — nunca contamina o downstream, nunca some em
> silêncio**.

A intuição vem da mesma disciplina de rejeição da casa (validação na borda, §12) e do
"suspeita ≠ achado" da `schematize-pentest`: dado que chega é **suspeito** até validar; só o que
passa o contrato é **confiável** e segue pro dataset bom.

## 1. Validar na borda de ingestão (não lá dentro)

- **Toda entrada de dado é validada no ponto de ingestão** — contra o **contrato**
  (`references/contratos.md`): forma (schema), tipos, obrigatoriedade, faixa, enum fechado,
  referência (FK lógica). Validar depois, no meio do pipeline, é deixar o lixo entrar e espalhar.
- **Rejeição explícita, não coerção silenciosa.** Coagir `"abc"` pra `0`, truncar string, assumir
  default onde faltou obrigatório — isso **fabrica** dado e esconde o problema. Reprovou o contrato,
  vai pra quarentena (§4). (Mesma guarda anti-coerção da pentest.)
- **PII na borda:** o que não pode nem pousar (`references/governanca.md`) é dropado/mascarado
  **aqui**, antes de gravar — não depois.

## 2. As dimensões da qualidade (o que se checa)

Qualidade de dado não é uma coisa só. Cubra as dimensões e declare o **esperado** por dataset:

| Dimensão | Pergunta | Exemplo de check |
|---|---|---|
| **Schema/validade** | a forma bate com o contrato? | tipo, nulabilidade, enum ∈ conjunto |
| **Completude** | faltou o que é obrigatório? | `not null`; % de nulos ≤ limite |
| **Unicidade** | há duplicata da chave? | `count(distinct id) == count(*)` |
| **Faixa/sanidade** | o valor faz sentido? | `amount ≥ 0`; data não no futuro; % ∈ [0,100] |
| **Integridade referencial** | a referência existe? | FK lógica resolve; sem órfão |
| **Freshness** | o dado está atrasado? | última partição ≤ SLA (`references/pipelines.md` §8) |
| **Volume** | o volume é plausível? | linhas por janela dentro de banda; queda súbita = origem quebrada |
| **Distribuição/deriva** | a estatística mudou? | média/cardinalidade fora do histórico (drift) |

## 3. Testes de dados — expectations como código, no CI e em runtime

- **Expectations declarativas** (à la Great Expectations / dbt tests / Soda / constraints do
  warehouse): cada dataset tem um **conjunto de asserções versionado** (`references/contratos.md`).
  Ex.: "`user_id` nunca nulo", "`status ∈ {open,paid,void}`", "`total == sum(itens)`".
- **Dois momentos:**
  - **No CI (build do pipeline):** testa a **transformação** com fixtures — a lógica produz a forma
    esperada. É o teste de código do dado.
  - **Em runtime (cada execução):** testa o **dado real** que passou — as expectations rodam sobre
    a janela ingerida/transformada. É o portão que pega o lixo da origem, que o CI não vê.
- **Piso da casa:** dado que a DoD (§35) marca como crítico (dinheiro, PII, caminho crítico) tem
  expectations **e** roda o portão em runtime. "Confiei na origem" não é controle de qualidade.
- **Expectation obrigatória em ambiente de não-produção:** **0 linhas com domínio de e-mail (e de
  contato em geral) fora da allowlist de teste**. Dado de contato **real** num ambiente que envia é
  **quarentena** e **refresh reprovado**, nunca warning — é ele que vira o disparo em massa no
  próximo laço (`references/governanca.md` §5.1).

> Bug de dado já corrigido não pode voltar: a expectation que o pegaria entra na suíte vista
> vermelha antes do fix (`schematize-regression` → `references/contrato-dados-ia.md`).

## 4. Quarentena e dead-letter — o destino do dado ruim (o piso)

O que reprova **não** entra no dataset bom e **não** é jogado fora:

- **Quarentena:** dado que falha a validação/expectation vai pra uma **área de quarentena**
  (dead-letter de dados) — a linha/registro **mais** o **motivo** (qual regra falhou), a **origem**
  (arquivo/offset/partição/`trace_id`) e o **timestamp**. Fica rastreável e reprocessável.
- **Nunca contamina o downstream:** o dataset "bom" só recebe o que passou. Um registro ruim no
  meio de um lote **não** derruba o lote inteiro (isola o registro) **nem** passa junto (não
  contamina) — vai pro side-channel de quarentena.
- **Nunca some em silêncio:** quarentena/DLQ **alerta** (§16) e é **revisada**. Quarentena que
  cresce sem ninguém olhar é dado perdido com aparência de tratado. Métrica de tamanho da
  quarentena é SLI de qualidade.
- **Reprocessável:** corrigida a origem (ou a regra), o dado quarentenado é **reprocessado** pelo
  mesmo pipeline (idempotente, `references/pipelines.md`) — não re-digitado à mão.

## 5. Contrato de qualidade / SLA de dados

- **SLA de dados por dataset:** freshness máxima, completude mínima, taxa de erro tolerada,
  disponibilidade. É a versão "dados" do SLO (§16.4) — e, como o SLO, **consumido o budget, freia**
  (não se promove pra gold um dado que estourou o SLA).
- **Owner explícito:** todo dataset tem **dono** (do catálogo, `references/governanca.md`) que
  responde pelo SLA. Dado sem dono é dado sem qualidade garantida.
- **Contrato de qualidade acoplado ao contrato de forma:** o `references/contratos.md` diz *a
  forma*; o SLA diz *o estado saudável*. Os dois juntos são a promessa completa ao consumidor.

## 6. Anti-padrões de qualidade (VETADOS)

- **Coerção silenciosa** pra "não quebrar o pipeline" — fabrica dado, esconde a origem quebrada.
- **`try/catch` que engole a linha ruim** sem quarentena nem alerta — dado sumiu, ninguém soube.
- **Derrubar o lote inteiro** por um registro ruim — indisponibiliza o bom junto com o ruim
  (isole o registro, não o lote).
- **Métrica de qualidade que ninguém olha** — dashboard de quarentena sem alerta é decoração.
- **"A origem é confiável"** como justificativa pra não validar — origem muda sem avisar; o portão
  é justamente pra isso.

## 7. Onde registrar (archive)

As expectations/SLA por dataset e os relatórios de qualidade (o que reprovou, quanto foi
quarentenado, deriva detectada) moram em `<projeto>_archive/data/quality/` (§28), versionados. A
série mostra a qualidade subindo ou apodrecendo ao longo do tempo.

> Regra de bolso: **dado é suspeito até validar; o que reprova vai pra quarentena com motivo e
> origem, alerta, e é reprocessável — nunca contamina o bom, nunca some.**
