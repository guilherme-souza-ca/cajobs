---
description: Lista os erros (HTTP 5xx + status:error) do order-to-cash-commercial nas últimas 24h (via agent datadog-5xx-order-to-cash-commercial)
argument-hint: "[env, ex.: production]"
---

Use o subagent `datadog-5xx-order-to-cash-commercial` para verificar e listar todos os erros (HTTP 5xx e `status:error` sem 5xx) do serviço `order-to-cash-commercial` no Datadog.

- Janela: últimas 24 horas (`now-24h` → `now`).
- Env: `$ARGUMENTS` (se vazio, não filtre por env e agrupe o resultado por env).

Devolva o relatório do agent exatamente no formato que ele produz, sem resumir nem omitir tabelas.

A resposta deve conter **somente** o relatório. Não acrescente nada antes nem depois dele — em especial:
- nenhum comentário sobre o ambiente/sessão (modo `sdk-cli`, não interativo, plugins como pattern-library, skills, hooks);
- nenhuma menção a conectores/servidores MCP sem autenticação ou indisponíveis que não sejam o Datadog;
- nenhum separador `---` ou nota de rodapé.

Exceção: se a consulta ao Datadog falhar, informe o erro dentro do próprio relatório.
