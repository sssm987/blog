---
layout: post
title: "[Consistency] MQ를 이용한 정합성을 보장시키는 법"
date: 2026-08-23 10:00:00 +0900
categories :
  - architecture
description : Consistency에 대해 알아보자
series: consistency
series_order: 4
---

## 개요
저번에는 재시도/이력 프로세스로 정합성을 맞추는 방법을 구현해봤다.    
서버쪽 문제로 인해 실패했을경우 재시도를 통해 정합성을 맞추는데 성공하였다.
하지만 재시도 대상이 없을때도 스케줄러를 통한 db조회가 호출이되고 또한 PG 장애로 작업이 대량 적체되면 결제 애플리케이션이 재시도까지 직접 처리하면서 정상 요청에 영향을 줄 수 있었다.   
이부분을 보완하기 위해 이번시간에는 MQ를 이용하여 정합성을 맞추는 실험을 해보겠다.   

## 시스템 구성
이번 시리즈에서는 간단하게 주문, 결제 도메인을 이용할것이다.    
주문, 결제 도메인은 엄청 복잡한 도메인이다 그렇기에 실무에서는 많은 예외 상황 처리와 많은 검증, 많은 테이블, 컬럼이 필요하지만 이번 시리즈에서는 정합성을 위해 최대한 간단하게 필요한 부분만 구현 할 것 이다.    
(Validation check, Exception 이런것들은 넘어갈것이다) .

시스템 구성은 주문서버와 결제서버와 API통신을 하는 방식으로 했고 결제서버는 Mock서버로 응답값만 넘어오도록 구성했다.    
DB는 주문서버에만 연동하였다.
<img src="{{ '/assets/images/concurrency/0820/configuration_diagram.png' | relative_url }}" alt="ConfigurationDiagram" />

DB테이블은 재시도/이력과 동일한 구조로 사용하였다.   
<img src="{{ '/assets/images/concurrency/0820/erd.png' | relative_url }}" alt="erd" />

## 실험 환경
실험 환경은 Spring boot,k6,PostgreSQL이며 모두 맥북에 도커를 이용하여 띄워 테스트 할 것 이다.

실험에서 사용한 전체 코드는
<a href="https://github.com/sssm987/payment/tree/rabbitmq-payment" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(주문 서버)]</a>
<a href="https://github.com/sssm987/pg" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(결제 서버)]</a>

## 설명
우선 기존과 비지니스 로직 구조를 변경했다.   
기존에는 orderUseCase에서 DB insert, 결제 api 호출, 보상프로세스를 책임졌다면 이번에는 DB insert, 승인 요청 발행 까지만 담당한다.  
그리고 PaymentConsumer에서 요청을 소비하고 PaymentUseCase에서 결제 api 호출,DB 후처리, 보상프로세스를 담당한다.    

이번 실험에서는 승인 API가 일시적으로 실패한 경우 RabbitMQ의 재시도를 통해 다시 처리하는 흐름까지만 구현하였다.    
설정한 재시도 횟수를 모두 소진하면 재시도 이력의 상태를 `FAILED`로 변경하지만, 주문 취소와 재고 원복 같은 최종 복구 처리는 구현 범위에서 제외하였다.    

실제 시스템에서는 최종 실패한 승인 요청을 그대로 방치하면 주문은 대기 상태로 남고 재고도 차감된 상태가 유지될 수 있다.   
따라서 PG 승인 여부를 조회한 뒤, 승인이 이루어지지 않았다면 주문을 취소하고 재고를 원복하는 별도의 복구 프로세스가 필요하다.
<img src="{{ '/assets/images/concurrency/0820/configuration_diagram2.png' | relative_url }}" alt="ConfigurationDiagram2" />

