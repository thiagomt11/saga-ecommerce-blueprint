# Decisões arquiteturais — Saga de compra

**Status:** proposta de arquitetura  
**Data:** 23/10/2026  
**Objetivo:** especificar a coordenação transacional, contratos, compensações e mecanismos de resiliência de uma compra em microsserviços.

## ADR-001 — Orquestração em vez de coreografia

### Contexto

O sistema possui três serviços com bancos próprios e um gateway financeiro externo. O fluxo requer coordenar reserva, cobrança, baixa de estoque e confirmação de pedido, além de compensações encadeadas quando ocorrem falhas.

### Decisão

Utilizar um **Saga Orchestrator** separado dos domínios, com máquina de estados persistente e comunicação assíncrona por comandos e respostas. Todos os participantes podem ser escalados de maneira independente, e o coordenador é replicável (com controle de concorrência no banco).

### Consequências

**Positivas:** visão unificada da Saga, ordem explícita de operações/compensações, auditoria, timeouts e métricas em um único fluxo. **Negativas:** mais um serviço crítico, necessidade de alta disponibilidade, complexidade em compensações e mudança de contratos. Não existe rollback global nem isolamento ACID entre serviços.

### Comparação

| Atributo | Orquestração | Coreografia |
|---|---|---|
| Acoplamento | Orquestrador conhece contratos dos domínios | Participantes dependem dos eventos uns dos outros |
| Complexidade evolutiva | Regras centralizadas e fluxo rastreável | Cresce a dificuldade de rastrear cadeias de eventos |
| Escalabilidade | Escala horizontal com Saga Store persistente | Escala horizontal por consumidores independentes |
| Disponibilidade | Coordenador exige failover/retomada | Não há coordenador único, porém tratamento é distribuído |
| Cenário escolhido | **Mais adequado** ao controle de pagamento e compensações múltiplas | Seria viável em fluxo curto e de poucos eventos |

## ADR-002 — Limites de transação, dados e consistência eventual

- **Order DB** é propriedade exclusiva do Order Service. Estados: `PENDING`, `CONFIRMED`, `CANCELLED`.
- **Inventory DB** é propriedade exclusiva do Inventory Service. Estados de uma reserva: `RESERVED`, `RELEASED`, `COMMITTED`, `RESTORED`.
- **Payment DB** é propriedade exclusiva do Payment Service. Estados de uma transação: `PENDING`, `CAPTURED`, `DECLINED`, `REFUNDED`, `UNKNOWN`.
- **Saga Store** é propriedade do orquestrador. Estados e comandos pendentes persistem com controle de versão.

Cada alteração de domínio e sua mensagem de saída são gravadas **na mesma transação local**. O *outbox relay* publica posteriormente a mensagem. O consumidor confirma a mensagem no broker apenas depois de gravar atomicamente o resultado de negócio e a Inbox. Nunca acessar o banco de outro serviço diretamente.

**Estoques concorrentes:** a reserva altera disponibilidade com condição atômica `available >= quantity` e bloqueio/versionamento por SKU; duas Sagas não podem consumir a mesma quantidade. Ao reservar, o produto deixa de estar disponível para outras compras. `CommitReservation` transforma a reserva em baixa, sem descontar a disponibilidade **pela segunda vez**. `ReleaseStock` devolve disponibilidade quando a reserva ainda está `RESERVED`; `RestoreStock` desfaz uma baixa `COMMITTED`. Operações devem verificar o estado anterior e ser idempotentes.

**Lacre semântico:** pedido `PENDING` não pode ser expedido. Somente `CONFIRMED` está elegível ao próximo processo de expedição (fora do escopo desta Saga). A reserva temporária deve sobreviver aos retries legítimos; um *lease* ou TTL (por exemplo, 15 minutos) é reconciliado com o estado da Saga antes de expirar, evitando liberação durante uma cobrança ainda indeterminada.

## ADR-003 — Contratos de comunicação

O início do fluxo é síncrono apenas na borda HTTP:

