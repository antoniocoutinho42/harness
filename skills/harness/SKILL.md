---
name: harness
description: "Projeta, cria, audita e amplia harnesses para projetos, definindo agentes especializados e as skills que eles utilizam. Acione quando o usuário pedir 'crie um harness', 'estruture um harness', 'desenhe um harness multiagente', 'planeje uma equipe de agentes', 'organize a orquestração dos agentes' ou quando precisar estruturar a automação de um domínio. Use também para 'audite o harness', 'revise a arquitetura', 'verifique a configuração', 'sincronize agentes e skills' e manutenção de harnesses existentes. Para aprender com execuções anteriores e incorporar feedback ao harness já existente, use harness:evolve."
---

# Harness v2 — Projeto de equipes de agentes e skills

Projete um harness adequado ao projeto. Defina os papéis dos agentes e as skills que orientam como cada agente deve trabalhar.

## Princípios essenciais

1. Crie as definições dos agentes em `<projeto>/.claude/agents/` e as skills em `<projeto>/.claude/skills/`. O agente define **quem executa**; a skill define **como executar**.
2. Escolha o modo de execução conforme o fluxo: se ordem e condições de repetição puderem ser expressas em código, orquestre por workflow; se especialistas precisarem iterar e trocar feedback, use agentes persistentes; se bastar um resultado pontual, delegue a subagentes. Consulte a fase 2.
3. Escolha os modelos segundo complexidade, duração, autonomia e latência. Use fable para trabalhos de altíssima complexidade que exijam planejamento autônomo prolongado; opus para arquitetura, programação, análises complexas e verificação cruzada; sonnet para atividades rotineiras como análise de logs, conversão de formatos e coleta simples. Consulte a fase 3.
4. Registre em `CLAUDE.md` **somente** as condições de acionamento e o histórico de alterações, para que a skill orquestradora seja descoberta em novas sessões.
5. Reutilize aprendizados de execução para aprimorar agentes, skills e `CLAUDE.md`. A revisão dos resultados e incorporação de feedback cabem à skill `harness:evolve`.
6. Escreva definições de agentes, skills, orquestradores e registros em `CLAUDE.md` no idioma da conversa com o usuário. Traduza também títulos e textos ilustrativos dos modelos de referência. Se o usuário escolher outro idioma, respeite-o; ao ampliar um harness existente sem orientação linguística explícita, mantenha o idioma dos arquivos existentes. O idioma deste documento não determina o idioma dos artefatos.

## Procedimento

### Fase 0 — Auditar a configuração atual

Antes de criar ou modificar qualquer coisa, avalie o que o projeto já possui.

1. Examine `<projeto>/.claude/agents/`, `<projeto>/.claude/skills/` e `<projeto>/CLAUDE.md`.
2. Defina o tipo de trabalho:
   - **Criar:** se os diretórios de agentes e skills estiverem ausentes ou vazios, execute todas as fases a partir da fase 1.
   - **Ampliar:** se for necessário acrescentar agentes ou skills, execute somente as fases relevantes segundo a tabela.
   - **Manter:** se a solicitação for auditar, corrigir ou sincronizar um harness existente, siga os procedimentos de manutenção da fase 7.

   | Alteração | Fase 1 | Fase 2 | Fase 3 | Fase 4 | Fase 5 | Fase 6 |
   |---|---|---|---|---|---|---|
   | Acrescentar agente | Usar a auditoria da fase 0 | Definir a equipe e fase de execução | Obrigatória | Se precisar de skill específica | Ajustar orquestrador | Obrigatória |
   | Acrescentar/alterar skill | Dispensável | Dispensável | Dispensável | Obrigatória | Se as conexões mudarem | Obrigatória |
   | Alterar arquitetura ou modo | Dispensável | Obrigatória | Somente agentes afetados | Somente skills afetadas | Obrigatória | Obrigatória |

