# Receitas de workflows: modelos de scripts e cuidados

Este documento apresenta padrões de scripts para skills orquestradoras no **modo A (Workflow)**. Escreva os scripts em JavaScript puro e envie-os pelo parâmetro `script` da ferramenta `Workflow`.

---

## Sumário

1. Estrutura básica
2. Fan-out e verificação adversarial
3. Painel de avaliadores
4. Busca até esgotamento (*loop-until-dry*)
5. Repetições limitadas por orçamento
6. Agentes personalizados e saídas estruturadas
7. Armadilhas e checklist de verificação

---

## 1. Estrutura básica

Um script completo e executável começa com um objeto literal `meta`. O título passado para `phase()` deve corresponder **exatamente** ao `title` de `meta.phases`.

```javascript
export const meta = {
  name: 'domain-task',
  description: 'Descrição de uma linha exibida no diálogo de permissão',
  phases: [
    { title: 'coleta', detail: 'Pesquisa as diferentes perspectivas em paralelo' },
    { title: 'verificacao', detail: 'Busca contraevidências para cada achado' },
  ],
}

phase('coleta')
const raw = await pipeline(args.items, item =>
  agent(`...${item}...`, { label: `coleta:${item}`, phase: 'coleta', schema: COLLECT_SCHEMA }))

phase('verificacao')
// ...

return { result } // O retorno final será entregue ao agente principal.
```

**Princípios:**
- **Prefira `pipeline()`; use `parallel()` somente se precisar de uma barreira de sincronização.** A barreira faz sentido quando a fase seguinte precisa comparar o conjunto **completo** de resultados, como deduplicar, decidir parar quando o total de achados for zero ou comparar achados de investigadores diferentes.
- Não escreva valores concretos da lista de itens diretamente no script. Sempre que possível, faça reconhecimento antecipado e entregue a lista em `args.items`.
- Quando precisar de um horário, passe-o em `args.now`; não use `Date.now()` dentro do workflow.

## 2. Fan-out e verificação adversarial

Reúna achados de perspectivas independentes e verifique cada achado individualmente, sem aguardar a conclusão de todas as perspectivas.

```javascript
export const meta = {
  name: 'review-fanout-verify',
  description: 'Revê cada perspectiva e busca contraevidências para os achados',
  phases: [{ title: 'revisao' }, { title: 'verificacao' }],
}

const FINDINGS = { type: 'object', required: ['findings'], properties: {
  findings: { type: 'array', items: { type: 'object',
    required: ['title', 'file', 'evidence'], properties: {
      title: { type: 'string' }, file: { type: 'string' }, evidence: { type: 'string' } } } } } }
const VERDICT = { type: 'object', required: ['status', 'reason'], properties: {
  status: { type: 'string', enum: ['confirmed', 'refuted', 'uncertain'] },
  reason: { type: 'string' } } }

const results = await pipeline(
  args.dimensions, // Exemplo: [{key:'security', prompt:'...'}, {key:'perf', prompt:'...'}]
  d => agent(d.prompt, { label: `revisao:${d.key}`, phase: 'revisao', schema: FINDINGS }),
  review => parallel((review?.findings ?? []).map(f => () =>
    agent(`Verifique este achado de forma crítica. Responda confirmed se houver evidências suficientes, refuted se puder refutar e uncertain se as evidências não forem conclusivas: ${JSON.stringify(f)}`,
      { label: `verificacao:${f.file}`, phase: 'verificacao', schema: VERDICT })
      .then(v => ({ ...f, verdict: v }))))
)
const confirmed = results.flat().filter(Boolean)
  .filter(f => f.verdict?.status === 'confirmed')
log(`Foram confirmados ${confirmed.length} dos ${results.flat().filter(Boolean).length} achados analisados.`)
return { confirmed }
```

Para tornar o julgamento mais rigoroso, execute **três verificadores** para cada achado. Somente aprove o item se pelo menos dois retornarem `confirmed` com evidências suficientes. Atribuir critérios diferentes aos avaliadores, como precisão, segurança e reprodutibilidade, ajuda a identificar erros que passariam despercebidos em avaliações idênticas.

## 3. Painel de avaliadores

Use quando existirem muitas soluções possíveis de arquitetura ou design. Produza N versões independentes, submeta-as a avaliadores em paralelo e construa uma versão final.