```java
public void createOrder(OrderCreateCmd cmd) {
        OrderContext orderContext = orderTransactionService.prepareOrder(cmd);

        paymentService.paymentApprovalPublication(PaymentApproveCmd.builder()
                .orderId(orderContext.orderId())
                .paymentId(orderContext.paymentId())
                .productId(orderContext.productId())
                .fee(orderContext.productPrice())
                .retryId(orderContext.retryId())
                .build());

    }
```
```java
public void approve(PaymentApproveCmd cmd) {

    if(retryHistoryService.isCompleted(cmd.retryId()))
        return;
    retryHistoryService.retryIncrease(cmd.retryId());
    paymentApiService.approve(cmd);
    try {
        orderTransactionService.completePayment(
                cmd.orderId(),
                cmd.paymentId(),
                cmd.retryId()
        );
    }catch(Exception e) {
        retryHistoryService.retryHistorySuccess(cmd.retryId());
        long retryId = retryHistoryService.retryHistoryCancelCreate(cmd.paymentId());
        paymentService.paymentCancelPublication(PaymentCancelCmd.builder()
                .fee(cmd.fee())
                .paymentId(cmd.paymentId())
                .orderId(cmd.orderId())
                .productId(cmd.productId())
                .retryId(retryId).build());
    }
}
public void cancel(PaymentCancelCmd cmd){
    if(retryHistoryService.isCompleted(cmd.retryId()))
        return;
    retryHistoryService.retryIncrease(cmd.retryId());
    paymentApiService.cancel(cmd);
    orderTransactionService.compensateCompletionFailure(cmd.productId(),cmd.orderId(),cmd.paymentId(),cmd.retryId());
}
```
첫번째 코드가 orderUseCase 코드이고 두번째 코드가 PaymentUseCase이다.   
기존 방식에서는 주문 요청을 처리하는 스레드가 PG API 호출과 재시도까지 직접 수행했다.    
이번에는 주문 생성과 결제 처리를 분리하고, 두 작업 사이에 RabbitMQ를 두었다.     

OrderUseCase는 주문, 결제, 재시도 이력을 DB에 저장한 뒤 결제 승인 메시지를 발행한다.     
메시지를 발행한 이후에는 PG API 응답을 기다리지 않으므로 주문 요청과 결제 처리가 비동기로 분리된다.     

발행된 승인 메시지는 PaymentConsumer가 소비하고 PaymentUseCase.approve()를 호출하며          
승인 API 호출 중 예외가 발생하면 예외가 RabbitMQ Listener까지 전달되고, Retry Interceptor가 동일한 메시지를 다시 처리한다.    

현재 설정은 최초 실행 이후 최대 2번 재시도하며, 재시도 간격은 1초부터 시작해 두 배씩 증가하도록 설정하였다.
모든 재시도가 실패하면 PaymentMessageRecoverer가 실행되어 해당 이력의 상태를 FAILED로 변경한다.

PG 승인은 성공했지만 주문 완료 DB 처리에 실패한 경우에는 승인 요청을 다시 실행하지 않고 취소 이력을 생성한 뒤 취소 메시지를 발행한다.
취소 메시지는 별도의 Consumer가 처리하며, PG 취소 API가 성공하면 재고를 원복하고 주문과 결제 상태를 시스템 취소로 변경한다.
취소 API 호출도 실패하면 승인 요청과 동일하게 RabbitMQ의 재시도 정책을 적용한다.

RabbitMQ가 실제 메시지 재시도를 담당하지만 재시도 횟수와 처리 상태는 DB에도 저장하였다.
RabbitMQ는 메시지 전달을 담당할 뿐 결제 승인이나 취소라는 비즈니스 작업의 처리 이력까지 관리하지 않기 때문이다.
DB 이력을 통해 애플리케이션 재시작 이후에도 처리 결과를 확인하고, 최종 실패한 요청을 조회하거나 수동으로 복구할 수 있다.    

## 가설
주문 생성 요청과 결제 처리를 RabbitMQ를 통해 비동기로 분리하면 주문 요청 스레드는 PG API의 응답을 기다리지 않고 작업을 종료할 수 있을 것이다.   
또한 PG 서버에 일시적인 장애가 발생하더라도 RabbitMQ Consumer의 재시도를 통해 결제 승인 또는 취소 요청이 최종적으로 처리될 가능성이 높아질 것이다.

승인 API 호출이 실패하면 설정한 횟수만큼 재시도될 것이다.   
취소 API 호출이 일시적으로 실패하면 동일한 메시지가 재처리되고, 최종적으로 취소에 성공하면 재고와 주문·결제 상태가 원복되어 시스템 간 정합성이 회복될 것이다.  

하지만 주문 데이터 저장과 메시지 발행은 하나의 트랜잭션으로 처리되지 않으므로, DB 저장 이후 메시지 발행에 실패하면 결제 요청이 유실될 수 있다.


## 실험
실험은 성공케이스, 승인 API 호출 실패 케이스, 승인 후처리 실패 케이스를 진행하겠다.   