```http
POST /orders HTTP/1.1
Idempotency-Key: compra-cliente-001
Content-Type: application/json

{"items":[{"sku":"SKU-001","quantity":2}],"totalCents":15990,"currency":"BRL"}
```

A resposta sugerida é `202 Accepted` com `{ "orderId": "ord-123", "status": "PENDING" }` e `Location: /orders/ord-123`. Repetir a mesma intenção/mesma chave devolve o **mesmo** pedido. Reutilizar a chave com conteúdo diferente deve ser rejeitado (por exemplo, HTTP 409).

### Envelope padrão

Todo comando e evento usa envelope versionado e identificadores para rastreio:

```json
{
  "schemaVersion": 1,
  "messageId": "msg-005",
  "commandId": "saga-123:ChargePayment",
  "sagaId": "saga-123",
  "orderId": "ord-123",
  "correlationId": "trace-abc",
  "causationId": "msg-004",
  "type": "ChargePayment",
  "occurredAt": "2026-10-23T18:00:00-03:00",
  "deadlineAt": "2026-10-23T18:00:30-03:00",
  "payload": {
    "amountCents": 15990,
    "currency": "BRL",
    "gatewayIdempotencyKey": "saga-123:ChargePayment"
  }
}
```

**Regras do envelope:** `messageId` identifica uma **publicação**; `commandId` é estável para a operação, mesmo com uma republicação e outro `messageId`. O par `(sagaId, operação)` funciona como *operationKey*. Todas as respostas informam `commandId` original e podem incluir `result`, `errorCode`, `reservationId` ou `gatewayTransactionId`. Regras de autenticação/autorização mTLS ou credenciais de serviço e assinatura/autenticidade dos eventos são exigidas; dados completos de cartão nunca entram nos eventos nem nos logs.

### Contratos por comando

| Comando publicado pelo orquestrador | Serviço | Resposta positiva | Falha de negócio | Compensação |
|---|---|---|---|---|
| `ReserveStock` | Inventory | `StockReserved` | `OutOfStock` | `ReleaseStock` |
| `ChargePayment` | Payment | `PaymentCaptured` | `PaymentDeclined` | `RefundPayment` somente se houve captura |
| `CommitReservation` | Inventory | `StockCommitted` | `CommitFailed`* | `RestoreStock` caso já consolidado |
| `ConfirmOrder` | Order | `OrderConfirmed` | `OrderConfirmationFailed`* | Verificar status antes de cancelar |
| `ReleaseStock` | Inventory | `StockReleased` | `ReleaseFailed` | Retry/DLQ, nunca pular silenciosamente |
| `RestoreStock` | Inventory | `StockRestored` | `RestoreFailed` | Retry/DLQ/reconciliação |
| `RefundPayment` | Payment | `PaymentRefunded` | `RefundFailed` | Retry/DLQ/reconciliação financeira |
| `CancelOrder` | Order | `OrderCancelled` | `CancelFailed` | Retry/DLQ/reconciliação |

\* Em erros/timeout ambíguos, verificar o estado real da operação antes de concluir que ela falhou.

Canais lógicos propostos: `order.events`, `saga.commands.order`, `saga.commands.inventory`, `saga.commands.payment`, `saga.replies` e `saga.dlq`. A implantação pode utilizar exchanges/queues do RabbitMQ, tópicos do Kafka ou serviço equivalente. Particionamento/ordem por `orderId` (quando suportado); compensações e confirmações verificam versão/estado no banco para proteger contra reordenação.

## ADR-004 — Sequência de sucesso e estados

| Passo | Estado persistido no orquestrador antes de enviar | Comando / resultado esperado | Estado alcançado |
|---|---|---|---|
| 0 | `STARTED` | Consome `OrderCreated` | `WAITING_STOCK` |
| 1 | `WAITING_STOCK` | `ReserveStock` → `StockReserved` | `WAITING_PAYMENT` |
| 2 | `WAITING_PAYMENT` | `ChargePayment` → `PaymentCaptured` | `WAITING_STOCK_COMMIT` |
| 3 | `WAITING_STOCK_COMMIT` | `CommitReservation` → `StockCommitted` | `WAITING_ORDER_CONFIRMATION` |
| 4 | `WAITING_ORDER_CONFIRMATION` | `ConfirmOrder` → `OrderConfirmed` | `COMPLETED` |

