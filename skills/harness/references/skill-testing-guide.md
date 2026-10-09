# Guia de testes e aprimoramento iterativo de skills

Complementa a fase 6 de `SKILL.md`. Descreve como verificar a qualidade de skills geradas pelo Harness e aprimorá-las de forma sistemática.

---

## Sumário

1. Estratégias de avaliação
2. Criar prompts de teste
3. Comparar execuções com e sem a skill
4. Executar testes A/B em workflows (v2)
5. Avaliar quantitativamente por assertions
6. Utilizar agentes especializados
7. Ciclo de aprimoramento
8. Verificar acionamento pela description
9. Estrutura de diretórios para testes

---

## 1. Estratégias de avaliação

Combine **avaliação qualitativa por pessoas** e **avaliação quantitativa por critérios verificáveis**.

| Tipo | Como avaliar | Skills adequadas |
|---|---|---|
| **Qualitativa** | Usuário examina diretamente o resultado. | Estilo, design, criação e outros trabalhos que exigem julgamento humano. |
| **Quantitativa** | Critérios objetivos são verificados automaticamente. | Criação de arquivos, extração de dados e programação, entre outros. |

Adote o ciclo **redigir → executar testes → avaliar → aprimorar → testar novamente**.

## 2. Criar prompts de teste

### Princípios

Escreva prompts **naturais, específicos e plausíveis**, semelhantes aos que usuários reais fariam. Solicitações abstratas ou artificiais não representam o comportamento esperado em produção.

**Ruins:** `"Processe um PDF"`, `"Extraia dados"`.

**Bom exemplo:**

```text
“Na planilha 'Receitas_4T_final_v2.xlsx' da pasta Downloads, use as colunas
C (receita) e D (custo) para criar uma coluna de margem (%).
Depois, ordene as linhas pela margem em ordem decrescente.”
```

### Variar a forma dos pedidos

- Combine linguagem formal e informal.
- Inclua pedidos explícitos e outros cujo objetivo seja inferido do contexto.
- Misture tarefas simples e complexas; alguns prompts devem conter abreviações, pequenos erros de digitação e expressões cotidianas.

### Abrangência mínima

Comece com dois ou três prompts: um para o cenário principal, outro para uma exceção e, se necessário, um terceiro que reúna várias tarefas.

## 3. Comparar execuções com e sem a skill

### 3.1. Como comparar

Para cada prompt, inicie **simultaneamente** dois subagentes na mesma mensagem:

- **Com a skill (*with-skill*):** leia a skill, execute a tarefa e salve em `_workspace/iteration-N/eval-{id}/with_skill/outputs/`.
- **Referência (*baseline*):** faça a mesma tarefa sem a skill e salve em `_workspace/iteration-N/eval-{id}/without_skill/outputs/`.

### 3.2. Definir o baseline

| Situação | Comparação adequada |
|---|---|
| Skill nova | Execução do mesmo pedido sem a skill. |
| Melhoria de skill existente | Execução com uma cópia da versão anterior. |

### 3.3. Registrar tempo e tokens

Salve `total_tokens` e `duration_ms` **assim que receber a notificação de conclusão**. Essas informações podem não estar disponíveis novamente depois.

## 4. Executar testes A/B com workflow (v2)

Se houver três ou mais casos de teste ou várias repetições, implemente o teste A/B como script de workflow. Execute somente se o usuário autorizar os testes.