3. Se um orquestrador contiver `TeamCreate`, `TeamDelete` ou `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` como dependências operacionais, ele possui artefatos da v1 incompatíveis com o runtime atual. Proponha a migração para a v2 conforme `references/execution-modes.md`, seção “Migração da v1 para a v2”.
4. Compare os agentes e skills existentes com o registro de `CLAUDE.md` e identifique divergências.
5. Apresente o diagnóstico e o plano de execução ao usuário e obtenha confirmação.

### Fase 1 — Analisar o domínio e as tarefas

1. Identifique a área de atuação e o objetivo do projeto a partir do pedido.
2. Separe os tipos de trabalho: criação, verificação, edição, análise e outros necessários.
3. Entenda o fluxo para escolher o modo de execução:
   - É possível enumerar antecipadamente as tarefas? Exemplo: processar N arquivos ou examinar M perspectivas.
   - O resultado exige ciclos de verificação e correção?
   - O trabalho melhora quando agentes trocam mensagens e discutem suas conclusões?
   - O mesmo especialista precisa manter contexto e participar de várias interações na sessão?
4. Identifique sobreposições e conflitos com agentes e skills encontrados na fase 0.
5. Examine código, stack tecnológica, modelos de dados e módulos principais do projeto.
6. Ajuste o nível de explicação à familiaridade técnica do usuário. Não use termos como *assertion* ou *JSON Schema* sem explicar quando o usuário não tiver experiência técnica.

### Fase 2 — Definir modo de execução e arquitetura de equipe

#### 2.1. Escolher o modo de execução

O Harness v2 utiliza três mecanismos multiagentes do Claude Code:

| Modo | Recursos | Casos adequados |
|---|---|---|
| **Orquestração por workflow** | Scripts `Workflow` com `agent()`, `pipeline()`, `parallel()` e `phase()` | Tarefas enumeráveis; regras de verificação e repetição expressas em código; saídas validadas por schema; dezenas de chamadas `Agent` |
| **Colaboração entre agentes persistentes** | `Agent(name:)`, `SendMessage`, `TaskCreate`, `TaskUpdate` | Especialistas nomeados que preservam contexto e precisam trocar feedback, negociar ou trabalhar juntos por várias interações |
| **Delegação a subagentes** | Uma chamada `Agent` por trabalho, normalmente em segundo plano e passível de paralelização | Trabalhos pontuais e independentes, sem necessidade de conversa entre agentes |

Siga esta ordem de decisão:

1. Se tarefas, critérios de verificação e condições de repetição puderem ser especificados antecipadamente em código, use workflow. Não delegue ao julgamento improvisado do modelo a ordem de um fluxo que pode ser determinístico.
2. Se não for possível codificar o fluxo e houver necessidade de discussão entre agentes ou preservação de contexto, use agentes persistentes.
3. Se nenhuma das condições anteriores se aplicar e bastar obter um resultado, use subagentes.
4. Se etapas distintas exigirem comportamentos diferentes, combine modos e identifique explicitamente o modo utilizado em cada fase do orquestrador.

O uso da ferramenta `Workflow` exige autorização explícita do usuário. Considere essa autorização concedida se ele tiver solicitado diretamente a execução do workflow ou acionado uma skill orquestradora que instrua expressamente chamar `Workflow`. Limite a escala inicial a poucos agentes e amplie somente se o usuário solicitar uma investigação abrangente ou exaustiva.

> Veja limites de concorrência, comparação de modos, schemas, orçamentos de tokens e condições de retomada em `references/execution-modes.md`.

#### 2.2. Selecionar o padrão de equipe

1. Divida o trabalho por especialidade.
2. Selecione um dos seis padrões abaixo, conforme o tipo de tarefa. Consulte `references/team-patterns.md` para os critérios detalhados.
   - **Pipeline:** uma etapa utiliza o resultado da anterior; adequado a `pipeline()`.
   - **Fan-out/Fan-in:** tarefas independentes são executadas em paralelo e consolidadas depois; combine `pipeline()` com `parallel()` somente quando necessário.
   - **Pool de Especialistas:** apenas os especialistas relevantes à entrada são acionados; adequado a subagentes ou agentes persistentes.
   - **Produção–Revisão:** um agente produz e outro avalia; use verificação adversarial em workflow ou uma dupla de agentes persistentes.
   - **Supervisor:** agente central acompanha o avanço e redistribui tarefas; adequado a agentes persistentes e lista compartilhada.
   - **Delegação Hierárquica:** agentes de nível superior delegam para níveis inferiores; limite a hierarquia a dois níveis e o aninhamento de workflows a um nível.
