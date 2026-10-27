---
description: Baseline do legado (inventory) da tabela de preço — p95, requisições e 5xx por EP antes do Canary (default 15/09 → 14/10/2026), via agent datadog-baseline-price-tables
argument-hint: "[--de AAAA-MM-DD] [--ate AAAA-MM-DD]"
---

Use o subagent `datadog-baseline-price-tables` para levantar o baseline do legado da tabela de preço no Datadog.

Argumentos recebidos: `$ARGUMENTS` → `[--de AAAA-MM-DD] [--ate AAAA-MM-DD]`, em qualquer ordem.

- **Janela** (dias UTC completos):
  - sem argumentos → `2026-09-15T00:00:00Z` → `2026-10-15T00:00:00Z`;
  - `--de AAAA-MM-DD` → início em `AAAA-MM-DDT00:00:00Z`; sem `--de`, 30 dias antes do fim;
  - `--ate AAAA-MM-DD` → fim em `(AAAA-MM-DD + 1 dia)T00:00:00Z` (inclui o dia); sem `--ate`, `2026-10-15T00:00:00Z`.
  - Data inválida ou `--de` depois de `--ate` → não consulte; devolva o erro no relatório.
- Passe a janela ao agent em ISO 8601 (`from`/`to`).

Devolva o relatório do agent exatamente no formato que ele produz, sem resumir nem omitir tabelas.

A resposta deve conter **somente** o relatório: nada sobre o ambiente/sessão (`sdk-cli`, plugins, skills, hooks), conectores MCP indisponíveis, separadores `---` ou notas de rodapé. Exceção: se a consulta ao Datadog falhar, informe o erro dentro do próprio relatório.
