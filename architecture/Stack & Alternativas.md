# Stack & Alternativas

Ver contexto geral em [[Visão Geral]].

## Stack completo

| Componente                             | Escolha sugerida                                    | Alternativa mais simples                                                       | Alternativa mais "enterprise"                  |
| -------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------ | ---------------------------------------------- |
| Mensageria (work queue, at-least-once) | Kafka (tópico + retry-count no header + tópico DLQ) | RabbitMQ (DLQ nativa via `x-dead-letter-exchange`, mais simples de configurar) | Kafka + Schema Registry (Avro)                 |
| Banco relacional                       | PostgreSQL                                          | —                                                                              | —                                              |
| Cache/contadores                       | Redis                                               | —                                                                              | Redis Cluster (se quiser mostrar HA)           |
| Rule engine                            | Regras hard-coded bem estruturadas                  | —                                                                              | Drools                                         |
| Busca/investigação                     | Elasticsearch (opcional)                            | Postgres full-text search                                                      | —                                              |
| Observabilidade                        | Prometheus + Grafana                                | Logs estruturados só                                                           | + Jaeger/OpenTelemetry pra tracing distribuído |
| Auth entre serviços                    | JWT M2M                                             | API key simples                                                                | mTLS                                           |
|                                        |                                                     |                                                                                |                                                |

## Kafka vs RabbitMQ — decisão

Comecem com RabbitMQ se o tempo for curto, migrem pra Kafka se sobrar fôlego — mas dado que "streaming com janela de tempo" é o componente que vocês querem mostrar, Kafka desde o início vale o investimento, porque RabbitMQ não tem replay de eventos nativo (importante se quiserem reprocessar histórico ao mudar uma regra).

Detalhe de implementação de retry/DLQ com cada opção está em [[Scoring-service]].
