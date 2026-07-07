# Modelo de Dados — Sistema Antifraude

Ver contexto geral em [[Visão Geral]].

## Postgres — `transactions`

```
id, account_id, amount, currency, type, counterparty_account,
device_id, ip_address, geo_lat, geo_lon, timestamp,
status (pending/approved/flagged/blocked), score, created_at
```

## Postgres — `cases`

```
id, transaction_id, score, triggered_rules (jsonb),
status (pending/approved/blocked), analyst_decision,
decided_at, sla_deadline
```

## Redis — estruturas

- `velocity:{account_id}` → sorted set com timestamps das últimas transações (janela deslizante).
- `avg_amount:{account_id}` → valor de média histórica (atualizado incrementalmente).
- `known_devices:{account_id}` → set de device_ids já vistos.

Usado principalmente pelo [[Scoring-service]] nas regras de velocidade, valor atípico e dispositivo desconhecido.
