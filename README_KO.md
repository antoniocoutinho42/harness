<!-- Nome legado mantido para não quebrar links externos que apontam para README_KO.md. A documentação passou a ser mantida em português brasileiro. -->

<p align="center">
  <img src="https://img.shields.io/badge/Versao-2.1.0-brightgreen.svg" alt="Versão">
  <a href="LICENSE"><img src="https://img.shields.io/badge/Licenca-Apache_2.0-blue.svg" alt="Licença"></a>
  <img src="https://img.shields.io/badge/Claude_Code-Plugin-purple.svg" alt="Plugin Claude Code">
  <img src="https://img.shields.io/badge/Modos_de_Execucao-3-teal.svg" alt="3 modos de execução">
  <img src="https://img.shields.io/badge/Padrões-6+Qualidade-orange.svg" alt="Padrões">
</p>

<p align="center">
  <a href="https://revfactory.github.io/harness-animation/">
    <img src="harness_reel_en.gif" alt="Harness em 15 segundos: de um pedido a uma equipe de agentes" width="800">
  </a>
  <br>
  <sub>Animação de 15 segundos · <a href="https://revfactory.github.io/harness-animation/">Veja a apresentação interativa completa de 2 minutos (em coreano) →</a></sub>
</p>

# Harness v2 — Fábrica de arquiteturas de equipes para Claude Code

