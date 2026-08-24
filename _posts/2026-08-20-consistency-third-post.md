---
layout: post
title: "[Consistency] 재시도/이력 프로세스로 정합성을 보장시키는 법"
date: 2026-08-20 23:00:00 +0900
categories :
  - architecture
description : Consistency에 대해 알아보자
series: consistency
series_order: 3
---

## 개요
저번에는 보상프로세스로 정합성을 맞추는 방법을 구현해봤다.    
실패시 정합성은 맞춰졌지만 보상프로세스도 실패 할 경우는 정합성을 맞출 수 없다.     
이번시간에는 재시도/이력 프로세스를 이용하여 정합성을 맞추는 실험을 해보겠다.   

## 시스템 구성
이번 시리즈에서는 간단하게 주문, 결제 도메인을 이용할것이다.    
주문, 결제 도메인은 엄청 복잡한 도메인이다 그렇기에 실무에서는 많은 예외 상황 처리와 많은 검증, 많은 테이블, 컬럼이 필요하지만 이번 시리즈에서는 정합성을 위해 최대한 간단하게 필요한 부분만 구현 할 것 이다.    
(Validation check, Exception 이런것들은 넘어갈것이다) .

시스템 구성은 주문서버와 결제서버와 API통신을 하는 방식으로 했고 결제서버는 Mock서버로 응답값만 넘어오도록 구성했다.    
DB는 주문서버에만 연동하였다.
<img src="{{ '/assets/images/concurrency/0820/configuration_diagram.png' | relative_url }}" alt="ConfigurationDiagram" />

DB테이블은 기존과 동일한 테이블에 추가로 이력 테이블을 하나 만들었다.   
이력 테이블은 API 호출시 상태값을 저장하는 테이블이다.   
<img src="{{ '/assets/images/concurrency/0820/erd.png' | relative_url }}" alt="erd" />

## 실험 환경
실험 환경은 Spring boot,PostgreSQL이며 모두 맥북에 도커를 이용하여 띄워 테스트 할 것 이다.

실험에서 사용한 전체 코드는
<a href="https://github.com/sssm987/payment/tree/validate-payment-approval" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(주문 서버)]</a>
<a href="https://github.com/sssm987/pg" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(결제 서버)]</a>

## 설명
```java
public void createOrder(OrderCreateRequestDTO dto) {
    log.info("주문 생성 시작. memberId={}, productId={}",dto.memberId(),dto.productId());
    OrderContext orderContext = orderTransactionService.prepareOrder(dto);
    log.info("주문 생성 완료. orderId={}",orderContext.orderId());
    PaymentApproveResponseCmd paymentApproveResponseCmd;

    try {
        paymentApproveResponseCmd = paymentApiService.approve(PaymentApproveCmd.builder()
                .orderId(orderContext.orderId())
                .paymentId(orderContext.paymentId())
                .fee(orderContext.productPrice())
                .build());
        log.info("PG 승인 성공 transactionId={}",paymentApproveResponseCmd.transactionId());
    }catch (Exception e){
        retryHistoryService.retryHistoryRetry(orderContext.retryId());
        log.info("PG 승인 실패 paymentId={}",orderContext.paymentId());
        throw e;
    }

    retryHistoryService.retryHistorySuccess(orderContext.retryId());

    try {
        orderTransactionService.completePayment(orderContext);
    }catch (Exception e){
        long retryId = retryHistoryService.retryHistoryCancelCreate(orderContext.productPrice(),
                orderContext.orderId(),
                orderContext.paymentId(),
                dto.productId(),
                paymentApproveResponseCmd.transactionId());
        log.info("주문 완료 변경 실패 orderId={}",orderContext.orderId());
        try {
            paymentApiService.cancel(PaymentCancelCmd.builder()
                    .transactionId(paymentApproveResponseCmd.transactionId())
                    .amount(paymentApproveResponseCmd.amount())
                    .build());
        }catch (Exception e2){
            log.info("PG 취소 API 실패 transactionId={}",paymentApproveResponseCmd.transactionId());
            retryHistoryService.retryHistoryRetry(retryId);
            throw e;
        }
        retryHistoryService.retryHistorySuccess(retryId);
        orderTransactionService.compensateCompletionFailure(dto.productId(), orderContext);
        throw e;
    }
}
```
보상 프로세스와 동일하게 usecase계층에서 트랜잭션 서비스를 호출하는 방식으로 사용하였다.   
대신에 다른점은 api호출 전에 재시도/이력 테이블에 insert를 한다는 점이다.  
그리고 api호출이 성공했을경우 status 성공으로 바꿔준다.    
실패시에는 status를 재시도로 바꿔준다.   
그리고 스케줄러로 주기적으로 재시도/이력 테이블을 조회한 후 재시도인것 들을 조회 후 재호출하는 구조이다.  


스케줄러 코드는 양이많아 따로 첨부하지않았고 깃을 통해 확인하기 바란다.   

## 가설
api호출 실패시 재시도를 통해 정합성을 맞출수있는 가능성이 더 높아질것이다.  
보상프로세스에서의 문제점인 결제완료 DB가 실패된 상황에서 취소 API 호출 실패의 문제점을 재시도하는 방식으로 보완이 가능해질것이다.   


## 실험
실험은 성공, 승인 api 실패, 취소 api 실패 케이스를 실험해보겠다.

