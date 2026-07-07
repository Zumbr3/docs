# Contrato de Eventos - Zumbre

## 1 - Convenções gerais

### Idioma e nomenclatura
- Tópicos, eventos e campos em inglês, `camelCase`
- Tópicos seguem `<domínio>.<estágio>`: `transaction.received`, `transaction.flagged`.
- Eventos nomeados como fato no passado (`transaction.flagged`),
  nunca como comando (`flag_transaction`).

### Envelope padrão
Todo evento, de qualquer serviço, carrega:

| Campo         | Tipo                  | Descrição                                   |
|---------------|-----------------------|---------------------------------------------|
| eventId       | string (UUID v6)      | id único desta mensagem                     |
| eventType     | string                | ex.: `transaction.received`                 |
| eventVersion  | int                   | versão do schema do `data`                  |
| occurredAt    | string (ISO-8601 UTC) | quando o fato ocorreu no mundo              |
| producer      | string                | serviço publicador, ex.: `ingestion-service`|
| data          | objeto                | payload específico do evento (seção 3)      |

### Formatos
- **Datas:** ISO-8601 em UTC, timezone explícito (`2026-07-07T14:00:00Z`).
  Data sem timezone é violação de contrato.
- **Monetário:**  string decimal ("amount": "1500.00"), parseada sempre como decimal
- **Ids:** UUID v6, sempre como string.
- **Coordenadas:** `{ "lat": -23.55, "lon": -46.63 }`, decimais, WGS-84.
- **Enums:** string minúscula, conjunto fechado listado no schema do evento.
  Consumer que encontrar valor desconhecido: marca a transação como erro, não processa
  apenas salva como DLQ.


### Particionamento
- Key de toda mensagem: `AccountId` — garante ordem por conta.

## 2. Tópicos

| nome                    | Partition key  | Partições | Produtor          | Consumers         | Retenção |
|-------------------------|----------------|-----------|-------------------|-------------------|----------|
| transaction.received    | AccoundId      | 6         | ingestion-service | scoring-service   | 7 dias   |
| transaction.approved    | AccoundId      | 6         | scoring-service   | injection-service | 7 dias   |
| transaction.flagged     | AccoundId      | 1         | scoring-service   | case-service      | 30 dias  |
| transaction.dlq         | AccoundId      | 1         | scoring-service   | ops (manual)      | 30 dias  |

Nº de partições e retenção: justificativas em ADR-003.

## 3. Eventos 
### 3.1 transaction.received (tópico: transaction.received)

**Versão atual do schema:** 1 · **Produtor:** injection-service · **Consumers:** scoring-service

**Payload (`data`):**
**Envelope: ver §1**

| Campo                | Tipo                                       | Obrig. | Descrição                                    |
|----------------------|--------------------------------------------|--------|----------------------------------------------|
| transactionId        | string (UUID)                              | sim    | id da transação                              |
| accountId            | string (UUID)                              | sim    | id da conta que fez a transação              |
| currency             | string (ISO-4217)                          | sim    | Moeda utilizada na transação                 |
| type                 | string                                     | sim    | Tipo da transação(pix, cartão, transferência)|
| counterpartyAccount  | string                                     | sim    | Conta da outra parte da transação            |
| deviceId             | string                                     | sim    | Id do dispositivo que fez a transação        |
| ipAddress            | string                                     | sim    | Ip que realizou a transferência              |
| geoLocation          | {"lat": 0.0, "lon": 0.0}                   | sim    | Localização do dispositivo                   |
| transactionDate      | string (ISO-8601 UTC)                      | sim    | Momento que a transação ocorreu              |
| channel              | string                                     | sim    | Canal que realizou a transação               |
| amount               | string (decimal)                           | sim    | Valor monetário da transação                 |
| accountDetails       | {"name": "", "account":"", "createdAt": ""}| sim    | Dados da conta que realizou a transação      |

**Exemplo:**
{
    "eventId": "9b2f-...",
    "eventType": "transaction.received",
    "eventVersion": 1,
    "occurredAt": "2026-07-07T14:00:00Z",
    "producer": "injection-service",
    "data": {
        "transactionId": "c41a-...",
        "accountId": "c41a-...",
        "currency": "BRL",
        "type": "pix",
        "counterpartyAccount": "99999",
        "deviceId": "c41a-...",
        "ipAddress": "0.0.0.0",
        "geoLocation": {
            "lat": 0.0,
            "lon": 0.0
        },
        "transactionDate": "2026-07-07T14:00:00Z",
        "channel": "app",
        "amount": "1500.00",
        "accountDetails": {
            "name": "fulano",
            "account": "0000-00",
            "createdAt": ""
        }
    }
}