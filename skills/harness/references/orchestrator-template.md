# Modelos de skills orquestradoras (v2)

O orquestrador coordena o trabalho de toda a equipe. Este documento oferece três modelos, um para cada modo de execução, e orientações para combiná-los.

- **Modelo A — Workflow:** fluxo antecipadamente definido em código; execução distribuída em grande escala ou verificações iterativas.
- **Modelo B — Agentes persistentes:** especialistas que precisam preservar contexto, trocar feedback, negociar e colaborar ao longo da sessão.
- **Modelo C — Subagentes:** tarefas independentes, delegadas pontualmente e em paralelo.
- **Modo híbrido:** combina o modo mais adequado a cada fase.

Todos os modelos devem começar recuperando o contexto na fase 0 e conter convenções para `_workspace/`, tratamento de falhas e cenários de teste.

---

## Modelo A — Orquestração por workflow

```markdown
---
name: {domain}-orchestrator
description: "Coordena o workflow de {domínio}. Use quando o usuário solicitar {expressões de acionamento inicial} e também em revisões de resultados, novas execuções parciais, atualizações, complementações ou melhorias baseadas em resultados anteriores."
---

# Orquestrador — {Domínio}

## Modo de execução: workflow

Esta skill utiliza a ferramenta Workflow para coordenar tarefas com fluxo de controle predefinido.
Acionar explicitamente esta skill constitui autorização para usar Workflow.
Por padrão, limite a execução a {N} agentes; amplie somente se o usuário
solicitar uma análise abrangente, exaustiva ou integral.

## Composição dos agentes

| Papel | `agentType` | Skill | Saída (resumo do `schema`) |
|---|---|---|---|
| {collector} | {customizado-ou-nativo} | {skill} | {findings: [...]} |
| {verifier} | {customizado-ou-nativo} | {skill} | {status, reason} |

Em {customizado-ou-nativo}, informe o nome do tipo personalizado ou de um tipo nativo.

## Procedimento

### Fase 0 — Recuperar contexto e tratar solicitações posteriores

1. Verifique se `_workspace/` existe:
   - Se não: iniciar execução completa.
   - Se sim e o pedido for uma correção localizada: reexecutar somente o trecho pertinente.
   - Se sim e houver novas entradas para execução completa: mover a pasta anterior para `_workspace_{ts}/` e iniciar novamente.
2. Para execução parcial, se houver um `runId` anterior em `_workspace/run_meta.json`:
   - Atualize somente a etapa relevante do script e retome com `resumeFromRunId`.
   - Chamadas `agent()` inalteradas podem recuperar resultados em cache.
3. Em uma nova execução, registre o novo `runId` em `_workspace/run_meta.json`.

### Fase 1 — Fixar a lista de trabalho (agente principal)

Antes de iniciar o workflow, examine rapidamente {arquivos/perspectivas/objetos de análise}
e organize a lista em array. Forneça-a ao script pelo objeto `args`.

### Fase 2 — Executar o workflow

Envie o script à ferramenta Workflow. Consulte os exemplos de
`workflow-recipes.md` da skill Harness.

- `meta`: `name` = '{domain}-run', `phases` = [{coleta}, {verificação}, {síntese}].
- Coleta: `pipeline(args.items, item => agent(..., {schema: COLLECT}))`.
- Verificação: para cada achado, executar revisão adversarial; aplicar `.filter(Boolean)`;
  encaminhar à etapa seguinte somente os confirmados.
- Síntese: devolver resultado estruturado em `return`.

Depois da chamada, aguarde a notificação de conclusão. Não afirme antecipadamente
que houve sucesso nem antecipe resultados ainda não recebidos.

### Fase 3 — Consolidar e entregar

1. Receba o retorno do workflow.
2. Registre claramente itens omitidos ou excluídos.
3. Salve o resultado final em `{output-path}/{filename}`.
4. Preserve os artefatos intermediários e `run_meta.json` em `_workspace/`.

## Tratamento de falhas

| Situação | Resposta |
|---|---|
| Falha em um `agent()`, que devolve `null` | Remova entradas vazias com `.filter(Boolean)`, conte-as por `log()` e informe omissões na entrega. |
| Falha do workflow inteiro | Consulte `journal`, identifique a etapa real da falha e retome apenas o trecho necessário com `resumeFromRunId`. |
| Falha irrecuperável (limite de uso, autenticação ou permissões) | Não tente novamente até que a causa seja resolvida; novas tentativas apenas consomem recursos. Inspecione `journal` e artefatos parciais, determine o avanço comprovado, registre pendências em `_workspace/` e informe o usuário. Se o horário de liberação do limite for conhecido, comunique-o. Após a correção da causa, poderá retomar com `resumeFromRunId`. O principal pode completar apenas fatos diretamente confirmados; não deve inventar o raciocínio de um agente interrompido, como a razão de descartar determinada fonte. |
| Resultado vazio | Não presuma que isso signifique êxito; consulte os retornos efetivos dos agentes em `journal`. |
| Informações divergentes | Conserve os registros e suas fontes, sem descartar uma versão arbitrariamente. |

## Cenários de teste

### Fluxo normal
1. O usuário apresenta {entrada}; a inspeção inicial identifica {M} itens.
2. O workflow coleta os {M} itens e confirma {K} após verificação.
3. Espera-se a criação de `{output-path}/{filename}` e o registro da execução em `run_meta.json`.

### Fluxo com falha
1. Dois itens da coleta retornam `null`.
2. O script filtra com `.filter(Boolean)` e informa “2 itens excluídos” via `log()`.
3. O relatório final registra “Não foi possível coletar {nomes de 2 itens}”.
```

