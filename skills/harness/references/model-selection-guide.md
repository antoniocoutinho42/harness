# Guia de escolha dos modelos Claude

Escolha o modelo de cada agente considerando **complexidade, duração prevista, autonomia necessária e latência**. Configure-o no `model:` do frontmatter YAML do agente, no parâmetro `model` de `Agent` ou em `opts.model` nas chamadas `agent()` de `Workflow`. Registre a justificativa na definição do agente ou no orquestrador.

## Tabela de decisão

| Modelo | Critério | Exemplos |
|---|---|---|
| **fable** | Trabalho longo e autônomo, que exige transformar um objetivo complexo em um plano de várias etapas e entregar um resultado final | Raciocínio de alta complexidade, coordenação de agentes, planejamento e execução prolongada, trabalho criativo aberto |
| **opus** | Problema técnico ou especializado que exige análise profunda e conclusões fundamentadas | Arquitetura, programação, análises complexas, verificação cruzada e criação |
| **sonnet** | Modelo versátil para a maioria das tarefas rotineiras de redação, programação, análise e pesquisa | Redação e programação comuns, tratamento de logs, conversão de formatos, inspeção estática, deploy e coleta simples |

## 1. Fable

**Papel principal:** realizar trabalhos muito difíceis que requerem planejamento autônomo e continuidade ao longo de diversas etapas. Em vez de responder apenas à pergunta, o modelo deve compreender o objetivo, definir um plano e conduzi-lo até uma entrega final.

### Trabalhos adequados

**① Projetos longos, compostos por etapas interdependentes.** Use quando uma única resposta não for suficiente:
- Pesquisa de mercado → análise de concorrentes → estratégia → relatório.
- Levantamento de requisitos → arquitetura de serviço → plano de desenvolvimento → implementação.
- Revisão de vários documentos → conclusão consolidada → plano de ação.
- Planejamento do ciclo completo de um projeto prolongado.

O ponto decisivo é que **resultados anteriores influenciam as decisões seguintes**.

**② Análise de grande volume de material complexo.** Adequado a conjuntos extensos de artigos, relatórios, atas e documentos técnicos quando se espera mais que um resumo:
- Identificar convergências e divergências.
- Verificar afirmações contraditórias.
- Distinguir evidência central de informação acessória.
- Integrar diversas fontes em um resultado coerente.

**③ Transformar ideias vagas em entregas concretas.** Quando requisitos ainda não foram definidos e o modelo precisa resolver lacunas:
- Desenvolver uma ideia preliminar de serviço em um plano de negócios.
- Criar estrutura e conteúdo de uma apresentação a partir de um tema geral.
- Converter uma ideia incompleta de produto em funcionalidades e plano de desenvolvimento.
- Produzir um relatório completo sem rascunho inicial.

Escolha quando importa mais **tomar decisões sobre o que ainda não foi especificado** do que seguir instruções fechadas.

**④ Problemas que combinam raciocínio profundo, planejamento, coordenação e execução longa.** Fable não é necessariamente superior a Opus em profundidade de raciocínio isolado. A vantagem proposta está em combinar análise com planejamento autônomo e coordenação de várias etapas.

### Quando optar por Fable

- O resultado exige várias etapas encadeadas.
- O usuário não consegue especificar todas as decisões antecipadamente.
- O modelo precisa determinar autonomamente a ordem das tarefas.
- A entrega deve integrar um volume grande de informações.
- O objetivo é concluir um projeto, não apenas responder a uma pergunta.

## 2. Opus

**Papel principal:** analisar problemas difíceis com profundidade e rigor. Enquanto Fable é adequado a execução autônoma prolongada, Opus é indicado quando o problema é **bem delimitado**, mas exige raciocínio especializado.

### Trabalhos adequados

**① Pesquisa e análise complexas.** Não basta encontrar informações; é preciso ponderar evidências e construir conclusões:
- Análise profunda de mercados ou setores complexos.
- Comparação de pesquisas e estudos.
- Avaliação de benefícios e riscos de políticas ou estratégias empresariais.
- Identificação de relações de causa e efeito em documentos e dados.
- Síntese de perspectivas conflitantes.

