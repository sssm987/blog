---
layout: post
title: "[Consistency] Outbox를 이용한 정합성을 보장시키는 법"
date: 2026-08-24 10:00:00 +0900
categories :
  - architecture
description : Consistency에 대해 알아보자
series: consistency
series_order: 5
---

## 개요
저번 시간에는 MQ를 이용한 정합성을 보장하는법을 해봤다.    
MQ를 이용했을때 장점도 있었지만 문제점으로 DB에 쓰기가 성공했지만 메세지 발행에 실패 했을 경우 PG서버와 정합성이 맞지 않는다는 단점이있다.     
그래서 이번시간에는 이 문제점을 보완하기위해 Outbox 패턴을 이용해 정합성을 맞춰보겠다.    

## 시스템 구성
이번 시리즈에서는 간단하게 주문, 결제 도메인을 이용할것이다.    
주문, 결제 도메인은 엄청 복잡한 도메인이다 그렇기에 실무에서는 많은 예외 상황 처리와 많은 검증, 많은 테이블, 컬럼이 필요하지만 이번 시리즈에서는 정합성을 위해 최대한 간단하게 필요한 부분만 구현 할 것 이다.    
(Validation check, Exception 이런것들은 넘어갈것이다) .

시스템 구성은 주문서버와 결제서버와 API통신을 하는 방식으로 했고 결제서버는 Mock서버로 응답값만 넘어오도록 구성했다.    
DB는 주문서버에만 연동하였다.
<img src="{{ '/assets/images/concurrency/0824/configuration_diagram.png' | relative_url }}" alt="ConfigurationDiagram" />

DB테이블에 MQ와 다른점은 재시도/이력 테이블에 재시도 횟수가 빠진것이다.   
빠진 이유는 설명에서 따로 기재하겠다.   
<img src="{{ '/assets/images/concurrency/0824/erd.png' | relative_url }}" alt="erd" />

## 실험 환경
실험 환경은 Spring boot,PostgreSQL이며 모두 맥북에 도커를 이용하여 띄워 테스트 할 것 이다.

실험에서 사용한 전체 코드는
<a href="https://github.com/sssm987/payment/tree/outbox" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(주문 서버)]</a>
<a href="https://github.com/sssm987/pg" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(결제 서버)]</a>

## 설명
우선 기존 코드를 활용하기 위해 클래스와 테이블에는 `retry`라는 명칭을 사용했지만, 이번 구현에서는 메시지 발행을 위한 Outbox 역할로 사용한다.

이전 MQ 방식과 달라진 점은 `OrderUseCase`에서 RabbitMQ 메시지를 직접 발행하지 않는다는 것이다.   
주문 생성 과정에서는 재고 차감, 주문·결제 생성과 Outbox 이벤트 저장을 하나의 DB 트랜잭션으로 처리한다.   
따라서 주문 데이터 저장에 성공하면 발행해야 할 이벤트도 DB에 함께 저장되고, 주문 처리 중 예외가 발생하면 Outbox 이벤트도 같이 롤백된다.

메시지 발행은 스케줄러가 담당한다.   
스케줄러는 주기적으로 `READY` 상태의 Outbox 이벤트를 조회하여 승인 또는 취소 메시지를 RabbitMQ에 발행한다.   
발행에 성공하면 Outbox 상태를 `SUCCESS`로 변경하고, 발행 중 예외가 발생하면 `READY` 상태로 남겨 다음 스케줄에서 다시 발행하도록 구성하였다.   
이번 실험에서는 발행 실패 시 횟수 제한 없이 다시 처리하도록 구성했기 때문에 Outbox 테이블에서 `retry_count`는 제외하였다.

RabbitMQ에 발행된 메시지는 `PaymentConsumer`가 소비하여 승인 또는 취소 로직을 호출한다.   
PG 승인은 성공했지만 주문 완료 DB 처리에 실패한 경우에는 취소 Outbox 이벤트를 새로 생성한다.   
이후 스케줄러가 취소 이벤트를 RabbitMQ에 발행하고, 취소 Consumer가 PG 취소 API와 주문·결제 상태 변경, 재고 원복 등의 보상 처리를 수행한다.

Outbox 메시지 발행 실패에 대한 재시도는 스케줄러가 담당하고, 메시지가 RabbitMQ에 전달된 이후 Consumer 처리 실패에 대한 재시도는 Spring AMQP의 Retry Interceptor가 담당한다.   
Consumer가 설정된 재시도 횟수를 모두 소진하면 PG 승인 여부 확인, 주문·재고 원복 또는 취소 재처리와 같은 별도의 복구 프로세스가 필요하다.   
하지만 이번 글은 Outbox 패턴을 이용해 DB 저장과 메시지 발행 사이의 정합성을 보장하는 과정에 집중하기 위해 Consumer 최종 실패에 대한 복구 처리는 구현 범위에서 제외하였다.

