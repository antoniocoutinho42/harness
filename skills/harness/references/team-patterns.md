# Padrões de equipes: seis arquiteturas, qualidade e definição de agentes

Complementa as fases 2.2 e 3 de `SKILL.md`. Para cada padrão, apresenta o modo de execução recomendado na v2. Consulte `execution-modes.md` para mais detalhes.

---

## Sumário

1. Seis padrões básicos
2. Padrões de verificação da qualidade
3. Combinação de padrões
4. Escolher o tipo de agente
5. Estrutura da definição de agentes
6. Critérios para separar agentes
7. Reutilizar agentes
8. Diferença entre skills e agentes

---

## 1. Seis padrões básicos

### 1.1. Pipeline

Encadeia o trabalho em uma ordem definida. A saída de cada agente alimenta o seguinte.

```text
[Análise] → [Projeto] → [Implementação] → [Verificação]
```

**Indicado:** quando cada fase depende fortemente da entrega anterior.

**Exemplo:** escrita de romance: mundo ficcional → personagens → enredo → redação → edição.

**Atenção:** uma fase lenta atrasa as seguintes; torne as etapas tão independentes quanto possível.

**Modo recomendado na v2:** **workflow** quando etapas e itens puderem ser enumerados. Use `pipeline()`; cada item segue para a próxima etapa assim que conclui a anterior, sem esperar todos os demais, reduzindo a duração total. Se houver revisão humana obrigatória entre fases, convoque subagentes sequencialmente.

### 1.2. Fan-out/Fan-in (distribuição e consolidação)

Executa trabalhos independentes em paralelo e reúne os resultados.

```text
              ┌→ [Especialista A] ─┐
[Distribuir] ─┼→ [Especialista B] ─┼→ [Consolidar]
              └→ [Especialista C] ─┘
```

**Indicado:** analisar a mesma questão sob perspectivas ou áreas diferentes.

**Exemplo:** pesquisa abrangente com fontes oficiais, imprensa, comunidade e contexto histórico, consolidados em um relatório.

**Atenção:** a qualidade da integração determina a qualidade do resultado final.

**Modo recomendado na v2:** priorize **workflow**. Defina as perspectivas em um array, execute com `pipeline()`, receba respostas por `schema` e consolide por código. Use `parallel()` somente quando for necessário aguardar todos os resultados para deduplicar ou comparar o conjunto completo. Para equipes de 2 a 4 pesquisadores que precisam compartilhar descobertas em tempo real, considere agentes persistentes.

### 1.3. Pool de Especialistas

Seleciona apenas os especialistas apropriados à entrada e ao contexto.

```text
[Classificador] → { Especialista A | Especialista B | Especialista C }
```

**Indicado:** quando diferentes tipos de entrada exigem métodos distintos.

**Exemplo:** revisão de código que convoca especialistas de segurança, desempenho ou arquitetura conforme a área.

**Atenção:** erros do classificador podem acionar especialistas inadequados.

**Modo recomendado na v2:** **subagentes** para consultas pontuais a especialistas selecionados. Se houver interações posteriores com o mesmo profissional, atribua `name` e utilize agentes persistentes.

### 1.4. Produção–Revisão (*Producer–Reviewer*)

Um agente produz e outro verifica os resultados.

```text
[Produção] → [Verificação] → (se houver problemas) → [Nova produção]
```

**Indicado:** entregas que exigem validação formal e permitem critérios de aprovação.

**Atenção:** limite a duas ou três rodadas para evitar correção interminável.

**Modo recomendado na v2:** **workflow** quando os critérios puderem ser codificados. Use verificação adversarial independente sobre cada achado. Se o julgamento exigir negociação qualitativa, inicie um produtor e um revisor **persistentes** com nomes e troque feedback por `SendMessage`; o produtor preservará o contexto para corrigir os pontos específicos.

### 1.5. Supervisor

Um coordenador central acompanha o estado das tarefas e redistribui o trabalho durante a execução.

```text
             ┌→ [Executor A]
[Supervisor] ┼→ [Executor B]  ← redistribuição conforme o progresso
             └→ [Executor C]
```

