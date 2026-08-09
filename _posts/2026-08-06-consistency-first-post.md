---
layout: post
title: "[Consistency] DB 트랜잭션만으로 정합성을 보장할 수 있을까?"
date: 2026-08-03 23:00:00 +0900
categories : [Consistency]
description : Consistency에 대해 알아보자
---

## 개요
애플리케이션은 데이터를 저장하고 조회하며, 비즈니스 로직에 따라 데이터를 변경한다.     
이 과정에서 여러 요청이 하나의 데이터에 동시에 접근하면 예상하지 못한 결과가 발생할 수 있다.     
이를 동시성 문제라고 하며, 락이나 원자적 연산 등의 방법으로 해결할 수 있다.    
      
하지만 하나의 작업이 여러 서버와 데이터베이스, 외부 API에 걸쳐 처리된다면 동시성 문제와는 다른 문제가 발생한다.      
일부 작업은 성공했지만 다른 작업은 실패하여 시스템 간 데이터가 서로 맞지 않는 상태가 될 수 있기 때문이다.       
정합성이란 각 시스템의 데이터가 정해진 규칙에 맞게 일관된 상태를 유지하는 것을 의미한다.     
예를 들어 결제 승인은 완료됐지만 주문 서버에는 결제 실패로 기록됐다면 데이터 정합성이 깨진 상태라고 볼 수 있다.    
      
이번 시리즈에서는 외부 시스템 연동 과정에서 정합성 불일치가 발생하는 상황을 재현하고, DB 트랜잭션부터 보상 처리, 재시도, Transactional Outbox 등의 방법을 단계적으로 적용해보며 각 방식의 한계와 해결 과정을 알아본다.      

이번 글에서는 DB트랜잭션을 이용한 정합성에 대해 알아볼것이다.   

## 시스템 구성
이번 시리즈에서는 간단하게 주문, 결제 도메인을 이용할것이다.    
주문, 결제 도메인은 엄청 복잡한 도메인이다 그렇기에 실무에서는 많은 예외 상황 처리와 많은 검증, 많은 테이블, 컬럼이 필요하지만 이번 시리즈에서는 정합성을 위해 최대한 간단하게 필요한 부분만 구현 할 것 이다.    
(valdaytion check, Exption 이런것들은 넘어갈것이다) .   

시스템 구성은 주문서버와 결제서버와 API통신을 하는 방식으로 했고 결제서버는 Mock서버로 응답값만 넘어오도록 구성했다.    
DB는 주문서버에만 연동하였다.
<img src="{{ '/assets/images/concurrency/0806/configuration_diagram.png' | relative_url }}" alt="ConfigurationDiagram" />

DB테이블은 최대한 간단하게 구성하였으며 실제 주문, 결제 DB는 이렇게 구성하면 안되고 더 많은 컬럼과 테이블이 있어야 한다.   
<img src="{{ '/assets/images/concurrency/0806/erd.png' | relative_url }}" alt="erd" />

## 실험 환경
실험 환경은 Spring boot,k6,PostgreSQL이며 모두 맥북에 도커를 이용하여 띄워 테스트 할 것 이다.   

실험에서 사용한 전체 코드는
<a href="https://github.com/sssm987/payment/tree/main" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(주문 서버)]</a>
<a href="https://github.com/sssm987/pg" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다(결제 서버)]</a>

## 설명
이번 실험에서 제일 중요한 비지니스 로직이다.    
유스케이스 계층에 트랜잭션을 걸고 주문에 필요한 요청들을 각 서비스에 알맞게 호출한다.   
유스케이스 계층을 사용한 이유는 원자성 때문이다 주문 생성중에 1개의 로직이 실패되더라도 롤백시키기 위해 했으며 유스케이스에서 트랜잭션 어노테이션을 선언하였으며 order 서비스 계층에서 각 서비스를 호출할 경우 order 서비스 계층의 책임이 너무 많아지고 순환참조도 걸릴 수 있기 때문에 계층을 만들었다.   
세부 코드는 깃을 통해 확인하기 바란다.   
```java
@Transactional
public void createOrder(OrderCreateRequestDTO dto) {

    productService.inventoryDeduction(dto.productId());
    long productPrice = productService.findProductPrice(dto.productId());
    long orderId = orderService.orderCreate(dto.memberId(),dto.productId());
    long paymentId = paymentService.paymentCreate(PaymentCreateRequestCmd.builder()
                .memberId(dto.memberId())
                .orderId(orderId)
                .fee(productPrice)
                .build());

    paymentApiService.approve(PaymentApproveCmd.builder()
            .orderId(orderId)
            .paymentId(paymentId)
            .fee(productPrice)
            .build());

    orderService.orderPaid(orderId);
    paymentService.paymentSuccess(paymentId);
}
```
<img src="{{ '/assets/images/concurrency/0806/business_diagram.png' | relative_url }}" alt="BusinessDiagram" />