3. Para resultados em que precisão e completude sejam críticas, combine mecanismos de qualidade:
   - **Verificação adversarial:** para cada item encontrado, convoque N verificadores adversariais; aprove como `confirmed` somente se a maioria concluir que há evidência suficiente. `refuted` e `uncertain` não contam como confirmação.
   - **Painel de avaliadores:** produza N versões independentes, avalie-as em paralelo e elabore a resposta final a partir da melhor.
   - **Busca até esgotamento (*loop-until-dry*):** continue pesquisando até ocorrerem K iterações consecutivas sem novos itens.
   - **Varredura por múltiplos critérios:** examine em paralelo perspectivas diferentes, como estrutura, conteúdo, entidades e datas.
   - **Revisor de omissões:** reserve a revisão final a um agente cuja tarefa seja encontrar exclusivamente o que ficou faltando.

#### 2.3. Quando separar agentes

Divida papéis considerando conhecimento especializado, possibilidade de execução paralela, necessidade de preservar contexto e potencial de reutilização. Consulte a seção “Critérios para separar agentes” de `references/team-patterns.md`.

### Fase 3 — Criar as definições dos agentes

Defina os especialistas reutilizáveis em `<projeto>/.claude/agents/{name}.md`. Não deixe um papel recorrente definido apenas no campo `prompt` de uma chamada `Agent`.

- A definição em arquivo permite reutilização em sessões futuras. Acione-a por `subagent_type: "{name}"` na ferramenta `Agent` ou `agentType: "{name}"` em `Workflow`.
- Defina antecipadamente os contratos de troca de informações para estabilizar a colaboração.
- Mantenha clara a separação entre papel do agente (**quem faz**) e skill (**como faz**).

Crie definições personalizadas apenas para especialistas recorrentes. Se um trabalho pontual puder usar um tipo nativo (`general-purpose`, `Explore`, `Plan`), não crie um arquivo desnecessário.

#### Conferir sobreposição com agentes existentes

Antes de criar um agente, verifique se algum arquivo em `<projeto>/.claude/agents/` já define papel semelhante. Faça essa verificação mesmo ao adicionar um agente sem repetir a fase 1. Execuções sucessivas podem acumular agentes com nomes diferentes e funções equivalentes. Reutilize ou amplie o agente existente, salvo se a especialização de domínio justificar a separação. Consulte “Reutilizar agentes”, em `references/team-patterns.md`.

#### Escolher o modelo por agente

Considere complexidade, duração, autonomia e latência. Defina o modelo no frontmatter YAML (`model:`) ou na chamada (`model`, `opts.model`). Registre a justificativa na definição ou no orquestrador.

| Modelo | Quando utilizar | Exemplos de tarefas |
|---|---|---|
| **fable** | Atividades de complexidade excepcional, em que o agente planeja e conduz autonomamente várias etapas por períodos prolongados | Orquestração de agentes, planejamento de longo prazo, integração de grande volume de informações e estruturação de problemas abertos |
| **opus** | Trabalhos especializados com escopo definido e forte exigência de raciocínio | Arquitetura, programação, análises complexas, verificação cruzada, crítica metodológica e criação |
| **sonnet** | Trabalhos rotineiros com procedimento definido, sensíveis a velocidade e custo | Logs, conversão de formatos, inspeção estática de arquivos, scripts de deploy, coleta simples, redação e resumos comuns |

