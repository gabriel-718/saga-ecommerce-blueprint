# Contratos de comunicação — compras com Saga

## 1. Convenções

**Protocolo externo:** HTTP/JSON, com `POST /orders`, `GET /orders/{orderId}`.  
**Integração interna:** comandos e eventos JSON no RabbitMQ (exchanges `commerce.commands` e `commerce.events`, com routing keys por domínio e ação).  
**Persistência:** um Outbox Relay entrega os registros confirmados pelo banco; a resposta de cada consumidor também passa pela Outbox.

| Categoria | Comandos | Eventos/resultados |
| --- | --- | --- |
| Pedido | `ConfirmOrder`, `CancelOrder`, `CheckOrderStatus` | `OrderCreated`, `OrderConfirmed`, `OrderConfirmationRejected`, `OrderCancelled`, `OrderStatusKnown` |
| Estoque | `ReserveStock`, `AllocateStock`, `ReleaseStock`, `CheckReservationStatus` | `StockReserved`, `StockReservationRejected`, `StockAllocated`, `StockAllocationRejected`, `StockReleased`, `ReservationExpired`, `ReservationStatusKnown` |
| Pagamento | `ChargePayment`, `RefundPayment`, `CheckPaymentStatus`, `CheckRefundStatus` | `PaymentCharged`, `PaymentDeclined`, `PaymentRefunded`, `RefundPending`, `PaymentStatusKnown`, `PaymentStatusUnknown`, `RefundStatusKnown` |

Os comandos solicitam operações; os eventos informam o resultado persistido pelo serviço responsável. Uma falha de rede não deve ser publicada como recusa definitiva de pagamento.

## 2. API do cliente

**Requisição (exemplo):**

```http
POST /orders HTTP/1.1
Content-Type: application/json
Idempotency-Key: checkout-8d5d1c

{
  "customerId": "customer-109",
  "items": [{ "sku": "SKU-ABC", "quantity": 2 }],
  "paymentToken": "tok_provider_example"
}
```

**Resposta:** `202 Accepted`, `Location: /orders/ord-4001`, corpo:

```json
{ "orderId": "ord-4001", "status": "PENDING", "statusUrl": "/orders/ord-4001" }
```

O `GET /orders/ord-4001` retorna `PENDING`, `CONFIRMED` ou `CANCELLED` de acordo com o estado persistido. Um mesmo `Idempotency-Key` dentro do escopo `(customerId, chave)` retorna o mesmo resultado; payload divergente retorna `409 Conflict`.

## 3. Envelope obrigatório de mensagens

```json
{
  "messageId": "msg-101",
  "schemaVersion": 1,
  "type": "ReserveStock",
  "sagaId": "saga-4001",
  "orderId": "ord-4001",
  "correlationId": "checkout-8d5d1c",
  "causationId": "msg-100",
  "operationId": "reserve-ord-4001",
  "occurredAt": "2026-10-21T12:00:00Z",
  "payload": {
    "items": [{ "sku": "SKU-ABC", "quantity": 2 }],
    "reservationExpiresAt": "2026-10-21T12:15:00Z"
  }
}
```

O exemplo usa identificadores ilustrativos e horários em UTC. O canal define o destinatário, e `schemaVersion` permite a evolução dos contratos. Para resposta de um participante, `causationId` deve apontar para o comando recebido; `messageId` é novo, e o `operationId` permanece o mesmo.

## 4. Contratos por etapa

