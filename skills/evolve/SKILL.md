---
name: evolve
description: "Skill para aprimorar harnesses existentes a partir das execuções realizadas. Coleta e generaliza feedback, atualiza agentes, skills e orquestradores, identifica diferenças em relação à configuração inicial e registra mudanças. Use obrigatoriamente quando o usuário pedir 'revisar o harness', 'evoluir o harness', 'incorporar feedback ao harness', 'melhorar o harness', 'o resultado ficou aquém do esperado', 'aplique esta lição no harness' ou solicitações equivalentes de melhoria baseada em execução. Para criar um harness, redesenhar sua arquitetura ou adicionar agentes, use a skill harness."
---

# Harness Evolve — Aprimoramento contínuo do harness

O harness não é uma estrutura estática: ele deve evoluir com a experiência. Esta skill identifica o que funcionou e o que não funcionou, incorpora as lições à configuração e busca tornar a próxima execução mensuravelmente melhor.

```text
Harness inicial ──▶ Uso em projetos reais ──▶ Harness atual
                                                 │
                                                 ▼ (identificar diferenças)
                                          Generalizar o feedback
                                                 │
                                                 ▼
                                   Atualizar agentes, skills e orquestrador
                                                 │
                                                 ▼
                                   Registrar mudanças e melhorar a próxima execução
```

## Processo de evolução

### Fase 1 — Identificar diferenças e coletar evidências

1. Leia `.claude/agents/`, `.claude/skills/` e a tabela de histórico de alterações em `CLAUDE.md`.
2. Se houver repositório Git, consulte o histórico dos arquivos do harness (`git log --oneline -- .claude/ CLAUDE.md`) para entender o que mudou, quando e por quê.
3. Se existir `_workspace/`, examine os artefatos das últimas execuções:
   - Os arquivos foram realmente produzidos nos caminhos definidos pelo orquestrador? Se não, o fluxo pode ter sido contornado ou conter código inativo.
   - Os resultados atendem aos formatos e critérios de qualidade definidos nas skills?
4. Solicite feedback ao usuário somente se ainda não tiver sido fornecido:
   - “O que poderia ter sido melhor no resultado?”
   - “Você mudaria algo na composição da equipe ou na ordem de execução?”
   - Não insista na ausência de feedback. Entretanto, proponha melhorias proativamente se encontrar os sinais abaixo.

**Indícios de evolução identificados por observação (mesmo sem feedback):**
- Solicitações de correção semelhantes repetidas duas ou mais vezes.
- Padrões recorrentes de falhas e novas tentativas dos agentes.
- Evidências de que o usuário contornou manualmente o orquestrador; isso pode indicar falha de acionamento e necessidade de ampliar a `description`.
- Vestígios da v1 (`TeamCreate`, `TeamDelete` ou flags experimentais); nesse caso, indique a migração pela skill `harness`.

### Fase 2 — Classificar o feedback e identificar o ponto de alteração

| Tipo de feedback | Onde alterar | Exemplo |
|---|---|---|
| Qualidade do resultado | Skill do agente responsável | “A análise ficou superficial” → especificar critérios de profundidade |
| Papel do agente | Definição `.md` do agente | “Precisamos avaliar segurança” → encaminhar à skill `harness` para adicionar um agente |
| Ordem do processo | Skill orquestradora | “A validação deveria vir antes” → reorganizar as fases |
| Composição da equipe | Orquestrador e agentes | “Esses dois papéis podem ser reunidos” → consolidar agentes |
| Falha de acionamento | Campo `description` da skill | “Este pedido não ativa o fluxo” → ampliar os gatilhos |
| Modo de execução inadequado | Orquestrador | “Sempre repete o mesmo fan-out e demora” → considerar o modo workflow |
| Escala ou custo | Orquestrador | “Consome tokens demais” → reduzir a escala padrão e vincular ao orçamento |

**Limite de escopo:** se for necessário adicionar ou remover agentes ou redesenhar a arquitetura, não faça isso diretamente nesta skill. Encaminhe a solicitação à skill `harness`, seguindo a ampliação da configuração existente na fase 0. A skill `evolve` deve se concentrar em **ajustar a configuração atual**.

### Fase 3 — Generalizar e incorporar as melhorias

1. **Generalize o feedback.** Uma regra específica demais para um caso é sobreajuste. Se “a introdução deste relatório ficou longa”, não imponha automaticamente “toda introdução terá no máximo 10% do texto”. Identifique a causa (por exemplo, ausência de critérios para distribuição do conteúdo) e corrija o princípio.
2. **Registre a justificativa.** Inclua o motivo das mudanças nas instruções revisadas, permitindo que agentes decidam bem também em situações excepcionais.
3. Aplique **uma alteração por vez** e execute a fase 4 após cada uma.
4. **Evite regressões.** Se uma alteração contrariar uma decisão anterior, informe o conflito e peça confirmação. Por exemplo, diante de feedbacks alternados de “longo demais” e “curto demais”, encontre um critério que considere ambos, em vez de simplesmente reverter a decisão.
5. **Respeite o idioma dos arquivos existentes.** Ao editar agentes, skills, orquestradores ou `CLAUDE.md`, use o idioma já adotado em cada arquivo. O idioma deste documento não deve contaminar conteúdos escritos em outro idioma.

### Fase 4 — Atualizar o histórico e validar

1. Registre a alteração na tabela de **Histórico de alterações** de `CLAUDE.md`:

```markdown
**Histórico de alterações:**
| Data | Alteração | Arquivo ou componente | Motivo |
|---|---|---|---|
| 2026-07-19 | Acrescentado guia de tom | skills/content-creator | Feedback de que o texto estava formal demais |
```

2. Valide a estrutura dos arquivos modificados (frontmatter e consistência das referências).
3. Se a `description` tiver mudado, verifique o acionamento com pelo menos três casos positivos (*should-trigger*) e três casos semelhantes que não devem acionar a skill (*near-miss*).
4. Confirme a correspondência entre `CLAUDE.md` e os arquivos existentes.

### Fase 5 — Relatar a evolução

Informe ao usuário:
- Resumo das diferenças identificadas (configuração inicial → atual).
- Alterações implementadas e fundamentos das generalizações.
- Feedbacks não incorporados, com justificativa, se houver.
- Melhorias esperadas na próxima execução.

## Princípios

- **A evolução acumulada é um ativo.** Um bom histórico faz com que o próximo harness no mesmo domínio comece mais próximo de um sistema pronto para uso. Não apague esse histórico.
- **Não incorpore feedback sem generalizá-lo.** Acrescentar uma regra para cada caso isolado torna a skill um acúmulo incoerente de exceções; converta observações em princípios.
- **Uma mudança de cada vez.** Alterações simultâneas dificultam atribuir os resultados a cada decisão.
