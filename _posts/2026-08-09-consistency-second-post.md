---
layout: post
title: "[Consistency] 보상프로세스로 정합성을 보장시키는 법"
date: 2026-08-09 23:00:00 +0900
categories :
  - architecture
description : Consistency에 대해 알아보자
series: consistency
series_order: 1
---

## 개요
저번에는 DB트랜잭션으로 정합성을 맞출수있는지 실험해봤지만 외부 호출이 포함된 순간 실패시 정합성이 틀어지는걸 볼수있었다.   
이번시간에는 보상프로세스를 이용하여 정합성을 맞추는 실험을 해보겠다.   


## 시스템 구성
이번 시리즈에서는 간단하게 주문, 결제 도메인을 이용할것이다.    
주문, 결제 도메인은 엄청 복잡한 도메인이다 그렇기에 실무에서는 많은 예외 상황 처리와 많은 검증, 많은 테이블, 컬럼이 필요하지만 이번 시리즈에서는 정합성을 위해 최대한 간단하게 필요한 부분만 구현 할 것 이다.    
(Validation check, Exception 이런것들은 넘어갈것이다) .

시스템 구성은 주문서버와 결제서버와 API통신을 하는 방식으로 했고 결제서버는 Mock서버로 응답값만 넘어오도록 구성했다.    
DB는 주문서버에만 연동하였다.
<img src="{{ '/assets/images/concurrency/0809/configuration_diagram.png' | relative_url }}" alt="ConfigurationDiagram" />

DB테이블은 최대한 간단하게 구성하였으며 실제 주문, 결제 DB는 이렇게 구성하면 안되고 더 많은 컬럼과 테이블이 있어야 한다.   
<img src="{{ '/assets/images/concurrency/0809/erd.png' | relative_url }}" alt="erd" />



## 실험 환경
실험 환경은 Spring boot,k6,PostgreSQL이며 모두 맥북에 도커를 이용하여 띄워 테스트 할 것 이다.

실험에서 사용한 전체 코드는
<a href="https://github.com/sssm987/payment/tree/compensation" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(주문 서버)]</a>
<a href="https://github.com/sssm987/pg" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(결제 서버)]</a>

