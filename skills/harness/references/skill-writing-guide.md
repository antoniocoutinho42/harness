# Guia de redação de skills

Este guia complementa a fase 4 de `SKILL.md` e detalha como produzir skills claras, reutilizáveis e eficientes em contexto.

---

## Sumário

1. Escrever a `description`
2. Redigir o corpo da skill
3. Definir formatos de saída
4. Construir exemplos
5. Carregar informações progressivamente
6. Decidir quando incluir scripts
7. Padronizar schemas de dados
8. Evitar conteúdo inadequado
9. Projetar para reutilização

---

## 1. Escrever a description

Os campos de skill visíveis ao Claude são `name` e `description`. O Claude consulta esses campos para decidir qual skill acionar. As condições específicas de acionamento devem constar da `description`.

### Como uma skill é acionada

O Claude pode optar por ferramentas básicas em tarefas simples, mesmo quando a `description` parece adequada. Por exemplo, “leia este PDF” pode não acionar uma skill especializada. A ativação costuma ser mais provável quando o trabalho exige várias etapas ou critérios profissionais.

### Princípios

1. Especifique **o que a skill faz** e **em quais situações ela deve ser usada**.
2. Diferencie solicitações semelhantes que estão fora do escopo.
3. Seja explícito sobre situações em que a skill deve ser utilizada; o acionamento automático tende a ser conservador.
4. Para skills de orquestração, inclua termos de **continuação e revisão**: reexecutar, corrigir, complementar, atualizar e melhorar resultados anteriores.

### Bom exemplo

```yaml
description: "Lê PDFs, extrai texto e tabelas, mescla, divide, gira páginas,
  aplica marcas d'água, criptografa, descriptografa e executa OCR.
  Use quando o usuário mencionar arquivos .pdf ou solicitar uma entrega em PDF,
  especialmente para conversão, edição e análise."
```

### Exemplos ruins

- `"Skill que processa dados"`: não especifica quais dados ou operações.
- `"Tarefas com PDFs"`: não explica o que faz nem quando deve ser acionada.

## 2. Redigir o corpo

### Explique os motivos

Modelos de linguagem tomam melhores decisões em casos excepcionais quando compreendem os motivos das regras. Descreva a ação e a razão, em vez de enumerar ordens arbitrárias.

**Ruim:**

```markdown
Use sempre pdfplumber para extrair tabelas. Nunca use PyPDF2 para tabelas.
```

**Bom:**

```markdown
Para extrair tabelas, use pdfplumber porque ele reconhece células e preserva
linhas e colunas. PyPDF2 é útil na extração de texto, mas não conserva
confiavelmente a estrutura tabular.
```

### Generalize correções

Ao identificar falhas durante testes, corrija critérios de decisão aplicáveis a vários casos, não um exemplo isolado.

**Sobreajuste:** `Se houver uma coluna "Receita 4T", converta-a em número.`

**Generalização:** `Se o nome de uma coluna indicar medida quantitativa (como "receita", "valor" ou "quantidade"), trate seus dados como numéricos quando apropriado e preserve valores que não possam ser convertidos.`

### Use instruções diretas

Empregue verbos como “Verifique”, “Registre”, “Compare” e “Gere”, em vez de descrições passivas ou promocionais. Uma skill orienta o comportamento de agentes.

### Reduza o contexto desnecessário

A janela de contexto precisa acomodar não apenas a skill, mas também as entradas e resultados do trabalho. Avalie cada instrução:

- “É algo que o Claude já sabe?” → Remova.
- “A ausência da instrução leva a erros?” → Mantenha.
- “Um exemplo concreto é mais esclarecedor que um parágrafo longo?” → Substitua por exemplo.

## 3. Definir formatos de saída

Se a entrega deve seguir uma estrutura, mostre o formato esperado.

```markdown
## Estrutura do relatório
Siga exatamente a organização abaixo:

# [Título]
## Resumo executivo
## Principais achados
## Recomendações
```

Prefira especificações curtas acompanhadas de um exemplo.

**Resultados consumidos por workflows:** se outra etapa processar a saída por código, defina um **schema JSON** em vez de apenas um modelo Markdown. Explique que a resposta final do agente é um objeto de dados para outra etapa, não uma mensagem ao usuário. O schema da skill deve ser compatível com o `schema` especificado em `Workflow.agent()`.

## 4. Construir exemplos

Exemplos concretos frequentemente esclarecem regras melhor que descrições abstratas.

```markdown
## Padrão de mensagem de commit

**Exemplo 1:**
Entrada: adicionar autenticação com tokens JWT
Saída: feat(auth): implementar autenticação JWT

**Exemplo 2:**
Entrada: corrigir botão de exibir senha na página de login
Saída: fix(login): corrigir alternância de visibilidade da senha
```

