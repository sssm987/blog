---
layout: post
title: "[Concurrency] 재고 차감에서 동시성 문제가 발생하는 이유"
date: 2026-04-27 21:10:00 +0900
categories: [Concurrency]
description: Concurrency가 일어나는 상황을 재현해보고 어떤 현상이 일어나는지 알아보자 
---

## 개요

실무에서 가장 조심해야하고 또 빈번히 일어나 늘 고려해야하는것중 하나가 바로 이 동시성 문제이다.    
실무에서는 이론상 알고있어 방지를 하지만 실제로 어떤 식으로 문제가 일어나고 또 서버, DB에서는 어떤 일이 일어나는지 잘 모르기 때문에 실험을 진행하게 되었다.    
이번글에서는 실험을 통해 문제를 발생시키고 측정할것이며 다음 글에 해결 방법과 성능 비교 식으로 진행 할 것이다.

## 실험 환경

실험 환경은 Spring boot,Grafana,Prometheus,k6,PostgreSQL이며 k6를 제외한 나머지를 로컬 미니 PC에 구축하고
동일한 네트워크에 있는 맥북에 k6를 이용하여 부하를 주어 테스트를 진행할 것 이다.    

테이블은 간단하게 구현하기 위해 회원은 따로 구현하지않고 숫자로 구분할것이며 상품과 주문은 일대다 관계로 구성했다.

<img src="{{ '/assets/images/concurrency/erd.png' | relative_url }}" alt="ERD" />

아래가 핵심 로직인 주문생성 로직이며 일부러 동시성 문제를 일으키기 위해 서비스에서 상품을 조회 후 재고를 차감 주문을 저장하는 식으로 구성했다.

```java
@Transactional
    public void create(OrderCreateCmd cmd){
        Product product = productRepository.findProductsById(cmd.getProductId())
                .orElseThrow(() -> new DomainException(ErrorCode.PRODUCT_NOT_FOUND));
        // 여러 트랜잭션이 같은 stock 값을 읽은 상태에서
        // 각각 메모리에서 stock-- 수행
        product.decrease();

        // 트랜잭션 커밋 시점에 update 발생
        // 마지막 update가 이전 update를 덮어쓸 수 있음
        Order order = Order.create(cmd.getMemberId(),product);
        orderRepository.save(order);
    }
```
```java
@Entity
@NoArgsConstructor
@Table(name = "product")
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Long id;

    @Column(name = "name")
    private String name;

    @Column(name = "stock")
    private Long stock;

    @Column(name = "initiative_stock")
    private Long initiativeStock;

    public void decrease(){
        if(stock > 0){
            stock--;
            return;
        }
        throw new DomainException(ErrorCode.PRODUCT_INVENTORY_SHORT);
    }
}
```

## 가설
락을 사용하지 않은 지금 방식은 worker thread가 오래 점유되는 상황이 적어 서버의 메모리 및 CPU의 성능은 좋을것이며 요청간의 대기가 없기에 사용자 응답 시간도 빠를 것 이어서 성능적으로는 가장 효율이 좋을 것이다.      
그렇지만 동시에 한명의 사용자만 상품을 주문하는경우는 문제가 없겠지만 여러사용자로 늘어나는 경우 바로 재고와 주문의 수가 맞지않는 정합성 문제가 일어날 것 이다.



## 테스트
테스트는 재고가 충분한 상황에 여러 요청이 들어왔을경우 정합성이 언제 얼마나 깨지는지 테스트할 예정이다. 
상품의 재고는 100개로 동시 요청 건 수는  1 / 5 / 10 / 50 / 100 이순서대로 그리고 진행할때마다 DB를 리셋하고 진행하여 해보겠다.

테스트는 k6를 이용하여 아래와 같은 요청을 보내었다.
```
VUS=1   ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
VUS=5   ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
VUS=10  ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
VUS=50  ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
VUS=100 ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
```
## 테스트 결과
테스트 결과는 같이 요청은 모두 성공 하였고 테스트 이미지를 전부 첨부하는것은 의미가 없을거같아 100개의 요청건만 첨부하였다.   

k6의 결과값을 보면 100명의 사용자가 각각 1번씩 주문 요청을 보냈고, 총 100건의 요청이 모두 정상적으로 처리되었음을 보여준다.     평균 응답시간은 약 84ms였고, p95는 약 142ms였다. 다만 동시성 테스트의 핵심은 응답시간 자체보다도, 성공 요청 수와 실제 주문 건수 및 남은 재고가 일치하는지 확인하는 데 있다.  

<img src="{{ '/assets/images/concurrency/success_1_k6.png' | relative_url }}" alt="success_1" />

스웨거로 상품을 조회한 결과값이다.   

productId : 상품 ID   
stock : 남은재고     
orderCount : 주문 개수    
initiativeStock : 초기 재고  

아래와 같이 주문 개수는 올바르게 100개가 들어갔지만 재고가 10개만 차감된걸 볼수있다.
원인은 상품을 조회한 순간 여러 사용자의 요청이 동시에 같은 재고값을 가져왔고 재고 차감을 같은 재고값에서 차감을 하였기 때문에 발생한 일이다.      
K6의 응답값만 본다면 성공으로 생각들지만 실제로 DB에는 정합성 문제가 일어났고 이런 정합성 문제는 장애나 에러로 도출되지 않기때문에 더욱 심각한 문제가 된다.

<img src="{{ '/assets/images/concurrency/success_1_swagger.png' | relative_url }}" alt="success_1_swagger" />   

ex)
현재 문제를 쉬운 예시를 들자면 3명의 사용자가 동시에 재고를 조회했을때 100개의 재고를 조회했고 다음 차감한후 update시 3명 모두 동일한 99를 update가 하여 LostUpdate가 일어난것이다.   

<img src="{{ '/assets/images/concurrency/ex_select.png' | relative_url }}" alt="ex_select" />   
<img src="{{ '/assets/images/concurrency/ex_update.png' | relative_url }}" alt="ex_update" />

서버 및 DB모니터링의 경우 아래와 같이 급격하게 생성된 쓰레드 풀을 제외하고는 큰 문제는 없었다.   

<img src="{{ '/assets/images/concurrency/success_1_grafana.png' | relative_url }}" alt="success_1_grafana" />

요청 갯수 1개의 경우 이러한 현상이 일어나지 않았지만 조금만 요청의 갯수가 많아질 경우 재고와 주문의 정합성은 맞지않았다.        

| 동시 요청 수 | 정상적으로 성공해야 하는 주문 수 | 최종 재고 |
| --- | --- |-------|
| 1명 | 1건 | 99    |
| 5명 | 5건 | 99    |
| 10명 | 10건 | 99    |
| 50명 | 50건 | 97    |
| 100명 | 100건 | 90    |


## 정리 
이번 실험에서 확인한 문제는 다음과 같다.    
- 주문 요청은 모두 성공했다.     
- 주문 데이터도 정상적으로 저장되었다.    
- 하지만 상품 재고는 주문 수만큼 차감되지 않았다.    
- 원인은 여러 트랜잭션이 같은 재고 값을 읽고 수정하면서 Lost Update가 발생했기 때문이다.
- 다음 글에서는 이 문제를 애플리케이션 단인 synchronized을 이용해 해결해보겠다