- Prefira fable quando a estratégia precisar ser redefinida autonomamente à medida que fases anteriores produzam resultados. Prefira opus para examinar em profundidade um problema bem delimitado; por exemplo, criticar um artigo científico específico. Para sintetizar dezenas de artigos e construir uma estratégia e um relatório de forma autônoma, considere fable.
- Use sonnet como padrão para tarefas rotineiras ambíguas.
- Não imponha fable ou opus a todos os agentes apenas pela importância hierárquica deles; escolha pela natureza do trabalho.
- Mesmo que um agente supervisor use fable, avalie separadamente os modelos dos especialistas subordinados.

> Critérios completos em `references/model-selection-guide.md`.

#### Conteúdo obrigatório das definições

Inclua `name` e `description` no frontmatter YAML. Se necessário, restrinja ferramentas em `tools` ou selecione um modelo em `model`. Para agentes que apenas revisam ou analisam, retire Edit e Write de `tools` para evitar alterações de arquivos.

Já os agentes que **modificam** artefatos precisam de Edit **e** Write. Sem Edit, pequenas correções exigem reescrever arquivos inteiros, podendo levar o agente a abandonar a alteração ou improvisar um desvio. Como ferramentas declaradas em `tools` podem não ser carregadas em determinados runtimes, peça aos agentes persistentes que informem em seu primeiro relatório quais ferramentas estão efetivamente disponíveis.

No corpo da definição, descreva papel central, princípios de trabalho, contratos de entrada e saída, tratamento de erros e forma de colaboração. Para agentes persistentes, acrescente `## Regras de comunicação`, indicando destinatários de `SendMessage` e o uso da lista compartilhada de tarefas.

> Estrutura completa, exemplos e cuidados ao restringir `tools`: `references/team-patterns.md`, seção “Estrutura da definição de agentes”.

#### Incluir um agente de QA

- Use um tipo com acesso a todas as ferramentas necessárias à validação. `Explore` é somente leitura e não executa scripts de teste.
- Não limite a verificação à presença de arquivos: compare, por exemplo, a estrutura da resposta de APIs com o formato esperado pelos hooks do frontend.
- Valide cada módulo assim que estiver pronto; não concentre todos os testes no final.
- Consulte `references/qa-agent-guide.md`.

### Fase 4 — Criar as skills

Escreva as instruções de cada agente em `<projeto>/.claude/skills/{name}/SKILL.md` conforme `references/skill-writing-guide.md`.

#### 4.0. Verificar skills já existentes

Antes de criar uma skill, procure capacidades equivalentes em `<projeto>/.claude/skills/`. Se houver sobreposição, conecte a skill existente ao novo agente ou amplie-a. Consulte os critérios e as exceções para especialização em “Projetar skills reutilizáveis”, em `references/skill-writing-guide.md`.

#### 4.1. Estrutura de diretórios

```text
skill-name/
├── SKILL.md              # obrigatório
│   ├── Frontmatter YAML  # name e description obrigatórios
│   └── Corpo Markdown
├── scripts/              # opcional: código para tarefas repetitivas ou determinísticas
├── references/           # opcional: documentos carregados somente quando necessários
└── assets/               # opcional: modelos, imagens e outros recursos do resultado
```

#### 4.2. Definir condições de acionamento na description

O Claude decide qual skill utilizar a partir de `name` e `description`. Explique claramente na `description` o que a skill faz, quando ela deve ser usada e quais situações semelhantes pertencem a outra ferramenta ou skill.

**Ruim:** `"Skill para processar documentos PDF."`

**Bom:** `"Lê arquivos PDF, extrai texto e tabelas, mescla, separa, gira páginas, aplica marca d'água, criptografa e realiza OCR. Use quando o usuário mencionar um arquivo .pdf ou solicitar um resultado em PDF."`

#### 4.3. Princípios para redigir skills

| Princípio | Como aplicar |
|---|---|
| **Explique o motivo** | Em vez de listar ordens `ALWAYS` e `NEVER`, justifique as regras para permitir decisões corretas em exceções. |
| **Seja conciso** | Mantenha `SKILL.md` com menos de 500 linhas. Remova conteúdo que não ajude a decidir ou transfira detalhes para `references/`. |
| **Generalize pelos princípios** | Não crie regras que só funcionem para o exemplo apresentado; descreva critérios aplicáveis a várias entradas. |
| **Inclua código repetitivo previamente** | Se vários agentes recriarem o mesmo script nos testes, centralize-o em `scripts/`. |
| **Use instruções inequívocas** | Adote um estilo verbal consistente, como “Verifique”, “Defina” e “Registre”, deixando clara a ação esperada. |

