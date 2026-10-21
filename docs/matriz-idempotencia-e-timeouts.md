# Matriz de decisão: duplicatas, timeouts e estados da Saga

## 1. Matriz de idempotência

| Operação | Chave estável | Proteção técnica | Se a mensagem repetir | Falha ambígua |
| --- | --- | --- | --- | --- |
| `CreateOrder` | `customerId + Idempotency-Key` | Unicidade `idempotency_requests` e hash do payload | Retorna `orderId` original | Consultar pedido por chave |
| `OrderCreated` | `orderId` | `UNIQUE(saga_instances.orderId)` + Inbox | Não inicia segunda Saga | Reconciliar Saga ausente |
| `ReserveStock` | `orderId + reservationOperationId` | Unicidade `reservations.operationId` + atualização condicional do estoque | Retorna a mesma reserva | Consultar reserva e validade |
| `StockReserved` | `messageId` + `reservationId` | Inbox do orquestrador + validação de estado/versão | Não envia segunda cobrança | Inspecionar `reservationId` |
| `AllocateStock` | `reservationId + allocationOperationId` | Transição atômica `HELD → ALLOCATED` com prazo ainda válido | Retorna a mesma alocação sem segundo débito | Consultar reserva; se `EXPIRED` e houve cobrança, estornar |
| `ChargePayment` | `paymentOperationId` | Unicidade local + chave idempotente externa | Não gera segunda cobrança | **Consultar gateway** antes de repetir |
| `PaymentDeclined` | `messageId` + `paymentOperationId` | Inbox + estado | Uma sequência de compensação | Confirmar que não houve cobrança |
| `ReleaseStock` | `reservationId` | Transição `HELD/ALLOCATED → RELEASED` condicional; `EXPIRED` já liberou saldo | Não devolve estoque duas vezes | Consultar estado da reserva |
| `CancelOrder` | `orderId + cancelOperationId` | Transição `PENDING → CANCELLED` condicional | Não cancela duas vezes | Consultar estado do pedido |
| `RefundPayment` | `refundOperationId` | Unicidade + idempotência/consulta no gateway | Não estorna duas vezes | Consultar estorno |
| `OrderConfirmed` | `messageId` + `orderId` | Inbox + estado terminal | Não reabre/recancela pedido | Reconsultar Order Service |

A Inbox identifica a repetição de uma mensagem pelo `messageId`. Já a chave da operação evita repetir o mesmo efeito comercial quando o comando recebe outro identificador de mensagem.

## 2. Regra de decisão por resultado observado

| Situação | O que o orquestrador deve fazer | Estado até resolução | Critério de saída |
| --- | --- | --- | --- |
| `StockReserved` | Avançar para cobrança | `WAITING_PAYMENT` | Receber resultado financeiro |
| `StockReservationRejected` definitivo | `CancelOrder` | `WAITING_CANCEL` | `OrderCancelled` |
| `PaymentCharged` | `AllocateStock` | `WAITING_ALLOCATE` | `StockAllocated` ou rejeição definitiva |
| `StockAllocated` | `ConfirmOrder` | `WAITING_ORDER` | `OrderConfirmed` |
| `StockAllocationRejected` após cobrança | `RefundPayment` → `ReleaseStock` → `CancelOrder` | `WAITING_REFUND` | `PaymentRefunded` e demais resultados comprovados |
| `PaymentDeclined` definitivo | `ReleaseStock` → `CancelOrder` | `WAITING_RELEASE` | Ambos os eventos confirmados |
| Timeout em `ReserveStock` | Consultar reserva; eventualmente reemitir comando idempotente | `RECONCILING` / `WAITING_STOCK` | Reserva comprovada, rejeitada ou liberação confirmada |
| Timeout em `ChargePayment` | Consultar status financeiro por ID; **não cobrar novamente por padrão** | `RECONCILING` | `CHARGED` ou `DECLINED` confirmado |
| Timeout em `AllocateStock` | Consultar estado; repetir comando idempotente só com guarda de validade | `RECONCILING` / `WAITING_ALLOCATE` | `ALLOCATED`, `EXPIRED` ou falha definitiva |
| Timeout em `ConfirmOrder` | Consultar pedido primeiro; não estornar por silêncio | `RECONCILING` / `WAITING_ORDER` | `CONFIRMED`, `PENDING` ou rejeição final validada |
| `RefundPending` | Aguardar/consultar o provedor | `RECONCILING` / `WAITING_REFUND` | `REFUNDED` comprovado |
| Timeout em `ReleaseStock` | Consultar reserva; repetir liberação se necessário | `WAITING_RELEASE` | `RELEASED` comprovado |
| Timeout em `CancelOrder` | Consultar pedido e repetir comando idempotente | `WAITING_CANCEL` | `CANCELLED` comprovado |
| Erro posterior com cobrança efetiva | `RefundPayment` → `ReleaseStock` → `CancelOrder` | `WAITING_REFUND` | Todas as compensações confirmadas |
| Retentativas esgotadas / DLQ | Gerar alerta, preservar estado e abrir reconciliação manual | `ACTION_REQUIRED` | Operador comprova e registra resolução |

