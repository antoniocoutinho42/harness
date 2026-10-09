# Guia de projeto de agentes de QA

Use este guia ao incluir um agente de garantia de qualidade (QA) no harness. Ele reúne falhas observadas em projetos reais e explica como identificar defeitos que passam despercebidos em verificações superficiais.

---

## Sumário

1. Padrões de defeitos ignorados pelo QA
2. Verificação de consistência nas integrações
3. Princípios para projetar agentes de QA
4. QA com workflows (v2)
5. Modelo de checklist de verificação
6. Modelo de definição de agente de QA

---

## 1. Padrões de defeitos ignorados pelo QA

### 1.1. Contratos incompatíveis entre componentes (*boundary mismatch*)

É uma das falhas mais frequentes: cada componente parece correto isoladamente, mas os dois lados de uma integração seguem contratos diferentes.

| Integração | Exemplo de incompatibilidade | Por que passa despercebida |
|---|---|---|
| Resposta da API → hook do frontend | A API retorna `{ projects: [...] }` e o hook espera `Project[]`. | Componentes são testados isoladamente, sem confronto de contratos. |
| Campo da API → definição de tipo | A API usa `thumbnailUrl` (camelCase) e o tipo usa `thumbnail_url` (snake_case). | Conversões ou asserções genéricas de tipo mascaram a divergência. |
| Caminho de arquivo → link `href` | A página fica em `/dashboard/create`, mas o link aponta para `/create`. | A estrutura de rotas não é comparada aos links. |
| Mapa de transições → atualização de `status` | O mapa define a transição, mas o código não atualiza o estado. | Verifica-se a existência do mapa, não sua implementação. |
| Endpoint da API → hook | O endpoint existe, mas nenhum hook o utiliza. | Não se comparam as listas de endpoints e de chamadas. |
| Resposta imediata → resultado assíncrono | A API retorna `{ status }` imediatamente, mas o frontend acessa campos do resultado final. | Confunde-se resposta inicial com resultado posterior. |

### 1.2. Por que inspeção estática não basta

- **Limites dos tipos genéricos:** `fetchJson<Project[]>()` pode compilar mesmo que a API retorne `{ projects: [...] }`.
- **Build bem-sucedido não garante execução correta:** `any`, asserções de tipo e uso indevido de genéricos podem ocultar erros até o runtime.
- **Existência e compatibilidade são verificações distintas:** confirmar que um endpoint existe não demonstra que sua resposta atende ao contrato esperado pelo cliente.

## 2. Verificar a consistência das integrações (*integration coherence verification*)

O QA deve **comparar os dois lados de cada integração**. Os exemplos usam Next.js, mas o princípio é independente da stack. Abra conjuntamente o código produtor e o consumidor.

### 2.1. Confrontar respostas de API e tipos dos hooks

```text
1. Localize onde a rota da API serializa a resposta (por exemplo, NextResponse.json()) e identifique a estrutura retornada.
2. Confira o tipo esperado pelo hook ou cliente (por exemplo, T em fetchJson<T>).
3. Compare a estrutura real com T. Se a API devolver { data: [...] }, confirme que o hook acessa .data.
```

Dê atenção a respostas paginadas incorretamente tratadas como arrays, conversões ausentes entre snake_case e camelCase e diferenças entre resposta imediata (202) e resposta final.

### 2.2. Confrontar arquivos, links e roteamento

```text
1. Extraia os padrões de URL da estrutura de arquivos de rotas. Retire (group) e trate [param] como segmento dinâmico.
2. Reúna os valores de href=, router.push( e redirect( no código.
3. Verifique se cada link corresponde a uma rota realmente existente.
```

### 2.3. Percorrer as transições de estado

```text
1. Extraia todas as transições permitidas pelo mapa de estados.
2. Localize o código que atualiza status.
3. Verifique se toda transição executada é permitida pelo mapa.
4. Identifique transições definidas, mas nunca executadas.
5. Procure especialmente a ausência de transições de estados intermediários para finais.
```

### 2.4. Conferir endpoints e hooks individualmente

```text
1. Liste endpoints por método HTTP.
2. Liste as URLs utilizadas em chamadas fetch do cliente.
3. Para endpoints sem consumidor, determine se são APIs administrativas intencionalmente independentes ou integrações que ficaram sem implementação.
```

## 3. Princípios para agentes de QA

### 3.1. Escolha um agente com ferramentas suficientes

O tipo `Explore` permite leitura, mas não executa scripts de verificação. Um agente de QA precisa poder procurar padrões com `Grep`, executar scripts e comparar resultados automaticamente; dependendo do escopo, também pode precisar alterar código. Use `general-purpose` ou um tipo personalizado com permissões adequadas, definindo o fluxo “verificar → relatar → solicitar correção”.

### 3.2. Priorize comparação de contratos, não apenas existência

| Checklist superficial | Checklist que realmente valida |
|---|---|
| O endpoint existe? | Sua resposta é compatível com o tipo do hook consumidor? |
| Há um mapa de transições? | Todas as atualizações de `status` respeitam o mapa? |
| A página existe? | Todos os links apontam para rotas válidas? |
| O modo estrito está ativado? | Alguma asserção genérica contorna a segurança de tipos? |

### 3.3. Leia juntos os componentes conectados

Para detectar erros de integração, não basta ler um lado: compare rotas de API e hooks, mapas de estados e atualizações de status, estrutura de rotas e links. Registre essa exigência na definição do agente.

### 3.4. Teste a cada módulo concluído