---

## Modelo B — Colaboração entre agentes persistentes

```markdown
---
name: {domain}-orchestrator
description: "Coordena especialistas persistentes em {domínio}. Acione com {expressões iniciais}, bem como para correções, reexecuções parciais, atualizações, complementações e melhorias de resultados anteriores."
---

# Orquestrador — {Domínio}

## Modo de execução: agentes persistentes

Não crie objetos explícitos de equipe como na v1.
Agentes iniciados com nome na sessão integram automaticamente
um único grupo de colaboração. TeamCreate e TeamDelete pertencem à v1.

## Composição dos agentes

| Nome | `subagent_type` | Papel | Skill | Artefato |
|---|---|---|---|---|
| {teammate-1} | {customizado-ou-nativo} | {papel} | {skill} | {output-file} |
| {teammate-2} | {customizado-ou-nativo} | {papel} | {skill} | {output-file} |

Em {customizado-ou-nativo}, informe o tipo personalizado ou nativo.

## Procedimento

### Fase 0 — Recuperar o trabalho anterior

Assim como no modelo A, examine `_workspace/` para distinguir primeira execução,
reexecução parcial e nova execução completa. Para reexecuções parciais,
informe no prompt do agente onde encontrar os artefatos anteriores.

### Fase 1 — Preparação

1. Analise a entrada e determine {informações a identificar}.
2. Crie `_workspace/` e salve as entradas em `_workspace/00_input/`.

### Fase 2 — Iniciar agentes e registrar tarefas

1. Inicie os agentes em paralelo, na **mesma mensagem**.
   É obrigatório atribuir `name`, usado como destinatário em `SendMessage`:
   - `Agent(name: "{teammate-1}", subagent_type: "{type}", prompt: "{papel + tarefa + caminho de saída}")`
   - `Agent(name: "{teammate-2}", subagent_type: "{type}", prompt: "...")`
2. Crie a lista compartilhada:
   - `TaskCreate({title: "{tarefa-1}", assignee: "{teammate-1}"})`
   - `TaskCreate({title: "{tarefa-3}", depends_on: [tarefa-1]})`
   Atribua, em geral, 3 a 6 tarefas a cada integrante e registre as dependências em `depends_on`.

### Fase 3 — Coordenar a colaboração

- Receba notificações de conclusão ou espera e acompanhe o conjunto por `TaskList`.
- Peça uma revisão com `SendMessage({to: "{teammate-2}"}, "Leia o rascunho de {teammate-1} em _workspace/01_... e indique correções")`.
  Encaminhe o feedback a {teammate-1}; o contexto persistente permite instruções como
  “corrija somente a segunda seção do rascunho anterior”.
- Para discussão de achados, o líder transmite mensagens entre especialistas.
  Registre sempre os resultados relevantes em arquivo.

**Arquivos intermediários:**

| Integrante | Caminho |
|---|---|
| {teammate-1} | `_workspace/{phase}_{teammate-1}_{artifact}.md` |
| {teammate-2} | `_workspace/{phase}_{teammate-2}_{artifact}.md` |

### Fase 4 — Congelar os artefatos

Agentes persistentes podem receber novas mensagens depois de declarar a tarefa concluída.
Ao responder perguntas tardias de outro agente, podem modificar um arquivo já entregue.
Essas perguntas cruzadas ajudam a descobrir erros e não devem ser proibidas;
no entanto, as próximas fases precisam de uma versão estável dos dados.

1. Confirme via `TaskList` e notificações que todos os integrantes da fase concluíram.
2. Avise cada agente com `SendMessage`: “Os artefatos da fase {fase} estão congelados.
   Não edite os arquivos existentes. Para ajustes, crie uma nova versão,
   como `{phase}_{teammate}_{artifact}_v2.md`, e informe o líder.”
3. Registre hashes: `shasum _workspace/{phase}_* > _workspace/freeze_{phase}.sha`
   (também pode usar `md5sum`).
4. Forneça à próxima fase tanto os caminhos dos resultados como o caminho do
   arquivo de hashes.

Repita o procedimento em **toda transição** entre fases colaborativas.

### Fase 5 — Integrar

1. Verifique em `TaskList` que todas as atividades terminaram.
2. Leia os artefatos e execute {lógica de consolidação/verificação}.
3. Antes de criar a entrega final, recalcule hashes com comando como
   `shasum -c _workspace/freeze_{phase}.sha`.
   Se houver divergência ou versões novas, identifique o que mudou e decida
   se é necessário refazer etapas que consumiram a versão antiga.
4. Salve o resultado final em `{output-path}/{filename}`.

### Fase 6 — Encerrar

1. Preserve `_workspace/` para auditoria posterior.
2. Informe ao usuário os resultados e limitações.
   Não existe etapa explícita para desfazer a equipe.
   Agentes encerram naturalmente; se necessário, use `TaskStop`.

## Tratamento de falhas

| Situação | Resposta |
|---|---|
| Agente não responde ou é interrompido | Solicite status via `SendMessage` e reenvie a instrução. Se falhar, inicie outro agente do mesmo tipo com nome diferente e repasse o contexto. |
| Interrupção por limite de uso, autenticação ou permissões | Não insista nem recrie agentes até resolver a causa. Examine arquivos parciais, registre pendências em `_workspace/` e informe o usuário, inclusive horário de liberação do limite se conhecido. O líder pode acrescentar somente fatos confirmados; não invente o julgamento do agente interrompido. |
| Mais da metade dos agentes falhou | Informe o usuário e confirme se deseja continuar. |
| Prazo de execução ultrapassado | Continue com os resultados disponíveis e deixe explícitas as lacunas. |
| Dados conflitantes | Preserve ambas as versões com suas fontes. |
| Estado de tarefa desatualizado | Confira por `TaskList` e corrija com `TaskUpdate`. |

## Cenários de teste

### Fluxo normal
1. A entrada {entrada} inicia {N} agentes e registra {M} tarefas.
2. Após {K} ciclos de feedback, os arquivos são congelados na fase 4 e consolidados na fase 5.
3. Espera-se a criação de `{output-path}/{filename}`.

### Fluxo com falha
1. Durante a fase 3, {teammate-2} deixa de responder.
2. Peça status por `SendMessage` e tente novamente.
   Se persistir, inicie o agente substituto “{teammate-2}b” com os caminhos dos artefatos anteriores.
3. Registre no relatório: “Parte do trabalho de {teammate-2} precisou ser refeita”.
```

