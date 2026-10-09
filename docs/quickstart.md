# Início rápido — seu primeiro harness em 5 minutos

**O que você terá ao final:** diretórios `.claude/agents/` e `.claude/skills/` contendo de 3 a 5 agentes especializados e suas skills, gerados a partir de um único pedido, além do resultado de uma execução de exemplo.

**Pré-requisitos:**
- Versão recente do Claude Code (confirme com `claude --version`).
- Acesso de rede a `github.com`.

> Diferentemente da v1, **não é necessário configurar variáveis de ambiente nem flags experimentais**. Se encontrar `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` em algum guia, ele se refere à v1.

---

## Etapa 1 — Adicionar o marketplace da tradução (30 segundos)

```
/plugin marketplace add antoniocoutinho42/harness
```

## Etapa 2 — Instalar o plugin (30 segundos)

```
/plugin install harness@harness-marketplace
```

Para usar a versão original, o marketplace é `revfactory/harness`.

**Se a instalação não aparecer:** confira com `/plugin list`. Se não estiver listado, repita a etapa 1; se estiver desativado, execute `/plugin enable harness@harness-marketplace`.

## Etapa 3 — Gerar um harness com uma única frase (2 minutos)

```bash
claude "Crie um harness para uma equipe de avaliação de riscos em fintechs"
```

Outros pedidos válidos:
- `claude "Crie um harness para detectar fraudes em comércio eletrônico"`
- `claude "Projete uma equipe de agentes para due diligence técnica de repositórios de código aberto"`

**Resultado esperado:** auditoria do estado atual (fase 0) → proposta de modo de execução e arquitetura → confirmação da criação de 3 a 5 arquivos de agentes, skills e orquestrador.

## Etapa 4 — Conferir os arquivos gerados (30 segundos)

```bash
ls -la .claude/agents/ .claude/skills/
```

**Se nenhum arquivo for criado:** confirme que o plugin está ativado (etapa 2). Se estiver, use uma instrução explícita, como “crie um harness”, para acionar a skill.

## Etapa 5 — Executar uma tarefa de exemplo (90 segundos)

Envie à equipe uma tarefa parecida com um ticket real:

```bash
claude "Ticket FIN-427: uma nova cliente corporativa (indústria de médio porte, receita de US$ 80 milhões, sediada na Coreia do Sul) solicita um limite de capital de giro de US$ 5 milhões. Avalie (1) sinais de alerta no histórico de crédito, (2) concentração setorial em relação à carteira atual e (3) exposição regulatória perante as autoridades coreanas de concorrência e financeiras. Produza um memorando de uma página com recomendação de aprovação ou rejeição."
```

A skill orquestradora deve ser acionada e encaminhar o trabalho ao padrão de equipe gerado (em avaliação de risco, geralmente Produção–Revisão ou Pool de Especialistas).

**Atenção ao custo:** uma execução multiagente pode consumir dezenas ou centenas de milhares de tokens por tarefa. O orquestrador gerado deve manter a escala padrão moderada e só ampliar a investigação quando o pedido solicitar abrangência ou exaustividade. Para estabelecer um orçamento de tokens, inclua uma instrução como `+500k` no prompt; harnesses no modo workflow podem usar essa indicação para dimensionar a execução.

## Próximos passos

- Se o resultado precisar melhorar: peça “revise e evolua o harness com base neste feedback” → `/harness:evolve` generaliza e incorpora o feedback.
- Se o projeto já usar um harness v1: peça “audite o harness” → será proposta a migração ([guia v1 → v2](migration-v1-to-v2.md)).
