# Guidelines de Engenharia — SOLID e Object Calisthenics

Vale documentar isso no README como "convenções do time" — mostra maturidade de engenharia além do "funciona". Aplicação prática pro [[Scoring-service]] (Java) e pros dois serviços em Go ([[Ingestion-service]], [[Case-service]]).

## SOLID — onde cada princípio aparece de verdade nesse projeto

| Princípio | Onde aplicar |
|---|---|
| **S**ingle Responsibility | Cada regra de fraude (velocidade, valor atípico, geo-impossível...) é uma unidade isolada, testável sozinha — não um `if` gigante dentro de uma função `avaliarTransacao()`. |
| **O**pen/Closed | Adicionar uma regra nova não deve exigir alterar o motor de scoring — só implementar uma nova regra que satisfaz uma interface comum (`Rule`/`Regra`) e registrá-la. |
| **L**iskov Substitution | Qualquer implementação de `Rule` deve poder substituir outra sem quebrar o motor — nenhuma regra específica pode exigir tratamento especial no orquestrador. |
| **I**nterface Segregation | Uma regra que só precisa do histórico da conta não deveria depender de uma interface gigante que também expõe dados de dispositivo, geo, etc. Separem por contrato de dados realmente necessário. |
| **D**ependency Inversion | O scoring-service depende de uma abstração de storage (`ContadorRepository`, `RuleRepository`), não diretamente do cliente Redis/Postgres — facilita trocar Redis por outra coisa e, principalmente, facilita mockar em teste unitário. |

**Aplicação concreta:** definam uma interface `Rule` (Java) / `Regra` (Go) com um método tipo `Evaluate(transaction) RuleResult` logo no início do projeto. Isso by design já força SRP e OCP — é a decisão de design mais importante do scoring-service.

## Object Calisthenics — regras mais relevantes pro escopo do projeto

Não precisa aplicar as 9 regras clássicas de forma dogmática (algumas, tipo "sem getters/setters", são difíceis de levar a sério em Go/Java de produção). As que valem a pena aplicar de verdade aqui:

1. **Um nível de indentação por método** — força quebrar a avaliação de regra em métodos pequenos (`Evaluate` → `checkThreshold` → `checkTimeWindow`), em vez de uma função de 80 linhas com `if` aninhado. Especialmente relevante na regra de "geo-impossível", que naturalmente tenta virar um emaranhado de condicionais.
2. **Não usar a palavra ELSE** — em vez de `if score > 70 { bloqueia } else if score > 30 { flag } else { aprova }`, usem early return ou uma tabela de decisão (`DecisionTable` com faixas e ação associada). Deixa a lógica de threshold mais fácil de testar isoladamente.
3. **Encapsular tipos primitivos** — em vez de passar `float64 amount` e `string currency` soltos pelo código, criem um tipo `Money` (ou `Amount`) que já carrega a lógica de comparação e formatação. Evita bug clássico de comparar valores em moedas diferentes sem querer.
4. **Coleções em wrapper próprio** — em vez de passar `[]Rule` cru entre camadas, um tipo `RuleSet` que expõe só os métodos necessários (`Evaluate`, `Add`) evita que qualquer parte do código manipule a lista de regras de forma inconsistente.
5. **Nomes sem abreviação e que revelam intenção** — principalmente nas regras de negócio (`isVelocityExceeded`, não `chkVel`), porque essa é a parte do código que vocês vão explicar em entrevista linha por linha.

**Onde não vale a pena forçar:** "sem getters/setters" e "máximo 2 variáveis de instância por classe" são regras didáticas boas pra exercício isolado, mas vão gerar fricção desnecessária num projeto de equipe com prazo — não briguem por isso.

Ver processo de revisão que aplica essas regras em [[Code Review & Qualidade]].