## 3. Esquema lógico da tabela `saga_instances`

| Campo | Finalidade |
| --- | --- |
| `saga_id` (PK) | Identificador durável do fluxo |
| `order_id` (UNIQUE) | Impede Sagas duplicadas por pedido |
| `state` | Estado atual e etapa de espera/compensação |
| `current_step` | `RESERVE_STOCK`, `CHARGE_PAYMENT`, `ALLOCATE_STOCK`, `CONFIRM_ORDER` ou compensação |
| `reservation_id` / `reservation_expires_at` | Referência para liberação/consulta do estoque e janela de validade |
| `allocation_operation_id` | Identificador estável da promoção de `HELD` para `ALLOCATED` |
| `compensation_required` | Etapas ainda exigidas (estorno, liberação, cancelamento) |
| `resume_state` | Estado anterior à reconciliação, para recuperar o fluxo sem perder contexto |
| `payment_operation_id` | Chave estável da cobrança no gateway |
| `payment_transaction_id` | Referência do efeito financeiro efetivo, quando conhecido |
| `refund_operation_id` | Chave estável do estorno, quando necessário |
| `deadline_at` | Limite da etapa atual para detecção de timeout |
| `retry_count` / `next_retry_at` | Orçamento e agendamento de nova tentativa |
| `last_error_code` | Diagnóstico e decisão |
| `version` | Controle otimista de concorrência |
| `created_at` / `updated_at` | Auditoria e métricas |

A tabela `saga_steps(saga_id, step_name, operation_id, status, attempt, last_updated_at)` mantém o histórico das etapas. Uma atualização usa predicado `WHERE saga_id = ? AND version = ?` e incrementa `version` ao confirmar. Em cenário com múltiplas instâncias, uma única transição vence; as demais recarregam o estado antes de decidir. A Inbox e a Outbox participam da **mesma transação local** do orquestrador.

## 4. Rotina de timeout e recuperação

1. Um agendador consulta Sagas **não terminais** cujo `deadline_at` venceu, aplicando bloqueio/versão para evitar varreduras concorrentes.
2. Revalida o estado antes de decidir, pois uma resposta pode ter sido persistida entre a leitura e o timer.
3. Consulta o **dono da transação local**: Inventory Service, Payment Service ou Order Service, usando identificador estável.
4. Se o resultado foi efetivado, registra o evento/reconciliação e avança; se negado em definitivo, inicia a compensação aplicável.
5. Se o resultado for desconhecido, marca `RECONCILING`, agenda backoff limitado com jitter e, esgotado o orçamento, marca `ACTION_REQUIRED` com alerta.
6. Não conclui `COMPLETED` ou `COMPENSATED` a partir de silêncio da rede. O encerramento exige **resultado comprovado** das etapas correspondentes.

## 5. Eventos tardios e transições inválidas

- `PaymentCharged` atrasado após uma decisão de cancelamento **não** deve ser ignorado como se nada tivesse ocorrido; deve abrir trilha de reconciliação financeira e, se cobrança comprovada, coordenar `RefundPayment`.
- `StockReserved` recebido após início do cancelamento deve provocar verificação/liberação idempotente dessa reserva.
- `OrderConfirmed` em uma Saga já compensada exige incidente de reconciliação; o serviço de pedidos não deve aceitar `ConfirmOrder` quando já está `CANCELLED`.
- Estado terminal não pode voltar a `PENDING` por evento duplicado. Alertar para estados contraditórios.

As reservas `HELD` têm prazo de validade. Antes de cobrar, o orquestrador precisa considerar se a reserva ainda pode ser utilizada. Depois que a reserva se torna `ALLOCATED`, ela não deve ser liberada por um timer de reserva temporária.

## 6. Regras complementares

- Para qualquer comando idempotente, persistir `operationId`, **hash normalizado de seus parâmetros** e resultado. Mesma chave com parâmetros diferentes exige conflito; não reinterpretar operação financeira com outros valores.
- Repetir `AllocateStock` não rebaixa disponibilidade e não prolonga reservas expiradas; ele consulta o status da mesma reserva.
- Eventos `ReservationExpired` concorrentes com `PaymentCharged` exigem **reconciliação**: jamais emitir `ConfirmOrder` sem `StockAllocated` confirmado.
- Uma chave de idempotência externa pode ter **prazo de retenção** determinado pelo gateway. Passado esse prazo, reutilizar a chave sem verificar a transação real pode gerar outra cobrança; priorizar referência do provedor e escalonamento.
- Inbox e Outbox **não** tornam instantaneamente consistentes os bancos independentes. Uma Saga pode ficar `PENDING`, `RECONCILING` ou `ACTION_REQUIRED` enquanto a decisão financeira é apurada.
- `COMPENSATED` só é permitido após conferir `refund_confirmed` (quando exigido), `release_confirmed` (quando houve reserva) e `cancel_confirmed`; `RefundPending` e DLQ não bastam.