**Indicado:** quando a quantidade ou a distribuição de tarefas muda durante o trabalho.

**Diferença em relação ao fan-out:** a distribuição tradicional é definida antes de começar; o supervisor ajusta as atribuições com base no andamento.

**Atenção:** subdivisões excessivas criam um gargalo de coordenação.

**Modo recomendado na v2:** **agentes persistentes**. O líder usa `TaskCreate`, acompanha conclusões e redistribui por `SendMessage` e `TaskUpdate`. Se a lista e a atribuição forem antecipadamente determináveis, prefira workflow; nesse caso não há motivo para decisões contínuas do supervisor.

### 1.6. Delegação Hierárquica

Um agente responsável por uma tarefa delega partes dela a agentes de nível inferior.

```text
[Coordenação geral] → [Líder A] → [Executores A1, A2]
                   → [Líder B] → [Executor B1]
```

**Indicado:** quando o problema se divide naturalmente em blocos e subblocos.

**Atenção:** hierarquias com três ou mais níveis tendem a aumentar latência e perda de contexto. **Prefira no máximo dois níveis.**

**Modo recomendado na v2:** workflows aninhados, em que o superior chama `workflow(nameOrRef, args)`. O aninhamento é limitado a um nível. Como alternativa, simplifique a hierarquia em `phase()` de um único workflow.

## 2. Padrões de verificação da qualidade (v2)

Acrescente estes padrões quando precisão e confiabilidade forem prioridades. Eles ajudam a rejeitar saídas plausíveis, mas incorretas.

| Padrão | Funcionamento | Quando utilizar |
|---|---|---|
| **Verificação adversarial** | Para cada achado, N verificadores independentes procuram contraevidências. Aprovar como `confirmed` somente com maioria absoluta de confirmações fundamentadas. | Auditorias, pesquisas e revisões em que cada afirmação precisa ser confiável. |
| **Verificação por perspectivas diferentes** | Atribuir critérios distintos (precisão, segurança, reprodutibilidade etc.) aos N verificadores, em vez de repetir o mesmo prompt. | Quando falhas podem surgir de várias categorias. |
| **Painel de avaliadores** | Produzir N alternativas independentes e avaliá-las em paralelo. Criar a versão final a partir da mais bem avaliada, incorporando méritos das outras quando apropriado. | Planejamento e design com múltiplas soluções possíveis. |
| **Busca até esgotamento (*loop-until-dry*)** | Repetir até ocorrerem K rodadas consecutivas sem novos achados. Registrar no conjunto `seen` inclusive itens refutados, evitando reapresentações infinitas. | Busca de bugs, riscos e casos extremos sem contagem conhecida. |
| **Varredura por múltiplos critérios** | Buscar em paralelo com critérios diferentes: estrutura, conteúdo, entidades e cronologia. | Quando um único filtro deixaria lacunas. |
| **Revisor de omissões** | Um agente pergunta somente “Quais critérios não foram aplicados? Que afirmações não foram verificadas? Que fontes não foram lidas?”. | Antes da consolidação, para identificar lacunas e reabrir investigação quando necessário. |
| **Transparência sobre itens excluídos** | Ao limitar a top-N ou amostragens, registrar com `log()` quantos itens ficaram de fora. | Em qualquer execução distribuída com escopo parcial. |

**Dimensionamento:** ajuste a quantidade de agentes à solicitação. Para “veja se há bugs”, poucos exploradores e um verificador podem bastar. Para “faça uma auditoria exaustiva”, amplie a investigação e use de 3 a 5 verificadores adversariais por achado, consolidando os resultados. Se a abrangência não estiver clara, conduza auditorias com atenção à completude e use escala pequena somente em pedidos de verificação rápida.

> Exemplos de implementação em `workflow-recipes.md`.

## 3. Combinar padrões

Trabalhos reais costumam combinar vários padrões.

