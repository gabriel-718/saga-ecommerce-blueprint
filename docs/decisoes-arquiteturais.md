# Decisões arquiteturais e resiliência

## 1. Saga por Orquestração

O processo de compra precisa coordenar três operações independentes: criar o pedido, reservar os produtos e processar o pagamento. Cada serviço é responsável por suas próprias transações, e a consistência entre eles é alcançada por uma Saga.

Foi adotada a **Orquestração**, em que o `Saga Orchestrator` envia comandos, acompanha os resultados e inicia as compensações. A Coreografia também permitiria integrar os serviços, mas, neste fluxo, distribuir as decisões de pagamento e cancelamento por vários consumidores aumentaria a dificuldade de acompanhar uma compra do início ao fim.

O custo dessa escolha é manter mais um componente. Para não depender de uma única instância, o orquestrador pode trabalhar com réplicas, desde que o estado seja persistido e cada transição seja protegida por controle de versão. Duas réplicas podem receber mensagens relacionadas ao mesmo pedido, mas apenas uma deve conseguir confirmar a mudança de estado.

## 2. Bancos de dados e limites dos serviços

Cada serviço mantém seu banco PostgreSQL, sem acesso direto às tabelas dos demais.

| Banco | Responsável | Registros principais |
| --- | --- | --- |
| Order DB | Order Service | `orders`, `idempotency_requests`, `inbox`, `outbox` |
| Inventory DB | Inventory Service | `inventory`, `reservations`, `inbox`, `outbox` |
| Payment DB | Payment Service | `payments`, `refunds`, `inbox`, `outbox` |
| Saga DB | Saga Orchestrator | `saga_instances`, `saga_steps`, `inbox`, `outbox` |

As alterações de estoque devem usar transações locais e atualização condicional da quantidade disponível. Assim, se duas compras disputarem a última unidade de um produto, apenas uma reserva poderá ser confirmada.

## 3. Mensageria, Outbox e Inbox

O RabbitMQ transporta os comandos e resultados entre os serviços. As filas de trabalho críticas são duráveis, preferencialmente do tipo **quorum**, com confirmação de publicação (`publisher confirms`) e confirmação de consumo (`manual ack`). O encaminhamento de mensagens sem rota também deve ser monitorado.

A publicação usa o padrão **Transactional Outbox**: a alteração de negócio e a mensagem a publicar são gravadas na mesma transação local. Depois, um processo de publicação (*Outbox Relay*) envia a mensagem ao RabbitMQ. Se o Relay parar depois de publicar, mas antes de marcar o registro como enviado, a mensagem poderá ser repetida.

Por esse motivo, a aplicação assume entrega **at-least-once**. Cada consumidor mantém uma Inbox para reconhecer `messageId` já processado, além de chaves de negócio para impedir efeitos duplicados, mesmo que duas mensagens diferentes solicitem a mesma operação. O `ACK` é enviado somente depois do commit local.

Não se pressupõe ordem global de mensagens. Os eventos são aplicados apenas quando correspondem a uma transição válida para o estado e a versão atuais do pedido ou da Saga.

## 4. Reserva e alocação de estoque

A reserva começa como `HELD` e tem um prazo de validade. A quantidade reservada deixa de estar disponível para novas compras. Se o prazo vencer, o Inventory Service pode mudar a reserva para `EXPIRED` e devolver a quantidade ao saldo disponível.

Depois da cobrança confirmada, o orquestrador envia `AllocateStock`. A mudança `HELD → ALLOCATED` só acontece se a reserva ainda estiver válida. A verificação do prazo e a atualização precisam ocorrer na mesma transação, evitando que expiração e alocação sejam confirmadas ao mesmo tempo.

Uma reserva `ALLOCATED` deixa de depender do prazo de uma reserva temporária, mas ainda não pode ser expedida enquanto o pedido não estiver `CONFIRMED`. Se houver falha definitiva antes da confirmação, `ReleaseStock` pode liberar essa alocação. A operação deve devolver a quantidade apenas uma vez, inclusive quando houver mensagens repetidas.