우선 성공 케이스이다.   
실제로 메세지 발행, 소비, DB 적재, api호출 모두 정상으로 실행되었다.   
<img src="{{ '/assets/images/concurrency/0823/first/configuration_swagger_test.png' | relative_url }}" alt="configuration_swagger_test" />
<img src="{{ '/assets/images/concurrency/0823/first/configuration_order_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0823/first/configuration_order_db_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0823/first/configuration_pg_test.png' | relative_url }}" alt="configuration_pg_test" />
<img src="{{ '/assets/images/concurrency/0823/first/configuration_inventory_db_test.png' | relative_url }}" alt="configuration_inventory_db_test" />
<img src="{{ '/assets/images/concurrency/0823/first/configuration_retry_db_test.png' | relative_url }}" alt="configuration_retry_db_test" />
<img src="{{ '/assets/images/concurrency/0823/first/configuration_payment_db_test.png' | relative_url }}" alt="configuration_payment_db_test" />

두번째는 승인 API 호출 실패 케이스이다.   
승인 API 호출 하는 로직에 의도적으로 익셉션을 넣어 한번은 실패하도록 만들고 다음은 성공하게 설정하여 실험 해보겠다.   
의도한대로 1번의 실패 후 메세지 발행, 소비, DB 적재, api호출 전부 정상으로 실행되었다. 
<img src="{{ '/assets/images/concurrency/0823/second/configuration_swagger_test.png' | relative_url }}" alt="configuration_swagger_test" />
<img src="{{ '/assets/images/concurrency/0823/second/configuration_order_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0823/second/configuration_pg_test.png' | relative_url }}" alt="configuration_pg_test" />
<img src="{{ '/assets/images/concurrency/0823/second/configuration_inventory_db_test.png' | relative_url }}" alt="configuration_inventory_db_test" />
<img src="{{ '/assets/images/concurrency/0823/second/configuration_order_db_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0823/second/configuration_payment_db_test.png' | relative_url }}" alt="configuration_payment_db_test" />
<img src="{{ '/assets/images/concurrency/0823/second/configuration_retry_db_test.png' | relative_url }}" alt="configuration_retry_db_test" />

세번째는 승인 후처리 실패 케이스를 진행하겠다.   
승인 API성공 후 후처리 과정에 익셉션을 넣어 무조건 실패하도록 만들고 취소 발행 후 취소 api호출까지 잘 되는지 실험 해보겠다.    
승인 실패시 보상프로세스도 잘 동작했고 취소 api, 후 처리도 잘 동작하였다.   
<img src="{{ '/assets/images/concurrency/0823/third/configuration_swagger_test.png' | relative_url }}" alt="configuration_swagger_test" />
<img src="{{ '/assets/images/concurrency/0823/third/configuration_order_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0823/third/configuration_pg_test.png' | relative_url }}" alt="configuration_pg_test" />
<img src="{{ '/assets/images/concurrency/0823/third/configuration_inventory_db_test.png' | relative_url }}" alt="configuration_inventory_db_test" />
<img src="{{ '/assets/images/concurrency/0823/third/configuration_order_db_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0823/third/configuration_payment_db_test.png' | relative_url }}" alt="configuration_payment_db_test" />
<img src="{{ '/assets/images/concurrency/0823/third/configuration_retry_db_test.png' | relative_url }}" alt="configuration_retry_db_test" />


## 결론
이로써 이번에는 메세지큐를 이용한 정합성 보정을 실험해봤다.    
물론 api호출 실패시 보상프로세스 및 api 실패 상황에 따른 프로세스가 추가되어야 한다.   
하지만 그것들까지 추가해서 만들기에는 범위가 너무 넓어져 제외했다.    
사실 메세지큐를 이용한 정합성 보정은 재시도/이력 프로세스와 정합성 보정에 큰 차이는 없다.   
메세지 큐를 이용한 보정은 비동기 호출이 가능하고 역할 분리가 확실하다 그리고 애플리케이션이 늘어나도 큰 문제는 없다.   
그렇지만 단점도 있다 메세지큐를 도입해야한다는거 자체가 관리 해야될 대상이 하나 늘어나는것이다.    
또 DB적재까지 성공했지만 메세지큐 발행이 유실되었을때 문제가 생긴다.    
재시도/이력 프로세스의 단점은 주기적으로 db를 조회해야하고 책임의 범위가 애매해진다.  
그리고 단일 서버일때는 문제가 없지만 다중서버로 갈땐 스케줄러가 여러개가 도는 문제가 생겨 또 배치서버를 따로 둬야하고 그러면 메세지큐의 단점인 관리대상이 다시 늘어나는 문제가 생긴다.   
그래서 이러한 각자의 단점을 보완하는 outBox패턴으로 다음에는 실험을 해보겠다.    