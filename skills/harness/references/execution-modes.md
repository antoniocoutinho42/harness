# Modos de execução — os três mecanismos do Harness v2

Este documento complementa a fase 2.1 de `SKILL.md` e apresenta as capacidades, limitações e critérios de escolha dos três modos de execução.

---

## Sumário

1. Modo A — Orquestração por workflow
2. Modo B — Colaboração entre agentes persistentes
3. Modo C — Delegação a subagentes
4. Árvore de decisão
5. Migração da v1 para a v2

---

## 1. Modo A — Orquestração por workflow

Execute scripts de coordenação com a ferramenta `Workflow`. O **código**, e não decisões improvisadas do modelo, determina o fluxo de controle, como fan-out, repetições e ramificações. Isso facilita reproduzir a estrutura de execução com as mesmas entradas e dimensionar o trabalho com muitos agentes.

```text
[Agente principal] → Workflow(script)
    ├── phase('pesquisa'): pipeline(items, item => agent(...))
    ├── phase('verificacao'): submeter cada achado à avaliação adversarial em paralelo
    └── return { confirmed }  ← resultado final estruturado
```

**Recursos principais:**
- `agent(prompt, opts)`: executa subagentes. `opts.schema` valida a resposta segundo um JSON Schema e devolve um objeto estruturado sem exigir parsing adicional; em caso de saída inválida, pode tentar novamente. `opts.agentType` permite usar definições personalizadas de `.claude/agents/`. `opts.effort` configura esforço de raciocínio; `opts.isolation: 'worktree'` isola modificações.
- `pipeline(items, stage1, stage2, ...)`: processa cada item por várias etapas independentes, sem barreira global entre etapas. **É o padrão para fluxos com várias etapas por item.**
- `parallel(thunks)`: barreira que espera todas as tarefas. Use apenas quando a próxima etapa precisar do conjunto **completo**, como remoção global de duplicatas ou parada definida pela contagem total.
- `phase(title)` / `log(msg)`: agrupam etapas e relatam o avanço.
- `budget`: respeita orçamento de tokens especificado pelo usuário (por exemplo, `+500k`), consultando `budget.total` e `budget.remaining()` para ajustar a escala.
- `workflow(nameOrRef, args)`: aninha outro workflow até um nível, permitindo delegação hierárquica.

**Características:**
- Estrutura de execução determinística para a mesma entrada.
- Transferência de dados entre etapas por saídas validadas por `schema`.
- `resumeFromRunId` permite retomar execuções interrompidas; chamadas `agent()` inalteradas reutilizam resultados em cache, reduzindo o custo de reexecução parcial.
- Execução em segundo plano, com notificação ao concluir.

**Limitações e cuidados:**
- **Exige solicitação prévia do usuário.** Use quando o usuário pedir diretamente coordenação por workflow ou acionar uma skill que instrua a chamada de `Workflow`. A skill orquestradora gerada pelo Harness atende à segunda condição. Mesmo assim, mantenha poucos agentes por padrão e use grande escala somente sob solicitação explícita.
- Scripts aceitam JavaScript puro, não TypeScript. Evite `Date.now()`, `Math.random()` e `new Date()` sem argumentos, pois prejudicam a retomada; forneça timestamps via `args`.
- O bloco `meta` aceita somente literais; variáveis, chamadas de função e spread não são permitidos.
- Existem limites de concorrência por sessão e de chamadas totais de agentes por workflow; tarefas excedentes aguardam em fila.
- Chamadas `agent()` que falham ou são ignoradas retornam `null`. `parallel()` não necessariamente lança exceção; aplique `.filter(Boolean)` aos resultados.
- A barreira `parallel()` sincroniza conclusões, mas não impede alterações posteriores. Em modo híbrido, um agente persistente acordado por `SendMessage` pode modificar artefatos já entregues. **Congele os arquivos na transição de fases** conforme a etapa 4 do modelo B em `orchestrator-template.md`.