## 가설
비지니스로직은 문제가 없으므로 성공시에는 알맞은 값이 생기고 저장 될 것이다.   
그리고 실패시에 DB롤백이 진행되어 재고도 원복이 되고 정합성에 문제가 없어보인다.   
하지만 이 코드의 문제점은 API호출에 있다.   
우선 실험을 통해 자세히 알아보자.      

## 실험
우선 첫번째 실험은 PG서버 주문서버 DB모두 정상일때 실험을 진행한다.
간단하게 주문 생성 스웨거를 통해 주문 1건을 생성하고 PG서버에 값이 잘 오는지 데이터가 잘 쌓이는지를 보겠다.  
우선 생성은 문제없이 잘 되었다.    

<img src="{{ '/assets/images/concurrency/0806/configuration_swagger_test.png' | relative_url }}" alt="configuration_swagger_test" />
<img src="{{ '/assets/images/concurrency/0806/configuration_order_test.png' | relative_url }}" alt="configuration_order_test" />
<img src="{{ '/assets/images/concurrency/0806/configuration_pg_test.png' | relative_url }}" alt="configuration_pg_test" />
<img src="{{ '/assets/images/concurrency/0806/configuration_db_test.png' | relative_url }}" alt="configuration_db_test" />


두번째 실험은 API호출 후 의도적으로 익셉션을 발생시켰다.   
그 결과 DB는 모두 롤백이 되었다 하지만 PG서버는 결제 완료가 되어있어 당연히 롤백이 되지않아 정합성이 틀어졌다.   
<img src="{{ '/assets/images/concurrency/0806/configuration_swagger_test2.png' | relative_url }}" alt="configuration_swagger_test2" />
<img src="{{ '/assets/images/concurrency/0806/configuration_order_test2.png' | relative_url }}" alt="configuration_order_test2" />
<img src="{{ '/assets/images/concurrency/0806/configuration_pg_test2.png' | relative_url }}" alt="configuration_pg_test2" />
<img src="{{ '/assets/images/concurrency/0806/configuration_db_test2.png' | relative_url }}" alt="configuration_db_test2" />
<img src="{{ '/assets/images/concurrency/0806/configuration_inventory_dg_test2.png' | relative_url }}" alt="configuration_inventory_dg_test2" />


세번째 실험은 PG서버를 죽이고 실험을 진행한다.   
보면 PG서버 연결이 실패되고 그대로 끝난다.   
DB의 같은경우 롤백이 되었고 PG서버는 따로 호출되지않았으니까 정합성은 맞다.   
원래라면 cheked에러이기 때문에 DB가 롤백이 안될 줄 알았지만 webClint는 저수준의 에러를 unchecked에러로 변환 후 익셉션을 발행해주기 때문에 롤백이 되었던것이였다.    
<img src="{{ '/assets/images/concurrency/0806/configuration_swagger_test3.png' | relative_url }}" alt="configuration_swagger_test3" />
<img src="{{ '/assets/images/concurrency/0806/configuration_order_test3.png' | relative_url }}" alt="configuration_order_test3" />
<img src="{{ '/assets/images/concurrency/0806/configuration_db_test3.png' | relative_url }}" alt="configuration_db_test3" />
<img src="{{ '/assets/images/concurrency/0806/configuration_inventory_dg_test3.png' | relative_url }}" alt="configuration_inventory_dg_test3" />

## 결론
DB트랜잭션만으로는 다른통신없이 내부의 행위는 정합성을 맞출수있지만 외부에 행위까지 롤백이 되지않는걸 알수있었다.        
이를 맞추기위서 보상프로세스가 필요하여 다음 글에는 보상프로세스를 사용하여 실험해보겠다.   
그리고 이글에서는 실험하지 않았지만 트랜잭션 메서드에서 API호출이 포함되어있으면 API호출이 완료될때까지 DB커녁션을 유지, 점유하고있기 때문에 따로 분리해야된다.  


