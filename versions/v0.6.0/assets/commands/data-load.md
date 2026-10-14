---
description: schematize-data — carrega à força TODO o corpo normativo (contratos, pipelines, qualidade, armazenamento, governança) e passa a aplicá-lo
---

Carregue **à força** e passe a aplicar **integralmente** os Padrões de Engenharia de Dados da Casa
(skill `schematize-data`) neste projeto. A partir de agora, nesta sessão, isto **não é opcional**.

1. **Leia agora, na íntegra, TODOS os references** — não trabalhe de memória. Caminho:
   `.claude/skills/schematize-data/references/*.md` (projeto) ou
   `~/.claude/skills/schematize-data/references/*.md` (global):
   - `contratos.md` — **dado é contrato**: onde há fronteira, modelagem, versionamento (`v1`/`v2`),
     matriz de **compatibilidade** (back/forward/full), schema registry com checagem no CI,
     política de evolução (o que é aditivo vs o que quebra — incl. mudança silenciosa de semântica).
   - `pipelines.md` — **movimento**: ETL vs ELT, batch (janela/watermark/backfill), streaming
     (entrega/ordem/backpressure/DLQ), CDC e **event sourcing**, **idempotência**, entrega via
     **Outbox** (dual-write vetado), **reprocessamento/replay** determinístico sem re-disparar efeito.
   - `qualidade.md` — **portão**: validação na borda (rejeição, não coerção), dimensões da
     qualidade, expectations no CI e em runtime, **quarentena**/dead-letter (dado ruim não
     contamina, não some), SLA de dados e alerta.
   - `armazenamento.md` — **onde pousa**: escolha do store (OLTP≠OLAP), particionamento,
     lakehouse bronze/silver/gold, e **migration expand-contract** reversível (nunca `DROP`/rename
     destrutivo num passo; backfill em lotes; `down` testado).
   - `governanca.md` — **PII/LGPD**: classificação, base legal + **retenção** aplicada, direitos do
     titular, **deny-by-default** no acesso (server-side, tenant no `WHERE`/RLS), mascaramento/
     anonimização, **lineage & catálogo** (nenhum dataset órfão, rastreável até a origem).

2. **Confirme ao usuário** que leu (1 linha por arquivo).

3. Deste ponto, aplique como regra inegociável: **dado é contrato** (breaking change → nova versão
   + janela; consumidor ignora campo desconhecido), **entrega via Outbox** (dual-write vetado, DLQ
   sempre), **idempotência e replay seguros**, **dado ruim vai pra quarentena** (nunca contamina o
   downstream), **migration expand-contract reversível**, **deny-by-default no acesso**, **nada de
   PII sem base legal e retenção** (fora de log/URL), e **lineage rastreável**. Toda decisão de
   dados durável mora em `<projeto>_archive/data/`.

4. **Atualize o `CLAUDE.md` da raiz** com `assets/CLAUDE.md` da skill (mescla se já houver de outra
   skill) — é o `/data-claude`.
