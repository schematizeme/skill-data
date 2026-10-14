---
description: Gera o handoff de contexto (context.md + checklist.md) no archive, SEM compactar
---

Gere o handoff do trabalho de dados **sem** compactar — pra fim de sessão ou troca de tarefa:

1. `<projeto>_archive/context/<YYYY-MM-DD-HH-MM-SS>-context.md` — datasets/tópicos/tabelas em jogo,
   contratos + versões vivas e janelas de coexistência abertas, mapa dos pipelines (origem →
   destino), estado das migrations (expand/migrate/contract + rollback), quarentena/DLQ e
   qualidade, governança (classificação/retenção/acesso/lineage), decisões, onde parou.
2. `<projeto>_archive/context/<YYYY-MM-DD-HH-MM-SS>-checklist.md` — **FEITO vs EM ABERTO**
   (contratos, backfills/replays, expectations, grants, retenção — o que falta).

Não rode `/compact` — só arquiva.