| Emissor → Consumidor | Mensagem | Conteúdo mínimo de `payload` | Pós-condição / efeito único |
| --- | --- | --- | --- |
| Order → Saga | `OrderCreated` | `orderId`, `items`, `amount`, `currency`, `paymentTokenRef` | Saga criada só uma vez por `orderId` |
| Saga → Inventory | `ReserveStock` | `items`, `reservationExpiresAt` | Uma reserva `HELD` por `orderId`/operação |
| Inventory → Saga | `StockReserved` | `reservationId`, `items`, `expiresAt` | Saga pode avançar para cobrança |
| Inventory → Saga | `StockReservationRejected` | `reasonCode` | Saga pode cancelar pedido sem cobrar |
| Saga → Payment | `ChargePayment` | `paymentOperationId`, `amount`, `currency`, `paymentTokenRef` | No máximo uma cobrança econômica por operação |
| Payment → Saga | `PaymentCharged` | `paymentOperationId`, `transactionId`, `amount` | Saga solicita `AllocateStock` antes de `ConfirmOrder` |
| Payment → Saga | `PaymentDeclined` | `paymentOperationId`, `reasonCode`, `final=true` | Saga inicia liberação e cancelamento |
| Saga → Inventory | `AllocateStock` | `reservationId`, `orderId`, `allocationOperationId` | `HELD → ALLOCATED` apenas com reserva válida, atômica e ainda não expirada |
| Inventory → Saga | `StockAllocated` | `reservationId`, `allocatedAt` | Libera a emissão de `ConfirmOrder` |
| Inventory → Saga | `StockAllocationRejected` | `reservationId`, `reasonCode` | Se houve cobrança, deve ser estornada antes do cancelamento |
| Inventory → Saga | `ReservationExpired` | `reservationId`, `expiredAt` | Correlacionar Saga e verificar operação financeira pendente |
| Saga → Order | `ConfirmOrder` | `orderId`, `paymentOperationId`, `reservationId` | Pedido `CONFIRMED` uma vez |
| Order → Saga | `OrderConfirmed` | `orderId`, `confirmedAt` | Saga `COMPLETED` |
| Order → Saga | `OrderConfirmationRejected` | `orderId`, `reasonCode`, `final=true` | Só compensar após rejeição final comprovada, não por timeout |
| Saga → Inventory | `ReleaseStock` | `reservationId`, `reasonCode` | Estado `HELD/ALLOCATED → RELEASED` no máximo uma vez; `EXPIRED` já teve saldo devolvido |
| Inventory → Saga | `StockReleased` | `reservationId`, `releasedAt` | Compensação avança para cancelamento |
| Saga → Order | `CancelOrder` | `orderId`, `reasonCode` | Pedido `PENDING → CANCELLED` uma vez |
| Order → Saga | `OrderCancelled` | `orderId`, `cancelledAt` | Saga `COMPENSATED` (se demais compensações concluídas) |
| Saga → Payment | `RefundPayment` | `transactionId`, `refundOperationId`, `amount` | Estorno idempotente (somente cobrança confirmada) |
| Payment → Saga | `PaymentRefunded` | `refundOperationId`, `refundTransactionId` | Pode prosseguir para liberação do estoque |
| Payment → Saga | `RefundPending` | `refundOperationId`, `providerStatus` | Não equivale a estorno concluído; manter reconciliação |
| Saga → Inventory | `CheckReservationStatus` | `reservationId` | Consulta estado/expiração sem mutação |
| Inventory → Saga | `ReservationStatusKnown` | `reservationId`, `status`, `expiresAt` | Resultado para reconciliar a reserva |
| Saga → Payment | `CheckPaymentStatus` | `paymentOperationId` | Consulta técnica, sem efetuar nova cobrança |
| Payment → Saga | `PaymentStatusKnown` / `PaymentStatusUnknown` | `paymentOperationId`, `status` | Resultado conhecido ou ainda incerto no gateway |
| Saga → Payment | `CheckRefundStatus` | `refundOperationId` | Consulta do estorno sem criar outra solicitação |
| Payment → Saga | `RefundStatusKnown` | `refundOperationId`, `status` | Resultado da consulta de estorno |
| Saga → Order | `CheckOrderStatus` | `orderId` | Consulta o estado persistido do pedido |
| Order → Saga | `OrderStatusKnown` | `orderId`, `status` | Permite confirmar se `ConfirmOrder` foi efetivado |

## 5. Exemplo de recusa