**Indicado para:** listas conhecidas de N arquivos ou M perspectivas, busca até esgotamento, verificação adversarial, migrações e auditorias em grande escala e relatórios estruturados.

**Não indicado para:** descobrir o escopo progressivamente por diálogo, sem lista inicial possível, ou interações frequentes com o usuário durante a execução.

> Veja modelos e armadilhas em `workflow-recipes.md`.

## 2. Modo B — Colaboração entre agentes persistentes

Inicie agentes identificados por nome, coordenados por lista compartilhada de tarefas e `SendMessage`. Esse modo substitui as antigas equipes explícitas da v1.

**Diferença fundamental para a v1:** `TeamCreate` e `TeamDelete` foram removidos. Não é necessário criar um objeto de equipe; agentes nomeados iniciados na sessão participam de um grupo colaborativo implícito.

```text
[Agente principal (líder)]
    ├── Agent(name: "researcher", subagent_type: "...", prompt: ...)  ← paralelo
    ├── Agent(name: "critic", ...)
    ├── TaskCreate(tarefa + dependências) → lista compartilhada
    ├── SendMessage({to: "researcher"}, ...) ← nova instrução com contexto preservado
    └── Receber conclusão → consolidar resultados → entregar
```

**Recursos principais:**
- `Agent(name: ..., subagent_type: ..., model: ..., prompt: ...)`: com `name`, o agente se torna persistente e pode ser chamado novamente via `SendMessage`. Executa em segundo plano por padrão e notifica a conclusão. Para aguardar diretamente, use `run_in_background: false`.
- `SendMessage({to: name})`: permite continuar trabalhos com o **contexto da conversa anterior**, inclusive ciclos de revisão.
- `TaskCreate`, `TaskUpdate`, `TaskList` e `TaskGet`: compartilham dependências, estados e atribuições de tarefas.
- `TaskStop`: encerra trabalhos em segundo plano que estejam consumindo recursos desnecessariamente.

**Características:**
- Os agentes mantêm contexto, permitindo pedidos como “corrija apenas a segunda seção do rascunho que você produziu antes”.
- Especialistas podem compartilhar descobertas, discutir conflitos e alterar rapidamente o plano.
- Um mesmo especialista pode atuar repetidas vezes durante a sessão.

**Limitações:**
- O agente principal concentra a coordenação; com muitos agentes, seu trabalho cresce. Em geral, prefira 3–5 especialistas.
- Hierarquias muito profundas aumentam latência e risco de perda de contexto. Limite a delegação a dois níveis.
- Costuma consumir mais tokens que uma única chamada pontual.
- Agentes persistentes podem receber mensagens depois de comunicar conclusão e **alterar artefatos já entregues**. Antes de outra fase ler os arquivos, congele-os; veja a etapa 4 do modelo B em `orchestrator-template.md`.

**Indicado para:** ciclos de Produção–Revisão, negociação de divergências de dados, redistribuição dinâmica de tarefas por supervisor e trabalhos que exigem um especialista ao longo da sessão.

**Não indicado para:** consultas pontuais com resultado único ou processamento em massa de uma lista conhecida.

## 3. Modo C — Delegação a subagentes

Faça chamadas pontuais com `Agent`; o resultado volta ao agente principal, mas os subagentes não mantêm comunicação entre si.

```text
[Agente principal] → Agent(subA) ─┐
                  → Agent(subB) ─┼→ paralelo, normalmente em segundo plano
                  → Agent(subC) ─┘  → conclusão → consolidação
```

**Características:**
- Menor custo de coordenação e execução rápida; o agente principal recebe resultados resumidos.
- Para paralelismo real, agrupe chamadas independentes **na mesma mensagem**.
- A execução acontece em segundo plano por padrão; para receber o resultado diretamente, use `run_in_background: false`.
- Após delegar uma pesquisa, o agente principal não deve refazer a mesma busca enquanto aguarda os resultados.

