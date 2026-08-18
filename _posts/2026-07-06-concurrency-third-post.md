---
layout: post
title: "[Concurrency] 재고 차감에서 동시성 문제를 해결하는 방법 2"
date: 2026-07-09 21:00:00 +0900
categories :
  - architecture
description : Concurrency의 문제를 해결해보자
series: concurrency
series_order: 3
---

## 개요
저번에 Synchronized를 이용하여 동시성 문제를 해결해 해결해 보려 했지만 예상과 다르게 트랜잭션이 프록시로 호출되어 락을 반납 후 트랜잭션을 커밋하는 방식이여서 동시성 문제를 해결할 수 없었다.   
이번에는 애플리케이션 단에서 해결이 아닌 DB단에서 해결하는 방식으로 해결을 해보겠다.   
오늘은 낙관적락을 이용하여 동시성 문제를 해결해볼것이다.   

실험에서 사용한 전체 코드는 <a href="https://github.com/sssm987/basic/tree/optimisticLock" target="_blank" rel="noopener noreferrer">[여기에서 확인할 수 있다]</a>
## 실험 환경

실험 환경은 Spring boot,Grafana,Prometheus,k6,PostgreSQL이며 k6를 제외한 나머지를 로컬 미니 PC에 구축하고
동일한 네트워크에 있는 맥북에 k6를 이용하여 부하를 주어 테스트를 진행할 것 이다.

이번 버전은 전과 다르게 상품에 있던 재고를 재고 테이블로 분리를 하여 상품과 주문은 일대다, 주문과 재고도 일대일 관계로 구성했다.    
그리고 재고테이블에 version 컬럼을 하나 추가하였다.   

<img src="{{ '/assets/images/concurrency/0706/erd.png' | relative_url }}" alt="ERD" />

그리고 기존은 OrderController에서 바로 OrderService의 create 메서드를 호출했지만 이번에는 중간에 OrderRetryService를 먼저 호출하는 순으로 변경했다.
```java
public void create(OrderCreateCmd cmd) {
        for (int retryCount = 0; retryCount < MAX_RETRY_COUNT; retryCount++) {
            try {
                orderService.create(cmd);
                return;
            } catch (ObjectOptimisticLockingFailureException e) {
                if (retryCount == MAX_RETRY_COUNT - 1) {
                    throw new DomainException(ErrorCode.ORDER_CONFLICT_RETRY_EXCEEDED);
                }
            }
        }
    }
```
위 코드는 OrderRetryService이다 이 메서드의 역할은 낙관적락의 관리부분이며 락 경합이 발생하여 update가 실패시 재시도 하게끔 만들었으며 retry 횟수는 5번으로 설정하였다.
```java
@Transactional
public void create(OrderCreateCmd cmd){
Inventory inventory = inventoryRepository.findByProductId(cmd.getProductId())
.orElseThrow(() -> new DomainException(ErrorCode.PRODUCT_NOT_FOUND));

        inventory.decrease();

        
        Product product = productRepository.getReferenceById(cmd.getProductId());

        Order order = Order.create(cmd.getMemberId(),product);
        orderRepository.save(order);
    }
```
위 코드는 OrderService이며 재고 차감 및 주문생성의 비지니스 로직이다.    
왜 이런식으로 서비스를 나눠 호출했냐면 낙관적락의 시도는 JPA기준으로 트랜잭션이 끝날때 완료되며 트랜잭션은 프록시를 타기 때문에 내부 메서드호춣로 실행 할 수 없기 때문에 호출 계층을 분리했다.
```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Table(name = "inventory",
uniqueConstraints = @UniqueConstraint(
name = "uk_inventory_product",
columnNames = "product_id"
))
public class Inventory {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Long id;

    @Column(name = "stock", nullable = false)
    private Long stock;

    @Column(name = "initiative_stock")
    private Long initiativeStock;

    @OneToOne
    @JoinColumn(name = "product_id")
    private Product product;

    @Version
    private Long version;

    public void decrease(){
        if(stock > 0){
            stock--;
            return;
        }
        throw new DomainException(ErrorCode.PRODUCT_INVENTORY_SHORT);
    }
}
```
재고 엔티티는 이런식으로 만들었으며 버전 컬럼을 만들어 select된 version값이랑 실제 DB의 버전값이 맞지않으면 update가 되지않는다. 