```json
{
  "messageId": "msg-205",
  "schemaVersion": 1,
  "type": "PaymentDeclined",
  "sagaId": "saga-4001",
  "orderId": "ord-4001",
  "correlationId": "checkout-8d5d1c",
  "causationId": "msg-204",
  "operationId": "charge-ord-4001",
  "occurredAt": "2026-10-21T12:00:08Z",
  "payload": {
    "paymentOperationId": "charge-ord-4001",
    "reasonCode": "CARD_DECLINED",
    "final": true
  }
}
```

Nenhum dado sensível de cartão é armazenado nos eventos. O `Payment Service` registra o diagnóstico no próprio domínio, distinguindo `DECLINED` (falha de negócio definitiva), `UNKNOWN` (resultado ambíguo) e `CHARGED` (efeito comprovado).

## 6. Protocolos de confirmação

1. Consumidor valida schema, correlação e estado permitido.
2. Abre transação local; tenta inserir Inbox `(consumerName,messageId)` e adquirir a transição por versão.
3. Executa a alteração idempotente e insere o resultado na Outbox; `COMMIT`.
4. Só então envia `ACK` ao RabbitMQ. Duplicata de `messageId` é reconhecida sem refazer a ação; duplicata por **chave de negócio** retorna resultado previamente persistido.
5. Relay publica com confirmação; se a publicação ficar incerta, pode republicar. Isso é esperado e protegido pela Inbox.

A chamada ao gateway não participa da transação do Payment DB. Por isso, o Payment Service registra a operação como `PENDING`, com chave estável, antes de chamar o provedor. Após a resposta ou reconciliação, confirma o estado e publica evento. Uma repetição reusa **a mesma** chave, desde que suportada pelo gateway; caso contrário, consulta e escala para atendimento humano em vez de cobrar novamente às cegas.

## 7. Contrato de validade e alocação da reserva

Após `StockReserved`, a reserva fica `HELD` com vencimento persistido. O evento **não** autoriza `ConfirmOrder`. Quando `PaymentCharged` é comprovado, o orquestrador chama `AllocateStock`. O Inventory Service usa controle de concorrência local para verificar `status=HELD` e `expiresAt > now()` no mesmo commit que efetiva `ALLOCATED`. Expiração concorrente vence ou perde a disputa atomicamente. `StockAllocated` torna a alocação não expirável pela política da reserva pendente, mas a operação de expedição continua bloqueada até `OrderConfirmed`.

Uma falha tardia antes de `OrderConfirmed` pode exigir `ReleaseStock` sobre `HELD` **ou** `ALLOCATED`. Reprocessamentos sobre `RELEASED` ou `EXPIRED` não somam unidades novamente. Uma reserva `EXPIRED` não pode ser reativada usando o mesmo `operationId`. Os eventos levam `reservationId`, `orderId` e `sagaId`; respostas atrasadas são conciliadas por identidade e versão.

## 8. Matriz de erros semânticos

| Resultado | É falha definitiva? | Próxima decisão |
| --- | --- | --- |
| `StockReservationRejected` | Sim: sem estoque | `CancelOrder`; não chamar pagamento |
| `PaymentDeclined` com `final=true` | Sim: não houve cobrança | `ReleaseStock` e `CancelOrder` |
| `PaymentStatusUnknown` | **Não** | Consulta por referência; não reenviar cobrança às cegas |
| `StockAllocationRejected` por `EXPIRED` após `PaymentCharged` | Sim para esta reserva | `RefundPayment` → confirmar estorno → normalizar liberação → cancelar |
| `OrderConfirmationRejected` permanente | Somente depois de estado confirmado por Order Service | Se não confirmou, estornar, liberar e cancelar |
| Timeout de `ConfirmOrder` | **Não** | Consultar Order Service; `CONFIRMED` implica concluir, `PENDING` permite retentar idempotente |
| `RefundPending` | **Não** para conclusão do estorno | Consultar até `PaymentRefunded` ou intervenção humana |

O campo `paymentToken` representa uma referência de pagamento. O token original não deve circular livremente entre os serviços; os eventos utilizam `paymentTokenRef`, com proteção adequada para dados pessoais e autorização na consulta de pedidos.