**② Leitura de documentação técnica extensa.** Adequado a especificações de software, arquitetura de sistemas, artigos científicos, padrões técnicos, requisitos de produto e documentação complexa de APIs. O propósito principal é **compreender o significado e a estrutura lógica**, não simplesmente encurtar textos.

**③ Avaliação metodológica de pesquisas.** Examine o processo que gerou as conclusões:
- Adequação do desenho do estudo, tamanho e possíveis vieses da amostra.
- Consistência das medidas e inferências estatísticas.
- Relação entre evidências e conclusões, inclusive interpretações alternativas.

**④ Programação com múltiplas etapas.** Para muito além de escrever trechos isolados:
- Entender o código existente e localizar causas de falhas.
- Modificar diversos arquivos, implementar funcionalidades e escrever testes.
- Melhorar arquitetura e decidir passos técnicos subsequentes.

### Quando optar por Opus

- O problema é difícil e especializado, mas tem **escopo relativamente claro**.
- É necessária análise lógica ou revisão aprofundada.
- Documentos técnicos e artigos precisam ser compreendidos precisamente.
- É necessário encontrar falhas em estudos ou argumentos.
- O trabalho envolve desafios complexos de programação.

## 3. Sonnet

**Papel principal:** atender à maioria das necessidades cotidianas de modo equilibrado, com bom compromisso entre velocidade e qualidade.

### Trabalhos adequados

**① Redação e conteúdo:** e-mails, artigos, rascunhos de relatórios, textos publicitários, edição, apresentações, redes sociais e brainstorming.

**② Programação comum:** funções, correção de erros, programas simples, explicação de código, refatorações, testes e scripts de processamento de dados.

**③ Análise e pesquisa:** visões gerais, comparação de produtos e serviços, leitura de documentos, estudos simples da concorrência, extração de argumentos e apresentação estruturada de resultados.

**④ Tarefas com várias etapas bem definidas:** resumir fontes e produzir uma apresentação, analisar requisitos e criar exemplos de código, tratar dados e explicar os resultados. Diferencie isso de projetos longos, abertos e autônomos, adequados a Fable.

**⑤ Rotinas operacionais:** análise de logs, conversão de formatos, inspeções estáticas, scripts de deploy e coleta simples.

### Quando optar por Sonnet

- Quando não houver uma razão clara para outro modelo, use sonnet como padrão.
- Quando for importante concluir atividades comuns com rapidez e estabilidade.
- Para combinar redação, programação e pesquisa em tarefas rotineiras.
- Para dificuldades intermediárias, sem necessidade de raciocínio excepcional.
- Quando velocidade e qualidade tiverem importância semelhante.

## Diferença entre Fable e Opus

- **Fable:** planeja o trabalho inteiro e o executa autonomamente ao longo do tempo.
- **Opus:** examina profundamente um problema complexo com limites definidos.

Criticar em profundidade um artigo específico tende a favorecer **Opus**. Revisar dezenas de artigos, desenvolver uma estratégia de pesquisa e produzir um relatório final pode favorecer **Fable**. Se o caminho muda conforme os resultados intermediários, considere Fable; se o desafio é principalmente a análise aprofundada de um problema delimitado, considere Opus.

## Regras para aplicação em harnesses

1. **Escolha pela tarefa, não pelo status do agente.** Avalie complexidade, duração, autonomia e latência; não selecione um modelo mais caro somente porque o agente tem papel importante.
2. **Evite uma configuração geral sem justificativa.** Atribuir fable ou opus a todos os agentes pode aumentar custos desnecessariamente. Documente o motivo da escolha de cada um.
3. **Separe coordenação de execução.** Mesmo que a sessão principal ou o supervisor use fable, selecione para os agentes executores os modelos apropriados a suas tarefas.
4. **Escolha por etapa do workflow.** Em `Workflow`, `opts.model` permite variar o modelo: sonnet para coleta e transformação repetitivas, opus para verificação, avaliação e arquitetura complexas, fable para planejamento e execução autônoma prolongada.