한편 RabbitMQ 메시지 발행에는 성공했지만 Outbox 상태를 `PUBLISHED`로 변경하기 전에 애플리케이션이 종료되면 동일한 이벤트가 다시 발행될 수 있다.   
따라서 Outbox 패턴은 일반적으로 최소 한 번 전달을 전제로 하며, 실제 시스템에서는 이벤트 식별자를 이용해 Consumer를 멱등하게 처리해야 한다.
<img src="{{ '/assets/images/concurrency/0824/configuration_diagram2.png' | relative_url }}" alt="ConfigurationDiagram2" />
OrderTransactionService에 주문 생성 비지니스 로직이다.   
```java
@Transactional
    public void prepareOrder(OrderCreateCmd dto){
        productService.inventoryDeduction(dto.productId());

        long productPrice = productService.findProductPrice(dto.productId());
        long orderId = orderService.orderCreate(dto.memberId(),dto.productId());
        long paymentId = paymentService.paymentCreate(PaymentCreateRequestCmd.builder()
                .memberId(dto.memberId())
                .orderId(orderId)
                .fee(productPrice)
                .build());

        retryHistoryService.retryHistoryApproveCreate(PaymentApproveRetryPayload.builder()
                .orderId(orderId)
                .productPrice(productPrice)
                .paymentId(paymentId)
                .productId(dto.productId())
                .build()
        );
    }
```
스케줄러에서 Outbox테이블을 조회한 후 메세지를 발행하는 로직이다.   
```java
@Component
@RequiredArgsConstructor
public class RetryUseCase {

    private final ObjectMapper objectMapper;
    private final PaymentService paymentService;
    private final RetryHistoryService retryHistoryService;

    public void retryPendingRequests() {
        List<RetryHistory> targets =
                retryHistoryService.findRetryTargets();

        for (RetryHistory history : targets) {
            retry(history);
        }
    }
    private void retry(RetryHistory history) {
        switch (history.getRetryApiType()) {
            case APPROVE -> retryApprove(history);
            case CANCEL -> retryCancel(history);
        }

        retryHistoryService.retryHistorySuccess(history.getId());
    }
    private void retryApprove(RetryHistory history) {
        PaymentApproveRetryPayload payload =
                objectMapper.readValue(
                        history.getRequestPayload(),
                        PaymentApproveRetryPayload.class
                );

        paymentService.paymentApprovalPublication(PaymentApproveCmd.builder()
                .orderId(payload.orderId())
                .paymentId(payload.paymentId())
                .productId(payload.productId())
                .fee(payload.productPrice())
                .build());
    }
    private void retryCancel(RetryHistory history) {
        PaymentCancelRetryPayload payload =
                objectMapper.readValue(
                        history.getRequestPayload(),
                        PaymentCancelRetryPayload.class
                );

        paymentService.paymentCancelPublication(PaymentCancelCmd.builder()
                .orderId(payload.orderId())
                .transactionId(payload.transactionId())
                .paymentId(payload.paymentId())
                .productId(payload.productId())
                .fee(payload.amount())
                .build()
        );

    }
}
```
메세지를 소비하는 로직이다.
```java
@Component
@RequiredArgsConstructor
public class PaymentUseCase {

    private final PaymentApiService paymentApiService;
    private final OrderTransactionService orderTransactionService;
    private final RetryHistoryService retryHistoryService;

    public void approve(PaymentApproveCmd cmd) {
        PaymentApproveResponseCmd responseCmd = paymentApiService.approve(cmd);
        try {
            orderTransactionService.completePayment(
                    cmd.orderId(),
                    cmd.paymentId()
            );
        }catch(Exception e) {
            retryHistoryService.retryHistoryCancelCreate(PaymentCancelRetryPayload.builder()
                    .transactionId(responseCmd.transactionId())
                    .amount(responseCmd.amount())
                    .orderId(cmd.orderId())
                    .productId(cmd.productId())
                    .paymentId(cmd.paymentId())
                    .build());
        }
    }
    public void cancel(PaymentCancelCmd cmd){
        paymentApiService.cancel(cmd);
        orderTransactionService.compensateCompletionFailure(cmd.productId(),cmd.orderId(),cmd.paymentId());
    }
}
```
중요한 코드만 첨부하였고 나머지 코드는 깃을 참조하길 바란다.
## 가설
주문, 결제, 재고 변경과 Outbox 이벤트 저장이 하나의 DB 트랜잭션으로 묶여 있으므로, 처리 과정에서 예외가 발생하면 모든 작업이 함께 롤백되어 로컬 데이터의 정합성이 유지될 것이다.
RabbitMQ 장애로 메시지 발행에 실패하더라도 Outbox 이벤트는 `READY` 상태로 DB에 남아 있을 것이다.   
이후 RabbitMQ가 복구되면 스케줄러가 해당 이벤트를 다시 조회하여 메시지를 발행하고, Consumer가 이를 처리함으로써 메시지 유실 없이 주문과 결제 처리가 완료될 것이다.
## 실험
실험은 3가지를 진행할 것 이다.
성공 케이스, Outbox이벤트 저장 실패, RabbitMQ장애