#### 4.4. Carregar informações progressivamente

Carregue cada tipo de recurso somente quando necessário:

| Recurso | Quando carregar | Tamanho recomendado |
|---|---|---|
| **Metadados** (`name`, `description`) | Sempre | Cerca de 100 palavras |
| **Corpo de `SKILL.md`** | Quando a skill for acionada | Menos de 500 linhas |
| **`references/`** | Somente quando as referências forem necessárias | Sem limite fixo |
| **`scripts/`** | Quando precisar de trabalho repetitivo ou determinístico | Sem limite; pode executar sem ler todo o código |

- Ao se aproximar de 500 linhas, transfira detalhes para `references/` e explique no corpo quando consultar cada arquivo.
- Adicione um sumário a referências com mais de 300 linhas.
- Separe orientações específicas de domínio ou framework em arquivos distintos, carregando somente as referências pertinentes.

#### 4.5. Relacionar agentes e skills

- Um agente pode usar várias skills.
- Vários agentes podem compartilhar a mesma skill.
- A skill estabelece o método; a definição do agente, o responsável.

### Fase 5 — Integrar os agentes e ordenar a execução

O orquestrador também é uma skill. Ele conecta agentes e métodos em um fluxo, especificando quem executa cada etapa, quando e em que ordem. Consulte os modelos por modo em `references/orchestrator-template.md` e os exemplos de scripts em `references/workflow-recipes.md`.

Ao ampliar um harness existente, modifique seu orquestrador, em vez de criar outro. Ao adicionar agentes, atualize a composição da equipe, a distribuição de tarefas, a transferência de dados e os gatilhos na `description`.

#### 5.0. Orquestradores por modo

**A. Orquestração por workflow**

Defina um script `Workflow` na skill orquestradora, utilizando `meta`, `phase()`, `pipeline()`, `parallel()` e `agent()`. Defina agentes personalizados por `agentType` e receba resultados validados por `schema`.

```text
[Skill orquestradora] → Workflow(script)
    ├── phase('coleta'): pipeline(items, ...)   ← processa uma lista conhecida
    ├── phase('verificação'): verificação adversarial ou painel
    └── return resultado estruturado → agente principal redige a síntese
```

**B. Colaboração entre agentes persistentes**

Inicie agentes identificados por nome e coordene-os com tarefas compartilhadas e `SendMessage`. `TeamCreate` e `TeamDelete` foram removidos. Não crie um objeto de equipe explícito: os agentes nomeados participam do grupo de colaboração implícito da sessão e podem receber novas mensagens preservando o contexto.

```text
[Agente principal (líder)]
    ├── Agent(name: "researcher", ...) / Agent(name: "critic", ...)  ← iniciar em paralelo
    ├── TaskCreate(tarefa + dependências)
    ├── SendMessage({to: "critic"}, "Revise o rascunho do researcher")
    └── Reunir e sintetizar os resultados
```

**C. Delegação a subagentes**

Faça N chamadas paralelas à ferramenta `Agent` na mesma mensagem e reúna os resultados conforme os agentes concluírem. Por padrão, as chamadas ocorrem em segundo plano.

Modos podem ser combinados entre fases. Por exemplo, colete informações por workflow, consolide-as por discussão entre agentes persistentes e submeta o resultado à verificação adversarial por outro workflow. Identifique cada fase com `**Modo de execução:**`.

#### 5.1. Transferir dados entre agentes e etapas

