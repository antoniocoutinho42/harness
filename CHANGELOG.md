# Histórico de versões

Este projeto segue o [Versionamento Semântico](https://semver.org/).

## [Não publicado] — Localização em português brasileiro

### Alterado

- Tradução editorial das skills principais e de seus nove documentos de referência para português brasileiro, com preservação de nomes de ferramentas, comandos, schemas, identificadores e caminhos.
- Localização dos guias de instalação, manutenção e contribuição, templates de issues e metadados públicos do plugin.
- Adequação dos prompts de exemplos à linguagem natural em português, sem tradução de identificadores de código e de contratos de integração.

## [2.1.0] — 2026-09-26

### Alterado

- **Política de modelos: de “herdar o modelo da sessão por padrão” para “escolher o nível conforme a tarefa”.** Ao definir um agente, considerar quatro dimensões: complexidade, duração, autonomia e latência. Usar fable para planejamento e execução autônoma prolongada de altíssima dificuldade; opus para arquitetura, programação, análises complexas e verificações cruzadas; e sonnet para rotinas como logs, conversões de formato e coleta simples. Usar sonnet em caso de dúvida e não impor um modelo único sem justificativa.
- **Revisão editorial das skills.** Eliminação de expressões artificiais de tradução e padronização de termos na skill Harness e nos nove arquivos de referência.
- **Sincronização da versão 2.1.0.** Alinhamento de `.claude-plugin/plugin.json`, `marketplace.json` e selos dos READMEs em inglês e coreano.

### Adicionado

- **`references/model-selection-guide.md`.** Guia detalhado sobre papéis e usos de cada modelo, critérios para diferenciá-los e regras de aplicação no harness, incluindo seleção por tarefa, separação entre coordenação e execução e escolha por etapa de workflow.
- **Regra de idioma dos artefatos.** Agentes, skills e orquestradores gerados devem adotar o idioma da conversa do usuário, não o idioma em que a documentação da skill foi escrita (#28).
- **Congelamento de artefatos nas transições de fase.** Na colaboração entre agentes persistentes, todos devem confirmar conclusão antes do início da próxima fase. O líder comunica o congelamento, registra hashes e os confere novamente na validação final (#53).
- **Detecção de agentes e skills redundantes.** Antes de criar novos, identificar responsabilidades existentes e ampliá-las quando houver sobreposição. Adaptação à v2 de contribuição incorporada à v1.x (#17).

### Corrigido

- **Tratamento de falhas irrecuperáveis.** Limite de uso esgotado, autenticação expirada e permissão negada não devem causar novas tentativas automáticas. O sistema deve examinar artefatos parciais, registrar lacunas e informar o avanço real. O orquestrador só pode completar fatos verificados, sem inventar julgamentos de agentes interrompidos (#53).
- **Cuidados com o campo `tools:`.** Alertas sobre falta de Edit em agentes que editam arquivos e sobre ferramentas carregadas sob demanda que podem não estar disponíveis (#53).
- **Comandos de instalação.** Correção de `harness@harness` para `harness@harness-marketplace` no README e no início rápido (#46).
- **E-mail do responsável no marketplace.** Preservação de `owner.email` da branch principal, evitando campo vazio na v2.

## [2.0.0] — 2026-07-19

Reconstrução integral para o runtime multiagente atual do Claude Code. A API experimental de Agent Teams utilizada pela v1 deixou de existir e foi acrescentada a ferramenta `Workflow`, voltada à orquestração determinística.

### Incompatibilidades resolvidas e correções

- **Remoção de `TeamCreate`, `TeamDelete` e `team_name`.** Reescrita dos modelos para a equipe implícita da sessão, com `Agent(name:)` e `SendMessage`. O orquestrador v1 podia tentar utilizar APIs inexistentes e, silenciosamente, degradar para um único agente.
- **Fim da dependência `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.** Remoção de `docs/experimental-dependency.md` e das referências à flag; a v2 funciona sem ela.
- **Fim da imposição geral de `model: "opus"`.** Na v2.0.0, passou-se a herdar o modelo da sessão por padrão, aceitando sobrescrita justificada por economia em tarefas mecânicas ou raciocínio de alto nível. Essa regra seria refinada na v2.1.0.
- **Coerência entre README e implementação.** A skill `/harness:evolve`, antes anunciada mas inexistente, foi efetivamente entregue.
- **Remoção da regra “uma equipe ativa por sessão” e dos procedimentos de recriação.** A restrição deixou de existir no novo modelo.

### Adicionado

- **Três modos de execução.** Orquestração por workflow (novo), colaboração entre agentes persistentes (substituindo a antiga equipe) e delegação a subagentes. Critério de escolha passou do tamanho da equipe para a **determinabilidade do fluxo de controle**.
- **Orquestração por workflow.** Scripts com `pipeline()` e `parallel()`, schemas de saídas estruturadas, dimensionamento vinculado a `budget`, retomada parcial por `resumeFromRunId`, `agentType` personalizado e `isolation: worktree`.
- **Catálogo de padrões de qualidade.** Verificação adversarial, verificação por perspectivas, painel de avaliadores, busca até esgotamento, varredura por vários critérios, revisão de omissões e transparência sobre itens excluídos.
- **`skills/evolve` (`/harness:evolve`).** Ciclo de cinco etapas: identificar diferenças, classificar feedback, generalizar ajustes, atualizar histórico e relatar evolução. Inclui sinais observacionais, como feedback recorrente e contornos do orquestrador.
- **Migração v1 → v2.** Identificação automática de elementos v1 na fase 0 e mapeamentos em `docs/migration-v1-to-v2.md` e `references/execution-modes.md`.
- **Novas referências.** `execution-modes.md` (modos e migração) e `workflow-recipes.md` (seis modelos de scripts e doze pontos de atenção).
- **Retorno estruturado nos protocolos de transferência.** Uso de `schema` para fornecer dados validados entre etapas como padrão recomendado em workflows.
- **Testes A/B por workflow.** Receita para comparar resultados com e sem skill.
- **Definições de agentes mais completas.** Suporte a `tools` no frontmatter para restrição de ferramentas e seção padronizada de instruções para reexecução.

### Alterado

- **Mapeamento dos seis padrões a modos da v2.** Preferência por workflows para pipeline e fan-out; supervisor persistente com tarefas compartilhadas; delegação hierárquica com aninhamento restrito.
- **Reescrita dos três modelos de orquestração.** A: workflow com reconhecimento, execução, síntese, `run_meta` e retomada. B: agentes persistentes com comunicação e tarefas. C: delegação pontual a subagentes.
- **Reorganização da fase 7.** A skill Harness cuida de operação e manutenção; a skill Evolve cuida de incorporar feedback.
- **Checklist de entrega.** Novas verificações de elementos v1 e armadilhas de workflows.
- **Referências de arquitetura.** `references/agent-design-patterns.md` foi convertido em `team-patterns.md`; cinco exemplos em `team-examples.md` foram atualizados para a v2.

### Removido

- **Recursos de marketing e site:** `harness_banner.png`, `harness_icon.png`, `harness_social.png`, `harness_team.png` (cerca de 9 MB), `index.html` e `privacy.html`.
- **Artefatos operacionais internos:** resultados de campanha em `_workspace/`, agentes de marketing (`.claude/agents/launch-strategist` e outros) e `docs/experimental-dependency.md`.
- **README_JA** em japonês, cuja manutenção não se justificava. Restaram as versões EN/KO na origem.

## [1.2.1] — 2026-04-18

### Corrigido

- **Coerência entre versões.** Os selos de README.md, README_KO.md e README_JA.md indicavam `v1.0.1`, enquanto `.claude-plugin/marketplace.json` indicava `1.1.0` e `.claude-plugin/plugin.json`, `1.2.0`. Os três foram alinhados a **v1.2.0**, considerando plugin.json como referência.
- **Preparação para tags de versões anteriores.** Foi criado um plano para registrar retroativamente v1.0.0, v1.0.1, v1.1.0 e v1.2.0, já que não existiam releases marcadas (`_workspace/release/audit-2026-04-18.md`, seção 4).

### Adicionado

- **Posicionamento como “harness factory”.** Descrição de categoria no início do README para diferenciar a proposta de gerar agentes e skills por domínio de frameworks para prompts ou agentes isolados.
- **CONTRIBUTING.md.** Guia de contribuições e metas de resposta (72 horas para primeira resposta em PRs e 48 horas para triagem de issues).
- **Pasta docs/.** Local previsto para documentação de longa duração (arquitetura, migração e catálogo de padrões), evitando que o README ficasse excessivamente extenso.
- **Política de resposta para issue #3.** Modelo de respostas oficiais e processo de triagem de assuntos da comunidade.

### Alterado

- Versão de `.claude-plugin/marketplace.json`: `1.1.0` → `1.2.0`.
- Selos dos READMEs em inglês, coreano e japonês: `Version-1.0.1` → `Version-1.2.0`.
- **Descrição de `.claude-plugin/plugin.json`.** Evolução de `"Agent Team & Skill Architect — Meta-skill that designs..."` para uma descrição de fábrica de equipes de agentes, com seis padrões e posicionamento como metafábrica. A versão original combinava inglês e coreano.
- **Palavras-chave de `.claude-plugin/plugin.json`.** Expansão de cinco para 17, incluindo `harness-factory`, `team-architecture-factory`, `claude-code-plugin`, `agent-scaffolding`, `multi-agent` e seis termos de padrões.

## [1.2.0] — 2026-04-08

### Alterado

- **Simplificação do registro em `CLAUDE.md`.** Na fase 5.4, “registrar contexto” virou “registrar referência”. Foram removidas de `CLAUDE.md` listas de agentes e skills, diretórios e regras detalhadas, preservando apenas **gatilhos e histórico**. Arquivos de agentes, skills e a skill orquestradora passaram a ser a fonte única.
- **Remoção de sincronizações provisórias nas fases 3 e 4.** Em vez de atualizar `CLAUDE.md` em cada criação de artefato, a referência final passou a ser registrada uma única vez na fase 5.4.
- **Terceiro princípio fundamental.** Mudança de “registrar o contexto do harness em `CLAUDE.md`” para “registrar uma referência do harness”.
- **Remoção da tabela de responsabilidades `CLAUDE.md` versus orquestrador.** A nova política de referência mínima tornou a tabela desnecessária.

### Adicionado

- **Fase 2.1: modo híbrido.** Combinação de equipes persistentes e subagentes em fases diferentes, com exemplos como coleta paralela e consolidação, criação e verificação, e reconfiguração de equipes entre etapas.
- **Comparação de modos na fase 2.1.** Quadro comparativo de equipes, subagentes e híbridos, com procedimento de decisão em três passos.
- **Padrão híbrido do orquestrador na fase 5.0.** Regra para identificar o modo de execução no início de cada fase.
- **Transferência de dados por retorno na fase 5.1.** Estratégia específica para subagentes.
- **Combinações de transferência recomendadas na fase 5.1.** Ampliação das opções de mensagens, tarefas e arquivos para cobrir os modos de subagentes e híbridos.

## [1.1.0] — 2026-04-05

### Adicionado

- **Fase 0: auditoria do estado atual.** Ao acionar a skill, verificar primeiro o harness existente e encaminhar para criação, ampliação ou manutenção.
- **Matriz de fases para ampliação.** Definir fases necessárias conforme adição de agentes, skills ou mudanças na arquitetura.
- **Sincronização provisória de `CLAUDE.md` nas fases 3 e 4.** Registrar os artefatos imediatamente após criá-los para resistir a interrupções.
- **Fase 5.4: registro do contexto do harness em `CLAUDE.md`.** Composição da equipe, skills, regras, estrutura de diretórios e histórico; incluía tabela de divisão de responsabilidades.
- **Fase 5.5: suporte a solicitações posteriores.** Gatilhos para continuação na `description` e interpretação automática de primeira execução, reexecução parcial ou nova execução.
- **Fase 5: alteração de orquestradores existentes.** Não duplicar a skill orquestradora ao ampliar um harness.
- **Fase 7: evolução do harness.** Coletar feedback após a execução, mapear tipos de correção a arquivos, registrar mudanças e identificar sinais de evolução.
- **Fase 7.5: operação e manutenção.** Auditoria → alterações graduais → sincronização com `CLAUDE.md` → validação.
- **Gatilhos de manutenção na `description`.** Expressões relativas a auditoria, estado atual e sincronização de agentes e skills.
- **Checklist reforçado.** Verificar atualização de `CLAUDE.md`, histórico e recuperação de contexto na fase 0.
- **Fase 0 em modelos de orquestrador.** Recuperação de contexto para equipes persistentes e subagentes.
- **Gatilhos de continuação nos modelos de `description`.**

### Alterado

- Ampliação dos princípios fundamentais de dois para quatro (registro em `CLAUDE.md` e evolução).
- **Terminologia “log de evolução” → “histórico de alterações”.** Nome e schema de quatro colunas (data, alteração, componente e motivo) padronizados.
- **Fase 1, passo 3.** Passou a usar o diagnóstico da fase 0 para identificar sobreposições.
- **Modelo `CLAUDE.md` da fase 5.4.** Correção de um bloco Markdown aninhado, com quatro crases em vez de três.
- **Tabela de responsabilidades.** Inclusão de skills, estrutura de pastas e histórico.
- **Modelos de orquestradores.** Acrescentada fase 0 e expressões de continuação.

## [1.0.1] — 2026-03-28

### Alterado

- Eliminação de duplicações entre `SKILL.md` e `references/` (de 330 para 285 linhas).
  - Fase 2.1: tabela de modos e lista de princípios substituídas por resumo e referência a `agent-design-patterns.md`.
  - Fase 2.3: critérios de separação de agentes resumidos em quatro dimensões, com referência a `agent-design-patterns.md`.
  - Fase 3: modelo completo de definição de agente substituído por seções obrigatórias e referência.
  - Fase 5.2: tabela de cinco casos de erros substituída por princípios e referência a `orchestrator-template.md`.

## [1.0.0] — 2026-03-27

### Adicionado

- Metaskill de configuração de harness por um processo de seis fases.
- Seis padrões de agentes: Pipeline, Fan-out/Fan-in, Pool de Especialistas, Produção–Revisão, Supervisor e Delegação Hierárquica.
- Modos de execução por equipes e subagentes.
- Guia de criação de skills com *Progressive Disclosure*.
- Modelos de orquestradores para equipes e subagentes.
- Guia para incorporar agentes de QA com base em sete bugs observados em projetos reais.
- Metodologia de testes de skills com comparação entre execuções com e sem a skill.
- Cinco exemplos práticos: pesquisa, romance, webtoon, revisão de código e migração.