---

## Modelo C — Delegação a subagentes

```markdown
---
name: {domain}-orchestrator
description: "Delega tarefas independentes de {domínio} a subagentes. Use com {expressões iniciais}, além de pedidos de correção, reexecução parcial, atualização, complementação e melhoria de resultados anteriores."
---

## Modo de execução: subagentes

## Procedimento

### Fase 0 — Recuperar contexto

Verifique `_workspace/` para distinguir primeira execução, reexecução parcial
e uma nova execução completa.

### Fase 1 — Preparar

Analise as entradas e crie `_workspace/`.

### Fase 2 — Executar em paralelo

Na mesma mensagem, faça N chamadas à ferramenta `Agent`.
Por padrão, os agentes executam em segundo plano.

| Agente | `subagent_type` | Entrada | Saída |
|---|---|---|---|
| {agent-1} | {type} | {source} | `_workspace/{phase}_{agent}_{artifact}.md` |
| {agent-2} | {type} | {source} | `_workspace/{phase}_{agent}_{artifact}.md` |

Em {type}, informe o nome de um tipo personalizado ou nativo.

Aguarde as notificações de conclusão. O agente principal não deve repetir
pesquisas que já delegou.

### Fase 3 — Consolidar

1. Reúna retornos e artefatos, consolide-os e produza a entrega final.

### Fase 4 — Encerrar

Preserve `_workspace/` e apresente ao usuário um resumo dos resultados.

## Tratamento de falhas

- Se um agente falhar, tente mais uma vez. Se falhar novamente, registre a lacuna e continue.
- Se a causa for limite de uso, autenticação expirada ou permissão negada, não tente novamente. Inspecione resultados parciais, registre pendências em `_workspace/` e informe o usuário, incluindo horário de normalização se conhecido. Não invente julgamentos não concluídos.
- Se mais da metade dos agentes falhar, informe o usuário e confirme se deseja continuar.
```

