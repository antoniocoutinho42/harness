# Exemplos práticos de composição de equipes

Os exemplos demonstram como escolher o modo de execução conforme a tarefa e apresentam o formato de orquestração do Harness v2. Consulte `execution-modes.md` para os modos e `workflow-recipes.md` para os scripts.

---

## Exemplo 1 — Equipe de pesquisa abrangente com workflow

**Arquitetura:** Fan-out/Fan-in + verificação adversarial.

**Motivo:** as perspectivas de pesquisa podem ser enumeradas, e a validação de cada afirmação pode ser expressa em código.

```text
[Principal] Reconhecimento: definir perspectivas (fontes oficiais, imprensa, comunidade, contexto)
    → Workflow(script, args: {axes, topic, ws})
        phase 'pesquisa': pipeline(axes, axis => agent(..., {schema: FINDINGS}))
        phase 'verificacao': verificar cada afirmação; aprovar apenas maioria com status confirmed
        phase 'sintese': revisor de omissões examina lacunas e solicita pesquisa adicional
    → Principal redige relatório com os resultados estruturados
```

Defina o método de pesquisa e a saída estruturada em `.claude/agents/researcher.md`; registre em `.claude/agents/fact-checker.md` as diretrizes de verificação com busca de contraevidências. Utilize `agentType` para chamar esses dois tipos no workflow.

Não elimine arbitrariamente informações conflitantes; preserve-as junto às fontes no schema de retorno.

## Exemplo 2 — Escrita de ficção científica com agentes persistentes e modo híbrido

**Arquitetura:** Pipeline + Produção–Revisão.

**Motivo:** mundo ficcional, personagens e enredo precisam permanecer coerentes por várias conversas; revisores independentes, por outro lado, podem entregar apenas um resultado pontual.

```text
Fase 1 (persistente): Agent(name: "worldbuilder") + Agent(name: "character-designer")
        + Agent(name: "plot-architect") em paralelo
        → TaskCreate(mundo, personagens, enredo; registrar dependências)
        → Líder transmite decisões: worldbuilder define estruturas sociais;
          usa SendMessage para informar character-designer.
        → Se profissões dos personagens conflitarem com o mundo, solicita
          ajuste a worldbuilder por SendMessage.
        → O contexto preservado permite pedir “modifique apenas
          a classe de comerciantes definida anteriormente”.
Fase 2 (subagente): chamar prose-stylist uma vez;
        ele lê os três resultados em _workspace/ e redige o texto.
Fase 3 (subagentes paralelos): science-consultant e continuity-manager revisam.
Fase 4 (persistente): não é possível enviar SendMessage a prose-stylist
        se ele foi chamado uma única vez sem name. Se alterações
        forem prováveis, inicie-o com name desde a fase 2.
```

Se houver probabilidade de revisões, atribua `name` ao agente desde sua primeira execução. Subagentes sem nome não continuam a conversa anterior.

## Exemplo 3 — Revisão completa de código por workflow

**Arquitetura:** Fan-out/Fan-in + verificação adversarial.

**Motivo:** segurança, desempenho, arquitetura e testes são dimensões predefinidas; cada achado também pode ser validado separadamente.

```javascript
// Examinar cada perspectiva e verificar adversarialmente cada achado.
// Não espere todas as perspectivas: assim que segurança concluir,
// seus achados podem ser verificados mesmo com desempenho em andamento.
const FINDINGS = { type: 'object', required: ['findings'], properties: {
  findings: { type: 'array', items: { type: 'object',
    required: ['title', 'file', 'evidence'], properties: {
      title: { type: 'string' }, file: { type: 'string' }, evidence: { type: 'string' } } } } } }
const VERDICT = { type: 'object', required: ['status', 'reason'], properties: {
  status: { type: 'string', enum: ['confirmed', 'refuted', 'uncertain'] },
  reason: { type: 'string' } } }
const results = await pipeline(
  [
    { key: 'security', prompt: 'Revise o código sob a ótica da segurança.' },
    { key: 'perf', prompt: 'Revise o código sob a ótica do desempenho.' },
    { key: 'arch', prompt: 'Revise o código sob a ótica da arquitetura.' },
    { key: 'test', prompt: 'Revise o código sob a ótica da cobertura de testes.' },
  ],
  d => agent(d.prompt, { phase: 'revisao', schema: FINDINGS }),
  r => parallel((r?.findings ?? []).map(f => () =>
    agent(`Valide este achado. Responda confirmed se houver evidências suficientes, refuted se houver refutação clara ou uncertain se não for possível concluir: ${JSON.stringify(f)}`,
      { phase: 'verificacao', schema: VERDICT })
      .then(verdict => ({ ...f, verdict }))))
)
const confirmed = results.flat().filter(Boolean)
  .filter(f => f.verdict?.status === 'confirmed')
```