## 설명
기존 시스템은 usecase계층에 트랜잭션을 걸었지만 이번에는 usecase계층에 트랜잭션을 제거 하고 따로 트랜잭션 서비스를 만들어 트랜잭션 서비스에서 조립하는 형태로 바꿨다.    
비지니스 로직은 아래와 같이 변경되었으며 구간별 try catch로 감싸 익셥션 발생될때 보상 프로세스를 호출하도록 만들었다.   
실제로는 저렇게 익셉션을 전부다 잡으면 안되고 설계자의 의도에 맞는 익셉션만 잡아야하며 예외처리를 의미에 맞게 넣어야한다.   
하지만 정합성을 잡는 토이프로젝트이기 때문에 전체로 잡고 진행하였다.   
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
        orderTransactionService.compensateApprovalFailure(dto.productId(), orderContext);
        log.info("PG 승인 실패 paymentId={}",orderContext.paymentId());
        throw e;
    }
    try {
        orderTransactionService.completePayment(orderContext);
    }catch (Exception e){
        paymentApiService.cancel(PaymentCancelCmd.builder()
                .transactionId(paymentApproveResponseCmd.transactionId())
                .amount(paymentApproveResponseCmd.amount())
                .build());
        orderTransactionService.compensateCompletionFailure(dto.productId(), orderContext);
        log.info("주문 완료 변경 실패 orderId={}",orderContext.orderId());
        throw e;
    }
}
```
성공시 비지니스 로직이다.  
이렇게 변경하먄 api호출부가 트랜잭션이랑 분리가 되어 api호출시간동안 DB커녁션을 점유하지 않는 장점이있다.   
하지만 단점으로는 주문 완료 DB로직에서 익셥션이 발생 할 경우 주문 생성 로직, 재고 차감로직 등 롤백이 되지않아 보상프로세스를 따로 만들어줘야한다.
<img src="{{ '/assets/images/concurrency/0809/success_business_diagram.png' | relative_url }}" alt="BusinessDiagram" />
실패시 비지니스 로직이다.
실패 단계의 따라 보상해야하는 로직이 다르며 결제 실패시와 DB오류랑 다른 상태값을 주기위해 다른 로직을 호출하는 쪽으로 설계를 하였다.    
이렇게 하면 보상 처리가 성공한다는 전제에서 정합성을 회복할 수 있다.
<img src="{{ '/assets/images/concurrency/0809/fail_business_diagram.png' | relative_url }}" alt="FailBusinessDiagram" />
세부 코드는 깃을 통해 확인하기 바란다.   

## 가설
비지니스 로직의 성공 데이터가 원활하게 들어갈것이며 실패시에도 보상 처리를 통해 정합성이 회복될 것이다.      
PG쪽도 마찬가지로 주문 실패시 결제 취소를 호출하여 다른 서버의 정합성도 함께 맞춰진다.   

## 실험
첫번째 실험은 모두가 정상일때 실헝을 하였다.   
간단하게 주문 생성 스웨거를 통해 주문 1건을 생성하고 PG서버에 값이 잘 오는지 데이터가 잘 쌓이는지를 실험하겠다.   
결과는 모두 문제없이 데이터가 쌓여고 각 서버간의 정합성도 모두 맞는다.   
<img src="{{ '/assets/images/concurrency/0809/configuration_swagger_test.png' | relative_url }}" alt="configuration_swagger_test" />
<img src="{{ '/assets/images/concurrency/0809/configuration_order_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0809/configuration_pg_test.png' | relative_url }}" alt="configuration_pg_test" />
<img src="{{ '/assets/images/concurrency/0809/configuration_db_test.png' | relative_url }}" alt="configuration_db_test" />
<img src="{{ '/assets/images/concurrency/0809/configuration_inventory_db_test.png' | relative_url }}" alt="configuration_inventory_db_test" />

두번째 실험은 API호출시 실패하는 상황을 실험하였다.    
pg서버를 끄고 주문생성을 호출한 결과 주문생성이 실패하였다.   
그리고 DB를 조회해보니 시스템 취소 상태로 주문생성이 되어있고, 재고도 다시 원상복구되어있고, 결제는 실패 상태로 잘되어있다.    
실험결과 API 실패시 정합성은 문제가 없었다. 
<img src="{{ '/assets/images/concurrency/0809/configuration_swagger_test2.png' | relative_url }}" alt="configuration_swagger_test2" />
<img src="{{ '/assets/images/concurrency/0809/configuration_order_db_test2.png' | relative_url }}" alt="configuration_order_db_test2" />
<img src="{{ '/assets/images/concurrency/0809/configuration_inventory_db_test2.png' | relative_url }}" alt="inventory_db_test2" />
<img src="{{ '/assets/images/concurrency/0809/configuration_order_test2.png' | relative_url }}" alt="configuration_order_test2" />
<img src="{{ '/assets/images/concurrency/0809/configuration_payment_db_test2.png' | relative_url }}" alt="configuration_payment_db_test2" />

세번째 실험은 API호출이 성공적으로 되었지만 완료상태를 저장하는 단계에서 실패 상황을 실험하였다.    
DB저장 부분이 실행될때 의도적으로 익셉션을 발생시켰고 실험결과는 시스템 취소 상태로 주문생성이 되었고, 재고도 다시 원상복구되어있고, 결제는 취소상태로 잘되어있다.   
그리고 PG서버에도 취소 요청이 성공적으로 들어왔다.   
<img src="{{ '/assets/images/concurrency/0809/configuration_swagger_test3.png' | relative_url }}" alt="configuration_swagger_test3" />
<img src="{{ '/assets/images/concurrency/0809/configuration_pg_test3.png' | relative_url }}" alt="configuration_pg_test3" />
<img src="{{ '/assets/images/concurrency/0809/configuration_order_db_test3.png' | relative_url }}" alt="configuration_order_db_test3" />
<img src="{{ '/assets/images/concurrency/0809/configuration_inventory_db_test3.png' | relative_url }}" alt="inventory_db_test3" />
<img src="{{ '/assets/images/concurrency/0809/configuration_order_test3.png' | relative_url }}" alt="configuration_order_test3" />
<img src="{{ '/assets/images/concurrency/0809/configuration_payment_db_test3.png' | relative_url }}" alt="configuration_payment_db_test3" />

## 결론
보상프로세스로 API 서버와도 정합성을 맞추는데 성공했다.    
하지만 여기서 의문점이 생긴다 만약 결제완료 DB가 실패된 상황에서 취소 API를 호출했지만 그것도 실패한 경우는 어떡해야할까??   
사람이 직접 테이블이랑 로그를 보고 조치를 취할 수는 있겠지만 우리는 시스템적으로 더 완벽한 정합성을 생각해야한다.   
사람이 직접 무언가 행위로 조치를 취하는건 가장 마지막에 해야하는 최후에 보루라고 생각한다.   
그래서 이런 상황을 막기위해 재시도/이력 프로세스를 사용한다.   
다음글에는 재시도/이력 프로세스를 이용하여 정합성을 맞추는 법을 실험해보겠다.  
