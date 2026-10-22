# agents

Projeto local para **orquestrar agents e automações do Claude Code** nesta máquina: checagens de observabilidade no Datadog, geração de relatórios e publicação no Slack — com disparo manual ou agendado (launchd), sem infraestrutura extra.

```
                    ┌───────────────────────── cajobs ─────────────────────────┐
 launchd (seg–sex   │  run erros-5xx-otc                                        │
  10:00) ou manual ─┼─► claude -p "/erros-5xx production"                       │
                    │      └─ agent datadog-5xx-order-to-cash-commercial ──► Datadog APM
                    │  grava reports/erros-5xx-otc/AAAA-MM-DD_HHMMSS.md         │
                    │  after: /enviar-5xx-slack                                 │
                    │      └─ skill enviar-report-slack ──► Slack #o2c-commercial-reports
                    └───────────────────────────────────────────────────────────┘
```

## Funcionalidades

- **Agent de erros do `order-to-cash-commercial`** — verifica no Datadog APM as respostas HTTP 5xx **e** as spans com `status:error` que não viraram 5xx (exceções tratadas, banco, chamadas a outros serviços), agrupa por endpoint/operação e tipo de erro e analisa a causa provável nos traces. Somente leitura.
- **Publicação no Slack** — skill reutilizável que envia qualquer relatório de `reports/` para qualquer canal/DM: só o **título** no canal e o **conteúdo na thread**, para não poluir o canal.
- **CLI `cajobs`** — cadastra, executa, agenda (launchd) e acompanha automações; também roda qualquer slash command do projeto pelo terminal.
- **Relatórios e histórico** — cada execução gera um `.md` datado com status no cabeçalho; histórico consolidado em `logs/history.tsv`.

## Pré-requisitos

- macOS com **Claude Code** (`claude`) logado na conta claude.ai.
- Conectores do claude.ai autenticados (via `/mcp`): **Datadog** (com acesso de leitura a APM) e **Slack**.
- `jq` instalado.
- Projeto **confiável** no Claude Code: rode `claude` uma vez dentro desta pasta e aceite o diálogo de confiança — sem isso, as permissões de `.claude/settings.json` são ignoradas e as execuções headless não acessam Datadog/Slack.
- Para canais privados do Slack, o app do Claude precisa estar no canal.

## Instalação

```bash
# CLI disponível em qualquer terminal (~/.local/bin já está no PATH)
ln -s ~/repos/local/agents/bin/cajobs ~/.local/bin/cajobs

# confiar no projeto (uma vez)
cd ~/repos/local/agents && claude   # aceite o diálogo e saia com /exit

# agendar a automação existente
cajobs enable erros-5xx-otc
```

## Estrutura

```
agents/
├── bin/cajobs                          # CLI orquestradora (bash + jq + launchd)
├── automations/                        # definições das automações (JSON)
│   └── erros-5xx-otc.json
├── .claude/
│   ├── agents/                         # subagents
│   │   └── datadog-5xx-order-to-cash-commercial.md
│   ├── commands/                       # slash commands (prompts prontos)
│   │   ├── erros-5xx.md
│   │   └── enviar-5xx-slack.md
│   ├── skills/
│   │   └── enviar-report-slack/SKILL.md
│   └── settings.json                   # permissões pré-aprovadas (headless)
├── reports/<automação>/*.md            # relatórios gerados   (não versionado)
├── logs/                               # logs e history.tsv   (não versionado)
├── CLAUDE.md                           # instruções para o Claude neste projeto
└── README.md
```

## Uso

### CLI `cajobs`

| Comando | O que faz |
|---|---|
| `cajobs list` | lista automações, agenda, se está no launchd e a última execução |
| `cajobs run <nome> [args]` | executa a automação agora (args extras vão para o fim do prompt) |
| `cajobs enable <nome>` | agenda no launchd conforme o `schedule` |
| `cajobs disable <nome>` | remove o agendamento |
| `cajobs status [nome]` | estado do job no launchd (execuções, último exit code) |
| `cajobs logs <nome> [linhas]` | últimas linhas dos logs da automação |
| `cajobs last <nome>` | mostra o último relatório |
| `cajobs new <nome>` | cria `automations/<nome>.json` a partir de um template |
| `cajobs commands` | lista os slash commands do projeto |
| `cajobs command <nome> [args]` | executa um slash command em modo headless (modelo via `CAJOBS_MODEL`, default `sonnet`) |

Exemplos:

```bash
cajobs run erros-5xx-otc                      # checa erros + publica no Slack
cajobs last erros-5xx-otc                     # lê o relatório gerado
cajobs command erros-5xx staging              # só a checagem, saída no terminal
cajobs command enviar-5xx-slack --preview     # mostra a mensagem sem enviar
cajobs command enviar-5xx-slack               # publica o último relatório
```

### Dentro do Claude Code

Abra uma sessão nesta pasta (`cd ~/repos/local/agents && claude`) e use:

| Comando | O que faz |
|---|---|
| `/erros-5xx [env]` | lista 5xx + `status:error` do `order-to-cash-commercial` nas últimas 24h |
| `/enviar-5xx-slack [--preview]` | envia o último relatório do `erros-5xx-otc` para `#o2c-commercial-reports` |
| `/enviar-report-slack <relatório> <canal> [--titulo "…{data}"] [--preview]` | envia qualquer relatório para qualquer canal/DM |

Parâmetros da skill `enviar-report-slack`:

- `<relatório>`: `erros-5xx-otc` (mais recente) · `erros-5xx-otc/2026-10-22` (mais recente do dia) · `reports/erros-5xx-otc/2026-10-22_093116.md`
- `<canal>`: `#nome`, `nome`, ID `C…`/`G…`, ou `U…` (DM)
- `--titulo`: texto do título; `{data}` vira a data do relatório em `DD/MM/AAAA` (default `Report <automação> - {data}`)

## Automações

Cada automação é um JSON em `automations/`:

```json
{
  "description": "5xx + status:error do order-to-cash-commercial nas últimas 24h (production)",
  "prompt": "/erros-5xx production",
  "model": "sonnet",
  "timeout_minutes": 15,
  "schedule": { "hour": 10, "minute": 0, "weekdays": [1, 2, 3, 4, 5] },
  "after": ["enviar-5xx-slack"]
}
```

| Campo | Obrigatório | Descrição |
|---|---|---|
| `description` | não | texto exibido no `cajobs list` |
| `prompt` | sim | prompt ou slash command executado com `claude -p` |
| `model` | não | default `sonnet` |
| `timeout_minutes` | não | default `15`; estourou → relatório `FALHOU (timeout)` |
| `schedule` | não | `hour`, `minute`, `weekdays` (0/7=dom, 1=seg … 6=sab; omitido = todo dia). Sem `schedule` = só manual |
| `after` | não | slash commands executados depois que o relatório é gravado (inclusive em falha) |

### Automação ativa

| Automação | Agenda | O que faz |
|---|---|---|
| `erros-5xx-otc` | seg–sex 10:00 | checa erros do `order-to-cash-commercial` em `production` e publica em `#o2c-commercial-reports` com o título `Report Diário De Error De Commercial - DD/MM/AAAA` |

### Criando uma nova automação

1. **Agent** em `.claude/agents/<agent>.md` — frontmatter com `name`, `description`, `tools` (lista explícita, read-only por padrão) e `model`.
2. **Command** em `.claude/commands/<cmd>.md` que chama o agent (e/ou uma skill).
3. **Permissões**: adicione no `allow` de `.claude/settings.json` toda tool MCP que o agent/skill usa — em modo headless não há prompt de permissão, tool fora do `allow` falha.
4. `cajobs new <nome>` → edite `prompt`, `schedule` e `after`.
5. `cajobs run <nome>` para validar → `cajobs enable <nome>` para agendar.

Para publicar no Slack, crie um command como `enviar-5xx-slack.md` apontando para a skill `enviar-report-slack` com relatório, canal e título, e coloque-o no `after` da automação.

## Saídas

- **Relatório**: `reports/<automação>/AAAA-MM-DD_HHMMSS.md`, com cabeçalho (`automação`, `prompt`, `modelo`, `início`, `fim`, `status`).
- **Histórico**: `logs/history.tsv` (`inicio`, `automacao`, `rc`, `status`, `relatorio`) — automações e commands.
- **Logs**: `logs/<automação>.log` (execução e `after`), `logs/<automação>.launchd.log` (saída do launchd), `logs/commands.log` (`cajobs command`).

## Limitações e troubleshooting

| Situação | O que acontece / o que fazer |
|---|---|
| Mac dormindo no horário | o launchd roda **uma vez ao acordar** |
| Mac desligado ou usuário deslogado | a execução do horário é perdida |
| `status: OK` mas relatório sem dados | `OK` só indica que o `claude` terminou com exit 0 — leia o relatório; geralmente é permissão (projeto não confiável ou tool fora do `allow`) |
| Log com `Ignoring … permissions.allow … not been trusted` | rode `claude` nesta pasta e aceite o diálogo de confiança |
| Erro de autenticação Datadog/Slack | reconecte o conector com `/mcp` numa sessão interativa |
| `já está em execução (lock …)` | outra execução em andamento; se um processo morreu à força, apague `.locks/<nome>` |
| `cajobs: command not found` | confira o link em `~/.local/bin/cajobs` |
| Testar o agendamento sem esperar | `launchctl kickstart -k gui/$(id -u)/com.local.agents.<nome>` |

## Segurança

- Agents são **somente leitura** no Datadog; ferramentas de escrita do Datadog não entram no `allow`.
- A única escrita liberada é o envio de mensagem no Slack, usada apenas pela skill `enviar-report-slack`, que recusa arquivos fora de `reports/` e para se encontrar segredos no conteúdo.
- Mensagens de erro nos relatórios são truncadas e têm tokens, e-mails, CPF/CNPJ e IDs de tenant mascarados.