Na v1, revisores persistentes compartilhavam achados via `SendMessage`. Quando cada achado puder ser verificado isoladamente, envie a evidência no prompt e valide imediatamente. Se for necessário confrontar achados de perspectivas diferentes, espere a revisão completa e adicione uma barreira antes da verificação. Use agentes persistentes somente se houver necessidade real de discussão interativa.

## Exemplo 4 — Migração de código em grande escala

**Arquitetura:** Fan-out/Fan-in se os lotes forem conhecidos; Supervisor se precisarem ser redefinidos durante a execução.

**Decisão:** a possibilidade de dividir os lotes antecipadamente determina o modo.

Se a lista de lotes for conhecida, use workflow:

```javascript
const MIGRATION_RESULT = { type: 'object', required: ['worktreePath', 'changedFiles'], properties: {
  worktreePath: { type: 'string', minLength: 1 },
  changedFiles: { type: 'array', minItems: 1, uniqueItems: true,
    items: { type: 'string', minLength: 1 } } } }
const MIGRATION_VERDICT = { type: 'object', required: ['status', 'reason'], properties: {
  status: { type: 'string', enum: ['confirmed', 'refuted', 'uncertain'] },
  reason: { type: 'string' } } }
const migrated = await pipeline(args.batches, // Fixe os lotes após estimar sua complexidade.
  b => agent(`Migre estes arquivos e retorne o caminho absoluto do worktree isolado e a lista real de arquivos modificados: ${b.files.join(', ')}`,
    { agentType: 'migrator', isolation: 'worktree', schema: MIGRATION_RESULT }),
  (r, b) => r && agent(
    `No worktree isolado ${r.worktreePath}, confronte o escopo previsto e o real. Previsto: ${b.files.join(', ')}. Alterado: ${r.changedFiles.join(', ')}. Identifique omissões e erros.`,
    { agentType: 'qa-inspector', schema: MIGRATION_VERDICT })
    .then(verdict => ({ ...r, verdict })))
const confirmed = migrated.filter(Boolean)
  .filter(r => r.verdict?.status === 'confirmed')
return { confirmed }
```

Envie aos verificadores o caminho do worktree onde a migração foi realizada. Ao terminar o workflow, o agente principal deve conferir os caminhos `worktreePath` com status `confirmed` e integrar as alterações à branch de base, por merge ou aplicação seletiva de commits. Resolva conflitos antes de integrar o próximo worktree e execute testes de integração a cada alteração. Não incorpore resultados `refuted` ou `uncertain`; relate as lacunas.

Se os lotes precisarem de redistribuição dinâmica, use agentes persistentes:

```text
O líder cria lotes por TaskCreate, incluindo depends_on.
→ Inicia Agent(name: "migrator-1"), Agent(name: "migrator-2"),
  Agent(name: "migrator-3") em paralelo.
→ A cada conclusão, verifica os artefatos.
→ Em falhas, investiga por SendMessage e redistribui via TaskUpdate.
→ Quando todos concluírem, executa testes de integração.
```

## Exemplo 5 — Produção de webtoon com autor persistente e revisor pontual

**Arquitetura:** Produção–Revisão.

**Motivo:** bastam um criador e um revisor. Como o criador só precisa receber até duas rodadas de feedback, um modo híbrido leve é suficiente.

```text
Fase 1: Agent(name: "artist") cria painéis em _workspace/panels/.
Fase 2: Agent(subagent_type: "webtoon-reviewer", prompt: "Revise os painéis")
        é chamado uma vez → decide PASS/FIX/REDO
        → _workspace/review_report.md
Fase 3: Somente os painéis marcados REDO retornam a "artist"
        por SendMessage({to: "artist"}). Repetir no máximo duas vezes.
        O contexto permite pedidos como “altere apenas o enquadramento do painel 3”.
Contingência: se duas revisões não resolverem, informe o problema ao usuário.
        Se mais de 50% dos painéis receberem REDO,
        sugira que o usuário revise o prompt.
```

---

## Convenções para salvar artefatos

- **Agentes:** definições em `<projeto>/.claude/agents/{name}.md`, com papel, princípios, entradas, saídas, forma de nova chamada, tratamento de erros e colaboração. Inclua regras de comunicação para persistentes e contratos estruturados para workflows.
- **Skills:** `<projeto>/.claude/skills/{name}/SKILL.md`, acrescentando `references/` e `scripts/` quando necessário.
- **Orquestrador:** indique obrigatoriamente o modo de execução. Utilize os modelos de `orchestrator-template.md`.
- **Artefatos intermediários:** salve em `_workspace/{phase}_{agent}_{artifact}.{ext}` e mantenha-os mesmo após a validação.
