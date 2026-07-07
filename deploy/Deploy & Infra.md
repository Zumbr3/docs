# Deploy & Infra

Ver estrutura de repositório em [[Organização & Time]] e processo de revisão em [[Code Review & Qualidade]].

## Docker

### Um Dockerfile por serviço (multi-stage build)

Padrão simples pros dois em Go:

```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY . .
RUN go build -o service ./cmd/service

FROM alpine:3.20
COPY --from=builder /app/service /service
ENTRYPOINT ["/service"]
```

Pro Java ([[Scoring-service]]), multi-stage com Maven/Gradle + JRE slim na imagem final (não subir JDK completo em produção, só JRE).

### docker-compose pra ambiente local

Um único `docker-compose.yml` na pasta `deploy/` subindo: os 3 serviços + Kafka (ou RabbitMQ) + Redis + Postgres + Prometheus/Grafana. Esse é o ambiente que qualquer um dos 3 (ou um entrevistador curioso) consegue rodar com `docker-compose up` e ver o sistema funcionando de ponta a ponta — isso é o "demo" mais barato e convincente que dá pra ter.

### LocalStack — simular serviços AWS localmente

Vale adicionar o **LocalStack** (`localstack/localstack`) como mais um container no `docker-compose.yml` pra emular serviços AWS sem custo e sem depender de conta real. Usos que fazem sentido nesse projeto:

- **S3** — armazenar exports/relatórios de casos analisados, ou payloads brutos da DLQ pra auditoria.
- **SQS/SNS** — alternativa gerenciada de mensageria caso queiram comparar com Kafka/RabbitMQ (dá pra documentar como ADR: "por que Kafka e não SQS").
- **Secrets Manager / SSM Parameter Store** — em vez de `Secret` do Kubernetes puro, simula como credenciais seriam geridas num ambiente AWS de verdade.
- **DynamoDB** — só se quiserem comparar contadores de velocidade em DynamoDB vs. Redis (mais como exercício de trade-off do que substituição real).

**Por que isso é bom pro portfólio:** conecta diretamente com certificações AWS já feitas — dá pra falar em entrevista "simulei X serviço AWS localmente com LocalStack pra não depender de conta paga" sem precisar de infra real na nuvem.

**Escopo hobby:** não tentem emular tudo — escolham 1-2 serviços AWS que fazem sentido pro caso de uso (S3 pra artefatos é o mais natural aqui) e documentem a decisão num ADR, em vez de adicionar por adicionar.

## Kubernetes — mantendo simples pra hobby

Não tentar replicar um setup de produção de fintech de verdade (multi-região, HPA agressivo, service mesh). Pro objetivo de portfólio, o que importa é mostrar que o time entende os conceitos, não que tem infraestrutura de banco real.

**O que vale a pena ter:**

- `Deployment` + `Service` por microserviço.
- `ConfigMap` pra configuração (thresholds de regra, por exemplo) e `Secret` pra credenciais.
- `HorizontalPodAutoscaler` simples no scoring-service (é o serviço que mais faz sentido escalar sob carga) — só pra mostrar que sabem configurar, não precisa de carga real justificando.
- **Kustomize** com `base/` + `overlays/local` e `overlays/staging` — é mais leve que Helm pra esse tamanho de projeto.

**O que NÃO investir tempo nesse projeto:**

- Service mesh (Istio/Linkerd) — adiciona complexidade operacional que não gera aprendizado proporcional ao esforço aqui.
- Multi-cluster ou multi-região — sem propósito real num projeto hobby.

**Onde rodar:** k3d ou kind (cluster local) é suficiente e gratuito. Só usar EKS/GKE real se quiser também mostrar competência de cloud.

## ArgoCD (GitOps)

Padrão **app-of-apps**: um `Application` raiz no ArgoCD aponta pra pasta `deploy/argocd/`, que referencia um `Application` por serviço (ou um só apontando pro `deploy/k8s/overlays/staging`, mais simples).

**Fluxo:**

1. Push na branch `main` → CI builda imagem → publica no GitHub Container Registry (`ghcr.io`), tagueada com o SHA do commit.
2. Um passo do CI (ou um bot tipo `Image Updater` do próprio ArgoCD) atualiza o manifest do overlay com a nova tag de imagem.
3. ArgoCD detecta a mudança no repo Git e sincroniza automaticamente no cluster.

**Por que isso é bom pro portfólio:** GitOps é exatamente o tipo de prática que aparece em vaga pleno/sênior de infra, e dá pra falar de "deploy declarativo, git como fonte da verdade" com propriedade, porque foi feito na prática, não só lido a respeito.

**Escopo hobby:** não precisa de múltiplos ambientes de verdade (dev/staging/prod separados fisicamente) — um único cluster local com um overlay "staging" já é suficiente pra demonstrar o conceito.

## Pipeline de CI/CD sugerido (GitHub Actions)

```
PR aberta
   │
   ▼
[lint] ──▶ golangci-lint (Go) / Checkstyle+PMD (Java)
   │
   ▼
[test] ──▶ unit tests por serviço (só o serviço que mudou, não rebuild tudo)
   │
   ▼
[review IA] (opcional, não bloqueante) ──▶ comenta na PR
   │
   ▼
merge na main
   │
   ▼
[build] ──▶ Docker build + push pro ghcr.io (tag = SHA do commit)
   │
   ▼
[deploy] ──▶ atualiza manifest do overlay staging → ArgoCD sincroniza
```

**Regra prática pra não estourar minutos de CI de graça do GitHub:** usar `paths:` no trigger do workflow pra só rodar lint/test do serviço que de fato mudou (monorepo com 3 serviços não precisa rebuildar tudo a cada commit).
