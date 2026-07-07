# ADR-001 — Partition key dos tópicos de transação

**Status:** proposto (a ratificar no kickoff) · **Data:** 07/07/2026

## Contexto
múltiplas instâncias consomem em paralelo, as regras mantêm estado por conta no Redis, e duas delas (velocidade, geo-impossível) dependem da ordem das transações da conta.

## Decisão
Definido Partition Key como accountId, mantém transações de uma mesma conta
em um só consumer

*Partition key -> chave que define o roteamento dos dados entre partições*
*Chave deduplicação -> chave que não mantém a idempotência das transações*

## Alternativas consideradas
transactionId -> duas instâncias leem o contador em "6" ao mesmo tempo, ambas aprovam, a 7ª e a 8ª transações do fraudador passam 
eventId -> Aleatoriza totalmente a distribuição

## Consequências
Eliminamos lost update e perda de ordem por design, sem lock
Aceitamos, em alguns momentos, sobrecarregar uma instância, se a conta tiver transações demais(hot-partition)