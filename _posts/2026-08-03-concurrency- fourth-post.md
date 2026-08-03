---
layout: post
title: "[Concurrency] 재고 차감에서 동시성 문제를 해결하는 방법 3"
date: 2026-08-03 21:00:00 +0900
categories : [Concurrency]
description : Concurrency의 문제를 해결해보자
---

## 개요
저번에 낙관적락을 이용하여 동시성 문제를 해결했지만 경합이 많이 예상되는 상황에서는 사용하기 어렵다는 단점이있었다.   
이번도 DB단에서 해결하는 방식으로 해결을 해보겠다.   
오늘은 비관적락을 이용하여 동시성 문제를 해결해볼것이다.

실험에서 사용한 전체 코드는 <a href="https://github.com/sssm987/basic/tree/pessimisticLock" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다]</a>

## 실험 환경
실험 환경은 Spring boot,Grafana,Prometheus,k6,PostgreSQL이며 k6를 제외한 나머지를 로컬 미니 PC에 구축하고
동일한 네트워크에 있는 맥북에 k6를 이용하여 부하를 주어 테스트를 진행할 것 이다.

테이블은 전과 동일하게 재고를 재고 테이블로 분리를 하여 상품과 주문은 일대다, 주문과 재고도 일대일 관계로 구성했다.

<img src="{{ '/assets/images/concurrency/0803/erd.png' | relative_url }}" alt="ERD" />

낙관적락과 다르게 비즈니스 로직은 원래 버전으로 변경하였고 대신 select를 해오는 메서드 쿼리에 비관적 락을 걸어 가져오는 방식으로 변경하였다.   
```java
@Transactional
    public void create(OrderCreateCmd cmd){
        Product product = productRepository.findProductsById(cmd.getProductId())
                .orElseThrow(() -> new DomainException(ErrorCode.PRODUCT_NOT_FOUND));

        Inventory inventory = inventoryRepository.findByProductIdForUpdate(product.getId())
                .orElseThrow(() -> new DomainException(ErrorCode.INVENTORY_NOT_FOUND));

        inventory.decrease();

        Order order = Order.create(cmd.getMemberId(),product);
        orderRepository.save(order);
    }
```
```java
public interface InventoryRepository extends JpaRepository<Inventory, Long> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select i from Inventory i where i.product.id = :productId")
    Optional<Inventory> findByProductIdForUpdate(@Param("productId") Long productId);
}
```
## 가설
DB의 비관적 락으로 인해 동시성 문제를 해결 할 수 있을것이다.   
하지만 단점으로는 동시에 많은 요청이 들어올시 락 대기로 인하여 DB의 커넥션 요청 증가, 서버 쓰래드 증가, 응답시간 증가를 예상해 볼 수 있다.   
그리고 서버단 락이 아니므로 다중 서버에서도 문제가 없을것으로 예상된다.   

## 테스트
테스트는 단일서버를 테스트 할것이며 상품의 재고는 100개로 동시 요청 건 수는 100 건을 한번에 요청하는 것으로 진행하여 해보겠다.   
테스트는 k6를 이용하여 아래와 같은 요청을 보내었다.
```
VUS=100 ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
```

## 테스트 결과
k6의 결과를 보면 낙관적 락과 다르게 실패한 결과가 없고 모두 성공으로 나온다.    
하지만 낙관적 락보다 응답시간이 더 증가하였다.    
<img src="{{ '/assets/images/concurrency/0803/success_1_k6.png' | relative_url }}" alt="success_1_k6" />


모니터링의 경우 낙관적 락보다 서버 cpu부하가 적어졌다 그리고 DB롤백도 일어나지 않았고 쓰레드 풀은 여전히 많이 생긴다.   
대신 응답 시간이 전보다 늘어났다.  
<img src="{{ '/assets/images/concurrency/0803/success_1_grafana.png' | relative_url }}" alt="success_1_grafana" />

중요한 재고는 다음과 같이 100개의 건수가 딱 맞게 줄어들었다.
<img src="{{ '/assets/images/concurrency/0706/success_1_swagger.png' | relative_url }}" alt="success_1_swagger" />

더 많은 요청을 보내도 재고가 음수가 되는지 test를 하기위해 재고를 초기화 후 요청 건 수를 120건으로 늘려 다시 요청을 보내봤다.   
k6를 보면 20건 실패로 정확히 나왔으며
<img src="{{ '/assets/images/concurrency/0803/success_2_k6.png' | relative_url }}" alt="success_2_k6" />

재고의 경우 정확히 100개만 줄어들었다.
<img src="{{ '/assets/images/concurrency/0803/success_2_swagger.png' | relative_url }}" alt="success_2_swagger" />

이번 실험을 통해 비관적 락 역시 재고 정합성을 지키는 데 효과적이라는 것을 확인할 수 있었다.
실제로 주문 성공 건수만큼만 재고가 감소했고, 초과 차감은 발생하지 않았다.

또한 낙관적 락과 달리 버전 충돌로 인한 재시도 실패가 없었고,
경합 상황에서도 요청은 순차적으로 처리되었다.
다만 동일한 row에 대한 락 대기가 발생하면서 전체 응답 시간은 증가했다.

즉, 비관적 락은 충돌이 자주 발생하는 환경에서 정합성과 안정적인 처리에는 유리하지만,
그 대가로 처리량과 응답 속도 측면의 trade-off가 존재한다.

실무에서는 비관적 락도 락 대기로 인한 성능 저하와 처리량 감소, 그리고 DB 락에 대한 부담 때문에 자주 선택되지는 않는다.  
특히 동일한 데이터에 대한 경합이 많아질수록 응답 시간이 길어지고, 전체 처리 흐름이 직렬화에 가까워질 수 있다는 단점이 있다.

다음 글에서는 이러한 단점을 보완할 수 있는 방식인 원자적 update에 대해 알아보겠다.