```javascript
export const meta = {
  name: 'design-judge-panel',
  description: 'Gera alternativas independentes, avalia-as e consolida a melhor solução',
  phases: [{ title: 'propostas' }, { title: 'avaliacao' }, { title: 'consolidacao' }],
}

const angles = args.angles?.length
  ? args.angles
  : ['priorizando o MVP', 'priorizando a gestão de riscos', 'priorizando a experiência do usuário']
const judgeCount = args.judgeCount ?? 3

phase('propostas')
const drafts = (await parallel(angles.map(a => () =>
  agent(`Elabore uma proposta independente ${a}: ${args.brief}`,
    { label: `proposta:${a}`, phase: 'propostas' })))).filter(Boolean)
log(`Foram produzidas ${drafts.length} de ${angles.length} propostas.`)
if (!drafts.length) return { error: 'Nenhuma proposta foi concluída.', drafts: [] }

phase('avaliacao') // Barreira necessária: todas as alternativas serão comparadas.
const SCORE = { type: 'object', required: ['scores'], properties: {
  scores: { type: 'array', items: { type: 'object',
    required: ['index', 'score', 'strengths'], properties: {
      index: { type: 'integer' }, score: { type: 'number' }, strengths: { type: 'string' } } } } } }
const judged = (await parallel(Array.from({ length: judgeCount }, (_, j) => () =>
  agent(`Avalie estas ${drafts.length} propostas:\n${drafts.map((d, i) => `[${i}] ${d}`).join('\n---\n')}`,
    { label: `avaliador:${j}`, phase: 'avaliacao', schema: SCORE })))).filter(Boolean)
log(`Foram recebidas ${judged.length} de ${judgeCount} avaliações.`)
if (!judged.length) return { error: 'Nenhuma avaliação foi concluída.', drafts }

phase('consolidacao')
const ranked = drafts.map((_, i) => {
  const scores = judged.flatMap(r =>
    r.scores.filter(x => x.index === i).map(x => x.score)).filter(Number.isFinite)
  return scores.length
    ? { index: i, average: scores.reduce((sum, score) => sum + score, 0) / scores.length }
    : null
}).filter(Boolean)
if (!ranked.length) return { error: 'Nenhuma proposta recebeu notas válidas.', drafts, judged }
const winner = ranked.reduce((best, item) =>
  item.average > best.average ? item : best).index
const finalDraft = await agent(
  `Desenvolva a solução final a partir da proposta de maior pontuação, incorporando também os pontos positivos relevantes das demais.\nProposta vencedora:\n${drafts[winner]}\nAvaliações:\n${JSON.stringify(judged)}`,
  { phase: 'consolidacao' })
if (!finalDraft) {
  log('Não foi possível elaborar a versão final.')
  return { error: 'Não foi possível elaborar a versão final.', drafts, judged, winner }
}
return finalDraft
```

## 4. Busca até esgotamento (*loop-until-dry*)

Use quando não for possível estimar quantos itens existem. Continue investigando até que ocorram K rodadas consecutivas sem achados inéditos.

Os códigos das seções 4 a 6 são **fragmentos** para inserir no modelo completo da seção 1. Para uso independente, comece com `meta`, defina os títulos das fases e os schemas referenciados.

```javascript
const finders = args.finders ?? []
const dryLimit = args.dryRuns ?? 2
const findingKey = finding => JSON.stringify([
  finding.file ?? '', finding.title ?? '', finding.evidence ?? ''
])
const seen = new Set(), confirmed = []
if (!finders.length) return { confirmed, error: 'Nenhum critério de investigação foi fornecido.' }
let dry = 0
while (dry < dryLimit) {
  const found = (await parallel(finders.map(f => () =>
    agent(f.prompt, { phase: 'busca', schema: FINDINGS })))).filter(Boolean).flatMap(r => r.findings)
  const fresh = found.filter(b => !seen.has(findingKey(b))) // Use seen, não confirmed, para deduplicar.
  if (!fresh.length) { dry++; continue }
  dry = 0
  fresh.forEach(b => seen.add(findingKey(b)))
  const judged = await parallel(fresh.map(b => () =>
    agent(`Examine criticamente o achado. Responda confirmed se houver evidências suficientes, refuted se for falso ou uncertain se inconclusivo: ${JSON.stringify(b)}`,
      { phase: 'verificacao', schema: VERDICT })
      .then(v => ({ b, ok: v?.status === 'confirmed' }))))
  confirmed.push(...judged.filter(Boolean).filter(x => x.ok).map(x => x.b))
  log(`Há ${confirmed.length} achados confirmados; esta rodada encontrou ${fresh.length} novos itens.`)
}
return { confirmed }
```

