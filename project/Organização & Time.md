# Organização do Repositório e do Time

## Monorepo vs. multi-repo — decisão real, não só preferência

Pra 3 pessoas e 3 serviços com deploy conjunto (docker-compose/k8s no mesmo ambiente), **monorepo é mais simples de administrar** nesse estágio. Multi-repo só compensa quando os serviços têm ciclos de release e times de verdade separados — não é o caso aqui.

## Estrutura sugerida (monorepo)

```
antifraude-platform/
├── README.md                     ← visão geral, arquitetura, como rodar tudo
├── docs/
│   ├── architecture.md           ← diagrama + decisões macro
│   ├── adr/                      ← Architecture Decision Records
│   │   ├── 001-mensageria.md
│   │   ├── 002-modelo-de-score.md
│   │   └── 003-idempotencia-dlq.md
│   └── code-review-checklist.md  ← SOLID + Object Calisthenics do time
├── services/
│   ├── ingestion-service/        ← Go
│   │   ├── cmd/
│   │   ├── internal/
│   │   ├── Dockerfile
│   │   └── README.md
│   ├── scoring-service/          ← Java
│   │   ├── src/
│   │   ├── Dockerfile
│   │   └── README.md
│   └── case-service/             ← Go
│       ├── cmd/
│       ├── internal/
│       ├── Dockerfile
│       └── README.md
├── deploy/
│   ├── docker-compose.yml        ← ambiente local completo
│   ├── k8s/
│   │   ├── base/                 ← manifests base (Kustomize)
│   │   └── overlays/
│   │       ├── local/
│   │       └── staging/
│   └── argocd/
│       └── app-of-apps.yaml
└── .github/
    └── workflows/
        ├── ci.yml                ← lint + test + build por serviço
        └── code-review.yml       ← revisão assistida (opcional)
```

**Por que isso funciona bem pra portfólio:** um recrutador ou entrevistador técnico consegue entender a arquitetura do projeto só de abrir a pasta, sem precisar clonar e rodar nada. Isso conta mais do que se imagina. Ver [[Portfólio & Apresentação]] pra detalhes de README/ADRs/demo.

## Convenção de commits e branches

- **Commits semânticos** (`feat(scoring): add velocity rule`, `fix(ingestion): dedup key ttl`) — padronizar entre os 3.
- **Branches:** `main` protegida, feature branches curtas (`feat/velocity-rule`), sem long-lived branches por serviço — isso evita merge hell num projeto de 3 pessoas.

## Divisão sugerida entre os 3

- **Ingestion-service** (Go) + observabilidade (Prometheus/Grafana) do sistema todo — aproveita direto experiência de API/auth. Ver [[Ingestion-service]].
- **Case-service** (Go) + dashboard simples de análise (front básico ou Postman/Insomnia collection bem documentada). Ver [[Case-service]].
- **Scoring-service** (Java) — é o serviço mais "vistoso" tecnicamente, faz sentido o dono de Java mostrar Kafka Streams/Drools. Ver [[Scoring-service]].

Isso evita que uma pessoa fique com "o serviço chato" — cada um tem uma parte com profundidade técnica própria pra defender em entrevista.

## Como organizar tasks sem virar processo pesado

Isso é o ponto mais importante, porque é onde projeto hobby de 3 pessoas costuma morrer — não por falta de conhecimento técnico, mas por gestão de tarefas virando trabalho em si.

### O que usar

**GitHub Projects (board simples, não Jira):** 3 colunas só — `A fazer`, `Em andamento`, `Feito`. Nada de sprints, story points, ou estimativa em horas. Isso é overhead de processo corporativo que não faz sentido sem cobrança externa.

Cada task = uma Issue do GitHub, referenciada no commit/PR (`closes #12`). Isso já documenta o "porquê" de cada mudança de graça, sem esforço extra.

### Ritmo sugerido

- **Sem daily, sem sprint.** Uma call curta (15-20 min) por semana pra alinhar o que cada um fez e o que trava o próximo passo. Mais que isso vira fricção que mata motivação de projeto sem cobrança.
- **Definir "pronto" por fase, não por task individual.** Ex: "Fase 1 pronta quando as 3 regras básicas do scoring estão testadas e o case-service lista casos" — isso já vem do [[Roadmap]], então tasks derivam disso, não o contrário.

### Como dividir tasks entre os 3 sem gerar dependência travada

- Cada serviço tem dono (definido acima).
- Definir o **contrato entre serviços primeiro** (schema do evento Kafka/queue, formato do payload) — isso é literalmente a primeira Issue de todas, porque desbloqueia os 3 trabalharem em paralelo sem esperar um pelo outro.
- Depois do contrato definido, cada dono trabalha isolado no próprio serviço, e só sincronizam quando alguém quer integrar de verdade (rodar o docker-compose local com os 3 juntos).

### Sinal de alerta de que virou processo demais

Se em algum momento o time estiver gastando mais tempo atualizando o board do que escrevendo código, é sinal de over-engineering de processo — simplificar de volta pras 3 colunas e seguir.
