---
description: Lista os erros (HTTP 5xx + status:error) do order-to-cash-commercial e a p95/5xx dos endpoints /price-tables — últimas 24h por padrão, ou um período com --de/--ate (via agent datadog-5xx-order-to-cash-commercial)
argument-hint: "[env] [--de AAAA-MM-DD] [--ate AAAA-MM-DD]"
---

Use o subagent `datadog-5xx-order-to-cash-commercial` para verificar e listar todos os erros (HTTP 5xx e `status:error` sem 5xx) do serviço `order-to-cash-commercial` no Datadog, incluindo a seção de latência (p95) e 5xx dos endpoints `/price-tables`.

Argumentos recebidos: `$ARGUMENTS` → `[env] [--de AAAA-MM-DD] [--ate AAAA-MM-DD]`, em qualquer ordem.

- **Env**: o argumento que não começa com `--` (ex.: `production`). Sem env, não filtre e agrupe o resultado por env.
- **Janela** (horário de Brasília, `-03:00`):
  - sem `--de`/`--ate` → últimas 24 horas (`now-24h` → `now`);
  - `--de AAAA-MM-DD` → início em `AAAA-MM-DDT00:00:00-03:00`;
  - `--ate AAAA-MM-DD` → fim em `AAAA-MM-DDT23:59:59-03:00`; sem `--ate`, o fim é `now`.
  - Data inválida ou `--de` depois de `--ate` → não consulte; devolva o erro no relatório.
- Passe a janela ao agent em ISO 8601 (`from`/`to`) e use-a no título do relatório (ex.: `15/10/2026 00:00 → agora`).

Devolva o relatório do agent exatamente no formato que ele produz, sem resumir nem omitir tabelas.

A resposta deve conter **somente** o relatório. Não acrescente nada antes nem depois dele — em especial:
- nenhum comentário sobre o ambiente/sessão (modo `sdk-cli`, não interativo, plugins como pattern-library, skills, hooks);
- nenhuma menção a conectores/servidores MCP sem autenticação ou indisponíveis que não sejam o Datadog;
- nenhum separador `---` ou nota de rodapé.

Exceção: se a consulta ao Datadog falhar, informe o erro dentro do próprio relatório.
