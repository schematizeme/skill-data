---
description: schematize-data — cria ou mescla o CLAUDE.md sempre-on de dados na raiz do repo (não sobrescreve blocos de outras skills)
---

Instale/atualize a regra **sempre-on** de engenharia de dados na raiz do repositório.

1. Pegue `assets/CLAUDE.md` da skill `schematize-data` (projeto ou `~/.claude/skills/...`).
2. Se **não existe** `CLAUDE.md` na raiz: crie com esse conteúdo.
3. Se **já existe** (de outra skill — engineering/go/rust/web/pentest/...): **mescle** — adicione a
   seção de Engenharia de Dados **sem sobrescrever** os blocos das outras skills. Em repo
   multi-skill, cada CLAUDE convive; o piso de dados é aditivo.
4. Se houver customização local, salve `./CLAUDE.md.bak` e reaplique por cima.
5. Confirme a versão aplicada e destaque o **piso**: dado é contrato (breaking change → nova
   versão); dual-write vetado (Outbox); dado ruim vai pra quarentena; deny-by-default no acesso;
   nada de PII sem base legal e retenção.
