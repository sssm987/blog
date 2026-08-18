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
실험 환경은 Spring boot,k6,PostgreSQL이며 모두 맥북에 도커를 이용하여 띄워 테스트 할 것 이다.

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