| Composição | Estrutura | Exemplo |
|---|---|---|
| **Fan-out/Fan-in + Produção–Revisão** | Produzir várias versões em paralelo e revisar cada uma. | Tradução em quatro idiomas, com revisão linguística independente. |
| **Pipeline + Fan-out/Fan-in** | Algumas fases sequenciais, outras paralelas. | Análise sequencial → implementação paralela → testes integrados. |
| **Supervisor + Pool de Especialistas** | Supervisor classifica e convoca especialistas segundo a demanda. | Encaminhamento de solicitações de clientes. |
| **Investigação + verificação adversarial + consolidação** | Pesquisar por critérios diversos, verificar cada achado, buscar omissões e produzir relatório. | Auditoria técnica e pesquisa aprofundada. |

Escolha o modo **por fase** da composição: o critério principal é se seu fluxo de controle pode ser antecipadamente expresso em código.

## 4. Escolher o tipo de agente

Utilize `subagent_type` na ferramenta `Agent` ou `agentType` em `Workflow`.

### Tipos nativos

| Tipo | Ferramentas disponíveis | Uso |
|---|---|---|
| `general-purpose` | Todas, incluindo WebSearch e WebFetch | Pesquisa web, trabalho geral e alterações em arquivos. |
| `Explore` | Somente leitura, sem Edit e Write | Localizar código; não é suficiente para revisões ou auditorias com execução de testes. |
| `Plan` | Somente leitura, sem Edit e Write | Arquitetura e planejamento de implementação. |

### Tipos personalizados

Defina em `.claude/agents/{name}.md` e chame por `subagent_type: "{name}"` ou `agentType: "{name}"`. Configure ferramentas e modelos no frontmatter YAML.

### Critérios de escolha

| Situação | Preferência | Justificativa |
|---|---|---|
| Especialidade complexa e reutilizável em várias sessões | **Tipo personalizado** | Mantém papel e critérios de forma consistente. |
| Pesquisa pontual com instrução suficientemente clara | `general-purpose` + prompt detalhado | Dispensa arquivo de definição. |
| Localizar código sem modificar nada | `Explore` | Reduz risco de alterações acidentais. |
| Elaborar somente plano ou arquitetura | `Plan` | Foco em análise, sem alteração do repositório. |
| Alteração pontual de arquivo | `general-purpose` + prompt | Resolve sem criar agente permanente. |
| Implementação especializada ou recorrente | **Tipo personalizado** | Permite padronizar ferramentas e regras. |
| Edição paralela de muitos arquivos | Tipo personalizado + `isolation: 'worktree'` | Evita conflitos, justificando o custo de isolamento. |

**Regra:** defina em arquivo os especialistas que devem persistir entre sessões. Para usos pontuais de tipos nativos, não crie definições desnecessárias.

**Política de modelos:** escolha pelo tipo de tarefa. Use **fable** quando houver planejamento e execução autônoma prolongada, **opus** para arquitetura, código, análises complexas e verificação cruzada e **sonnet** para logs, conversão de formatos, inspeção estática, deploy e coleta simples. Se não houver motivo para outro modelo, prefira sonnet. Não configure o mesmo modelo para todos indiscriminadamente. Registre justificativas e consulte `model-selection-guide.md`.

## 5. Estrutura da definição de agentes

```markdown
---
name: agent-name
description: "Descreve o papel em uma ou duas frases e informa quando acioná-lo."
# tools: Read, Grep, Glob, Bash  ← opcional: restrições de ferramentas para agentes de leitura
#                                Para editores, inclua Edit e Write.
# model: sonnet                ← selecionar pela tarefa; documentar a justificativa
---

# Nome do agente — resumo do papel

Você é especialista em [papel] no domínio [domínio].

## Papel principal
1. Responsabilidade 1.
2. Responsabilidade 2.

## Princípios de trabalho
- Princípio 1, com justificativa para decisões em situações excepcionais.
- Princípio 2.

## Contratos de entrada e saída
- Entrada: [o que recebe e onde: arquivo, args ou artefatos anteriores].
- Saída: [onde grava: caminho em _workspace/ ou retorno estruturado].
- Formato: [estrutura de arquivo ou schema].

## Regras de comunicação (agentes persistentes)
- Primeiro relatório: informar as ferramentas realmente disponíveis,
  pois algumas declaradas em tools podem não ter sido carregadas.
- Recebimento: [de quem e com que finalidade recebe mensagens].
- Envio: [a quem e sobre o quê envia mensagens].
- Tarefas compartilhadas: [quais atividades registra e solicita].

## Instruções para nova chamada
- Leia artefatos anteriores antes de aprimorá-los.
- Se houver feedback, altere somente os pontos pertinentes.

## Tratamento de erros
- [Como agir após uma falha].
- [Como agir se as entradas estiverem ausentes ou ambíguas].

## Colaboração
- Relações com outros agentes.
```

