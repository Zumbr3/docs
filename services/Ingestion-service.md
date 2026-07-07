# ingestion-service (Go)

Ver contexto geral em [[Visão Geral]].

## Responsabilidade

Ponto de entrada único. Valida payload, enriquece com dados básicos (ex: busca conta no cadastro), publica no Kafka, retorna resposta síncrona rápida (ack, não o resultado da análise).

## Endpoints

- `POST /v1/transactions` — recebe a transação, valida schema, publica evento, retorna `202 Accepted` com `transaction_id`.
- `GET /v1/transactions/{id}/status` — consulta status atual (pending/approved/flagged/blocked) — lê de um cache (Redis) alimentado pelo [[Scoring-service]].
- `GET /v1/accounts/{id}/history` — histórico recente da conta (últimas N transações), usado pelo scoring-service e por analistas.

## Regras de negócio

- Validação de schema (campos obrigatórios, tipos, moeda válida).
- Idempotência: se o mesmo `transaction_id` chegar duas vezes, não duplica processamento (chave idempotente no Redis/Postgres).
- Enriquecimento: busca dados da conta (tempo de conta aberta, KYC status) pra já anexar ao evento — evita que o scoring-service tenha que consultar isso toda hora.

## Componente técnico interessante

Aqui é onde entra idempotência de verdade, algo que já vale a pena aplicar com cuidado (dado histórico de valorizar isso no trabalho atual).

## Dono

Serviço + observabilidade (Prometheus/Grafana) do sistema todo — ver [[Organização & Time]].
