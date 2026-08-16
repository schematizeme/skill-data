# schematize-data

> **Engenharia de dados** da casa — a disciplina "o dado é um contrato". Rege modelagem e
> **contratos de dados** (schema evolutivo, versionamento, compatibilidade back/forward/full),
> pipelines **ETL/ELT** batch e streaming, **event sourcing / CDC** (via Outbox, sem dual-write),
> **qualidade** com quarentena e dead-letter (dado ruim não contamina o downstream), **lineage &
> catálogo**, particionamento / warehouse / **lakehouse**, **migrations** de dados seguras
> (expand-contract, reversível), **idempotência** e reprocessamento, e **PII/governança/retention/
> LGPD** (deny-by-default no acesso, base legal + retenção obrigatórias).

Pacote de **skill normativa para [Claude Code](https://claude.com/claude-code)**. Parte do catálogo
**schematize skills**. Disciplina **agnóstica de linguagem** que pareia com a
`schematize-engineering` (a base: dados-eventos, segurança/LGPD, observabilidade, archive, DoD); a
implementação por stack fica nas skills irmãs (`schematize-go/rust/elixir/csharp/zig/ruby`);
segurança ofensiva na `schematize-pentest`.

## Instalar

### Pelo app schematize (recomendado)

```bash
schematize install data       # requer o CLI schematize instalado
```

### Última versão (a partir de um clone)

```bash
git clone https://github.com/schematizeme/skill-data.git
cd skill-data && ./install.sh             # instala no projeto atual
# ./install.sh /caminho/do/projeto          # ou aponte para outro projeto
```

Ou baixe o `.zip` da última release e descompacte em `.claude/skills/`:

```bash
curl -L -o skill-data.zip \
  https://github.com/schematizeme/skill-data/releases/latest/download/skill-data.zip
unzip skill-data.zip -d .claude/skills/
```

## O que tem dentro

- **SKILL.md** — o contrato: 9 pisos inegociáveis (dado é contrato → nova versão pra breaking
  change; Outbox sem dual-write; idempotência/replay seguros; dado ruim vai pra quarentena;
  migration expand-contract reversível; deny-by-default no acesso; nada de PII sem base legal e
  retenção; lineage rastreável; saída durável no archive) + mapa de references.
- **references/** — `contratos` (dado é contrato + compatibilidade + registry), `pipelines`
  (ETL/ELT/streaming/CDC + idempotência + Outbox + replay), `qualidade` (validação + expectations +
  quarentena/DLQ + SLA), `armazenamento` (particionamento + lakehouse + migration expand-contract),
  `governanca` (PII/LGPD + retenção + deny-by-default + lineage & catálogo).
- **assets/commands/** — `/data-help`, `/data-contract`, `/data-pipeline`, `/data-quality`,
  `/data-load`, `/data-claude`, `/data-cc`, `/data-handoff`.
- **assets/CLAUDE.md** — regra sempre-on: dado é contrato; dual-write vetado; quarentena pro dado
  ruim; deny-by-default; nada de PII sem base legal e retenção.

## Regra de ouro

**O dado é um contrato.** Quem lê depende da forma que você prometeu — quebrar sem versão quebra o
consumidor; publicar dado sujo contamina o downstream; vazar PII é passivo jurídico. Versione a
mudança com janela, quarentene o que reprova, mantenha o lineage rastreável e trate dado pessoal
como material perigoso: **base legal, retenção e deny-by-default, sempre**.

## Relação com as outras skills

- **schematize-engineering** — a base: dados-eventos (§9-12), resiliência/jobs (§18-19),
  segurança/auth/multi-tenancy/LGPD (§13-15/§32), observabilidade (§16), archive (§28), DoD (§35).
- **skills irmãs (go/rust/elixir/csharp/zig/ruby)** — a implementação por stack (o gate do código:
  `/<slug>-review`, testes, lint). O piso de dados é o mesmo em qualquer linguagem.
- **schematize-pentest** — o oráculo ofensivo: cross-tenant leak, IDOR em endpoint de dados, PII
  vazando em log/resposta, injeção na borda de ingestão.

Co-autoria / patrocínio: Lucassa — https://lucassa.me

MIT.