Não concentre o QA no fim da implementação. Isso acumula erros e permite que contratos incompatíveis contaminem módulos seguintes. Ao concluir cada endpoint, verifique-o imediatamente com seu consumidor: **QA incremental**.

## 4. QA executado por workflow (v2)

Se as integrações a verificar forem enumeráveis, organize a análise como workflow:

```javascript
// Inspecionar integrações separadamente e verificar adversarialmente cada achado.
const FINDINGS = { type: 'object', required: ['findings'], properties: {
  findings: { type: 'array', items: { type: 'object',
    required: ['title', 'file', 'evidence'], properties: {
      title: { type: 'string' }, file: { type: 'string' }, evidence: { type: 'string' } } } } } }
const VERDICT = { type: 'object', required: ['status', 'reason'], properties: {
  status: { type: 'string', enum: ['confirmed', 'refuted', 'uncertain'] },
  reason: { type: 'string' } } }
const boundaries = args.boundaries // Integrações previamente identificadas: [{api: '...', consumer: '...'}, ...]
const found = await pipeline(
  boundaries,
  b => agent(`Leia os dois lados e encontre contratos incompatíveis: ${b.api} ↔ ${b.consumer}`,
    { agentType: 'qa-inspector', phase: 'inspecao', schema: FINDINGS }),
  r => parallel((r?.findings ?? []).map(f => () =>
    agent(`Verifique criticamente se a incompatibilidade realmente causa erro: ${JSON.stringify(f)}`,
      { phase: 'verificacao', schema: VERDICT }).then(verdict => ({ ...f, verdict }))))
)
const confirmed = found.flat().filter(Boolean)
  .filter(f => f.verdict?.status === 'confirmed')
return { confirmed }
```

A verificação adversarial elimina achados que parecem problemas, mas não têm impacto real, deixando somente os itens com status `confirmed` no relatório final.

## 5. Modelo de checklist de verificação

Inclua o checklist abaixo na definição do agente de QA para aplicações web.

```markdown
### Consistência das integrações (aplicação web)

#### Integração entre API e frontend
- [ ] A estrutura das respostas das APIs é compatível com os tipos genéricos dos hooks.
- [ ] Os hooks extraem o conteúdo de respostas encapsuladas em objetos, como { items: [...] }.
- [ ] Conversões entre snake_case e camelCase são coerentes.
- [ ] O frontend distingue respostas imediatas (202) de resultados finais.
- [ ] Os endpoints necessários têm consumidores e são efetivamente chamados.

#### Coerência do roteamento
- [ ] Todos os valores de href/router.push apontam para páginas existentes.
- [ ] Grupos de rotas (group) são removidos da URL conforme a convenção do framework.
- [ ] Segmentos dinâmicos, como [id], recebem parâmetros válidos.

#### Consistência da máquina de estados
- [ ] Todas as transições definidas são implementadas e alcançáveis.
- [ ] Toda atualização de status corresponde a uma transição permitida.
- [ ] Existem transições corretas de estados intermediários para estados finais.
- [ ] Os valores X de condições como if (status === "X") são estados alcançáveis.

#### Consistência do fluxo de dados
- [ ] Campos do banco e da resposta da API possuem mapeamento consistente.
- [ ] Os tipos do frontend correspondem aos campos reais da API.
- [ ] Campos opcionais tratam null e undefined de modo coerente.
```

## 6. Modelo de definição de agente de QA

```markdown
---
name: qa-inspector
description: "Agente de QA que verifica conformidade com requisitos, consistência das integrações e qualidade visual."
---

# Agente de QA

## Papel principal
Verificar a implementação e, principalmente, a **consistência dos contratos entre módulos**.

## Prioridades
1. **Integração e contratos** — incompatibilidades entre componentes são uma causa frequente de erros em runtime.
2. **Conformidade funcional** — validar APIs, máquinas de estados e modelos de dados.
3. **Qualidade visual** — cores, tipografia e responsividade.
4. **Qualidade do código** — código morto e convenções de nomenclatura.

## Método: ler os dois lados de cada integração
| Verificação | Produtor | Consumidor |
|---|---|---|
| Estrutura da API | Ponto de serialização da resposta | Tipo esperado pelo cliente |
| Roteamento | Arquivos de rotas | href e router.push |
| Máquina de estados | Mapa de transições | Código que atualiza status |
| DB → API → UI | Colunas do banco | Resposta da API e tipos de frontend |

## Como relatar e pedir correções
- Ao identificar uma falha, envie ao responsável um pedido específico de correção com arquivo, linha e abordagem sugerida.
- Em problemas de integração, notifique os responsáveis pelos dois lados.
- Informe ao líder separadamente as verificações aprovadas, reprovadas e não realizadas.
```

---

## Casos reais: falhas e causas

| Falha | Integração | Causa |
|---|---|---|
| `projects?.filter is not a function` | API → hook | A API retorna `{projects:[]}`, mas o hook espera um array. |
| Todos os links do painel retornam 404 | Rota → `href` | Falta o prefixo `/dashboard/`. |
| Miniaturas não aparecem | API → UI | Há incompatibilidade entre `thumbnailUrl` e `thumbnail_url`. |
| O valor escolhido não é salvo | API → hook | A API existe, mas não há hook correspondente. |
| A página de criação permanece aguardando | Transição → código | Falta a atualização para o estado final. |
| Erro ao acessar `data.failedIndices` | Resposta imediata → frontend | O frontend acessa dados do processamento posterior já na resposta inicial. |
| Página de detalhes retorna 404 após conclusão | Rota → `href` | Prefixos de caminhos incompatíveis. |