A transição de estado e o próximo comando (na Outbox do orquestrador) são persistidos juntos. Cada resultado é aceito apenas se `sagaId`, `commandId`, passo esperado e versão da instância forem compatíveis. Eventos atrasados de passos anteriores são registrados como duplicatas/atrasados, nunca aplicados em um estado subsequente.

## ADR-005 — Compensações e matriz de decisão

As compensações são **operações de negócio inversas**, não um `ROLLBACK` SQL distribuído. Executá-las de forma serial, aguardando a confirmação de cada uma. Se uma compensação falhar, **manter a Saga em compensação pendente** e reprocessar/reconciliar.

| Situação detectada | Evidência mínima | Ação do orquestrador | Estado terminal possível |
|---|---|---|---|
| `OutOfStock`, sem reserva | Resposta de negócio definitiva | `CancelOrder` | `COMPENSATED` |
| `PaymentDeclined`, estoque `RESERVED` | Recusa definitiva do gateway/serviço | `ReleaseStock → CancelOrder` | `COMPENSATED` |
| Timeout no `ChargePayment` | Resultado financeiro **desconhecido** | `RECONCILING`: consultar status por chave e só então avançar/compensar | `COMPLETED`, `COMPENSATED` ou `MANUAL_REVIEW` |
| `CommitReservation` falha **sem ter sido efetivado**, após captura | Consulta confirma reserva ainda `RESERVED` | `RefundPayment → ReleaseStock → CancelOrder` | `COMPENSATED` após todos ACKs |
| `ConfirmOrder` falha **sem confirmar pedido**, estoque `COMMITTED` | Consulta confirma pedido ainda `PENDING` | `RestoreStock → RefundPayment → CancelOrder` | `COMPENSATED` após todos ACKs |
| ACK de `ConfirmOrder` perdido | Consulta comprova pedido `CONFIRMED` | Reconciliar como sucesso; **não** cancelar | `COMPLETED` |
| Falha temporária em `RefundPayment` | Estorno pendente ou desconhecido | Retry idempotente + conciliação com gateway; alarme | `COMPENSATING_*` ou `MANUAL_REVIEW` |

**Ordem inversa:** depois de cobrança capturada e baixa do estoque, a compensação desfaz primeiro a baixa (`RestoreStock`), depois o pagamento (`RefundPayment`) e por último o pedido (`CancelOrder`). A política de produto pode exigir estado `CANCELLATION_PENDING` enquanto a reversão financeira não foi confirmada, evitando comunicar cancelamento final prematuramente.

**Distinção crítica:** `DECLINED` é falha de negócio; `TIMEOUT`, `503` e perda de ACK são falhas técnicas e não provam que a operação deixou de ocorrer.

## ADR-006 — Controle de estado, concorrência, timeouts e idempotência

### Esquema lógico da Saga Store

**`saga_instances`**

| Campo | Finalidade |
|---|---|
| `saga_id` (PK), `order_id` (UNIQUE) | Identificação única e rastreabilidade |
| `status`, `current_step`, `step_status` | Máquina de estados e passo em execução |
| `last_command_id`, `last_event_id` | Correlação e descarte de respostas indevidas |
| `deadline_at`, `next_retry_at`, `attempt` | Timeout e política de reenvio |
| `version` | Concorrência otimista via compare-and-set |
| `last_error`, `created_at`, `updated_at` | Diagnóstico e auditoria |

**`saga_steps`** guarda `(saga_id, step_name, attempt, operation_key, status, started_at, finished_at, result_json)` para histórico, auditoria e retomada. **`outbox_messages`** guarda mensagens não publicadas até confirmação de publicação. Cada serviço participante também guarda sua Inbox de mensagens recebidas e uma tabela/índice de efeitos executados por `operation_key`.