| Método | Implementação | Modos adequados | Quando utilizar |
|---|---|---|---|
| **Retorno estruturado** | `Workflow`: `agent(prompt, {schema})` retorna JSON validado | Workflow | Quando a próxima etapa precisa processar os dados por código |
| **Retorno comum** | Mensagem de retorno de `Agent` | Subagentes | Quando o agente principal consolida diretamente os resultados |
| **Mensagens** | Comunicação direta por `SendMessage` | Agentes persistentes | Coordenação e feedback em tempo real |
| **Lista compartilhada de tarefas** | `TaskCreate` e `TaskUpdate` | Agentes persistentes | Acompanhar progresso, dependências e reatribuição |
| **Arquivos** | Gravação/leitura em caminhos definidos | Todos | Grandes volumes de dados ou exigência de trilha auditável |

Ao transferir por arquivos:

- Salve artefatos intermediários em `_workspace/`, dentro do diretório de trabalho.
- Nomeie-os conforme `{phase}_{agent}_{artifact}.{ext}`. Exemplo: `01_analyst_requirements.md`.
- Entregue somente artefatos finais no caminho solicitado pelo usuário. Preserve `_workspace/` para verificações posteriores.

#### 5.2. Tratar erros

Inclua uma política de falhas no orquestrador:

- Faça uma tentativa adicional após uma falha recuperável. Se persistir, prossiga sem o resultado e registre explicitamente a lacuna no relatório. Não apague dados conflitantes: mantenha as versões e suas fontes.
- **Não repita falhas irrecuperáveis**, como limite de uso esgotado, autenticação expirada ou permissão negada. Verifique os resultados parciais para determinar o avanço efetivo, registre em arquivo as pendências e informe o usuário. Se houver um horário conhecido para restabelecimento do limite, comunique-o.
- Se o orquestrador precisar completar uma lacuna deixada por agente interrompido, use apenas fatos que tenha verificado diretamente. **Não invente o julgamento do agente**, como os motivos para excluir uma fonte; isso destruiria a rastreabilidade da execução.
- Em workflows, `agent()` com falha ou ignorado pode devolver `null`, e `parallel()` não necessariamente lança uma exceção. Filtre as saídas com `.filter(Boolean)` e registre a quantidade de itens ausentes com `log()`.
- Se um agente persistente não responder, solicite status via `SendMessage` e reenvie a instrução. Se continuar falhando, inicie um novo agente do mesmo tipo com outro nome e forneça o contexto necessário.

> Consulte as orientações específicas em “Tratamento de erros”, nos modelos de `references/orchestrator-template.md`.

#### 5.3. Dimensionar o trabalho

| Escala | Agentes persistentes | Chamadas `Agent` em workflow |
|---|---|---|
| Pequena: menos de 10 tarefas | 2–3 | 2–5 |
| Média: 10–20 tarefas | 3–5 | Aproximadamente 10; excedentes aguardam quando o limite de concorrência for atingido |
| Grande: mais de 20 tarefas | Supervisor + 3–5 executores | Dezenas a centenas; máximo global de 1.000 |

- Equipes maiores aumentam o custo de coordenação. Prefira uma configuração inicial enxuta.
- Se o usuário definir orçamento como `+500k`, consulte `budget.remaining()` no script `Workflow` para ajustar a escala.

#### 5.4. Registrar a referência mínima no CLAUDE.md

Ao concluir a configuração, informe em `CLAUDE.md` que existe um harness e quando acioná-lo. Como esse arquivo é lido em novas sessões, **não duplique instruções detalhadas**.

````markdown
## Harness: {nome-do-domínio}

**Objetivo:** {objetivo do harness em uma frase}

**Condição de acionamento:** Para tarefas relacionadas a {domínio}, use a skill `{orchestrator-skill-name}`. Para perguntas simples, uma resposta direta é suficiente.

**Histórico de alterações:**
| Data | Alteração | Componente | Motivo |
|---|---|---|---|
| {YYYY-MM-DD} | Criação inicial com Harness v2 | Todos | - |
````

Não copie listas de agentes, skills, estruturas de diretórios ou regras de execução para `CLAUDE.md`. A fonte dessas informações deve ser o orquestrador e os arquivos em `.claude/agents/` e `.claude/skills/`.

#### 5.5. Tratar solicitações posteriores

O orquestrador deve funcionar tanto na primeira execução quanto ao revisar resultados anteriores.

