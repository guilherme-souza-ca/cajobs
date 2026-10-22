# agents

Projeto local de agents do Claude Code para automações de observabilidade (Datadog).

## Estrutura

- `.claude/agents/*.md` — subagents do projeto (carregados ao abrir o Claude Code neste diretório).
- `.claude/commands/*.md` — prompts prontos (slash commands) que disparam os agents. Ex.: `/erros-5xx [env]`, `/enviar-5xx-slack [--preview]` (último report do `erros-5xx-otc` → `#o2c-commercial-reports`).
- `.claude/skills/*/SKILL.md` — skills do projeto. Ex.: `/enviar-report-slack <relatório> <canal> [--preview]`.
- `.claude/settings.json` — permissões pré-aprovadas: ferramentas **de leitura** do MCP `claude.ai Datadog` + busca de canais e envio de mensagem do `claude.ai Slack` (usado pela skill `enviar-report-slack`).

## Agents

| Agent | O que faz |
|---|---|
| `datadog-5xx-order-to-cash-commercial` | Verifica e lista erros do serviço `order-to-cash-commercial` (APM): HTTP 5xx + spans `status:error` sem 5xx, agrupados por endpoint/operação e tipo de erro. |

## Skills

| Skill | Uso | O que faz |
|---|---|---|
| `enviar-report-slack` | `/enviar-report-slack erros-5xx-otc #meu-canal` | Envia um relatório de `reports/` (mais recente da automação, de uma data, ou caminho) para um canal/DM do Slack: só o título (`--titulo`, aceita `{data}`) no canal e o conteúdo na thread, dividido em partes se passar de 5.000 caracteres. `--preview` só mostra. |

## Convenções

- Responder em PT-BR; identificadores técnicos em inglês.
- Agents são somente leitura por padrão: nunca criar/alterar/apagar recursos no Datadog sem confirmação explícita.
- Ferramentas de escrita do Datadog **não** entram no `allow` do `settings.json`. A única escrita liberada é `slack_send_message`, e só a skill `enviar-report-slack` deve usá-la.
- Novo agent: um arquivo `.claude/agents/<nome>.md` com frontmatter `name`, `description`, `tools` (lista explícita) e `model`.

## Automações (`bin/cajobs` + launchd)

Cada automação é um JSON em `automations/<nome>.json`, executado em modo headless (`claude -p`) a partir da raiz deste projeto — então herda agents, commands, `CLAUDE.md` e o `allow` do `settings.json`.

```json
{
  "description": "texto curto",
  "prompt": "/meu-command args",
  "model": "sonnet",
  "timeout_minutes": 15,
  "schedule": { "hour": 9, "minute": 0, "weekdays": [1, 2, 3, 4, 5] },
  "after": ["meu-command-de-envio"]
}
```

- `after` (opcional): slash commands rodados em sequência depois que o relatório é gravado (mesmo efeito de `cajobs command <nome>`), inclusive quando a execução falha. Ex.: `erros-5xx-otc` usa `["enviar-5xx-slack"]` para publicar no Slack. Falha num `after` faz o `cajobs run` sair com código ≠ 0 e fica registrada em `logs/<nome>.log`.
- `schedule` omitido = só disparo manual. `weekdays`: 0/7=dom, 1=seg … 6=sab; omitido = todo dia.
- Saída: `reports/<nome>/AAAA-MM-DD_HHMMSS.md` (cabeçalho com status). Histórico: `logs/history.tsv`. Stderr: `logs/<nome>.log`; launchd: `logs/<nome>.launchd.log`.

| Comando | O que faz |
|---|---|
| `cajobs list` | automações, agenda, se está no launchd, última execução |
| `cajobs run <nome> [args]` | executa agora (args extras vão para o fim do prompt) |
| `cajobs enable <nome>` / `cajobs disable <nome>` | agenda/remove em `~/Library/LaunchAgents/com.local.agents.<nome>.plist` |
| `cajobs status [nome]` | estado no launchd (runs, last exit code) |
| `cajobs logs <nome>` / `cajobs last <nome>` | logs / último relatório |
| `cajobs new <nome>` | cria o JSON a partir do template |
| `cajobs command <nome> [args]` | executa o slash command `.claude/commands/<nome>.md` em headless (ex.: `cajobs command enviar-5xx-slack --preview`); modelo via `CAJOBS_MODEL` (default `sonnet`); log em `logs/commands.log` |
| `cajobs commands` | lista os slash commands do projeto |

Teste o job pelo launchd sem esperar o horário: `launchctl kickstart -k gui/$(id -u)/com.local.agents.<nome>`.

### Nova automação

1. Agent em `.claude/agents/<agent>.md` (tools explícitas, read-only por padrão).
2. Command em `.claude/commands/<cmd>.md` que chama o agent.
3. Tools que o agent usa no `allow` do `.claude/settings.json` — em `-p` não há prompt de permissão: tool fora do `allow` falha.
4. `cajobs new <nome>` → editar prompt/schedule → `cajobs run <nome>` → `cajobs enable <nome>`.

### Limitações

- O projeto precisa estar **confiável** (trust dialog aceito uma vez rodando `claude` aqui), senão o `allow` é ignorado.
- launchd só roda com seu usuário logado. Mac dormindo no horário → roda uma vez ao acordar; desligado → execução perdida.
- `status: OK` significa que o `claude` terminou com exit 0; leia o relatório para o resultado em si.
- Lock por automação em `.locks/<nome>`; se um processo morrer à força, apague o diretório.