### Algoritmo conceitual do Idempotent Consumer

1. Receber comando com `messageId` e `commandId`/`operationKey`.
2. Abrir **transação local** e tentar registrar `messageId` na `inbox` com UNIQUE.
3. Consultar o resultado de `operationKey` UNIQUE. Se já executado, recuperar o resultado sem repetir efeito financeiro/estoque.
4. Se inédito, validar pré-condições e executar a transação local **uma única vez**; gravar resultado e evento de resposta na Outbox.
5. Confirmar a transação local e depois ACK no broker. Em crash antes do ACK, o reenvio reutiliza o resultado anterior; em crash antes do commit, não houve efeito persistido.

**Nota sobre gateway:** registrar previamente a intenção/chave estável e usar a **idempotency key do próprio gateway**. Uma chamada externa não pode participar da transação local; se ocorrer crash após cobrança e antes de persistir a resposta, reconciliar o status pelo identificador/chave em vez de fazer outra cobrança com chave nova.

### Política sugerida de timeouts

| Situação | Regra proposta (ajustável por SLO) |
|---|---|
| Sem resposta de serviço | `deadline_at` por etapa, por exemplo 30 s |
| Falha transitória | Até 3 retries, backoff exponencial `1 s → 2 s → 4 s` com jitter |
| Falha definitiva de negócio | Sem retry automático do mesmo resultado; compensação |
| Falha indeterminada em gateway | Consulta transação pela chave; não assumir recusa |
| Sem conclusão dentro de prazo máximo | `RECONCILING`; se impossível, `MANUAL_REVIEW` + alerta |
| Reserva temporária expirada | Conferir Saga/pagamento; liberar somente quando for seguro |

**Watcher/scheduler:** periodicamente consulta `saga_instances` cujos `deadline_at`/`next_retry_at` venceram; adquire a instância por `version`/lease, reconcilia ou republica o **mesmo comando lógico**. Reiniciar o orquestrador não perde o progresso.

## ADR-007 — Resiliência, Dead-Letter Queue e observabilidade

### Retry e Circuit Breaker

**Retry** apenas em erros provavelmente transitórios (`timeout`, HTTP `429`, `502`, `503`), com limite, jitter e respeito ao tempo total de Saga; nunca repetir cegamente uma cobrança sem idempotência. `PaymentDeclined` e `OutOfStock` não são falhas transitórias a retentar. **Circuit Breaker** nas chamadas externas ao gateway: abrir após anomalias sucessivas, rejeitar novas tentativas durante o período de proteção e sondar recuperação (half-open). O Circuit Breaker protege recursos, mas não decide sozinho a compensação financeira.

**Bulkhead:** limitar concorrência do cliente do gateway e isolar a capacidade de pagamentos dos consumidores de estoque/pedidos. **Outbox Relay:** republicar sem perder mensagens mesmo quando o broker está indisponível. **Reconciliador:** corrigir situações em que a realidade do banco/gateway não coincide com o estado de Saga.

### DLQ — política de tratamento

1. Publicar na **dead-letter queue** após o limite de retentativas técnicas ou em caso de mensagem inválida (*poison message*), preservando cabeçalhos, `sagaId`, `orderId`, `commandId`, `messageId`, `errorCode`, `firstSeenAt`, `lastSeenAt` e contador de tentativas.
2. Alarmar a equipe e manter a Saga em `RECONCILING`, `COMPENSATING_*` ou `MANUAL_REVIEW`, jamais marcá-la falsamente `COMPLETED`/`COMPENSATED`.
3. Classificar a causa: erro de contrato, falha de infraestrutura, serviço indisponível, duplicata legítima ou estado externo desconhecido.
4. Corrigir causa raiz e reprocessar a partir da DLQ **com a mesma operationKey** e verificações de estado; não repetir estorno/cobrança com nova chave.
5. Registrar operador, data, decisão e resultado em trilha de auditoria; fazer teste de reprocessamento sem duplicação.