첫번째 실험은 모두가 정상일때 실헝을 하였다.   
간단하게 주문 생성 스웨거를 통해 주문 1건을 생성하고 PG서버에 값이 잘 오는지 데이터가 잘 쌓이는지를 실험하겠다.   
결과는 모두 문제없이 데이터가 쌓여고 각 서버간의 정합성도 모두 맞는다.   
<img src="{{ '/assets/images/concurrency/0820/configuration_swagger_test.png' | relative_url }}" alt="configuration_swagger_test" />
<img src="{{ '/assets/images/concurrency/0820/configuration_order_db_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0820/configuration_pg_test.png' | relative_url }}" alt="configuration_pg_test" />
<img src="{{ '/assets/images/concurrency/0820/configuration_inventory_db_test.png' | relative_url }}" alt="configuration_inventory_db_test" />
<img src="{{ '/assets/images/concurrency/0820/configuration_retry_db_test.png' | relative_url }}" alt="configuration_retry_db_test" />

두번째 실험은 승인 api 실패시를 실험하였다.   
api 호출로직에 의도적으로 익셉션을 발생시켜 pg서버 호출이 되기전 실패 처리를 하게 만들었다.   
의도한대로 재시도 이력 테이블에 잘 쌓였고 재시도에서 pg서버 호출도 성공하였다.    
재시도는 2번시도 하게끔 만들었으며 실패시 카운트가 증가하게끔 하였으며 재시도 횟수는 1번으로 알맞게 증가되었다.
<img src="{{ '/assets/images/concurrency/0820/configuration_swagger_test2.png' | relative_url }}" alt="configuration_swagger_test2" />
<img src="{{ '/assets/images/concurrency/0820/configuration_order_test2.png' | relative_url }}" alt="configuration_order_test2" />
<img src="{{ '/assets/images/concurrency/0820/configuration_order_retry_test2.png' | relative_url }}" alt="configuration_order_retry_test2" />
<img src="{{ '/assets/images/concurrency/0820/configuration_inventory_db_test2.png' | relative_url }}" alt="configuration_inventory_db_test2" />
<img src="{{ '/assets/images/concurrency/0820/configuration_pg_test2.png' | relative_url }}" alt="configuration_pg_test2" />
<img src="{{ '/assets/images/concurrency/0820/configuration_retry_db_test2.png' | relative_url }}" alt="configuration_retry_db_test2" />


세번째 실험은 취소 api 실패 케이스를 실험해보겠다.
이번에는 api호출 성공 후 DB저장시 의도적으로 익셥센은 발생시키고 취소api호출시에 의도적으로 익셥센을 발생시켜 실패 처리를 하게만들었다. 
의도한대로 실패 후 재시도에서 취소 api호출을 잘했도 db의 정합성과 pg서버의 정합성도 잘 맞는걸 확인 할 수 있었다.   
그리고 후속 처리인 취소 후 재고 원복과 상태값 변경도 잘 이루어졌다.   
<img src="{{ '/assets/images/concurrency/0820/configuration_swagger_test3.png' | relative_url }}" alt="configuration_swagger_test3" />
<img src="{{ '/assets/images/concurrency/0820/configuration_order_test3.png' | relative_url }}" alt="configuration_order_test3" />
<img src="{{ '/assets/images/concurrency/0820/configuration_order_retry_test3.png' | relative_url }}" alt="configuration_order_retry_test3" />
<img src="{{ '/assets/images/concurrency/0820/configuration_pg_test3.png' | relative_url }}" alt="configuration_pg_test3" />
<img src="{{ '/assets/images/concurrency/0820/configuration_inventory_db_test3.png' | relative_url }}" alt="configuration_inventory_db_test3" />
<img src="{{ '/assets/images/concurrency/0820/configuration_payment_db_test3.png' | relative_url }}" alt="configuration_payment_db_test3" />
<img src="{{ '/assets/images/concurrency/0820/configuration_retry_db_test3.png' | relative_url }}" alt="configuration_retry_db_test3" />
<img src="{{ '/assets/images/concurrency/0820/configuration_order_db_test3.png' | relative_url }}" alt="configuration_order_db_test3" />


## 결론
재시도/이력 프로세스로 승인 api가 성공했을경우 db쪽 문제시 API 서버와도 정합성을 맞추는데 성공했다.    
즉 보상프로세스에서 할수없었던 api성공시 서버쪽 문제가 생겨도 db에 상태를 보고 정합성을 맞출 수 있다.    
물론 여기선 멱등성을 따로 설계하지 않았지만 멱등성까지 고려해서 만들면 더더욱 안전한 시스템이 될것이다.   
재시도/이력 프로세스는 시스템적으로 많은 부분을 보완 할 수 있었다.   
하지만 재시도 대상이 없을때도 스케줄러를 통한 db조회가 호출이되고 또한 PG 장애로 작업이 대량 적체되면 결제 애플리케이션이 재시도까지 직접 처리하면서 정상 요청에 영향을 줄 수 있었다.    
이 부분을 보완하기위해 다음 글에서는 재시도 작업을 RabbitMQ를 통해 비동기로 분리하고, DB 저장과 메시지 발행 사이에서 발생하는 새로운 정합성 문제도 함께 실험해보겠다. 