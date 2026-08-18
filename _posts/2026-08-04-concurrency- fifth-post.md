---
layout: post
title: "[Concurrency] 재고 차감에서 동시성 문제를 해결하는 방법 4"
date: 2026-08-04 23:00:00 +0900
categories :
  - architecture
description : Concurrency의 문제를 해결해보자
series: concurrency
series_order: 5
---

## 개요
저번에 비관적락을 이용하여 동시성 문제를 해결했지만 락 대기로 인한 성능 저하와 처리량 감소 락에 대한 부담이 있다는걸 알았다.     
오늘은 원자적 update를 이용하여 동시성 문제를 해결해볼것이다.

실험에서 사용한 전체 코드는 <a href="https://github.com/sssm987/basic/tree/atomicUpdate" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다]</a>

## 실험 환경
실험 환경은 Spring boot,Grafana,Prometheus,k6,PostgreSQL이며 k6를 제외한 나머지를 로컬 미니 PC에 구축하고
동일한 네트워크에 있는 맥북에 k6를 이용하여 부하를 주어 테스트를 진행할 것 이다.

테이블은 전과 동일하게 재고를 재고 테이블로 분리를 하여 상품과 주문은 일대다, 주문과 재고도 일대일 관계로 구성했다.

<img src="{{ '/assets/images/concurrency/0804/erd.png' | relative_url }}" alt="ERD" />

전과 유사하지만 비즈니스 로직은 조금 변경사항이있었다.
원자적 update는 다른 방법과 다르게 update된 row로 update가 성공했는지 알수있기 때문에 이런 식으로 변경하엿다.
메서드 쿼리는 락부분을 제외시키고 update문으로 변경하였다. 
```java
@Transactional
    public void create(OrderCreateCmd cmd){
        Product product = productRepository.findProductsById(cmd.getProductId())
                .orElseThrow(() -> new DomainException(ErrorCode.PRODUCT_NOT_FOUND));


        if(inventoryRepository.decrease(cmd.getProductId()) == 0){
            throw new DomainException(ErrorCode.PRODUCT_INVENTORY_SHORT);
        }

        Order order = Order.create(cmd.getMemberId(),product);
        orderRepository.save(order);
    }
```
```java
@Modifying
@Query("update Inventory i set i.stock = i.stock - 1  where i.product.id = :productId  and i.stock > 0")
int decrease(@Param("productId") Long productId);
```
## 가설
원자적 update는 다른 방법과 다르게 크게 서버에 부하가 있지도, 응답 속도도 늦지 않을 것 이다.
update시에 DB단에서 잠깐 lock이 걸리지만 사실 체감이 되지않는 시간이며 많은 요청에도 크게 부담이 있지 않을 것 이다.
그리고 서버단 락이 아니므로 다중 서버에서도 문제가 없을것으로 예상된다.   

## 테스트
테스트는 단일서버를 테스트 할것이며 상품의 재고는 100개로 동시 요청 건 수는 100 건을 한번에 요청하는 것으로 진행하여 해보겠다.   
테스트는 k6를 이용하여 아래와 같은 요청을 보내었다.
```
VUS=100 ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
```

## 테스트 결과
k6의 결과를 보면 비관적 락과 동일하게 실패한 결과가 없고 모두 성공으로 나온다.    
그리고 크게는 아니지만 응답시간이 비관적 락보다 빨라진걸 알 수 있다.   
<img src="{{ '/assets/images/concurrency/0804/success_1_k6.png' | relative_url }}" alt="success_1_k6" />

모니터링의 경우 비관적 락과 매우 흡사했다.   커넥션 풀을 봐야지 다른점을 알 수 있지만 커넥션 풀은 차후에 다른실험에서 측정할 예정이다.    
쓰레드 갯수가 조금 줄긴 했지만 요청수가 적어서 그런지 크게 유의미 할 정도 줄진 않았다.
<img src="{{ '/assets/images/concurrency/0804/success_1_grafana.png' | relative_url }}" alt="success_1_grafana" />

중요한 재고도 다음과 같이 100개의 건수가 딱 맞게 줄어들었다.
<img src="{{ '/assets/images/concurrency/0804/success_1_swagger.png' | relative_url }}" alt="success_1_swagger" />

비관적 락과 동일하게 요청을 보내도 재고가 음수가 되는지 test를 하기위해 재고를 초기화 후 요청 건 수를 120건으로 늘려 다시 요청을 보내봤다.   
k6를 보면 20건 실패로 정확히 나왔으며 이것도 동일하게 비관적 락 보다 응답시간이 빨랐다.   
<img src="{{ '/assets/images/concurrency/0804/success_2_k6.png' | relative_url }}" alt="success_2_k6" />

재고의 경우 동일하게 정확히 100개만 줄어들었다.
<img src="{{ '/assets/images/concurrency/0804/success_2_swagger.png' | relative_url }}" alt="success_2_swagger" />

이번 실험을 통해 원자적 update도 정합성을 지키는 데 효과적이라는 것을 확인할 수 있었다.
실제로 주문 성공 건수만큼만 재고가 감소했고, 초과 차감은 발생하지 않았다.

또한 다른락과 다르게 이론처럼 서버 부하와 응답 대기 시간이 더 효율적이라는 것도 알 수 있었지만 많은 요청을 보내지 않아서 그런지 크게 확인 할 수는 없었다. 
살무에서는 원자적 update를 많이 사용을 하고 대기열 서버를 함께 사용하여 많은 경합이 일어나지 않게 대기열 서버를 함께두어 부하를 줄이는 방법을 사용한다.  

추후에는 앞에서 사용하는 대기열서버도 함께 구축하여 TEST하는 글을 쓰도록 하겠다.