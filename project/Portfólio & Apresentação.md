# Como Apresentar Isso como Projeto de Fato

1. **README de arquitetura** — diagrama (o ASCII de [[Visão Geral]] refeito em Mermaid ou excalidraw), explicação do fluxo, decisões de trade-off (por que Kafka, por que threshold em 3 faixas, etc). Isso é o que recrutador técnico e entrevistador realmente leem.
2. **ADRs** — documentar pelo menos 3 decisões: escolha de mensageria, modelo de score, estratégia de idempotência (ver estrutura `docs/adr/` em [[Organização & Time]]).
3. **Demo gravado (2-3 min)** — mostrando uma transação normal passando, uma sendo flagged, e o analista decidindo. Vale mais que 10 páginas de texto.
4. **Métricas reais** — rodar um pequeno load test (k6 ou vegeta) e documentar: throughput, latência p99 do scoring, quantos falsos positivos numa simulação com dados sintéticos.
5. **README individual por serviço** — cada um documenta o seu ([[Ingestion-service]], [[Scoring-service]], [[Case-service]]), mas com um README raiz linkando tudo (isso mostra que sabem organizar um mono-repo ou multi-repo de microserviços).
6. **Testes** — cobertura em unit tests das regras de negócio (scoring) é o que mais impressiona, porque mostra que o time testa lógica de negócio, não só "o servidor sobe".
7. **No LinkedIn/GitHub profile:** um post curto explicando o "porquê" do projeto — "construímos um antifraude simplificado pra entender como bancos reais decidem bloquear uma transação em milissegundos" gera mais engajamento e curiosidade de recrutador do que "fizemos um projeto de fraude".
