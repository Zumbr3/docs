# case-service (Go)

Ver contexto geral em [[Visão Geral]].

## Responsabilidade

Gerencia os casos suspeitos (flagged), fila de análise, decisão do analista, e feedback loop.

## Endpoints

- `GET /v1/cases?status=pending` — lista casos pendentes de análise.
- `GET /v1/cases/{id}` — detalhe do caso: transação, score, quais regras dispararam, histórico da conta.
- `POST /v1/cases/{id}/decision` — analista decide: `approve` ou `block`. Isso vira um evento de feedback.
- `GET /v1/cases/stats` — métricas: quantos casos, taxa de falso positivo estimada, tempo médio de análise.

## Regras de negócio

- SLA de análise: caso não decidido em X horas pode escalar prioridade ou ser auto-bloqueado por segurança.
- Feedback loop: a decisão do analista é publicada de volta no Kafka (`case.decided`), permitindo, no futuro, retreinar ou ajustar pesos das regras — isso é o embrião de "human-in-the-loop", conceito que aparece bastante em antifraude de verdade.

## DLQ

Deve expor também uma visão da fila de dead-letter (`transactions.dlq`) do [[Scoring-service]] pra reprocessamento manual — é um caso de uso de análise manual por falha técnica, separado da fila de casos suspeitos.

## Dono

Segunda pessoa (Go) — junto com dashboard simples de análise (front básico ou collection Postman/Insomnia bem documentada). Ver [[Organização & Time]].
