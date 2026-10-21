# Máquina de estados, invariantes e política de compensação

As transições abaixo definem quando o orquestrador pode avançar uma compra e quando precisa iniciar uma compensação. Cada serviço continua responsável por validar as alterações no próprio banco.

## 1. Invariantes de negócio

1. **I1 — Pagamento único:** a mesma intenção de cobrança (`paymentOperationId`) não gera duas cobranças econômicas; idempotência externa deve ser confirmada com o gateway contratado. Retenção expirada da chave exige busca por referência comercial e reconciliação.
2. **I2 — Não vender estoque inexistente:** reservas decrementam `available_quantity` com atualização condicional atômica; não existe disponibilidade negativa.
3. **I3 — Confirmação exige evidências:** `ConfirmOrder` só é emitido após **cobrança comprovada** e **alocação `ALLOCATED` confirmada** para a reserva vinculada ao mesmo pedido.
4. **I4 — Alocação não é expedição:** `ALLOCATED` significa quantidade comprometida sem TTL; pedido só é liberado para expedição após `OrderConfirmed`. Antes disso, `ReleaseStock` é compensação permitida mesmo para `ALLOCATED`.
5. **I5 — Compensação tem efeitos observáveis:** `COMPENSATED` exige cancelamento comprovado e, se aplicável, liberação e estorno confirmados. DLQ, agendamento de retry ou `RefundPending` não equivalem à compensação concluída.
6. **I6 — Timeout não é estado de pagamento:** silêncio do gateway leva a `RECONCILING`, não a `PaymentDeclined` nem a `PaymentCharged`.
7. **I7 — Estados terminais protegidos:** `CONFIRMED` não aceita `CancelOrder` normal; `CANCELLED` não aceita `ConfirmOrder`. Qualquer contradição vira incidente com reconciliação, não mutação silenciosa.
8. **I8 — Resultado financeiro tardio não desaparece:** mesmo que uma Saga entre em compensação, `PaymentCharged` tardio exige verificação e eventual estorno; nunca descartar apenas porque o estado não é o esperado.

## 2. Estados locais (fonte de verdade por domínio)

| Domínio | Estados principais | Regras |
| --- | --- | --- |
| Order | `PENDING`, `CONFIRMED`, `CANCELLED` | `PENDING → CONFIRMED` só com ordem de orquestração e evidências; `PENDING → CANCELLED`; transições terminais não se revertem pelo fluxo normal |
| Inventory | `HELD`, `ALLOCATED`, `RELEASED`, `EXPIRED` | `HELD → ALLOCATED` somente se `expires_at > db_now` sob lock; `HELD/ALLOCATED → RELEASED` enquanto não expedido; `HELD → EXPIRED` ao vencer; `RELEASED/EXPIRED` não liberam saldo de novo |
| Payment | `PENDING`, `UNKNOWN`, `DECLINED`, `CHARGED`, `REFUND_PENDING`, `REFUNDED` | `DECLINED` apenas conclusão financeira definitiva; `UNKNOWN` exige consulta; `REFUND_PENDING` não equivale a `REFUNDED` |

**Modelo de saldo:** `available_quantity` representa unidades livres para novas compras. Uma reserva `HELD` reduz disponibilidade **uma vez**; a transição para `ALLOCATED` **não** reduz novamente. `ReleaseStock` restaura a disponibilidade **uma vez** para uma reserva ainda retida/alocada. `EXPIRED` devolve disponibilidade uma vez e uma chamada posterior `ReleaseStock` confirma o resultado anterior sem novo incremento. A baixa de estoque físico na expedição é um processo externo ao escopo da Saga de compra.

A expiração e a alocação concorrem pela **mesma linha** de reserva. Uma atualização condicional, sob controle de concorrência, garante que apenas uma transição seja confirmada.

## 3. Tabela de transições do Saga Orchestrator

A tabela descreve o estado **após emitir um comando** ou processar um resultado. A publicação e a mudança de estado do coordenador entram na **mesma transação local** via Outbox.

