# Cenários de validação

Os cenários abaixo ajudam a verificar se os serviços mantêm as regras de pedido, estoque e pagamento, inclusive quando há mensagens repetidas, indisponibilidade ou respostas atrasadas. Cada caso apresenta a situação e o resultado esperado.

| ID | Cenário | Situação | Resultado esperado |
| --- | --- | --- | --- |
| T01 | Fluxo completo | Estoque disponível + gateway aprova | `CONFIRMED`, Saga `COMPLETED`, reserva `ALLOCATED` e uma cobrança |
| T02 | Pagamento recusado | Reserva confirmada + `DECLINED` | `StockReleased`, `OrderCancelled`, Saga `COMPENSATED`, sem estorno |
| T03 | Estoque insuficiente | `StockReservationRejected` | Pedido cancelado; gateway não é invocado |
| T04 | `ReserveStock` duplicado | Mensagem repetida com o mesmo `messageId` | Uma reserva e um único débito de disponibilidade |
| T05 | `ReserveStock` reemitido | Mesmo `operationId`, outro `messageId` | Retorna a reserva original, sem novo débito |
| T06 | Resposta de cobrança perdida | Gateway cobrou, mas conexão caiu | Consulta por `paymentOperationId`; nenhuma segunda cobrança |
| T07 | `ReleaseStock` duplicado | Mesmo comando duas vezes | Estoque restaurado uma vez |
| T08 | Reinício do coordenador | Queda depois de persistir comando na Outbox | Retoma pelo DB/Relay sem perder a etapa |
| T09 | Queda do Relay | Publicou e caiu antes de marcar entrega | Evento repetido é absorvido por Inbox/chave de negócio |
| T10 | Falha na compensação | Inventory indisponível durante `ReleaseStock` | Retentativas limitadas, Saga não termina até confirmação, DLQ/alerta |
| T11 | Falha após cobrança | Pedido é rejeitado definitivamente após `PaymentCharged` e `StockAllocated` | `RefundPayment` → `ReleaseStock` → `CancelOrder`; não declarar encerrado antes de estornar |
| T12 | Concorrência de estoque | Duas compras disputam última unidade | Apenas uma reserva aceita; a outra rejeitada sem vender além do estoque disponível |
| T13 | Evento fora de ordem | `PaymentCharged` atrasado após início de cancelamento | Alerta e reconciliação/estorno necessário; não ignorar pagamento |
| T14 | Repetição da requisição HTTP | Mesma `Idempotency-Key` no `POST /orders` | Retorna o mesmo pedido sem criar outra Saga |
| T15 | Chave HTTP com payload alterado | Mesma chave e itens diferentes | `409 Conflict`, sem operação duplicada |
| T16 | Circuit Breaker aberto | Gateway falhando repetidamente | Falha rápida/estado pendente sem tempestade de cobrança |
| T17 | Reserva expira pendente | TTL vencido antes de cobrar | Reconciliar/revalidar reserva, sem cobrar estoque inexistente |
| T18 | Duas instâncias da Saga | Mesmo `sagaId` processado em paralelo | Apenas uma transição efetiva por controle de versão |
| T19 | Alocação após cobrança | `StockReserved` e `PaymentCharged` com reserva válida | `AllocateStock` → `StockAllocated`; só então `ConfirmOrder` |
| T20 | Reserva expira enquanto o gateway processa | `HELD → EXPIRED`, depois `PaymentCharged` | `StockAllocationRejected`, estorno comprovado e cancelamento; nunca `CONFIRMED` |
| T21 | Corrida expiração × alocação | As duas ações tentam mudar a mesma reserva | Apenas uma transição vence; `ALLOCATED` ou `EXPIRED`, nunca os dois |
| T22 | Duplicata de `AllocateStock` | Mesmo `allocationOperationId` entregue duas vezes | Uma alocação, nenhum desconto de saldo adicional |
| T23 | `ReleaseStock` para `ALLOCATED` | Cobrança estornada e pedido ainda `PENDING` | Liberar uma vez a alocação não expedida |
| T24 | `ReleaseStock` após `EXPIRED` | Expiração já devolveu saldo | Confirmar estado equivalente sem devolver unidades novamente |
| T25 | Timeout de `ConfirmOrder` com commit bem-sucedido | Evento `OrderConfirmed` atrasado | Consultar Order Service, encerrar Saga como sucesso; não efetuar refund |
| T26 | Estorno assíncrono | Gateway responde `RefundPending` | Ficar aguardando/reconciliando; não marcar `COMPENSATED` prematuramente |
| T27 | Retenção externa de idempotência vencida | Tentativa de repetição após janela do gateway | Consultar referência comercial antes de qualquer nova cobrança |
| T28 | Evento contraditório tardio | `PaymentCharged` após decisão compensatória | Abrir incidente e avaliar necessidade de estorno; jamais ignorar cobrança |
| T29 | Comando com mesma chave e payload diferente | `ChargePayment` reemitido com valor alterado | Recusar conflito de idempotência, sem débito adicional |
| T30 | Compensação na DLQ | `ReleaseStock` ou `RefundPayment` falham persistentemente | Saga permanece pendente / `ACTION_REQUIRED` com alerta, sem falsa conclusão |

## Como verificar

Em cada cenário, devem ser comparados os registros do Saga DB com os estados do pedido, da reserva e da operação financeira. Os eventos publicados, a Inbox, a Outbox e as filas de mensagens também precisam manter a mesma correlação por `sagaId` e `orderId`.

Nos testes com o gateway, é importante conferir o resultado da operação no ambiente de testes do provedor, especialmente quando uma chamada expira ou quando o estorno é assíncrono. Para os casos de concorrência, é necessário observar se as operações sobre a mesma reserva respeitam as transições permitidas e não alteram o estoque duas vezes.

Os casos que terminam em `ACTION_REQUIRED` devem continuar visíveis para tratamento operacional, sem que a Saga seja marcada como concluída indevidamente.
