---
layout: post
title: "[Concurrency] 재고 차감에서 동시성 문제를 해결하는 방법 1"
date: 2026-06-29 21:10:00 +0900
categories : [Concurrency]
description : Concurrency의 문제를 해결해보자
---

## 개요

저번에 동시성 테스트를 한 결과 요청이 조금이라도 동시에 들어오면 재고랑 주문의 정합성이 깨지는것을 봤다.    
이번글에서는 이 동시성 문제를 해결을 해보고 서버와, DB의 성능 측정을 해볼것이다.    
동시성 문제는 해결 방법이 여러가지 있지만 하나씩 사용해보고 측정해보며 어떤 장점과 단점이있는지 알아볼것이며 이번 글에서는 Synchronized를 사용하여 해결해 볼 것 이다.       


## 실험 환경

실험 환경은 Spring boot,Grafana,Prometheus,k6,PostgreSQL이며 k6를 제외한 나머지를 로컬 미니 PC에 구축하고
동일한 네트워크에 있는 맥북에 k6를 이용하여 부하를 주어 테스트를 진행할 것 이다.

테이블은 간단하게 구현하기 위해 회원은 따로 구현하지않고 숫자로 구분할것이며 상품과 주문은 일대다 관계로 구성했다.

<img src="{{ '/assets/images/concurrency/erd.png' | relative_url }}" alt="ERD" />

아래가 핵심 로직인 주문생성 로직이며 동시성 문제해결를 하기위해 메서드에 Synchronized를 선언 하였다.      
로직은 상품을 조회 후 재고를 차감 주문을 저장하는 식으로 구성했다.

```java
@Transactional
public synchronized void create(OrderCreateCmd cmd){
    Product product = productRepository.findProductsById(cmd.getProductId())
            .orElseThrow(() -> new DomainException(ErrorCode.PRODUCT_NOT_FOUND));

    product.decrease();

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
메서드에 락이 걸려 여러건이 동시에 접근을 해도 동시성 문제는 일어나지 않을것이다.    
하지만 앞에 사용자의 요청이 끝날때까지 다른 사용자들은 대기를 해야하므로 응답시간이 늘어날 것이며, 그만큼 서버에 쓰래드의 점유시간도 길어질 것 이다.   
최악의 경우 서버의 쓰레드풀을 넘겨 다른 사용자의 요청을 받을 수 없는 상황이 일어날 수 있을 것 이며 다중서버로 변경되었을때 동시성 문제가 다시 일어날것이다.

## 테스트
테스트는 단일서버를 테스트 할것이며 상품의 재고는 100개로 동시 요청 건 수는  1 / 5 / 10 / 50 / 100 이순서대로 그리고 진행할때마다 DB를 리셋하고 진행하여 해보겠다.   
테스트는 k6를 이용하여 아래와 같은 요청을 보내었다.
```
VUS=1   ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
VUS=5   ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
VUS=10  ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
VUS=50  ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
VUS=100 ITERATIONS=1 BASE_URL=http://192.168.219.103:8080 PRODUCT_ID=1 k6 run k6/order-test.js
```
## 테스트 결과
테스트 결과는 같이 요청은 모두 성공 하였고 전글과 동일하게 100개의 요청건만 첨부하였다.    
k6의 결과값을 보면 100명의 사용자가 각각 1번씩 주문 요청을 보냈고, 총 100건의 요청이 모두 정상적으로 처리되었음을 보여준다.   
평균 응답시간은 약 182ms였고, p95는 약 230ms였다.    
락이 없던 버전과 비교했을때 평균 응답시간은 약 100ms시간이 차이가 났고, p95은 15ms밖에 차이가 나지 않았다.   
Synchronized를 사용하면 확실히 모든 시간이 지연된것을 느낄수있다.     
그렇지만 예상대로라면 100개의 재고가 정확히 0개가 되어야한다.   

<img src="{{ '/assets/images/concurrency/0629/success_1_k6.png' | relative_url }}" alt="success_1" />

스웨거로 상품을 조회한 결과값이다.    

productId : 상품 ID   
stock : 남은재고     
orderCount : 주문 개수    
initiativeStock : 초기 재고  

<img src="{{ '/assets/images/concurrency/0629/success_1_swagger.png' | relative_url }}" alt="success_1_swagger" />   

하지만 우리의 예상과 다르게 100개의 주문요청이 들어왔고 86개의 재고가 남아있는상태다.    
왜 그런지 생각해보면 첫번째 요청이 락을 획득하여 로직을 수행하는동안 두번째 요청이 락을 대기하고 있는 상황에서 첫번째 요청이 로직을 수행하고 트랜잭션 커밋을 한다음 락을 반납하는게 아닌 락을 반납하고 커밋을 진행하기 때문에 B에서는 100의 재고를 읽고 99로 차감하는 상황이 생긴다.     

<img src="{{ '/assets/images/concurrency/0629/ex_update.png' | relative_url }}" alt="ex_update" />   

내 예상과 다르게 단일서버때 부터 동시성문제가 일어나 결국 Synchronized으로는 DB동시성 문제를 해결 할 수는 없었다.   
다중서버일때는 더 심하게 나올것으로 예상되어 다중서버의 테스트는 진행하지 않았다.  
그리고 서버 및 DB상태는 서버의 쓰레드 갯수가 늘어난것 그리고 응답시간이 증가한것을 제외하고는 특별한 사항은 없었다.

<img src="{{ '/assets/images/concurrency/0629/success_1_grafana.png' | relative_url }}" alt="success_1_grafana" />

다음 글은 DB의 기능을 이용하여 동시성을 해결하는 방법을 알아보겠다.  