| Estado anterior | Evento/condição | Ação única | Próximo estado |
| --- | --- | --- | --- |
| Ausente | `OrderCreated` válido | Criar Saga + Outbox(`ReserveStock`) | `WAITING_STOCK` |
| `WAITING_STOCK` | `StockReserved` | Registrar `reservationId`, `expiresAt`; emitir `ChargePayment` | `WAITING_PAYMENT` |
| `WAITING_STOCK` | `StockReservationRejected` definitivo | Emitir `CancelOrder` | `WAITING_CANCEL` |
| `WAITING_PAYMENT` | `PaymentCharged` comprovado | Registrar `transactionId`; emitir `AllocateStock` | `WAITING_ALLOCATE` |
| `WAITING_PAYMENT` | `PaymentDeclined` definitivo | Emitir `ReleaseStock` | `WAITING_RELEASE` |
| `WAITING_ALLOCATE` | `StockAllocated` | Emitir `ConfirmOrder` | `WAITING_ORDER` |
| `WAITING_ALLOCATE` | `StockAllocationRejected` / reserva expirada | Emitir `RefundPayment` | `WAITING_REFUND` |
| `WAITING_ORDER` | `OrderConfirmed` | Marcar conclusão, sem novo comando | `COMPLETED` |
| `WAITING_ORDER` | `OrderConfirmationRejected` definitivamente válido, após consulta | Emitir `RefundPayment` | `WAITING_REFUND` |
| `WAITING_REFUND` | `PaymentRefunded` comprovado | Emitir `ReleaseStock` | `WAITING_RELEASE` |
| `WAITING_REFUND` | `RefundPending` | Agendar nova consulta | `RECONCILING` com `resume_state=WAITING_REFUND` |
| `WAITING_RELEASE` | `StockReleased` comprovado (inclusive se a reserva já havia expirado) | Emitir `CancelOrder` | `WAITING_CANCEL` |
| `WAITING_CANCEL` | `OrderCancelled` comprovado | Marcar conclusão compensatória | `COMPENSATED` |
| Qualquer espera | Timeout / resultado desconhecido | Consultar serviço dono sem deduzir falha | `RECONCILING` + `resume_state` |
| `RECONCILING` | Resultado efetivo conhecido | Aplicar transição correspondente, com guarda e versão | Estado correspondente |
| `RECONCILING` | Orçamento de recuperação esgotado | Abrir incidente, preservar pendências | `ACTION_REQUIRED` |
| `ACTION_REQUIRED` | Reconciliação auditada por operador | Retomar ação idempotente apropriada | Estado correspondente |

O estado `WAITING_RELEASE` aparece tanto na recusa de pagamento quanto nas falhas posteriores à cobrança. O coordenador guarda `compensation_required` e uma lista de etapas comprovadas (`refund_required`, `refund_confirmed`, `release_confirmed`, `cancel_confirmed`). Só transita para `COMPENSATED` quando **todas** as compensações requeridas estão confirmadas. Se a Saga foi cancelada por falha de estoque **sem ter reservado**, `release_required=false` e apenas `OrderCancelled` é exigido.

## 4. Casos de corrida e respostas tardias

### Reserva vence enquanto a cobrança está em andamento

A reserva `HELD` pode expirar enquanto `Payment Service` processa a cobrança. O agendador de estoque libera unidades usando transição atômica `HELD → EXPIRED` e publica `ReservationExpired`. Se a cobrança vier aprovada depois, `AllocateStock` falha (pois `EXPIRED` não vira `ALLOCATED`). A Saga deve executar `RefundPayment`, confirmar o estorno, normalizar a liberação de estoque (sem duplicar saldo) e cancelar o pedido. Não pode confirmar o pedido com reserva expirada.

### Resposta de `ConfirmOrder` se perde

O Order Service pode ter persistido `CONFIRMED`, embora o evento esteja atrasado. Em timeout, o orquestrador **consulta o pedido** e só estorna após comprovar impossibilidade definitiva de confirmar. Um `OrderConfirmed` tardio requer conciliação e não autoriza o estado `COMPENSATED` contraditório.

### Resultado financeiro chega após início de cancelamento

Se `PaymentCharged` aparece após `PaymentDeclined` aparente, a Saga abre incidente de correlação; verifica a situação no gateway e **não finaliza** compensação sem assegurar ausência de cobrança ou estorno confirmado. Para evitar que a Saga já tenha sido encerrada antes do evento, `PaymentDeclined` só é aceito como definitivo quando a integração tem prova de recusa final do provedor para aquela operação. Um evento terminal contraditório após conclusão também dispara alerta e reconciliação fora de banda.

### Falha após alocação ou confirmação

Uma falha definitiva **antes** de `OrderConfirmed` permite `RefundPayment → ReleaseStock (ALLOCATED) → CancelOrder`. Depois do pedido `CONFIRMED`, não se executa automaticamente esta Saga reversa: cancelamento/devolução de compra confirmada constitui **outro processo de negócio**, com políticas próprias (faturamento, separação e envio). Isso evita 'desconfirmar' um pedido já concluído.

## 5. Exemplo: pagamento recusado

```text
Order(PENDING)
  → ReserveStock → StockReserved(HELD)
  → ChargePayment → PaymentDeclined(DECLINED final; sem cobrança)
  → ReleaseStock → StockReleased(RELEASED)
  → CancelOrder → OrderCancelled(CANCELLED)
  → Saga(COMPENSATED)
```

Nesse caminho não se executa `RefundPayment`, pois o gateway confirmou a recusa sem efetuar cobrança. Se o resultado financeiro for desconhecido, o fluxo permanece em reconciliação até que a situação seja esclarecida.