**Métricas e alertas:** número de Sagas iniciadas/concluídas/compensadas, idade das `PENDING`, tentativas por etapa, tempo p95/p99, atraso de Outbox, profundidade da DLQ, quantidade de estados `UNKNOWN` e falhas de reconciliação. Incluir `sagaId`, `orderId`, `commandId` e `correlationId` em traces e logs estruturados, sem dados sensíveis.

## ADR-008 — Riscos e mitigação

| Risco | Efeito possível | Mitigação arquitetural |
|---|---|---|
| Evento perdido na janela DB/broker | Pedido fica pendente indefinidamente | Transactional Outbox + relay + métricas de atraso |
| Evento duplicado | Duas cobranças ou reservas | Inbox + operationKey UNIQUE + chave idempotente no gateway |
| Falha do orquestrador | Fluxo interrompido | Saga Store durável + réplicas + lease/versionamento + scheduler |
| ACK perdido após cobrança | Cancelamento ou segunda cobrança indevida | Estado `UNKNOWN` + consulta/reconciliação gateway |
| Excesso de retries | Sobrecarga em cascata | Backoff/jitter + limite + Circuit Breaker + Bulkhead |
| Compensação parcial | Estoque ou financeiro divergente | Estado `COMPENSATING_*`, retries, DLQ e reconciliação |
| Reordenação de respostas | Estado inválido | commandId/correlationId + versão/passos esperados |
| Dupla reserva do último item | Overselling | Atualização de estoque condicional, transação local e concorrência |
| Reserva expira durante pagamento | Pedido pago sem estoque | TTL supervisionado pelo fluxo + confirmação da situação antes de liberar |
| Estorno aprovado sem ACK | Duplo estorno | Chave idempotente do gateway + consulta do estorno |

## ADR-009 — Validação arquitetural (casos de teste)

| Caso | Injeção de falha / evento | Resultado esperado |
|---|---|---|
| T01 — compra normal | Todas as etapas respondem | `COMPLETED`, Order `CONFIRMED`, estoque `COMMITTED`, pagamento `CAPTURED` |
| T02 — recusa financeira | `PaymentDeclined` após `StockReserved` | `StockReleased`, `OrderCancelled`, `COMPENSATED`, **sem estorno** |
| T03 — falta de estoque | `OutOfStock` | Cancelar pedido sem chamar gateway |
| T04 — duas mensagens iguais | Reentregar `ReserveStock` | Uma única reserva |
| T05 — reenvio da cobrança | Crash antes do ACK de `ChargePayment` | Uma única cobrança no gateway |
| T06 — queda após Outbox | Persistir e reiniciar relay antes de publicar | Comando/evento publicado após retomada |
| T07 — timeout do gateway | Captura ocorre, resposta desaparece | `RECONCILING` → consulta → retoma, sem nova cobrança |
| T08 — commit de estoque falha | Pagamento já capturado; reserva não consolidada | `RefundPayment → ReleaseStock → CancelOrder` |
| T09 — confirmar pedido falha | Estoque `COMMITTED`, pedido não confirmado | `RestoreStock → RefundPayment → CancelOrder` |
| T10 — compensação falha | `ReleaseStock` indisponível | Saga fica pendente, vai a DLQ/alerta; não finaliza como compensada |
| T11 — orquestrador duplicado | Dois workers recebem mesmo resultado | Apenas um avança a versão da instância |
| T12 — estorno sem resposta | Gateway estorna, ACK se perde | Consulta e marca estornado, sem repetir com chave nova |

## Referências

1. Richardson, Chris. [Pattern: Saga](https://microservices.io/patterns/data/saga.html).
2. AWS. [Saga orchestration pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-orchestration.html).
3. AWS. [Saga choreography pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/saga-choreography.html).
4. AWS. [Transactional outbox pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/transactional-outbox.html).
5. AWS. [Retry with backoff pattern](https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/retry-backoff.html).