## 5. Idempotência financeira

Toda cobrança utiliza um `paymentOperationId` estável, vinculado a valor, moeda, pedido e referência de pagamento imutáveis. O Payment Service registra a tentativa antes de chamar o gateway e usa a chave de idempotência oferecida pelo provedor.

Quando a resposta não chega, a prioridade é consultar a operação existente; não se deve emitir outra cobrança sem confirmar o que aconteceu com a primeira. A retenção das chaves e a possibilidade de consulta dependem do gateway contratado. Se o provedor não permitir identificar o resultado com segurança, o pedido permanece em reconciliação e exige acompanhamento.

O estorno tem sua própria chave, `refundOperationId`, ligada à cobrança original. `RefundPending` significa que o pedido de estorno ainda não foi concluído; apenas um resultado financeiro confirmado permite considerar a compensação dessa etapa como encerrada. Isso não implica necessariamente crédito imediato na fatura do cliente.

## 6. Timeouts e Retry

Os limites abaixo são valores iniciais para dimensionamento. Precisam ser ajustados conforme a latência dos serviços e o acordo de nível de serviço do gateway.

| Operação | Prazo inicial | Reação ao timeout |
| --- | --- | --- |
| `ReserveStock` | 15 s | Consultar a reserva e repetir o comando com a mesma chave, se necessário |
| `ChargePayment` | 30 s | Consultar a transação por `paymentOperationId`; não repetir cobrança às cegas |
| `AllocateStock` | 15 s | Consultar a reserva e verificar se ainda pode ser alocada |
| `ConfirmOrder` | 15 s | Consultar o pedido antes de iniciar qualquer estorno |
| `RefundPayment` | 30 s | Consultar o estorno e manter a Saga pendente até obter confirmação |
| `ReleaseStock` | 15 s | Consultar reserva e repetir liberação idempotente |
| `CancelOrder` | 15 s | Consultar o pedido e repetir cancelamento idempotente |
| Gateway HTTPS | 3 s conexão / 10 s resposta | Tratar falha de rede como resultado incerto até consultar o provedor |

Falhas técnicas transitórias podem receber até três tentativas com intervalos crescentes, por exemplo 1, 2 e 4 segundos, acrescidos de *jitter*. O agendamento deve acontecer em filas de atraso ou no controlador da Saga, sem manter o consumidor bloqueado. Recusa de cartão, estoque insuficiente e erro de validação são resultados de negócio e não entram nesse Retry.

Os prazos e as tentativas ficam registrados no Saga DB. Após esgotar o limite de recuperação automática, a Saga pode ir para `ACTION_REQUIRED`, preservando as pendências para tratamento operacional. Um timeout nunca deve ser convertido automaticamente em confirmação ou recusa de pagamento.

## 7. Circuit Breaker

O **Circuit Breaker** protege a conexão HTTPS do Payment Service com o gateway. Quando há falhas técnicas sucessivas, o circuito abre temporariamente, evitando novas chamadas que provavelmente falhariam. Depois de um intervalo, libera uma tentativa controlada para verificar a recuperação.

Como configuração inicial, pode-se avaliar a abertura após cinco falhas técnicas em trinta segundos e nova tentativa após sessenta segundos. Esses valores são parâmetros de projeto, não números medidos. Uma recusa legítima do cartão não conta como indisponibilidade técnica.

## 8. Dead-Letter Queue e recuperação

As mensagens que não podem ser processadas após as tentativas permitidas são encaminhadas para uma **Dead-Letter Queue (DLQ)**. É possível separar comandos, eventos e compensações para facilitar a investigação.

| Fila | Mensagens | Tratamento |
| --- | --- | --- |
| `saga.commands.dlq` | Comando inválido ou com falha persistente | Verificar contrato, destinatário e estado da operação |
| `saga.events.dlq` | Evento que não pôde ser processado | Conferir identificadores, versão e estado da Saga |
| `saga.compensation.dlq` | Estorno, liberação ou cancelamento não concluído | Priorizar reconciliação e manter a compensação pendente |