```javascript
export const meta = {
  name: 'skill-ab-test',
  description: 'Executa as versões com e sem a skill e avalia os resultados às cegas',
  phases: [{ title: 'execucao' }, { title: 'avaliacao' }],
}
const RUN_RESULT = { type: 'object', required: ['saved', 'files'], properties: {
  saved: { type: 'boolean' },
  files: { type: 'array', minItems: 2, items: { type: 'string', minLength: 1 } } } }
const SLOT_GRADE = { type: 'object', required: ['expectations', 'summary'], properties: {
  expectations: { type: 'array', items: { type: 'object',
    required: ['text', 'passed', 'evidence'], properties: {
      text: { type: 'string' }, passed: { type: 'boolean' }, evidence: { type: 'string' } } } },
  summary: { type: 'object', required: ['passed', 'failed', 'total', 'pass_rate'], properties: {
    passed: { type: 'integer' }, failed: { type: 'integer' }, total: { type: 'integer' },
    pass_rate: { type: 'number' } } } } }
const GRADE = { type: 'object', required: ['A', 'B', 'comparison'], properties: {
  A: SLOT_GRADE,
  B: SLOT_GRADE,
  comparison: { type: 'object', required: ['preferred', 'reason'], properties: {
    preferred: { type: 'string', enum: ['A', 'B', 'tie'] },
    reason: { type: 'string' } } } } }

const results = await pipeline(
  args.evals, // [{id, prompt, assertions, skillPath, withSkillSlot: 'A' | 'B'}]
  e => {
    const withSkillSlot = e.withSkillSlot === 'B' ? 'B' : 'A'
    const baselineSlot = withSkillSlot === 'A' ? 'B' : 'A'
    return parallel([
      () => agent(
        `${e.prompt}\n\nLeia e siga primeiro ${e.skillPath}. Salve o resultado em ${args.ws}/${e.id}/with_skill/outputs/ e também em ${args.ws}/${e.id}/blind/${withSkillSlot}/outputs/. Retorne todos os caminhos criados.`,
        { label: `com-skill:${e.id}`, phase: 'execucao', schema: RUN_RESULT }),
      () => agent(
        `${e.prompt}\n\nSalve o resultado em ${args.ws}/${e.id}/without_skill/outputs/ e também em ${args.ws}/${e.id}/blind/${baselineSlot}/outputs/. Retorne todos os caminhos criados.`,
        { label: `baseline:${e.id}`, phase: 'execucao', schema: RUN_RESULT }),
    ])
  },
  (runs, e) => {
    const completed = (runs ?? []).filter(Boolean)
    if (completed.length !== 2 || completed.some(r => !r.saved || r.files.length < 2)) {
      log(`${e.id}: faltam resultados de uma ou ambas as execuções; avaliação ignorada.`)
      return null
    }
    return agent(
      `Confirme que existem resultados em ${args.ws}/${e.id}/blind/A/outputs/ e ${args.ws}/${e.id}/blind/B/outputs/. Se algum estiver vazio, interrompa a avaliação e explique. Se ambos existirem, avalie A e B separadamente e escolha A, B ou tie. Para preservar o cegamento, NÃO consulte eval_metadata.json, with_skill/ ou without_skill/. Critérios: ${JSON.stringify(e.assertions)}`,
      { label: `avaliacao:${e.id}`, phase: 'avaliacao', schema: GRADE })
      .then(grade => {
        if (!grade) log(`${e.id}: avaliação às cegas não recebida; caso ignorado.`)
        return grade
      })
  }
)
const mapped = results.map((grade, index) => {
  if (!grade) return null
  const e = args.evals[index]
  const withSkillSlot = e.withSkillSlot === 'B' ? 'B' : 'A'
  const baselineSlot = withSkillSlot === 'A' ? 'B' : 'A'
  return {
    evalId: e.id,
    withSkill: grade[withSkillSlot],
    baseline: grade[baselineSlot],
    comparison: {
      preferred: grade.comparison.preferred === 'tie'
        ? 'tie'
        : grade.comparison.preferred === withSkillSlot ? 'with_skill' : 'baseline',
      reason: grade.comparison.reason,
    },
  }
}).filter(Boolean)
return { results: mapped }
```

Alterne `withSkillSlot` entre A e B nos casos de teste. O avaliador só deve consultar diretórios anônimos `blind/A/` e `blind/B/`, sem saber a qual execução a skill foi aplicada. Avalie somente quando os dois resultados estiverem disponíveis. Depois, o agente principal associa as notas às execuções correspondentes e salva o resultado em `grading.json`. Um caso pode ser avaliado assim que suas duas execuções terminarem, sem aguardar os demais. Se a skill for alterada, use `resumeFromRunId` para reaproveitar chamadas que não mudaram.

## 5. Avaliação quantitativa por assertions

### 5.1. Boas condições de verificação

Uma *assertion* deve produzir um resultado objetivo (verdadeiro/falso), explicar claramente o que verifica e medir algo que a skill pretende melhorar. Evite condições triviais, como “a saída existe”, e subjetivas, como “o texto está bem escrito”.

### 5.2. Verificações automatizáveis

Quando a condição puder ser conferida por código, escreva um script e reutilize-o entre iterações. Isso é mais rápido e reproduzível que inspeções manuais.

### 5.3. Critérios sem poder de discriminação

Se a execução com a skill e o baseline alcançarem 100% no mesmo critério, ele não revela benefício da skill. Substitua-o por uma condição suficientemente exigente ou remova-o.

### 5.4. Schema da avaliação