**Para agentes exclusivos de workflow:** substitua “Regras de comunicação” por “Saída estruturada”. Defina o schema JSON e esclareça que o texto final é **retorno de dados**, não mensagem ao usuário.

**Cuidados com `tools`:**
- Agentes que alteram artefatos precisam de Edit **e** Write. Com Write apenas, uma pequena modificação exige reescrever todo o arquivo; em artefatos extensos, o agente pode abandonar a edição e criar um arquivo alternativo, deixando o original incorreto.
- Ferramentas carregadas sob demanda, como `TaskCreate` e `TaskUpdate`, podem faltar mesmo estando em `tools` (observado no Claude Code 2.1.226, podendo variar por ambiente). `SendMessage`, também carregada sob demanda, estava disponível na observação, de modo que a regra não é simples. Exija no primeiro relatório dos agentes persistentes uma relação das ferramentas efetivamente disponíveis. Se algo faltar, o orquestrador assume a operação, como atualizar tarefas por `TaskUpdate`, ou remove-se a restrição de `tools`.

## 6. Critérios para separar agentes

| Critério | Separe quando... | Considere reunir quando... |
|---|---|---|
| Conhecimento especializado | Os domínios de responsabilidade forem diferentes. | As responsabilidades forem semelhantes. |
| Paralelismo | As tarefas puderem ser independentes. | O processo for estritamente sequencial. |
| Volume de contexto | Uma responsabilidade exigir muitos dados. | As entradas forem curtas e a tarefa pequena. |
| Reutilização | O especialista puder ser usado por outras equipes. | O papel só fizer sentido nesta equipe. |

## 7. Reutilizar agentes

Antes de criar outro agente, compare papéis em `.claude/agents/`. Ampliações sucessivas podem produzir agentes idênticos com nomes diferentes.

| Relação com agente existente | Ação |
|---|---|
| Já cobre integralmente o papel | Reutilizar sem criar outro. |
| Há sobreposição e o agente pode ser generalizado | Ampliar o existente. |
| Há sobreposição, mas especialização intencional de domínio | Criar agente distinto. |
| Responsabilidades totalmente independentes | Criar agente novo. |

**Princípio:** papéis bem delimitados são mais reutilizáveis. Se a ampliação produzir dois papéis claramente distintos, avalie separá-los.

**Antes de ampliar:** identifique todos os orquestradores que chamam o agente. A mudança poderá afetá-los. Atualize a `description` e, depois, simule o fluxo antigo conforme a fase 6.5 de `SKILL.md`.

## 8. Skills versus agentes

| Aspecto | Skill | Agente |
|---|---|---|
| O que define | Procedimento e uso das ferramentas | Papel especializado e princípios de atuação |
| Caminho | `.claude/skills/` | `.claude/agents/` |
| Acionamento | Solicitação corresponde à `description` | Chamada explícita por `Agent` ou `Workflow` |
| Pergunta respondida | **Como fazer?** | **Quem faz?** |

### Conectar skills aos agentes

| Método | Implementação | Quando utilizar |
|---|---|---|
| **Chamar pela ferramenta Skill** | No prompt do agente, instrua: “Acione `/skill-name` pela ferramenta Skill”. | Skill independente, também invocável diretamente pelo usuário. |
| **Inserir instruções no prompt** | Incluir o conteúdo da skill na definição do agente. | Skill curta (até 50 linhas), usada somente por aquele agente. |
| **Ler referências sob demanda** | Acessar `references/` por `Read` somente quando necessário. | Skill extensa, com detalhes pertinentes a situações específicas. |

Para skills compartilhadas entre agentes, prefira a ferramenta Skill. Para regras breves e exclusivas, inclua-as no prompt; para conteúdo extenso, carregue apenas a referência relevante.
