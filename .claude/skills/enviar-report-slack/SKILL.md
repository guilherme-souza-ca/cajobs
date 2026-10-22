---
name: enviar-report-slack
description: Envia um relatório da pasta reports/ para um canal (ou DM) do Slack. Posta só um título no canal e o conteúdo na thread. Recebe dois parâmetros — qual relatório (nome da automação para o mais recente, nome/data, ou caminho do .md) e o canal (#nome, ID C…/G… ou usuário U… para DM). Use quando pedirem "manda o último report do erros-5xx-otc no #canal", "envia esse relatório no Slack", ou ao final de uma automação do cajobs que precise publicar o resultado.
argument-hint: "<relatório> <canal> [--titulo \"Texto - {data}\"] [--preview]"
allowed-tools: Read, Glob, mcp__claude_ai_Slack__slack_search_channels, mcp__claude_ai_Slack__slack_send_message
---

# Enviar relatório para o Slack

Publica no Slack um relatório gerado pelas automações (`reports/<automação>/AAAA-MM-DD_HHMMSS.md`) no formato **título + thread**: no canal fica só uma mensagem curta com o título; o conteúdo do relatório vai como resposta na thread dela. Assim o canal não fica poluído e cada relatório é achado pelo título. Responda em PT-BR.

## Parâmetros

Entrada: `$ARGUMENTS` → `<relatório> <canal> [--titulo "<texto>"] [--preview]`

**`<relatório>`** — um destes formatos (caminhos relativos à raiz do projeto):

| Formato | Exemplo | Resolve para |
|---|---|---|
| nome da automação | `erros-5xx-otc` | relatório **mais recente** de `reports/erros-5xx-otc/` |
| automação + data | `erros-5xx-otc/2026-10-22` | o mais recente daquele dia |
| caminho do arquivo | `reports/erros-5xx-otc/2026-10-22_093116.md` | o próprio arquivo |

**`<canal>`** — `#nome-do-canal`, `nome-do-canal`, ID de canal (`C…`/`G…`) ou ID de usuário (`U…`, envia DM).

**`--titulo "<texto>"`** (opcional) — título da mensagem principal. Aceita o marcador `{data}`, substituído pela data do relatório (`início` do cabeçalho) em `DD/MM/AAAA`. Default: `Report <automação> - {data}`.

**`--preview`** (opcional) — monta e mostra as mensagens, mas **não envia**.

Se faltar `<relatório>` ou `<canal>`, pare e peça o que falta com um exemplo de uso. Não suponha valores.

## Passo a passo

1. **Resolver o relatório**
   - Nome ou nome/data: use `Glob` com `reports/<automação>/*.md` (ou `reports/<automação>/<data>_*.md`) e escolha o de **maior nome** (o nome é o timestamp).
   - Caminho: confira que existe e que está dentro de `reports/`. Recuse caminhos fora de `reports/`.
   - Nada encontrado → pare e liste as automações que existem em `reports/`.

2. **Ler e separar** o arquivo com `Read`:
   - O cabeçalho `<!-- … -->` traz `automação`, `início` e `status`. Use esses campos e **remova o comentário** do corpo.
   - A data do título vem do `início` do relatório (não da data de hoje).

3. **Resolver o canal**
   - ID (`C…`, `G…`, `U…`): use direto.
   - Nome: tire o `#` e busque com `slack_search_channels` (`keywords: ["<nome>"]`, `channel_types: "public_channel,private_channel"`). Use **só** correspondência exata do nome; se não houver, pare e mostre os canais parecidos encontrados. Nunca envie para um canal "parecido".

4. **Montar as mensagens**

   **Mensagem principal (canal)** — só o título, entre crases, em uma linha:
   ```
   `<título>`
   ```
   - Ex.: `` `Report Diário De Error De Commercial - 22/10/2026` ``
   - Status `FALHOU…` → acrescente após o título: ` ⚠️ falhou`. Nada além disso na mensagem principal.

   **Conteúdo (thread)** — o relatório em markdown padrão (o Slack renderiza títulos, negrito, tabelas e links):
   ```
   📊 *<automação>* — <início> · <status>

   <corpo do relatório>

   _Arquivo: reports/<automação>/<arquivo>.md_
   ```
   - Status `FALHOU…` → a primeira linha começa com `⚠️` em vez de `📊`.
   - Não reescreva nem resuma o conteúdo: o relatório vai como está (só sem o comentário de cabeçalho).
   - Nunca coloque segredos/tokens na mensagem; se o relatório tiver algo assim, pare e avise.

5. **Dividir o conteúdo se necessário** — o limite é **5.000 caracteres por mensagem**:
   - Cabe → uma resposta só na thread.
   - Não cabe → quebre em blocos ≤ 4.800 caracteres **nos limites de seção (`##`/`###`)**; nunca no meio de uma tabela ou bloco de código (se uma tabela sozinha passar do limite, quebre entre linhas e repita o cabeçalho). Todas as partes vão na thread, em ordem, marcadas `(1/N)`, `(2/N)`…

6. **Enviar** com `slack_send_message`, nesta ordem:
   1. Mensagem principal no `channel_id` → guarde o `ts` retornado.
   2. Cada parte do conteúdo com `thread_ts` = esse `ts` (sem `reply_broadcast`, para não repostar no canal).
   - Se a mensagem principal saiu mas uma resposta da thread falhar, informe qual parte falhou e o link da thread — não reenvie a principal (evita título duplicado no canal).
   - Com `--preview`, pule este passo e mostre a mensagem principal e as partes da thread.

## Resposta

Uma linha por envio, sem comentários extras — nada sobre a sessão/ambiente (`sdk-cli`, plugins como pattern-library, hooks) nem sobre outros conectores MCP. Com `--preview`, mostre apenas o destino, o título, a contagem de partes da thread e o conteúdo montado.

```
✔ Enviado: reports/<automação>/<arquivo>.md → #<canal> — "<título>" + <N> resposta(s) na thread — <link da mensagem principal>
```

Em erro (canal não encontrado, sem permissão, bot fora do canal privado, Slack Connect), devolva o erro exato e o que fazer — por exemplo, adicionar o app do Claude ao canal privado.