**Português brasileiro** · Tradução e adaptação da documentação do projeto [revfactory/harness](https://github.com/revfactory/harness).

> **O Harness constrói arquiteturas de equipes para Claude Code.** Com um único pedido — **“Crie um harness para este projeto”** — o plugin transforma a descrição do seu domínio em uma equipe de agentes e nas skills que eles utilizarão.

## Novidades da v2

A v2 foi reconstruída para o runtime multiagente atual do Claude Code:

- **Três modos nativos de execução.** A v1 usava a API experimental `TeamCreate`, que deixou de existir. A v2 usa os recursos disponíveis no runtime atual:
  1. **Orquestração por workflows:** scripts determinísticos (`pipeline()`, `parallel()`, schemas e orçamentos) para atividades em paralelo, ciclos de verificação e execuções em grande escala.
  2. **Colaboração entre agentes persistentes:** agentes identificados por nome, `SendMessage` e listas de tarefas compartilhadas; o contexto é mantido entre interações.
  3. **Delegação a subagentes:** chamadas pontuais e leves, inclusive em paralelo.
- **Padrões de qualidade integrados aos workflows.** Verificação adversarial, painéis de avaliadores, busca iterativa até esgotamento (*loop-until-dry*), varreduras por diferentes perspectivas e revisão de completude ajudam a filtrar resultados plausíveis, mas incorretos.
- **Sem flags experimentais.** A dependência de `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` foi removida.
- **Política de escolha de modelos.** Em vez de fixar `model: "opus"` em todos os agentes, a v2 escolhe entre fable, opus e sonnet conforme complexidade, duração, autonomia e necessidade de baixa latência. Não se permite definir um mesmo modelo para todos os agentes sem justificativa.
- **Skill `/harness:evolve` funcional.** A evolução, que na v1 existia apenas na documentação, identifica diferenças entre a configuração inicial e o harness atual, generaliza o feedback e incorpora melhorias em agentes, skills e orquestradores.
- **Migração integrada da v1.** A fábrica detecta vestígios da v1 (`TeamCreate`, `TeamDelete` e flags experimentais) e propõe a migração.

## Funcionalidades principais

- **Desenho de equipes:** seis padrões (Pipeline, Fan-out/Fan-in, Pool de Especialistas, Produção–Revisão, Supervisor e Delegação Hierárquica), cada um associado ao modo de execução mais adequado na v2.
- **Geração de skills:** instruções que economizam contexto por meio de divulgação progressiva (*Progressive Disclosure*).
- **Orquestração:** protocolos de transferência de dados (schemas estruturados, arquivos, mensagens e tarefas), tratamento de erros e retomada de execuções.
- **Verificação:** avaliação de acionamento, execuções simuladas (*dry runs*) e testes A/B com e sem a skill, inclusive como workflows.
- **Evolução:** `/harness:evolve` converte o feedback de uso em melhorias mensuráveis.

## Etapas do trabalho

```
Fase 0: Auditar o harness existente (criar, ampliar ou manter; detectar artefatos v1)
Fase 1: Analisar o domínio e o fluxo de controle das tarefas
Fase 2: Definir modo de execução e arquitetura da equipe
Fase 3: Criar as definições de agentes (.claude/agents/)
Fase 4: Gerar as skills (.claude/skills/)
Fase 5: Integrar a orquestração e registrar a referência no CLAUDE.md
Fase 6: Verificar e testar
Fase 7: Manter e evoluir por meio de /harness:evolve
```

## Instalação

### Instalar esta tradução pelo marketplace

```shell
/plugin marketplace add antoniocoutinho42/harness
/plugin install harness@harness-marketplace
```

### Como skills globais

```shell
cp -r skills/harness ~/.claude/skills/harness
cp -r skills/evolve ~/.claude/skills/harness-evolve
```

Para instalar a versão original em inglês e coreano, o marketplace permanece em `revfactory/harness`.

Não são necessárias variáveis de ambiente ou flags experimentais.

## Como usar

```
Crie um harness para este projeto
Projete uma equipe de agentes para meu domínio
Estruture um harness multiagente para o processo de pesquisa
```

Depois de executar o harness:

```
Revise o harness e incorpore as lições desta execução
```

### Como escolher o modo de execução

| Modo | Recurso utilizado | Quando usar |
|---|---|---|
| **Orquestração por workflows** | Scripts `Workflow` | Fluxo determinístico, tarefas enumeráveis, ciclos de verificação, grande escala e saídas estruturadas |
| **Agentes persistentes** | `Agent(name:)`, `SendMessage` e tarefas | Especialistas que mantêm contexto e precisam iterar, negociar ou receber feedback |
| **Delegação a subagentes** | Chamadas pontuais `Agent` | Trabalho paralelo independente, em que basta receber o resultado |

A fábrica escolhe o modo conforme o **formato do fluxo de controle**, não o tamanho da equipe. Também pode combinar modos entre fases.

## Arquivos gerados

```text
seu-projeto/
├── .claude/
│   ├── agents/          # definições de agentes (quem executa)
│   │   ├── analyst.md
│   │   ├── builder.md
│   │   └── qa.md
│   └── skills/          # skills (como executar) e um orquestrador (quem, quando e em que ordem)
│       ├── analyze/SKILL.md
│       └── build/SKILL.md
└── CLAUDE.md            # referência mínima: regra de acionamento e histórico de alterações
```

## Migração da v1

Consulte [docs/migration-v1-to-v2.md](docs/migration-v1-to-v2.md). Em resumo: remova referências a `TeamCreate`, `TeamDelete`, broadcasts e flags experimentais; converta fan-outs para scripts `Workflow`; reescreva a colaboração remanescente usando agentes identificados por nome e `SendMessage`; e elimine a imposição indiscriminada de `model: "opus"`. A fábrica automatiza esse processo ao encontrar artefatos v1 na fase 0.

## Resultados anteriores (v1)

Um teste A/B controlado com 15 tarefas de engenharia de software avaliou o efeito de configurações estruturadas na qualidade dos resultados de agentes de programação: qualidade média de 49,5 para 79,3 (+60%), vitória em 15/15 tarefas e redução de 32% na variância dos resultados (n=15; experimento conduzido pelos próprios autores; veja [revfactory/claude-code-harness](https://github.com/revfactory/claude-code-harness)). Esses números são dos autores; realize um piloto próprio antes de decidir pela adoção.

## Licença

Apache 2.0
