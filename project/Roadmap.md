# Roadmap por Fases

## Fase 1 (MVP funcional)

- [[Ingestion-service]] recebendo e publicando no Kafka.
- [[Scoring-service]] com 3-4 regras simples (velocidade, valor atípico, blocklist).
- [[Case-service]] listando e decidindo casos manualmente.

## Fase 2 (profundidade)

- Sliding window de verdade no Redis (não só contagem simples).
- Idempotência completa no ingestion e no scoring (chave de dedup por `transaction_id`).
- Retry com backoff (máx. 3 tentativas) + Dead Letter Queue configurada.
- Observabilidade: métricas de latência por serviço, dashboard Grafana, contagem de mensagens na DLQ.

## Fase 3 (diferencial, se sobrar tempo)

- Endpoint de simulação de regras (`/rules/simulate`).
- Feedback loop: decisão do analista ajusta contadores/pesos.
- Load test documentado com números reais.

Recomendação: fechar bem a Fase 1 e 2 antes de tentar a Fase 3 — um projeto completo até a Fase 2 vale muito mais em entrevista do que um projeto ambicioso na Fase 3 mas com buracos.

## Ordem de prioridade pra não travar no começo

1. Definir o contrato de eventos entre serviços (primeira Issue) — ver [[Organização & Time]].
2. Subir `docker-compose.yml` local funcional (ambiente de dev compartilhado) — ver [[Deploy & Infra]].
3. CI básico (lint + test) — sem IA de review ainda.
4. Implementar Fase 1 do roadmap (regras básicas, casos manuais).
5. Só depois disso funcionando: Kubernetes + ArgoCD + revisão por IA no CI — essas três coisas são "polimento de portfólio", não bloqueiam o aprendizado central do projeto.