**Limitações:**
- Os agentes não conversam entre si e não preservam contexto após a chamada. Novas tarefas pontuais devem ser iniciadas sem `name`.
- Toda a coordenação permanece com o agente principal.

**Indicado para:** pesquisa e coleta pontuais, seleção de especialistas para tarefas independentes e verificações isoladas.

**Não indicado para:** ciclos de revisão com diálogo ou grandes processamentos cujo fluxo pode ser definido em código.

## 4. Árvore de decisão

```text
É possível expressar em código, antecipadamente, tarefas, validações e repetições?
├── Sim → Modo A: workflow
│         O código, não o modelo, governa o fluxo.
│         Respeite a autorização do usuário e evite excesso de agentes.
│
└── Não → É necessário trocar feedback e manter contexto para garantir qualidade?
           ├── Sim → Modo B: agentes persistentes
           └── Não → Há mais de um agente?
                      ├── Sim → Modo C: subagentes em paralelo
                      └── Não → Modo C: chamada única
```

**Combinações de modos:** se as respostas forem diferentes entre fases, use um fluxo híbrido.

| Combinação | Sequência | Exemplo |
|---|---|---|
| Coleta por workflow → consolidação persistente | A → B | Coletar grandes volumes em paralelo e discutir divergências antes da síntese |
| Produção persistente → verificação por workflow | B → A | Equipe cria a primeira versão e validadores adversariais examinam cada achado |
| Reconhecimento pontual → execução por workflow | C → A | Agente identifica previamente a lista de itens e a passa em `args` para processamento em escala |

**Reconhecimento antes da orquestração:** identifique previamente arquivos, objetos de análise e perspectivas para fixar a lista de trabalho. Isso simplifica scripts e evita chamadas desnecessárias.

## 5. Migração da v1 para a v2

Se encontrar artefatos de agentes e orquestração da v1, proponha a adaptação:

| Elemento v1 | Situação | Substituição v2 |
|---|---|---|
| `TeamCreate(team_name, members)` | **Removido** | Iniciar `Agent(name: ...)` em paralelo na mesma mensagem; agentes nomeados integram o grupo implícito da sessão |
| `TeamDelete` e etapa “encerrar equipe” | **Removidos** | Eliminar a etapa; agentes terminam naturalmente ou podem ser interrompidos por `TaskStop` |
| Restrição “uma equipe ativa por sessão” | **Formato alterado** | Como não há objeto explícito de equipe, a antiga restrição não se aplica |
| Broadcast `SendMessage({to: "all"})` | Alterado | Enviar `SendMessage` individual a cada destinatário necessário |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | **Desnecessário** | Remover das instruções e scripts |
| `model: "opus"` obrigatório para todos | Política revogada | Escolher por natureza da tarefa; usar sonnet como padrão quando houver dúvida, ou herança somente por decisão consciente |
| Fan-out de grande escala usando equipe | Melhorado | Migrar para `Workflow` com fluxo em código, saídas estruturadas e retomada |
| Fase 0 de recuperação de contexto | Mantida | Acrescentar `resumeFromRunId` nos workflows |
| Convenção de arquivos `_workspace/` | Mantida | Preservar |
| Referências e histórico em `CLAUDE.md` | Mantidos | Preservar |

**Procedimento:**
1. Elimine do orquestrador chamadas a `TeamCreate`, `TeamDelete`, broadcast e flags experimentais.
2. Localize fan-outs e ciclos de verificação. Se o fluxo puder ser expresso em código, migre para workflow.
3. Reescreva o restante da colaboração usando `Agent(name:)`, `SendMessage`, `TaskCreate` e `TaskUpdate`.
4. Remova a atribuição indiscriminada de `model: "opus"`, conservando apenas escolhas justificadas.
5. Repita a fase 6 (verificações estruturais e simulações) e registre a migração no histórico de `CLAUDE.md`.
