# Visão Geral — Sistema Antifraude

> Plataforma de detecção de fraude em transações financeiras em tempo (quase) real, dividida em microserviços. Equipe de 3: 2 serviços em Go, 1 em Java.

## O que é

O sistema recebe transações financeiras (PIX, cartão, transferência) via API, avalia cada uma contra um conjunto de regras e heurísticas de risco, gera um score de fraude, e decide: **aprovar**, **bloquear** ou **enviar para análise manual**. Casos suspeitos viram um "case" que um analista (humano, simulado por você mesmo numa tela simples) pode revisar e decidir.

Isso não é um classificador de ML sofisticado — é um **rule engine + streaming de eventos**, que é exatamente o que a maioria das fintechs reais usa como primeira camada (ML vem depois, em cima disso). Isso é uma vantagem: dá pra fazer bem-feito sem precisar de dataset de ML.

## Input — o que o sistema recebe

Uma transação chega com algo assim:

```json
{
  "transaction_id": "uuid",
  "account_id": "uuid",
  "amount": 1500.00,
  "currency": "BRL",
  "type": "pix",
  "counterparty_account": "uuid-ou-chave-pix",
  "device_id": "string",
  "ip_address": "string",
  "geo_location": { "lat": -23.5, "lon": -46.6 },
  "timestamp": "2026-07-06T14:00:00Z",
  "channel": "mobile"
}
```

Esse payload é o "evento de entrada" que passa por todo o pipeline.

## Arquitetura — visão macro

```
Cliente/API externa
       │
       ▼
┌─────────────────┐
│  API Gateway     │  (opcional, pode ser um dos 3 serviços)
└────────┬─────────┘
         │
         ▼
┌─────────────────┐      publica evento       ┌──────────────────────┐
│ ingestion-service │ ───────────────────────▶ │   work queue          │
│      (Go)          │                          │ "transactions.raw"    │
└─────────────────┘                            └──────────┬────────────┘
                                                            │ consome (at-least-once)
                                                            ▼
                                                  ┌──────────────────┐
                                                  │  scoring-service  │
                                                  │      (Java)        │
                                                  │  rule engine +     │
                                                  │  contadores Redis  │
                                                  └─────────┬─────────┘
                                            ┌────────────────┼──────────────────────┐
                                            │ sucesso                                │ falha no processamento
                                            ▼                                       ▼
                          publica score/decisão                          retry (backoff)
                                            │                                       │
                     ┌──────────────────────┼──────────────────┐        tentativa 1, 2, 3
                     ▼                                         ▼                    │
        queue "transactions.approved"        queue "transactions.flagged"          │
                     │                                         │            falhou nas 3 tentativas
                     ▼                                         ▼                    ▼
            (fim do fluxo, log)                    ┌──────────────────┐   ┌──────────────────┐
                                                    │   case-service    │   │  DLQ:              │
                                                    │      (Go)          │   │  "transactions.dlq"│
                                                    │ gerencia alertas,   │   │  (revisão manual)   │
                                                    │ fila de análise     │   └──────────────────┘
                                                    └──────────────────┘
```

**Por que work queue com at-least-once em vez de "fire and forget":** numa fila de trabalho (work queue), cada mensagem é confirmada (`ack`) só depois de processada com sucesso. Se o consumer cair no meio do processamento, a mensagem volta pra fila e é reentregue — isso garante que nenhuma transação "some" silenciosamente, ao custo de o consumer poder processar a mesma mensagem mais de uma vez (por isso idempotência no scoring-service é obrigatória, não opcional).

## Serviços

- [[Ingestion-service]] — ponto de entrada, validação, enriquecimento
- [[Scoring-service]] — rule engine, score, decisão
- [[Case-service]] — fila de análise manual, feedback loop

## Ver também

- [[Modelo de Dados]]
- [[Stack & Alternativas]]
- [[Roadmap]]
