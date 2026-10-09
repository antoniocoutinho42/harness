# Guia de migração da v1 para a v2

Este guia explica como adaptar para o runtime v2 os artefatos de harness gerados pela v1 (até a versão 1.2.x). Se a skill Harness detectar artefatos v1 durante a fase 0, ela deverá propor e executar automaticamente o procedimento. Para uma migração manual, siga as etapas abaixo.

## Por que migrar?

A v1 dependia das APIs `TeamCreate`, `SendMessage` e `TaskCreate` habilitadas pela flag experimental `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. No Claude Code atual:

- `TeamCreate`, `TeamDelete` e `team_name` **foram removidos**. Há uma equipe implícita por sessão, e os agentes criados com `Agent(name: ...)` são seus integrantes.
- `SendMessage` e tarefas compartilhadas (como `TaskCreate`) **funcionam sem a flag experimental**.
- A ferramenta **`Workflow`** permite orquestração determinística e substitui os grandes fan-outs e ciclos de verificação antes simulados com equipes.

Executar um orquestrador v1 sem migração pode provocar tentativas de chamar ferramentas inexistentes e degradar silenciosamente a execução para um único agente. Essa foi uma das principais razões para a v2.

## Mapeamento das alterações

| v1 | v2 |
|---|---|
| `TeamCreate(team_name, members: [...])` | Criar agentes em paralelo com `Agent(name: "...", ...)` em uma única mensagem |
| `TeamDelete` ou “Fase N: encerrar equipe” | Remover; os agentes encerram naturalmente ou por `TaskStop`, se necessário |
| Restrição de uma equipe ativa por sessão e rotinas de recriação | Remover; a restrição não existe mais |
| Broadcast `SendMessage({to: "all"})` | Enviar mensagens individuais somente aos destinatários necessários |
| `export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | Remover |
| `model: "opus"` em todos os agentes | Escolher o nível fable/opus/sonnet segundo complexidade, duração, autonomia e latência (`references/model-selection-guide.md`) |
| Fan-out em grande escala e ciclos Produção–Verificação baseados em equipes | Scripts `Workflow`, com `pipeline()`, `schema` e verificação adversarial |
| Inspeção de contexto da fase 0 e ramificações de `_workspace/` | Manter; adicionar `resumeFromRunId` para workflows |
| Convenções de arquivos em `_workspace/`, referências mínimas e histórico no `CLAUDE.md` | Manter integralmente |

## Procedimento manual

1. **Auditoria:** procure `TeamCreate`, `TeamDelete`, `team_name`, `to: "all"`, `EXPERIMENTAL_AGENT_TEAMS` e `model: "opus"` nas definições de agentes e orquestradores.
2. **Reavaliar o modo:** para cada fase, pergunte se a lista de tarefas, os critérios de verificação e as condições de repetição podem ser expressos antecipadamente em código.
   - **Sim:** migre para workflows (consulte `references/workflow-recipes.md`).
   - **Não, e há necessidade de negociação iterativa:** use agentes persistentes (`Agent(name:)`, `SendMessage` e tarefas compartilhadas).
   - **Não, e basta receber um resultado:** simplifique para uma chamada pontual de subagente.
3. **Revisar os agentes:** substitua a configuração geral `model: "opus"` por uma escolha justificada entre fable/opus/sonnet; adapte a seção “Protocolo de comunicação da equipe” ao modo usado (nos agentes exclusivos de workflow, substitua por “Saída estruturada”) e acrescente “Instruções para nova execução”.
4. **Atualizar a documentação:** remova orientações sobre a flag experimental de README e `CLAUDE.md`.
5. **Validar:** realize verificações estruturais e um dry run conforme a fase 6. Confira ao final se não restaram ocorrências dos elementos obsoletos em contextos operacionais.
6. **Registrar:** acrescente “Migração para v2” ao histórico de alterações de `CLAUDE.md`.

## Migração automática

No projeto, basta pedir:

```
Audite meu harness e proponha a migração para a v2, se necessária.
```
