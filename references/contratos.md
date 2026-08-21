# Contratos de dados — o dado é uma promessa de forma

> O núcleo desta skill: **um schema é um contrato**. Todo dado que cruza uma fronteira (evento num
> tópico, tabela/view que outro serviço lê, arquivo publicado no lake, resposta de uma API de
> dados) carrega uma **promessa de forma** para quem consome. Mudar essa forma sem cerimônia
> quebra o consumidor — normalmente em silêncio, longe no tempo. Este reference define como se
> **modela, versiona e evolui** o contrato sem quebrar quem depende dele.

A regra-mestre é da `schematize-engineering` (§9): *eventos são imutáveis e versionados; novos
campos são opcionais com default seguro; nunca remover/renomear campo sem nova versão; consumidor
ignora campo desconhecido*. Aqui ela vira a disciplina de **todo** dado de fronteira, não só evento.

## 1. Onde há contrato (a fronteira)

Tem contrato **todo** ponto onde um produtor e um consumidor **diferentes** trocam dado:

| Fronteira | Contrato é |
|---|---|
| Evento / mensagem em tópico (Kafka/NATS) | o **schema do evento** (`v1`, `v2`) + a convenção de nome `<dominio>.<entidade>.<evento>` |
| Tabela/view que outro serviço ou o BI lê | o **schema público** (colunas, tipos, nulabilidade, semântica) |
| Arquivo no lake (Parquet/Avro/Delta/Iceberg) | o **schema do dataset** + partição + evolução declarada |
| API de dados / export | o **payload versionado** (`/v1`), erro RFC 7807, paginação por cursor (§12) |
| CDC / stream de mudança | o **shape da linha** + a semântica de `insert/update/delete` |

Dado **interno** de um serviço (tabela privada, struct em memória) **não** é contrato de fronteira
— pode mudar livre, desde que a fronteira pública permaneça. A disciplina protege a **borda**, não
engessa o interior.

## 2. Modelagem — antes de versionar, modelar direito

Um contrato só é bom se o modelo é bom. Pisos de modelagem (agnósticos):

- **Uma unidade semântica por evento/registro.** Evento gordo que carrega meio domínio acopla
  tudo a ele. Modele o fato mínimo (`billing.invoice.paid` com o essencial), não um dump.
- **Chave natural/estável explícita.** Todo registro/evento tem um **id** estável (UUIDv7/ULID) e,
  quando fizer sentido, uma **chave de negócio** (dedupe, idempotência, join). Sem chave, não há
  replay nem idempotência (`references/pipelines.md`).
- **Tipos honestos.** Dinheiro é inteiro em centavos (ou decimal), nunca float. Timestamp é
  **UTC** ISO-8601 com timezone. Enum é fechado e versionado (valor novo ≠ quebra só se o
  consumidor ignora o desconhecido). Nulo é decisão, não descuido — nulabilidade faz parte do
  contrato.
- **Semântica documentada.** O nome do campo não basta: `status` significa o quê em cada valor?
  `amount` é bruto ou líquido? O contrato carrega a **semântica**, senão cada consumidor inventa a
  sua.
- **Separe fato de projeção.** O evento/fato é imutável; *views* e agregações são derivadas e
  podem mudar. Não misture "o que aconteceu" com "como eu gosto de ver".

## 3. Versionamento — `v1`, `v2`, e a coexistência

- **Versão explícita** no contrato: no nome do tópico/schema (`orders.v1`), no header do evento,
  no namespace do dataset, ou na URL da API (`/v1`). Consumidor sabe **qual** versão está lendo.
- **Nunca reescreva `v1` no lugar.** Uma nova forma é `v2`, publicada **ao lado** da `v1`. As duas
  coexistem durante a **janela de migração** (documentada): produtor emite as duas (ou um conversor
  faz a ponte) até todo consumidor migrar; só então `v1` é aposentada.
- **Deprecação com prazo.** Quebra de contrato exige **janela de deprecação** (a base pede ≥ 90
  dias para API, §12, com headers `Deprecation`/`Sunset`); para evento/dataset, a janela é
  declarada e comunicada aos consumidores conhecidos (o **catálogo/lineage** diz quem são —
  `references/governanca.md`).

## 4. Compatibilidade — a matriz que decide "quebra ou não"

Compatibilidade é sobre **quem consegue ler o quê** depois da mudança. Três modos:

| Modo | Garante | Quem se protege | Regra prática |
|---|---|---|---|
| **Backward** | consumidor **novo** lê dado **antigo** | o **consumidor** | pode **adicionar campo opcional/com default**; pode **remover** campo — mas **ler a nota abaixo antes** |
| **Forward** | consumidor **antigo** lê dado **novo** | o **produtor** pode andar na frente | pode **adicionar** campo (o velho ignora o desconhecido); **não** pode remover campo que o velho exige |
| **Full** | os dois ao mesmo tempo | ambos | só **adicionar campo opcional com default** e **remover campo opcional** |

Piso da casa: **evolução é aditiva e opcional-com-default**, e consumidor **ignora campo
desconhecido** (forward-compatible por construção, §9). Escolha o modo por fronteira e **declare-o**
no registry:

- **Mudanças SEGURAS (evolutivas):** adicionar campo opcional com default; alargar um tipo de forma
  compatível (int32→int64 quando o serializador permite); adicionar valor de enum **se** os
  consumidores ignoram o desconhecido; afrouxar obrigatoriedade.
- **Mudanças que QUEBRAM (exigem `v2`):** remover/renomear campo; mudar o tipo de um campo;
  estreitar um tipo; tornar um campo opcional em obrigatório; mudar a **semântica** de um campo
  (mesmo nome, novo significado — a pior, porque passa nos testes de schema e engana o consumidor);
  mudar unidade (reais→centavos) sem renomear.

> A mudança **silenciosa de semântica** é a mais perigosa: o schema continua "válido", os testes de
> forma passam, e o consumidor lê `amount` achando que é bruto quando virou líquido. Semântica faz
> parte do contrato — mudou o significado, mudou a versão.

> **"Backward permite remover campo" e "remover campo é breaking change" não se contradizem — são
> duas perguntas diferentes, e confundi-las quebra produção.**
>
> - A **matriz acima é a regra do REGISTRY**: ela responde *"o schema novo consegue ler o dado
>   escrito com o schema velho?"*. Remover um campo passa nessa checagem, porque o **leitor** novo
>   simplesmente não pede mais aquele campo. O registry aprova.
> - O **piso da casa** (piso 1 do `SKILL.md`, *"Dado é contrato"*) responde outra coisa: *"algum consumidor lá fora ainda lê
>   esse campo?"*. Se sim, remover **quebra**, e o registry **não sabe disso** — ele valida schema
>   contra schema, não schema contra a realidade dos consumidores.
>
> **A regra que vale:** o `BACKWARD` do registry é **condição necessária, não suficiente**. Remover
> ou renomear campo exige, além do verde do registry: **inventário de consumidores** (quem lê, medido
> — não presumido), **janela de depreciação** anunciada, e `v2` quando houver consumidor que não
> migrou. Sem inventário, trate como quebra: *"achei que ninguém usava" é a frase que precede o
> incidente de dados*.

## 5. Schema registry — o contrato como código, verificado no CI

- **Schema como artefato versionado** (Avro/Protobuf/JSON Schema/Iceberg schema), no repo e/ou num
  **schema registry** (ex.: Confluent Schema Registry, Buf, Iceberg catalog). É a **fonte da
  verdade** da forma — como o OpenAPI é pra API (§12).
- **Checagem de compatibilidade no CI:** o pipeline **falha** se uma mudança viola o modo de
  compatibilidade declarado da fronteira (ex.: `BACKWARD` no registry rejeita remoção de campo).
  Contrato não é convenção de boa vontade — é **gate**.
- **Contrato consumer-driven** quando houver poucos consumidores conhecidos (Pact-like): o
  consumidor declara o que precisa; o produtor não pode quebrar isso sem sinal.
- **Gerar tipos do schema**, não escrever à mão dos dois lados: o código do produtor e do consumidor
  derivam do **mesmo** contrato. Divergência entre schema e código é bug de contrato.

## 6. Onde registrar (archive)

O contrato e sua história moram no archive (§28), versionados: `<projeto>_archive/data/contracts/`
(ou no registry, com o snapshot no repo). Cada contrato registra: schema atual + versões vivas,
**modo de compatibilidade**, janela de coexistência aberta, consumidores conhecidos (do lineage) e
o histórico de evolução. Contrato que só existe "no acordo verbal" não é contrato — é acidente
esperando acontecer.

> Regra de bolso: **se outro serviço lê, é contrato; se é contrato, é versionado; se quebra, é
> `v2` com janela.** O consumidor nunca deveria descobrir uma mudança de forma em produção.