성공 케이스 이다.   
실제로 메세지 발행, 소비, DB 적재, api호출 모두 정상으로 실행되었다.   
<img src="{{ '/assets/images/concurrency/0824/first/configuration_order_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0824/first/configuration_pg_test.png' | relative_url }}" alt="configuration_pg_test" />
<img src="{{ '/assets/images/concurrency/0824/first/configuration_order_db_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0824/first/configuration_inventory_db_test.png' | relative_url }}" alt="configuration_inventory_db_test" />
<img src="{{ '/assets/images/concurrency/0824/first/configuration_retry_db_test.png' | relative_url }}" alt="configuration_retry_db_test" />
<img src="{{ '/assets/images/concurrency/0824/first/configuration_payment_db_test.png' | relative_url }}" alt="configuration_payment_db_test" />

두 번째는 Outbox 이벤트 저장 후 주문 트랜잭션이 실패하는 상황이다.
정확히는 Outbox이벤트 저장 후 의도적인 익셉션 발생을 진행하여 실험해보겠다.    
의도한대로 익셉션 발생으로 주문 저장된 모든것들이 롤백이 되어 아무것도 저장이 안되었으며 재고도 원복 되어있고 Outbox 이벤트도 생성이 안되어 스케줄러에서 조회가 안되는것을 볼수있다.    
<img src="{{ '/assets/images/concurrency/0824/second/configuration_order_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0824/second/configuration_inventory_db_test.png' | relative_url }}" alt="configuration_inventory_db_test" />
<img src="{{ '/assets/images/concurrency/0824/second/configuration_order_db_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0824/second/configuration_payment_db_test.png' | relative_url }}" alt="configuration_payment_db_test" />
<img src="{{ '/assets/images/concurrency/0824/second/configuration_retry_db_test.png' | relative_url }}" alt="configuration_retry_db_test" />

마지막으로 RabbitMQ장애시를 실험 해보겠다.   
우선 주문을 생성하기전 RabbitMQ를 종료 한다음 발행이 안되는것을 확인하고 후에 다시 가동시켜 정상동작하는지 실험 해보겠다.     
RabbitMQ가 실행되어있지 않아 보는거와 같이 주문은 생성 되었지만 메세지 발행, pg호출, Outbox이벤트가 완료처리가 안되어있고 준비중으로 되어있다.   
<img src="{{ '/assets/images/concurrency/0824/third/configuration_order_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0824/third/configuration_order_db_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0824/third/configuration_payment_db_test.png' | relative_url }}" alt="configuration_payment_db_test" />
<img src="{{ '/assets/images/concurrency/0824/third/configuration_inventory_db_test.png' | relative_url }}" alt="configuration_inventory_db_test" />
<img src="{{ '/assets/images/concurrency/0824/third/configuration_retry_db_test.png' | relative_url }}" alt="configuration_retry_db_test" />
이제 RabbitMQ 실행시 이다.  
예상한대로 메세지 발행, 소비, DB상태변경 모두 수행되었다.   
하지만 예상을 한대로 바로 되지는 않았고 RabbitMQ를 재가동한 직후 Publisher가 먼저 연결되어 Outbox 이벤트를 발행하였지만   
Consumer는 연결 복구 과정에서 일시적으로 RabbitMQ 호스트를 찾지 못했고, Listener Container가 자동으로 재시작되었다.    
발행된 메시지는 큐에 보관되었으며 Consumer가 재연결된 이후 정상적으로 처리되었다.
<img src="{{ '/assets/images/concurrency/0824/third/configuration_order_db_test2.png' | relative_url }}" alt="configuration_order_db_test2" />
<img src="{{ '/assets/images/concurrency/0824/third/configuration_payment_db_test2.png' | relative_url }}" alt="configuration_payment_db_test2" />
<img src="{{ '/assets/images/concurrency/0824/third/configuration_retry_db_test2.png' | relative_url }}" alt="configuration_retry_db_test2" />

## 결론
이번에는 Outbox를 이용해 정합성을 보장하는 방법에 대해 실험 해봤다.   
물론 PG호출 실패시 보상 프로세스까지는 하지않았지만 이부분까지 설계/구현한다면 더욱 제대로된 정합성 보장을 할수있을 것 이다.   
그리고 MQ를 이용한 정합성에서 문제점인 Dual Write 문제는 해결되었다.   
하지만 생각해보면 우리는 재시도/이력 테이블의 단점중 하나인 항상 이력테이블 조회를 막기위해 MQ를 사용하였지만 결국 다시 이 단점이 드러났다.   
결국 이방식은 비동기 호출, 역할분리, Dual Write방지라는 장점을 얻었지만 단점으로 MQ의 관리부담, DB의 주기적인 조회를 얻었다.   
다음시간에는 이중 DB의 주기적인 조회를 없애는 CDC를 이용한 정합성 보장을 실험해보겠다.