## 가설
DB의 낙관적 락으로 인해 이번에는 진짜 동시성 문제를 해결 할 수 있을것이다.   
하지만 단점으로는 동시에 많은 요청이 들어올시 재시도 로직으로 인하여 DB의 커넥션 요청 증가, DB의 CPU 사용량 증가, 서버 CPU 사용량 증가, 서버 쓰래드 증가를 예상해 볼 수 있다.   
그리고 서버단 락이 아니므로 다중 서버에서도 문제가 없을것으로 예상된다.   

## 테스트
테스트는 단일서버를 테스트 할것이며 상품의 재고는 100개로 동시 요청 건 수는 100 건을 한번에 요청하는 것으로 진행하여 해보겠다.   
테스트는 k6를 이용하여 아래와 같은 요청을 보내었다.
```
VUS=100 ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
```
## 테스트 결과
우선 k6의 결과값을 봐보자 보면 성공 횟수와 락 경합이 일어나서 실패 나온 수치를 볼 수 있다.    
45건이 주문의 성공하였고 55건이 실패한걸 볼 수 있다.    
그리고 락경합으로 인해 응답시간이 다른 실험보다 조금 더 늦어진걸 볼 수 있다.    
동시성 초과 문제는 해결 했지만 
<img src="{{ '/assets/images/concurrency/0706/success_1_k6.png' | relative_url }}" alt="success_1_k6" />

스웨거를 통해 조회를 본결과 k6의 성공, 실패건과 동일하게 재고량이 감소되었다.   

<img src="{{ '/assets/images/concurrency/0706/success_1_swagger.png' | relative_url }}" alt="success_1_swagger" />

모니터링 결과 서버 CPU 사용량은 일부 증가했지만, 요청 수 자체가 크지 않아 급격한 상승은 없었다.  
반면 서버 스레드 수는 이전 실험과 유사하게 증가했다.  
가장 유의미했던 지표는 DB Rollback 수치였다. 낙관적 락 충돌이 발생한 요청들이 재시도 과정에서 롤백되었기 때문이다.

HikariCP 관련 지표에 큰 변화가 보이지 않은 이유는, 해당 지표가 순간적인 커넥션 풀 상태를 나타내는 게이지이기 때문이다.  
이번 실험에서는 요청이 매우 짧게 처리되어 Prometheus의 스크레이프 시점에는 이미 대부분의 DB 작업이 종료된 상태였고, 그 결과 active/pending connection 변화가 두드러지게 관측되지 않았다.  
반면 commit/rollback 지표는 누적 카운터이므로 변화가 그래프에 남아 확인할 수 있었다.
<img src="{{ '/assets/images/concurrency/0706/success_1_grafana.png' | relative_url }}" alt="success_3" />

그리고 과연 재고가 부족한 상황일때도 요청이 되면 재고가 음수가 안되는지 측정하기위해 2번의 요청을 더 보내보았고 스크린샷과 같이 정합성은 지켜졌다.   

<img src="{{ '/assets/images/concurrency/0706/success_2_k6.png' | relative_url }}" alt="success_2_k6" />
<img src="{{ '/assets/images/concurrency/0706/success_2_swagger.png' | relative_url }}" alt="success_2_swagger" />
<img src="{{ '/assets/images/concurrency/0706/success_3_k6.png' | relative_url }}" alt="success_3_k6" />
<img src="{{ '/assets/images/concurrency/0706/success_3_swagger.png' | relative_url }}" alt="success_3_swagger" />

이번 실험을 통해 낙관적 락은 재고 정합성을 지키는 데에는 효과적이라는 것을 확인할 수 있었다.
실제로 주문 성공 건수만큼만 재고가 감소했고, 초과 차감은 발생하지 않았다.

다만 경합이 심한 상황에서는 재시도 로직으로 인해 일부 요청이 최종 실패할 수 있었고,
그 과정에서 rollback 증가와 응답 시간 증가도 함께 확인할 수 있었다.
즉, 낙관적 락은 정합성 보장에는 강하지만, 충돌이 잦은 환경에서는 성능 측면의 trade-off가 존재한다.  
경합이 많지 않은 경우 사용하기 적합하겠지만 그렇지 않은 경우 서버에 많은 부하를 주기 때문에 실무적으로는 잘 사용하지 않는 방법이다.   
다음은 비관적락에 대해 알아보자