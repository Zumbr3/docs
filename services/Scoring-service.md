# scoring-service (Java — Spring Boot + Kafka Streams)

Ver contexto geral em [[Visão Geral]].

## Responsabilidade

O coração do sistema. Consome o evento, aplica regras, calcula score, decide, publica decisão.

Não expõe API pública (é um consumer puro), mas pode ter endpoints administrativos:

- `GET /v1/rules` — lista regras ativas e seus pesos.
- `PUT /v1/rules/{id}` — atualiza peso/threshold de uma regra (permite ajuste sem redeploy).
- `POST /v1/rules/simulate` — roda uma transação hipotética contra as regras atuais e retorna o score (útil pra testar antes de mudar regra em produção — isso é uma feature que fintech de verdade tem).

## Regras de negócio (exemplos concretos pra implementar)

| Regra | Lógica | Peso sugerido |
|---|---|---|
| Velocidade | Mais de N transações na mesma conta em X minutos | Alto |
| Valor atípico | Transação > 3x a média histórica da conta | Médio-alto |
| Geo-impossível | Duas transações em locais fisicamente impossíveis de estarem no intervalo de tempo (ex: São Paulo e Miami em 10 min) | Alto |
| Conta nova + valor alto | Conta com menos de 7 dias e transação acima de um limiar | Médio |
| Dispositivo desconhecido | `device_id` nunca visto nessa conta antes | Médio |
| Horário atípico | Transação em horário fora do padrão histórico da conta (ex: 3h da manhã) | Baixo |
| Destinatário na blocklist | `counterparty_account` está em lista de contas suspeitas conhecidas | Crítico (bloqueio automático) |

Cada regra retorna um score parcial (0-100) e um peso. O score final é uma soma ponderada (ou média), e thresholds definem a decisão:

- `< 30` → aprovado automaticamente
- `30-70` → flagged, vai pra fila de análise manual ([[Case-service]])
- `> 70` → bloqueado automaticamente

## Componente complexo real

Contadores de velocidade e comparação com histórico exigem estado — aqui entra Redis com janelas deslizantes (sliding window counter), que é o tipo de coisa que dá pra explicar bem em entrevista (por que Redis e não só Postgres, trade-off de latência). Ver estruturas em [[Modelo de Dados]].

## Alternativas de ferramenta

- Kafka Streams (Java, nativo pro streaming) **ou** consumer simples com Spring Kafka se quiser reduzir a complexidade inicial.
- Drools (rule engine Java) é uma opção mais "enterprise" se quiserem impressionar com regras configuráveis via DSL, mas adiciona complexidade — sugestão é regras hard-coded bem estruturadas primeiro, e só migrar pra Drools se sobrar tempo.

## Estratégia de entrega e retry (at-least-once + DLQ)

- **Semântica:** at-least-once. O scoring-service só confirma (`ack`/commit do offset) a mensagem depois que a regra terminou de rodar e a decisão foi publicada com sucesso. Se o processo cair antes disso, a mensagem é reentregue.
- **Consequência direta:** o processamento de uma transação precisa ser idempotente (aplicar a mesma transação duas vezes não pode gerar dois scores/casos diferentes). Usem o `transaction_id` como chave de deduplicação (ex: `SETNX` no Redis com TTL, ou constraint de unicidade no Postgres).
- **Retry com backoff, máximo de 3 tentativas:**
  1. 1ª tentativa: imediata (falha original).
  2. 2ª tentativa: após backoff curto (ex: 2s).
  3. 3ª tentativa: após backoff maior (ex: 10s) — evita que uma falha transitória (ex: Redis momentaneamente indisponível) já mate a mensagem de cara.
- **Dead Letter Queue (`transactions.dlq`):** se as 3 tentativas falharem, a mensagem vai pra fila de dead-letter em vez de ser descartada ou re-tentada infinitamente. Isso evita duas coisas ruins: (1) uma mensagem "presa" bloqueando o processamento das próximas (poison message), e (2) perda silenciosa de uma transação que não pôde ser avaliada.
- **O que fazer com o que cai na DLQ:** o [[Case-service]] (ou um endpoint administrativo do scoring-service) deve expor uma visão da DLQ pra reprocessamento manual — isso é literalmente um caso de uso de "análise manual", só que por falha técnica, não por suspeita de fraude. Documentar isso como uma decisão separada da fila de casos suspeitos (não misturar os dois conceitos).
- **Alternativas de implementação:**
  - **Kafka:** não tem DLQ nativa — implementa-se com um tópico separado + contagem de retry no header da mensagem (padrão comum: `retry-count` no header, consumer decide se reprocessa ou manda pra DLQ).
  - **RabbitMQ:** tem suporte nativo mais direto via `x-dead-letter-exchange` e TTL por mensagem — se o grupo achar Kafka complexo demais pra esse mecanismo específico, RabbitMQ facilita bastante essa parte.

Ver comparação Kafka vs RabbitMQ em [[Stack & Alternativas]].

## Dono

Terceira pessoa (Java) — é o serviço mais "vistoso" tecnicamente, faz sentido ser o dono de Java mostrar Kafka Streams/Drools. Ver [[Organização & Time]].