---

## Combinação de modos entre fases (híbrido)

Escolha o modo por fase. Identifique cada uma com `**Modo de execução:**`.

```markdown
## Modo de execução: híbrido

| Fase | Modo | Por que foi escolhido |
|---|---|---|
| 1 — Reconhecimento | Agente principal ou subagente | É preciso definir a lista de itens a processar. |
| 2 — Coleta e validação em escala | Workflow | O conjunto é conhecido e admite distribuição e verificação adversarial. |
| 3 — Discussão e consolidação | Agentes persistentes | Divergências precisam ser negociadas pelos especialistas. |
| 4 — Verificação independente | Subagente | Um revisor especializado realiza validação isolada. |
```

**Ao mudar o modo:**
- **Workflow → agentes persistentes:** salve a saída estruturada em `_workspace/` e forneça os caminhos nos prompts dos especialistas.
- **Agentes persistentes → workflow:** congele os artefatos conforme a fase 4 do modelo B e passe a lista de caminhos em `args`.
- Documente o caminho de transferência em cada fronteira, evitando que a fase seguinte não encontre o resultado anterior.

---

## Princípios para todos os modelos

1. **Indique o modo de execução no início.** Em modo híbrido, use uma tabela por fase.
2. **Detalhe scripts em workflows.** Informe `phases`, schemas e parâmetros esperados em `args`.
3. **Detalhe agentes persistentes.** Relacione nomes, destinatários de `SendMessage`, criação de tarefas por `TaskCreate`, dependências e congelamento dos artefatos.
4. **Detalhe as chamadas pontuais.** Especifique `subagent_type`, `prompt`, caminhos e ausência de `name` quando não se deseja persistência.
5. **Use caminhos explícitos.** Centralize intermediários em `_workspace/` e nomeie arquivos `{phase}_{agent}_{artifact}.{ext}`.
6. **Declare dependências entre fases.** Em fronteiras entre modos, explique como os dados passam de um mecanismo ao outro.
7. **Prepare contingências plausíveis.** Não presuma que todos os agentes terão sucesso.
8. **Defina cenários de teste.** Inclua pelo menos um caso normal e um com falha.
9. **Não reutilize APIs v1 obsoletas.** `TeamCreate`, `TeamDelete`, `team_name`, broadcasts e flags experimentais não devem ser dependências operacionais.

## Expressões de continuação para a description

A skill deve voltar a ser escolhida após sua primeira execução. Inclua expressões como:

- Reexecutar, executar novamente, atualizar, corrigir, complementar.
- “Refaça apenas {parte} de {domínio}”.
- “Com base no resultado anterior” e “melhore o resultado”.
- Termos frequentes no domínio: por exemplo, se o harness for de lançamento de produtos, “lançamento”, “divulgação” e “tendências”.

Sem gatilhos para solicitações posteriores, o Claude pode não voltar a selecionar o harness quando o usuário pedir alterações nos resultados.