1. Inclua na `description` expressões como “execute novamente”, “atualize”, “corrija”, “complemente”, “refaça somente {parte da tarefa} de {domínio}”, “com base no resultado anterior” e “melhore o resultado”.
2. Na fase 0 do orquestrador, determine o estado da execução:
   - Se `_workspace/` existir e o usuário pedir apenas correções, reexecute somente a fase ou o agente afetado.
   - Se `_workspace/` existir e houver novas entradas para uma execução completa, mova a pasta anterior para um diretório com timestamp antes de iniciar novamente.
   - Se `_workspace/` não existir, comece do início.
   - Se houver um `runId` anterior de workflow, avalie `resumeFromRunId`; chamadas `agent()` sem alterações podem reaproveitar o cache.
3. Documente como reexecutar cada agente. Se existir um resultado anterior, leia-o antes de alterar somente os pontos afetados pelo feedback.

### Fase 6 — Verificar e testar

Valide o harness produzido conforme `references/skill-testing-guide.md`.

#### 6.1. Estrutura de arquivos e referências

- Confirme que os arquivos dos agentes estão nos diretórios corretos.
- Verifique `name` e `description` no frontmatter YAML de todas as skills.
- Confirme que os nomes dos agentes são idênticos em todas as referências.
- Certifique-se de não criar arquivos de comando em `.claude/commands/`.
- Verifique que nenhum artefato operacional dependa de `TeamCreate`, `TeamDelete`, `team_name` ou flags experimentais da v1.

#### 6.2. Validações por modo de execução

- **Workflow:** `meta` contém apenas literais? `Date.now()` e `Math.random()` foram evitados? `parallel()` é usado somente quando é preciso esperar todos os resultados? O script utiliza `.filter(Boolean)`? Os nomes das `phase()` coincidem com `meta.phases`?
- **Agentes persistentes:** verifique remetentes e destinatários de `SendMessage`, dependências entre tarefas e tamanho da equipe.
- **Subagentes:** confira os contratos de entrada e saída, o agrupamento das chamadas paralelas na mesma mensagem e a coleta de todos os resultados.
- **Modo híbrido:** identifique o modo em cada fase e confirme que a transferência de dados entre fases não possui lacunas.

#### 6.3. Testes de execução de skills

1. Para cada skill, formule duas ou três solicitações concretas e plausíveis de um usuário real.
2. Execute em paralelo uma versão **com a skill** (*with-skill*) e outra **sem a skill** (*baseline*). Compare o ganho proporcionado. Se for preciso repetir os testes, implemente o próprio A/B como workflow.
3. Combine avaliações qualitativas do usuário com verificações quantitativas baseadas em *assertions*. Se não for possível definir critérios objetivos, priorize o julgamento humano.
4. Quando detectar um problema, corrija o princípio geral, não apenas o exemplo que falhou. Repita o teste até que alterações adicionais tragam ganhos marginais.
5. Se vários agentes recriarem o mesmo código, mova-o para `scripts/`.

#### 6.4. Verificar condições de acionamento

1. Crie dez solicitações que **devem** acionar a skill, variando linguagem e grau de explicitação.
2. Crie outras dez solicitações semelhantes que **não devem** acioná-la, por pertencerem a outra ferramenta ou skill.

Casos claramente sem relação não testam bem o limite. Para uma skill de geração de imagens, por exemplo, “extraia o gráfico deste Excel como PNG” é um caso limítrofe melhor do que “escreva uma função de Fibonacci”: embora o resultado seja uma imagem, uma ferramenta de planilhas é mais apropriada. Confira também conflitos com outras skills.

#### 6.5. Simular o fluxo

- Verifique se a sequência das etapas é coerente.
- Confira a continuidade do fluxo de dados entre fases.
- Confirme que cada agente recebe saídas compatíveis das fases anteriores.
- Valide a viabilidade dos procedimentos de contingência.

#### 6.6. Documentar cenários de teste

Acrescente `## Cenários de teste` à skill orquestradora, com pelo menos um fluxo normal e um fluxo de falha.

