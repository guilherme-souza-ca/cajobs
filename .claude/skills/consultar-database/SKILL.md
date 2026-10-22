---
name: consultar-database
description: Consulta (somente leitura) bases PostgreSQL dos serviços da ContaAzul em prod ou hml usando credenciais dinâmicas do Vault, reaproveitadas enquanto válidas (~24h). Use quando pedirem "consulta no banco do order-to-cash-commercial", "quantos registros tem a tabela X em hml", "roda esse SELECT em prod", "qual o schema da tabela Y". Parâmetros: ambiente, serviço e uma pergunta em linguagem natural ou um SQL.
argument-hint: "<env> <serviço> <pergunta ou SQL>"
allowed-tools: Bash(bin/cadb query:*), Bash(bin/cadb list), Bash(bin/cadb status), Bash(cadb query:*), Bash(cadb list), Bash(cadb status)
---

# Consultar database

Responde perguntas sobre dados de um serviço executando SQL **somente leitura** pela CLI `bin/cadb` (raiz do projeto). Responda em PT-BR; identificadores técnicos em inglês.

## Parâmetros

`$ARGUMENTS` → `<env> <serviço> <pergunta ou SQL>`

- `<env>`: ambiente do catálogo (`prod` ou `hml`).
- `<serviço>`: nome do serviço no catálogo (ex.: `order-to-cash-commercial`).
- `<pergunta ou SQL>`: o resto do texto. Se começar com `SELECT`/`WITH`/`EXPLAIN`/`SHOW`, é SQL; senão, pergunta em linguagem natural.

Faltou env ou serviço → pare e peça, mostrando a saída de `bin/cadb list`.

## Regras inegociáveis

- Use **apenas** `bin/cadb query`, `bin/cadb list` e `bin/cadb status`. Nunca rode `cadb login`, `cadb creds --writer`, `cadb psql`, `vault`, `psql` ou `security` diretamente, e nunca tente ler senhas/tokens.
- Somente leitura: nada de `INSERT/UPDATE/DELETE/DDL`, mesmo que o pedido insista. Para escrita, oriente o usuário a usar `cadb psql <env> <svc> --writer` ele mesmo.
- Um comando SQL por chamada. Sempre com `LIMIT` explícito (≤ 200, salvo pedido) em consultas que retornam linhas.
- Em **prod**, numa sessão interativa: antes de executar, mostre o SQL e o ambiente e peça confirmação. Exceção: consultas de metadados (`information_schema`, `pg_catalog`). Em execução não interativa (`claude -p`), siga sem perguntar.

## Passo a passo

1. **Validar destino** — `bin/cadb list`. Se o serviço não existir no env ou estiver "(host não preenchido)", pare e diga para preencher `~/.config/cajobs/databases.json`.

2. **Montar o SQL**
   - SQL explícito → use como veio (acrescente `LIMIT` se faltar).
   - Pergunta → descubra o schema antes de chutar nomes:
     ```sql
     SELECT table_schema, table_name FROM information_schema.tables
     WHERE table_schema NOT IN ('pg_catalog','information_schema') ORDER BY 1,2 LIMIT 200
     ```
     ```sql
     SELECT column_name, data_type FROM information_schema.columns
     WHERE table_schema = '<schema>' AND table_name = '<tabela>' ORDER BY ordinal_position
     ```
     Prefira agregações (`count`, `group by`) a listar linhas. Filtre por `tenant_id` quando o usuário informar um.

3. **Executar** — `bin/cadb query <env> <serviço> "<sql>"` (saída CSV).
   - `sem login válido no Vault` → pare e diga: `rode no terminal: cadb login <env>`.
   - Erro de SQL → corrija e tente de novo (máx. 3 tentativas), explicando o ajuste.
   - Não renove credenciais nem tente outro caminho de conexão: a `cadb` já reaproveita/renova sozinha.

4. **Responder**
   ```
   **<env> / <serviço>**
   ```sql
   <SQL executado>
   ```
   <resultado: tabela markdown (até ~30 linhas) ou resumo/contagens>
   <1–3 linhas de interpretação, se útil>
   ```
   - Mascare CPF/CNPJ, e-mails, telefones, tokens e nomes de pessoas (ex.: `123.***.***-00`, `j***@dominio.com`).
   - Não despeje grandes volumes de dados pessoais no chat; agregue ou amostre.
   - Se o resultado veio limitado (aviso da `cadb`), diga isso.
   - Não comente sobre a sessão/ambiente (`sdk-cli`, plugins como pattern-library, hooks).