## 5. Carregar informações progressivamente (*Progressive Disclosure*)

### Alternativa 1: separar referências por domínio

```text
bigquery-skill/
├── SKILL.md (visão geral e como selecionar o domínio)
└── references/
    ├── finance.md (receitas e faturamento)
    ├── sales.md (oportunidades e pipeline comercial)
    └── product.md (uso de API e funcionalidades)
```

Quando o usuário perguntar sobre receita, carregue apenas `finance.md`.

### Alternativa 2: ler detalhes somente sob demanda

```markdown
## Edição de documentos
Para edições simples, altere o XML diretamente.
**Se for necessário registrar alterações**, consulte
[REDLINING.md](references/redlining.md).
```

### Alternativa 3: incluir sumário em referências extensas

Acrescente um sumário na parte inicial dos documentos que ultrapassarem 300 linhas.

## 6. Decidir quando incluir scripts

Leia os registros dos agentes durante os testes. Se encontrar os padrões abaixo, incorpore procedimentos ou scripts reutilizáveis:

| Observação recorrente | Ação |
|---|---|
| Três testes geram o mesmo script auxiliar. | Salve-o em `scripts/`. |
| Os agentes repetem `pip install` ou `npm install` a cada execução. | Documente a instalação das dependências. |
| O mesmo procedimento longo aparece repetidamente. | Defina-o como fluxo padrão da skill. |
| Os agentes contornam o mesmo erro conhecido. | Documente a causa e a solução. |

Execute e valide qualquer script incluído na skill.

## 7. Schemas padronizados

Use os formatos abaixo para compartilhar dados coerentemente entre skills e avaliar resultados.

### eval_metadata.json

```json
{
  "eval_id": 0,
  "eval_name": "nome-descritivo",
  "prompt": "Solicitação apresentada pelo usuário",
  "assertions": [
    "O resultado inclui X",
    "O arquivo é gerado no formato Y"
  ]
}
```

### grading.json

```json
{
  "expectations": [
    {
      "text": "O resultado inclui a palavra 'São Paulo'",
      "passed": true,
      "evidence": "Na terceira etapa, foi encontrada a expressão 'dados da cidade de São Paulo'"
    }
  ],
  "summary": { "passed": 1, "failed": 0, "total": 1, "pass_rate": 1.0 }
}
```

**Não renomeie chaves:** mantenha `text`, `passed` e `evidence`. Não substitua por `name`, `met` ou `details`.

### timing.json

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

Ao receber a conclusão de um subagente, registre imediatamente `total_tokens` e `duration_ms`; essas métricas são disponibilizadas na notificação de conclusão e podem não ser recuperáveis posteriormente.

## 8. O que não incluir em skills

- Arquivos auxiliares como `README.md`, `CHANGELOG.md` e `INSTALLATION_GUIDE.md` dentro da skill.
- Informações úteis somente durante a criação da skill, como logs de testes e revisões intermediárias.
- Manuais escritos para usuários, quando o objetivo é instruir agentes.
- Conhecimentos gerais que o Claude já possui.
- Convenções obsoletas da v1 (`TeamCreate`, `TeamDelete`, flags experimentais, modelo imposto a todos os agentes).

## 9. Projetar skills reutilizáveis

Antes de criar uma skill, compare as capacidades com o que já existe em `.claude/skills/`. Repetir construções ou ampliações pode gerar skills equivalentes com nomes diferentes.

| Relação com skill existente | Ação |
|---|---|
| A skill existente já cobre toda a capacidade desejada | Reutilize a skill e associe-a ao agente. |
| Há sobreposição parcial e é possível generalizar a skill existente | Amplie a skill existente. |
| Há sobreposição, mas uma especialização de domínio é deliberada | Crie uma skill separada. |
| Os escopos são independentes | Crie uma skill nova. |

**Princípio:** quanto mais bem delimitada a responsabilidade da skill, maior seu potencial de reutilização. Se houver duas responsabilidades, considere separá-las.

### Até onde generalizar

Generalização não deve ser infinita. Pare nos limites da responsabilidade pretendida, conservando especializações intencionais e removendo apenas dependências acidentais.

Exemplo de generalização de uma skill para “PDF de avaliação de riscos em fintechs”:

| Dependência removida | Resultado |
|---|---|
| “Em fintechs” | “Relatório PDF de avaliação de riscos”; se a responsabilidade for produzir relatórios de avaliação, pare aqui. |
| “Avaliação de riscos” | “Formatação de PDFs”; se já existir uma skill de formatação, reutilize-a. |

Se a responsabilidade original for deliberadamente “avaliação de riscos em fintechs”, preserve a skill especializada.

Ampliar uma skill altera potencialmente o comportamento de todos os agentes que a utilizam. Identifique esses agentes antes de editar e atualize a `description` para refletir o novo escopo.