No RabbitMQ, o encaminhamento usa uma *Dead-Letter Exchange* vinculada às filas de destino. Para obter encaminhamento **at-least-once** com quorum queues, é preciso configurar a política correspondente, incluindo `dead-letter-strategy=at-least-once` e `overflow=reject-publish`, conforme os requisitos da versão utilizada.

A existência de uma mensagem na DLQ não significa que o pedido foi compensado. O Saga DB conserva o estado pendente. Depois de identificar a causa do problema e consultar a situação real nos serviços, a operação pode ser reprocessada com a mesma chave de negócio, sem duplicar seus efeitos.

## 9. Ordem das compensações

As compensações dependem de quais etapas foram realmente concluídas:

| Situação | Ações |
| --- | --- |
| Estoque indisponível antes da cobrança | `CancelOrder` |
| Pagamento recusado, sem cobrança | `ReleaseStock` → `CancelOrder` |
| Cobrança aprovada, mas alocação ou confirmação falhou definitivamente | `RefundPayment` → aguardar `PaymentRefunded` → `ReleaseStock` → `CancelOrder` |
| Resultado de pagamento ou confirmação desconhecido | Consultar/reconciliar antes de decidir a compensação |

Uma Saga só passa para `COMPENSATED` quando as compensações necessárias estiverem confirmadas. Se a confirmação do pedido já foi persistida, uma devolução ou cancelamento posterior pertence a outro processo de negócio, com suas próprias regras.

## 10. Monitoramento e segurança

As mensagens e os logs carregam `sagaId`, `orderId`, `correlationId` e `traceId`. Esses identificadores permitem seguir uma compra por todos os serviços, localizar mensagens duplicadas e descobrir em qual etapa houve atraso.

Entre os indicadores acompanhados estão tempo por etapa, quantidade de Sagas pendentes, falhas por serviço, idade das mensagens na Outbox, volume das DLQs e operações financeiras aguardando reconciliação. Casos em `ACTION_REQUIRED` precisam gerar alerta.

O tráfego utiliza TLS e credenciais específicas para cada serviço. Segredos ficam fora do código e dos logs. Os eventos financeiros transportam apenas referências de pagamento, sem número completo de cartão ou CVV. Também são necessários controle de acesso, política de retenção e evolução compatível dos contratos de mensagens.

## 11. Riscos principais

| Risco | Controle adotado |
| --- | --- |
| Pedido criado e evento não publicado | Outbox com Relay e monitoramento de atraso |
| Duplicação de comandos | Inbox e unicidade por operação de negócio |
| Duas compras usando a última unidade | Reserva atômica no Inventory Service |
| Cobrança confirmada, mas resposta perdida | Consulta pelo identificador financeiro antes de tentar novamente |
| Reserva expirada após cobrança | Alocação condicional e estorno quando a compra não pode prosseguir |
| Orquestrador indisponível | Estado persistente, réplicas e retomada de prazos |
| Mensagem de compensação sem processamento | DLQ, alerta e reconciliação com a Saga ainda aberta |

## Referências

- RICHARDSON, Chris. [Saga](https://microservices.io/patterns/data/saga.html), [Transactional Outbox](https://microservices.io/patterns/data/transactional-outbox.html) e [Idempotent Consumer](https://microservices.io/patterns/communication-style/idempotent-consumer.html). Microservices.io.
- RABBITMQ. [Quorum Queues](https://www.rabbitmq.com/docs/quorum-queues) e [Dead Letter Exchanges](https://www.rabbitmq.com/docs/dlx).
- AMAZON WEB SERVICES. [Timeouts, retries, and backoff with jitter](https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/).
- MICROSOFT. [Circuit Breaker pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker).
- STRIPE. [Idempotent requests](https://docs.stripe.com/api/idempotent_requests).
