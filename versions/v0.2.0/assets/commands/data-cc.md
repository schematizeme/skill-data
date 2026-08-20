---
description: Context Compact — gera handoff (context.md + checklist.md) no <projeto>_archive e compacta
---

Antes de compactar, **arquive o handoff** (não perca o estado do trabalho de dados):

1. `<projeto>_archive/context/<YYYY-MM-DD-HH-MM-SS>-context.md` — estado: datasets/tópicos/tabelas
   em jogo, contratos e versões em coexistência, pipelines mapeados, migrations em andamento
   (fase expand/migrate/contract), quarentena/DLQ pendente, itens de governança (PII/retenção/
   lineage), onde parou.
2. `<projeto>_archive/context/<YYYY-MM-DD-HH-MM-SS>-checklist.md` — **FEITO vs EM ABERTO**
   (contratos definidos vs faltantes, backfills/replays pendentes, expectations por escrever,
   grants por revisar).
3. Só então rode `/compact` (foco na tarefa corrente).