### Fase 7 — Operar, manter e aprimorar

Um harness não termina quando é criado.

Para revisar resultados de execução e incorporar feedback, acione `harness:evolve`. Isso inclui pedidos como “revise o harness”, “evolua o harness” e “incorpore este feedback”. A skill de evolução identifica diferenças entre a configuração inicial e a atual, generaliza as lições e atualiza agentes, skills, orquestrador e o histórico em `CLAUDE.md`.

A skill `harness` continua responsável pela manutenção estrutural:

1. **Auditar o estado:** compare `.claude/agents/`, `.claude/skills/` e o orquestrador; relacione divergências e informe o usuário.
2. **Alterar gradualmente:** implemente uma mudança por vez e valide-a imediatamente.
3. **Registrar o histórico:** inclua data, alteração, componente e justificativa em `CLAUDE.md`.
4. **Validar mudanças:** verifique a estrutura; teste gatilhos se tiverem sido alterados; se o escopo for amplo, faça também testes de execução e simulações. Confirme por fim a coerência entre `CLAUDE.md` e os arquivos reais.

Sugira `harness:evolve` proativamente quando:
- Surgirem duas ou mais solicitações semelhantes de correção.
- Os agentes falharem repetidamente pela mesma causa.
- O usuário executar manualmente e repetidas vezes atividades que deveriam passar pelo orquestrador.

## Checklist de entrega

- [ ] Todas as definições personalizadas reutilizáveis estão em `<projeto>/.claude/agents/`. Tipos nativos usados em tarefas pontuais não foram redefinidos desnecessariamente.
- [ ] As skills e referências estão em `<projeto>/.claude/skills/`.
- [ ] Existe uma skill orquestradora que define transferência de dados, tratamento de falhas e cenários de teste.
- [ ] O modo de execução (workflow, agentes persistentes ou subagentes) está documentado; se for híbrido, cada fase indica seu modo.
- [ ] Os modelos foram escolhidos segundo complexidade, duração, autonomia e latência, com justificativas registradas. Nenhum modelo caro foi imposto indiscriminadamente.
- [ ] Nenhum artefato operacional preserva dependências da v1, como `TeamCreate`, `TeamDelete` ou flags experimentais.
- [ ] Workflows usam `.filter(Boolean)`, `meta` apenas com literais e `parallel()` somente quando necessário esperar todos os resultados.
- [ ] Não foram criados arquivos em `.claude/commands/`.
- [ ] Foram verificadas sobreposições antes de criar agentes e skills; papéis redundantes foram reutilizados ou ampliados, exceto quando especialização de domínio justificar a separação.
- [ ] Agentes, skills, orquestrador e `CLAUDE.md` estão no idioma da conversa, salvo orientação explícita do usuário ou idioma predominante do harness existente.
- [ ] Cada `description` especifica objetivo, condições de acionamento e termos de solicitação posterior.
- [ ] `SKILL.md` possui menos de 500 linhas, com detalhes transferidos para `references/`.
- [ ] Cada skill foi testada com duas ou três solicitações realistas.
- [ ] Foram executados testes positivos e casos limítrofes negativos de acionamento.
- [ ] `CLAUDE.md` contém apenas regras de acionamento e histórico de alterações.
- [ ] A fase 0 do orquestrador distingue primeira execução, continuação, nova execução e reexecução parcial; workflows também oferecem retomada.

## Referências

- **Modos de execução:** `references/execution-modes.md`.
- **Escolha de modelos:** `references/model-selection-guide.md`.
- **Padrões de equipes e definições de agentes:** `references/team-patterns.md`.
- **Exemplos de equipes:** `references/team-examples.md`.
- **Scripts de workflow e armadilhas:** `references/workflow-recipes.md`.
- **Modelos de orquestradores:** `references/orchestrator-template.md`.
- **Redação de skills:** `references/skill-writing-guide.md`.
- **Testes e avaliação:** `references/skill-testing-guide.md`.
- **Agentes de QA:** `references/qa-agent-guide.md`.
