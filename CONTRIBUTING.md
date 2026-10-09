# Como contribuir com o Harness

Contribuições são bem-vindas. Este guia é intencionalmente breve: prioriza princípios em vez de regras excessivas.

## Princípios

1. **Skills são instruções para agentes.** Não inclua nas skills manuais para usuários, textos promocionais ou conhecimentos gerais que o Claude já possui.
2. **O contexto é um recurso compartilhado.** Mantenha o corpo de `SKILL.md` com até 500 linhas e transfira detalhes para `references/`. Cada frase precisa justificar seu custo em tokens.
3. **Explique os motivos.** Em vez de depender de `ALWAYS` e `NEVER`, justifique as regras. Conhecer o motivo permite tomar boas decisões mesmo em casos-limite.
4. **Considere somente o runtime atual.** Não introduza em PRs flags experimentais, APIs removidas (como `TeamCreate`) nem a escolha fixa de um modelo para todos os agentes. Se uma alteração do runtime quebrar a documentação, sua correção passa a ser prioritária.

## Checklist para pull requests

- [ ] As alterações são coerentes entre `SKILL.md` e os arquivos correspondentes em `references/`?
- [ ] Se a mudança na `description` afetar o acionamento da skill, ela foi testada com consultas que devem acioná-la e consultas semelhantes que não devem?
- [ ] A alteração foi registrada em `CHANGELOG.md`?
- [ ] As versões coincidem entre `plugin.json`, `marketplace.json` e o selo do README?

## Issues

- Bugs: inclua o prompt para reprodução, os comportamentos esperado e observado e a saída de `claude --version`.
- Incompatibilidade com o runtime: aplique o rótulo `compat`; prioridade máxima.

## Prazos de resposta almejados

- Primeira resposta a PRs: até 72 horas.
- Triagem de issues: até 48 horas.

Esses prazos são compromissos com a comunidade, não um SLA comercial.