Adote a estrutura `grading.json` definida em `skill-writing-guide.md`, incluindo `text`, `passed`, `evidence` e `summary`.

## 6. Agentes especializados

| Papel | Responsabilidade | Quando utilizar |
|---|---|---|
| **Avaliador (*Grader*)** | Registra aprovações, reprovações e evidências para cada critério; confronta afirmações factuais e critica a qualidade dos critérios. | Em cada iteração. |
| **Comparador cego (*Comparator*)** | Compara os resultados sem saber qual deles recebeu a skill, alternando a ordem A/B. | Ao medir se uma versão realmente supera a anterior. |
| **Analista (*Analyzer*)** | Identifica critérios pouco discriminatórios, dispersão de resultados e relações entre tempo e tokens. | Após pelo menos três rodadas. |

## 7. Ciclo de aprimoramento

### 7.1. Princípios

1. **Generalize o feedback.** Corrija princípios aplicáveis a vários casos, em vez de ajustar a skill a um único teste.
2. **Remova instruções inúteis.** Se o registro do agente revelar trabalho desnecessário causado pela skill, elimine a instrução.
3. **Explique o motivo.** Identifique a causa da falha e reflita-a na orientação revisada.
4. **Prepare ferramentas recorrentes.** Se todos os testes recriarem o mesmo script, disponibilize-o em `scripts/`.

### 7.2. Sequência

```text
1. Alterar a skill.
2. Reexecutar todos os testes em uma nova pasta iteration-{N+1}/.
3. Comparar os resultados com a iteração anterior e apresentá-los ao usuário.
4. Incorporar feedback e repetir.
```

**Quando parar:** quando o usuário estiver satisfeito, não houver feedback a incorporar ou o ganho incremental deixar de justificar novas revisões.

### 7.3. Revisar a partir de outra perspectiva

Crie primeiro um rascunho da alteração. Depois, releia-o como um revisor que nunca viu a skill. Não tente tornar o primeiro texto perfeito.

## 8. Verificar as condições de acionamento da description

### 8.1. Construir o conjunto de avaliação

Crie 20 prompts: dez casos `should-trigger` e dez `should-NOT-trigger`.

**Critérios:**
- Linguagem concreta e natural, semelhante à de usuários reais.
- Detalhes como caminhos de arquivo, contexto do usuário, nomes de colunas e empresas.
- Ênfase em **casos limítrofes**, nos quais a escolha da skill seja menos óbvia.

**Casos positivos (`should-trigger`):** intenções equivalentes com frases variadas, pedidos sem formato explicitado, usos incomuns e solicitações que poderiam acionar outra skill, mas pertencem a esta.

**Casos negativos (`should-NOT-trigger`):** pedidos parecidos que exigem outra ferramenta ou skill (*near misses*). Frases sem relação alguma não são testes úteis.

### 8.2. Detectar conflitos com skills existentes

1. Reúna todas as `description` das skills instaladas.
2. Verifique se os casos positivos da nova skill acionam incorretamente uma skill existente.
3. Se houver conflito, explicite na `description` os critérios de inclusão e exclusão.

### 8.3. Otimizar automaticamente (opcional e avançado)

1. Divida os 20 prompts em 60% para treinamento e 40% para teste.
2. Meça a precisão de acionamento com a `description` atual.
3. Examine falhas e revise a `description`.
4. Escolha a melhor versão **pela precisão no conjunto de teste**, não no de treinamento; caso contrário, há risco de sobreajuste.
5. Repita no máximo cinco vezes.

> Execute a avaliação por scripts em modo sem interface (`claude -p`). Como pode consumir muitos tokens, deixe essa avaliação para quando a skill estiver razoavelmente estável.

## 9. Diretórios para testes

```text
_workspace/
├── iteration-1/
│   ├── eval-nome-descritivo/
│   │   ├── eval_metadata.json
│   │   ├── with_skill/     (outputs/ + timing.json + grading.json)
│   │   ├── without_skill/ (outputs/ + timing.json + grading.json)
│   │   └── blind/         (A/outputs/ + B/outputs/)
│   └── benchmark.json
├── iteration-2/
└── evals/evals.json
```

**Preservação:**
- Dê nomes descritivos aos casos, como `eval-multi-page-table-extraction`, não apenas números.
- Use pastas independentes por iteração; nunca sobrescreva um diretório anterior.
- Preserve `_workspace/` para auditoria posterior e rastreabilidade das mudanças.