Se a deduplicação usar `confirmed`, achados rejeitados poderão aparecer novamente a cada iteração e impedir o encerramento. Deduplicate com `seen`, que registra todo item já encontrado, inclusive os refutados.

## 5. Repetir conforme o orçamento de tokens

Quando o usuário definir orçamento como `+500k`, ajuste a exploração de acordo com o valor restante. Em sessões sem limite explícito, `budget.remaining()` pode retornar `Infinity`; por isso, verifique `budget.total` **antes** de executar o loop.

```javascript
const findings = []
while (budget.total && budget.remaining() > 50_000) {
  const r = await agent('Prossiga com a próxima tarefa de investigação.', { schema: FINDINGS })
  if (r) findings.push(...r.findings)
  log(`Encontrados ${findings.length} itens; restam ${Math.round(budget.remaining() / 1000)} mil tokens no orçamento.`)
}
// Alternativa de dimensionamento prévio: const FLEET = budget.total ? Math.floor(budget.total / 100_000) : 3
```

## 6. Agentes personalizados com saída estruturada

Utilize diretamente em workflows as definições em `.claude/agents/{name}.md` geradas pelo Harness.

```javascript
const r = await agent(
  'Verifique o arquivo _workspace/02_draft.md e devolva os achados.',
  { agentType: 'qa-inspector',    // definição em .claude/agents/qa-inspector.md
    schema: FINDINGS,            // saída deve obedecer ao schema, também para tipos personalizados
    effort: 'high' })            // revisões podem justificar maior esforço de raciocínio
```

- Se `agentType` for omitido, o workflow utiliza o subagente padrão.
- Escolha `model` conforme a etapa: `sonnet` para coleta e transformação rotineiras, `opus` para análise profunda e verificação, `fable` para planejamento e execução autônoma de longo prazo. Consulte `model-selection-guide.md`.
- Para edições paralelas com risco de conflito, use `isolation: 'worktree'`. Como o isolamento tem custo, evite-o quando não houver risco real.

## 7. Armadilhas e checklist de verificação

Na fase 6.2, valide cada script `Workflow` contra esta tabela.

| Armadilha | Consequência | Correção |
|---|---|---|
| Variáveis ou expressões em `meta` | Falha de parsing. | Use somente literais. |
| Sintaxe TypeScript (como `: string[]`) | Falha de parsing. | Use JavaScript puro. |
| `Date.now()`, `Math.random()` ou `new Date()` | Falhas por restrições de retomada. | Passe timestamps e seeds em `args`; para diversidade, use variações determinísticas por índice. |
| Ausência de `.filter(Boolean)` | Resultados `null` de agentes com falha interrompem fases seguintes. | Filtre retornos antes de processá-los. |
| Barreiras desnecessárias | Agentes rápidos aguardam os mais lentos, aumentando duração. | Use `pipeline()` quando não precisar comparar o conjunto completo. |
| Títulos de `phase` divergentes | Progresso agrupado incorretamente. | Faça `phase()` coincidir com `meta.phases.title`. |
| `phase()` global dentro de etapas paralelas | Conflitos na exibição de progresso. | Indique `opts.phase` nas chamadas. |
| Deduplicação por `confirmed` | Itens refutados ressurgem e o loop não termina. | Deduplicate por `seen`. |
| Loop por orçamento sem testar `budget.total` | Sessões ilimitadas continuam até atingir o teto de agentes. | Use `while (budget.total && ...)`. |
| Limitar amostras sem informar excluídos | Relatar conclusão integral de trabalho parcial. | Mostre em `log()` quantos itens não foram processados. |
| Diagnosticar sem examinar o `journal` | Resultado vazio obtido de cache pode parecer êxito. | Consulte o `journal` no diretório `transcript` antes de concluir a análise. |
| Criar manualmente o arquivo do script antes de executar | Etapa redundante. | Passe o código diretamente em `script`. O script é salvo automaticamente; depois, use `Edit` e `scriptPath` para revisões. |
