---
description: Envia o último relatório do erros-5xx-otc para o canal #o2c-commercial-reports (via skill enviar-report-slack)
argument-hint: "[--preview]"
---

Use a skill `enviar-report-slack` com estes parâmetros:

- Relatório: `erros-5xx-otc` (o mais recente de `reports/erros-5xx-otc/`)
- Canal: `C0C9T85830W` (`#o2c-commercial-reports`) — use o ID direto, sem buscar o canal.
- Título: `Report Diário De Error De Commercial - {data}` (`{data}` = data do relatório em DD/MM/AAAA). Vai sozinho no canal, entre crases; o relatório vai na thread.
- Opções extras: `$ARGUMENTS` (ex.: `--preview` para só montar a mensagem sem enviar).

Ou seja: `/enviar-report-slack erros-5xx-otc C0C9T85830W --titulo "Report Diário De Error De Commercial - {data}" $ARGUMENTS`

Se a ferramenta `Skill` não devolver as instruções da skill (ex.: só "Execute skill: enviar-report-slack"), não pare: leia `.claude/skills/enviar-report-slack/SKILL.md` com `Read` e siga o passo a passo dele com os parâmetros acima.

Responda apenas com a linha de resultado da skill (ou o preview), sem comentários extras — nada sobre a sessão/ambiente (`sdk-cli`, plugins, hooks).
