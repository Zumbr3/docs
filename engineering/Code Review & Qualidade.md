# Agente de Revisão de Código (SOLID + Object Calisthenics)

> Foco: manter isso viável como projeto hobby de 3 pessoas, sem virar overhead de processo maior que o próprio código.

Não precisa de infraestrutura de ML pra isso — dá pra fazer com o que já se tem acesso, em duas camadas complementares. Regras aplicadas descritas em [[Guidelines de Engenharia]].

## Camada estática (linters) — roda em segundos, pega o óbvio

- **Go:** `golangci-lint` com os linters `gocyclo` (complexidade ciclomática — pega funções que violam "1 nível de indentação"), `revive` (regras de nomenclatura e SRP básico), `dupl` (duplicação de código).
- **Java:** Checkstyle + PMD. PMD tem regra pronta pra complexidade ciclomática e tamanho de método; Checkstyle cobre nomenclatura e convenções.
- **O que essa camada NÃO cobre:** não sabe dizer se uma regra de negócio devia estar isolada em vez de dentro de um `if`, ou se uma abstração faz sentido. Isso é julgamento de design, não sintaxe.

## Camada de revisão assistida por IA — pra julgamento de design

Aqui sim entra um "agente" de verdade. Duas formas práticas, da mais simples pra mais robusta:

**Opção simples — prompt de revisão versionado no repo:**
Criar um arquivo `docs/code-review-checklist.md` no repo com os princípios SOLID/Object Calisthenics definidos pelo time, e a cada PR, antes de pedir review humano, alguém roda esse checklist via Claude (ou outra IA) colando o diff. Não precisa de automação — é um hábito de time, documentado.

**Opção automatizada — GitHub Actions + IA na PR:**
- Existe a action `anthropics/claude-code-action` (GitHub Action oficial da Anthropic) que roda Claude Code diretamente no CI, comentando na PR. Passa-se o checklist do time (o mesmo `code-review-checklist.md`) como contexto/instrução, e ele comenta achados na PR automaticamente.
- Fluxo sugerido: PR aberta → linter roda primeiro (rápido, barato) → só se passar, roda a revisão por IA (mais lenta, mais cara) comentando pontualmente onde SRP/OCP foram violados.
- **Importante pro escopo hobby:** não bloquear o merge por causa do comentário da IA — usar como sugestão, não gate. Vira ruído e desmotiva o time se virar bloqueio obrigatório num projeto sem prazo de produção real.

**Recomendação:** começar só com linter automatizado + checklist manual (opção simples). Só adicionar a IA no CI se o time sentir que tá relaxando a disciplina — não adicionar complexidade de automação antes de sentir a dor que ela resolve.

Ver onde isso entra no pipeline em [[Deploy & Infra]].
